This is where I'll be documenting my journey of learning detection engineering. 

---
# I need to edit this:

# SSH Brute-Force Detection with Wazuh + pfSense

## What I built

I set up Wazuh to detect SSH brute-force attacks against my pfSense firewall. The final result: when I brute-force pfSense's SSH from Kali with Medusa or Hydra, Wazuh fires **rule 5763 ("sshd: brute force trying to get access to the system")** at level 10, mapped to MITRE T1110.001 (Password Guessing).

Getting there was not a straight line. I wrote a custom decoder and custom rules, then threw them out once I found the real root cause. That ended up being the most useful part of the whole thing.

## Lab setup

| Machine | Role | IP |
|---|---|---|
| pfSense (`firewall-lab.corp.lab`) | Target, sends syslog | LAN `10.10.10.1`, WAN `10.0.0.39` |
| Ubuntu (`sec-lab`) | Wazuh manager 4.14.7 + rsyslog | `10.10.10.20` |
| Kali | Attacker (Medusa, Hydra) | `10.0.0.222` |

I whitelisted Kali in pfSense's sshguard so it wouldn't block me mid-test. That removes a real defense layer, so it's only for testing.

---

## Part 1: How Wazuh works 

Wazuh is three services, each with one job:

- **Manager:** the brain. It receives logs, decodes them, runs rules, and decides what counts as an alert. It also manages agents.
- **Indexer:** storage and search (an OpenSearch fork). It holds alerts and answers queries. It doesn't analyze anything.
- **Dashboard:** the UI (an OpenSearch Dashboards fork). It queries the indexer. If it went down, detection would still work; I'd just be blind.

The manager thinks, the indexer remembers, the dashboard shows.

**Getting logs in:**
- **Agents** run on endpoints and collect OS logs by default (Windows Security/System/Application, Linux auth/syslog). They also do file integrity monitoring, config assessment (SCA), inventory and rootcheck. Extra sources like Sysmon need a `<localfile>` entry.
- **Agentless syslog** is for devices that can't run an agent, like pfSense. The device forwards syslog, and the manager has to be configured to listen.

Agent settings live in each agent's own `ossec.conf`. The manager can also push central config to agent groups through `agent.conf`, which overrides the local file.

**The pipeline every log goes through:**

1. **Pre-decoding (phase 1):** pulls the standard syslog header: timestamp, hostname, program name.
2. **Decoding (phase 2):** a decoder extracts fields like `srcip` and `dstuser`. It never decides if something is bad.
3. **Rules (phase 3):** rules look at the fields and judge them: level, description, alert or not.

Only events that match a rule at level 3 or higher become alerts, and **the dashboard only shows alerts**, not every raw log. That fact explained half my problems later.

My main debugging tool was `wazuh-logtest`. You paste in a raw log line and it shows all three phases.

---

## Part 2: Forwarding pfSense logs to Wazuh

### pfSense side

**Status → System Logs → Settings → Remote Logging Options:** I enabled remote logging on pfsense's GUI, pointed it at the Wazuh server on port `514`. That said, I ran into a few problems while trying to confirm everything was working:

### Problem 1: logs arriving, but nobody listening

I checked whether packets were reaching the Wazuh box:

```bash
sudo tcpdump -i any udp port 514 -n
```
- `-i any`: listen on every interface
- `udp port 514`: only syslog traffic
- `-n`: show raw IPs, no DNS lookups

Packets were showing up from `10.10.10.1` as `SYSLOG auth.*`. So pfSense was sending fine. Then I ran:
```bash
sudo ss -ulnp | grep 514
```
- `-u` UDP, `-l` listening, `-n` numeric ports, `-p` which process owns the socket

That returned **nothing**. The packets hit the network card, but no program was listening, so the OS threw them away. Wazuh was running, but it just hadn't been told to accept syslog.

**Fix:** add a second `<remote>` block to `/var/ossec/etc/ossec.conf`, leaving the agent one on 1514 alone:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.10.10.1</allowed-ips>
</remote>
```

### Problem 2: the manager wouldn't start

After the restart, `wazuh-manager` failed with `wazuh-csyslogd: ERROR: (1226)`, but it was just a config syntax error. My block looked like this:

```xml
<connection>syslog<connection>
```

**I forgot the `/` on every closing tag.** Without it, XML reads the second tag as a new opening tag, nothing ever closes, and the parser gives up. Fixing the closing tags fixed it.

XML notes for myself:
- Indentation doesn't matter.
- Every opening tag needs a matching closing tag with a `/`.
- Tags must nest properly, and tag names are case-sensitive.
- The block has to sit inside `<ossec_config>`.

After rrunning `ss` command again, it showed `wazuh-remoted` listening on 514 this time.

### Proving ingestion

The dashboard only shows alerts, so I turned on archiving to see *everything* that Wazuh receives. In `ossec.conf` under `<global>`:

```xml
<logall_json>yes</logall_json>
```

Then I watched it while doing deliberate failed SSH logins:
```bash
sudo tail -f /var/ossec/logs/archives/archives.json | grep -i sshd
```
- `tail -f`: follow the file live
- `grep -i`: case-insensitive filter

Logs were coming through, but there was a problem. (look at next section)

---

## Part 3: The decoding problem

Every event had `"decoder":{}`, which means **no decoder matched**. Logtest showed why:

```
Nov  8 18:57:38 sshd[3788]: error: PAM: Authentication error for root from 10.0.0.222

Phase 1:  hostname: 'sshd[3788]:'
Phase 2:  No decoder matched.
Phase 3:  id: '2501'  syslog: User authentication failure.
```

A normal syslog line is `timestamp hostname program[pid]: message`.
- **pfSense was sending it without a hostname**, so Wazuh grabbed the next word, `sshd[3788]:`, as the hostname. Now it couldn't tell the program was sshd, so none of the sshd decoders ran.

The only reason I got any alert was rule **2501**, a generic catch-all that matches the words "authentication error" anywhere in a syslog line.

**Why this matters:** with no decoder, there are no fields. The attacker IP and the username are just text. Brute-force detection is "X failures **from the same source IP** in Y seconds," so without a `srcip` field, no correlation rule can work.

### First attempt: a custom decoder

In `/var/ossec/etc/decoders/local_decoder.xml`:

```xml
<decoder name="pfsense-sshd">
  <prematch>PAM: Authentication error for </prematch>
</decoder>

<decoder name="pfsense-sshd-fields">
  <parent>pfsense-sshd</parent>
  <regex>for (\S+) from (\S+)$</regex>
  <order>dstuser, srcip</order>
</decoder>
```

- **Parent:** `prematch` is a cheap text check that answers "is this the right log?"
- **Child:** only runs if the parent matched. `\S+` grabs a whole word, `( )` marks what to capture, `$` means end of line. `<order>` names the captures in order.

**Problem:** the manager failed again. I'd named the parent `pfsense_sshd` with an underscore, but the child pointed at `pfsense-sshd` with a hyphen. The child was looking for a parent that didn't exist, which is a fatal config error. I made them match and it worked:
```
Phase 2:  name: 'pfsense-sshd'   dstuser: 'root'   srcip: '10.0.0.222'
```

Phase 1 still said the hostname was `sshd[3788]:`. A decoder can't change pre-decoding, but it can work around it by matching the message text. The `location` field (`10.10.10.1`) still told me which device sent the log.

**Decoder vs rule (I mixed these up):** A decoder answers "what does this log *say*?" A rule answers "does this *matter*?" 
- Phase 3 still showed 2501 because I hadn't written anything to judge the decoded fields yet.

---
# <font color="#4bacc6">Start Here</font>
## Part 4: My first custom rules

In `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="pfsense,sshd,">

  <rule id="100100" level="5">
    <if_sid>2501</if_sid>
    <decoded_as>pfsense-sshd</decoded_as>
    <description>pfSense SSH: failed login for $(dstuser) from $(srcip)</description>
    <group>authentication_failed,</group>
  </rule>

  <rule id="100101" level="10" frequency="8" timeframe="120">
    <if_matched_sid>100100</if_matched_sid>
    <same_srcip />
    <description>pfSense SSH: possible brute force from $(srcip)</description>
    <mitre>
      <id>T1110.001</id>
    </mitre>
  </rule>

</group>
```

How it works:
- **Custom rule IDs** must be 100000–120000.
- **`<if_sid>`** makes 100100 a *child* of 2501. It only checks events that already matched 2501, and when a child matches, it **replaces** the parent in the alert.
- **`<decoded_as>`** only matches logs my decoder parsed.
- **`$(srcip)`** puts decoded fields into the alert text.
- **`frequency` + `timeframe`** mean "8 matches in 120 seconds."
- **`<if_matched_sid>`** looks *back in time* at earlier matches of 100100. `if_sid` only checks the current event.
- **`<same_srcip />`** only counts failures from the same IP. Without it, 8 failures from 8 different machines would trip it.

**Problem 4:** more closing-tag typos: `<if_sid>2501<if_sid>`, `</decoded-as>` instead of `</decoded_as>`, and `</d>` instead of `</id>`. This time I checked the config *before* restarting:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
```
- `-t`: test mode. It parses config, decoders and rules, then exits. Silence means it's clean.

This now runs before every restart, so I stop taking the manager down over typos.

**Logtest result:** I pasted the failure line repeatedly in one session. Logtest remembers earlier events, so the counting works. Pastes 1–7 showed **100100**, and **paste 8 switched to 100101** at level 10 with the MITRE tags. It fired exactly on the threshold.

---

## Part 5: Live testing broke everything

I ran Medusa from Kali. Only **2501** showed up in the dashboard, and only about 2 alerts per run instead of 8.

The live attack produced *different log lines* than my test line:

```
Disconnecting authenticating user root 10.0.0.222 port 41482: Too many authentication failures [preauth]
```

My decoder only knew `PAM: Authentication error for`, which came from my manual SSH attempts (keyboard-interactive login). Medusa's per-guess lines matched **no rule at all**, so they never became alerts.

### Digging in with the PID

I tested with Hydra next:

```bash
hydra -l root -P test_pass.txt -t 1 ssh://10.0.0.39
```
- `-l` one username, `-P` password file, `-t 1` one task, `ssh://` protocol + target

The number in brackets, `sshd[34350]`, is the **process ID**. sshd starts a new process for every connection, so the PID groups every line from one session. I searched the archive by PID:

```bash
sudo grep 'sshd\[34350\]' /var/ossec/logs/archives/archives.json | grep -o '"full_log":"[^"]*"'
```
- `\[ \]`: escaped brackets, because `[34350]` in regex means "any one of these characters"
- `grep -o`: print only the matching part, here the raw log text
- I used `grep` on the whole file instead of `tail -f`, because the attack had already happened

The output:

```
6 × Failed password for root from 10.0.0.222 port 56570 ssh2
error: maximum authentication attempts exceeded for root ...
Disconnecting authenticating user root ...: Too many authentication failures [preauth]
```

What I learned from this:
- **That's not sshguard, that's sshd.** sshd allows 6 password attempts per connection by default (`MaxAuthTries`), then kicks the client.
- **The logs don't differ per cracking tool.** sshd writes the logs, not the tool. There's a small fixed set of OpenSSH messages (`Failed password`, `PAM: Authentication error`, `maximum authentication attempts exceeded`, `Accepted password`, `Invalid user`), and every tool produces the same ones.
- **Wazuh already has decoders and rules for all of them.** They weren't firing only because of the missing hostname. I'd been rebuilding Wazuh's sshd ruleset by hand when the real fix was the root cause.

(Side note: John the Ripper doesn't work for this. It's an offline cracker for stolen hashes and never touches the SSH service, so it generates no logs.)

---

## Part 6: Fixing the root cause

### Failed attempt: RFC 5424

pfSense can send **syslog (RFC 5424)**, which includes the hostname. I switched to it, and it made things worse:

```
1 2026-11-14T15:18:32.169410-05:00 firewall-lab.corp.lab sshd 40405 - - Failed password for root ...

Phase 1:  (nothing extracted)
Phase 2:  No decoder matched.
Phase 3:  1002  Unknown problem somewhere in the system.
```

Wazuh's pre-decoder doesn't understand RFC 5424: the leading `1` is a version number, and the program and PID are separate fields (`sshd 40405`) instead of `sshd[40405]:`. But the line *had* everything I needed, including a real hostname. It was just in a layout Wazuh can't read.

### The fix: rsyslog as a relay

Instead of Wazuh receiving syslog directly, **rsyslog**, Linux's standard syslog daemon, receives it, rewrites it into the classic layout, and writes it to a file that Wazuh reads like any local log. A "syslog collector → file → SIEM" setup is common in real environments.

(rsyslog is officially "the rocket-fast system for log processing," but the "r" originally meant *reliable*, from planned syslog-reliable support.)

Before building it, I checked rsyslog's docs. Its built-in `RSYSLOG_TraditionalFileFormat` template rebuilds the classic layout, but there's a known rsyslog issue (#557) where RFC 5424 messages get a tag **without the trailing colon**, which could break Wazuh's pre-decoder again. So I built the tag myself.

**1. Free port 514:** I deleted the syslog `<remote>` block from `ossec.conf`, since only one program can listen on a port.

**2. Create the log folder:**

```bash
sudo mkdir /var/log/pfsense
sudo chown syslog:adm /var/log/pfsense
```
- `chown syslog:adm`: rsyslog runs as the `syslog` user on Ubuntu, so it needs write access

**3. rsyslog config,** `/etc/rsyslog.d/10-pfsense.conf`:

```
module(load="imudp")
input(type="imudp" port="514" ruleset="pfsense")

template(name="WazuhFormat" type="string"
  string="%TIMESTAMP% %HOSTNAME% %APP-NAME%[%PROCID%]:%msg:::sp-if-no-1st-sp%%msg:::drop-last-lf%\n")

ruleset(name="pfsense") {
  if $fromhost-ip == "10.10.10.1" then {
    action(type="omfile" file="/var/log/pfsense/pfsense.log" template="WazuhFormat")
  }
}
```

- `imudp`: UDP input module, listening on 514
- `ruleset`: keeps pfSense traffic separate from the box's own logs
- `$fromhost-ip`: only accept from pfSense, replacing Wazuh's old `allowed-ips`
- `%APP-NAME%[%PROCID%]:` builds `sshd[40405]:` by hand, so the colon is guaranteed
- `omfile`: write to a file

**4. Tell Wazuh to read the file,** in `ossec.conf`:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/pfsense/pfsense.log</location>
</localfile>
```

**5. Delete my custom decoder and rules 100100/100101,** so they don't grab lines before the built-in sshd decoder does.

### Problem 5: the log file didn't exist

`tail` said `No such file or directory`. rsyslog creates the file on the *first message*, so nothing had arrived yet. `rsyslogd -N1` (config check only) passed, and the folder permissions were right. The real issue was in `ss`:

```
[::]:514   rsyslogd
```

**rsyslog was only listening on IPv6.** It had started while `wazuh-remoted` still held IPv4 port 514, so the IPv4 bind failed. pfSense sends over IPv4, so everything was being dropped. One `sudo systemctl restart rsyslog` fixed it, and I got both `0.0.0.0:514` and `[::]:514`.

(Tip: `tail -F` with a capital F keeps retrying until the file exists.)

### It worked

```
Nov 14 17:07:37 firewall-lab.corp.lab sshd[46599]: Failed password for root from 10.0.0.222 port 60198 ssh2

Phase 1:  hostname: 'firewall-lab.corp.lab'   program_name: 'sshd'
Phase 2:  name: 'sshd'   dstuser: 'root'   srcip: '10.0.0.222'   srcport: '60198'
Phase 3:  id: '5760'  sshd: authentication failed.   mitre: T1110.001, T1021.004
```

Real hostname, the built-in sshd decoder, the built-in rule, `srcport` (my decoder never got that), and MITRE mapping for free. One rsyslog relay replaced my entire custom decoder.

---

## Part 7: The accuracy problem

Now I was relying on the built-in brute-force rule **5763**, which counts 5760 events from the same IP. A Medusa run with 8 passwords **didn't trigger it**. The dashboard showed only **4** 5760 events for 8 guesses. Half my failures were missing.

The cause was rsyslog's **repeated message reduction**. When identical lines arrive back to back, rsyslog writes the first one and collapses the rest into "message repeated N times." The 6 failures in one SSH connection are identical (same PID, same port, same text), so most of them vanished before Wazuh ever saw them. rsyslog's own docs call this feature a "misfeature" because log analysis tools need every line. Ubuntu turns it on by default.

**Fix:** in `/etc/rsyslog.conf`:

```
$RepeatedMsgReduction off
```

It has to be the global setting, since it applies per input, not per action. Then `rsyslogd -N1` and restart.

I could have just used a bigger wordlist to force 5763 to fire, but that would have hidden the gap. If half the failures disappear, "8 failures" secretly means about 16, and a slow attacker could stay under it. Getting **8 guesses = 8 failures** mattered more than seeing the alert fire.

---

## Part 8: Final results

### Medusa, 8 passwords

```bash
medusa -h 10.0.0.39 -u root -P test_pass.txt -M ssh
```

Result: **7 × 5760 + 1 × 5763, plus 2 × 5758 and 2 × 2501.**

| Rule | Level | Triggered by | Meaning |
|---|---|---|---|
| 5760 | 5 | `Failed password for root from ...` | One wrong guess |
| 5758 | 8 | `error: maximum authentication attempts exceeded ...` | One connection hit sshd's 6-try limit |
| 2501 | 5 | `Disconnecting authenticating user ...: Too many authentication failures` | sshd kicking that connection; no sshd-specific rule covers it, so the generic one catches it |
| **5763** | **10** | 8 × 5760 from the same IP | **Brute force** |

Why only 7 × 5760: the 8th `Failed password` line matched 5760 first, then its child rule 5763 saw that it was the 8th match from that IP. **The child replaces the parent**, so that event shows up only as 5763. Same log line, labeled with the more serious rule. Nothing was lost.

The 5758/2501 pairs come in twos because each connection that hit the limit produced one of each.

### Hydra, same 8 passwords

5763 fired again, so the detection is tool-agnostic. The differences:

| | Medusa | Hydra |
|---|---|---|
| Failures logged | 8 | **10** |
| Connections that hit the 6-try limit | 2 | **1** |
| 5763 fired | Yes | Yes |

- Hydra packs guesses into one connection until sshd kicks it, then reconnects.
- **10 failures from an 8-password list:** I confirmed the list has 8 lines, so Hydra **resent 2 guesses**, most likely the ones in flight when sshd cut the connection. The same wordlist produces a different log volume depending on the tool, which changes how fast a frequency rule trips.
- Back before the rsyslog fix, Hydra looked like it stopped after the first kick and still reported "0 valid passwords found," silently skipping guesses. That's the same kind of Hydra false negative I'd hit before.

### Ordering alerts in the dashboard

In the export, 5763 looked like it came *before* the failures. It can't; it only fires after 7 earlier matches. Nine of the eleven events shared one millisecond (`20:29:22.832`), so the dashboard had no real order to sort them by.

**Fix:** add the top-level **`id`** field as a column and sort ascending. The alert ID is `<epoch seconds>.<byte offset in alerts.json>`, and Wazuh writes alerts in the order it processes them, so the offset is the true order. Caveat: it sorts as text, so it breaks if the offset rolls over to more digits mid-burst. That's rare.

Also: the dashboard's "Lucene syntax warning" is harmless. Quoting the value (`decoder.name:"pfsense-sshd"`) silences it. DQL is the default and enough for Wazuh. Splunk SPL and Microsoft KQL are the query languages worth actually studying.

---

## Lessons

1. **Check in layers, from the network up.** tcpdump (packets arriving?) → ss (anything listening?) → archive (ingested?) → alerts (rule matched?). Each layer tells me exactly where it broke.
2. **The dashboard only shows alerts.** Unmatched events are invisible there. The archive and the raw log file are ground truth.
3. **Logtest every decoder and rule before going live**, and **`wazuh-analysisd -t` before every restart.**
4. **Fix root causes instead of piling workarounds on top.** My custom decoder worked in logtest and failed against real traffic. Fixing the log format let Wazuh's built-in ruleset do the work and covered every SSH failure type at once.
5. **sshd writes the logs, not the attack tool.** Detect on the server's messages, not on tool signatures.
6. **Accuracy over "it fired."** A detection that only works because you flooded it has a hidden blind spot.
7. **Most of my outages were XML closing tags.** Check the `/`, check hyphens vs underscores, check names match.

## Next steps

- [ ] Turn `logall_json` back off (it fills the disk fast); `pfsense.log` is now my raw view
- [ ] Capture an `Accepted password` line and build **success after brute force from the same IP**, the detection that says an attack actually worked
- [ ] Remove the sshguard whitelist and detect sshguard's own block events
- [ ] Look at `ignore` to suppress repeat 5763 alerts during long attacks
- [ ] Add pfSense Firewall Events for network-level detections