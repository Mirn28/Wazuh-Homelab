I want to infiltrate my home lab network using a Kali Linux VM that is external to the network.

The firewall has a WAN interface that got its IP address from my ISP, so I'll try to figure that out 
by scanning my entire home subnet: 
![[Pasted image 20260829182412.png]]

Then after looking at the result, I noticed that there was an IP address that nmap assumed to be running a FreeBSD OS. So that's how i identified the firewall's WAN interface.

![[Pasted image 20260829184456.png]]

Now that I've found the IP, I'll use metasploit to exploit the vulnerability. 
Steps: I'll start by starting msfconsole and then finding an exploit that would be ideal for my use case.
![[Pasted image 20260829184237.png]]


- I'll go with that 1st one. Now I've gotta set the options for the exploit so it knows its target. Once I've done that, ill run the exploit:
![[Pasted image 20260829184702.png]]

Now I'm in. But this shell is pretty clanky and since I'm authenticated as root user, I think I'll just enable ssh so I can ssh into this. `pfSsh.php plaback enablesshd` 
![[Pasted image 20260829185838.png]]

Alright ssh is enabled, but the issue is about finding that ssh password. I'll try to bruteforce it, since the creds for this machine are default. I tried hydra and medusa to bruteforce the password but it didn't work due to ssh guard.  But I looked it up and found out that the default password for pfsense firewalls is literally: "pfsense". So I used that and got in, via this command: `ssh root@<IP>`.

Now that I confirmed that ssh access is working, I want to set up a SOCKS Proxy. 
- A SOCKS Proxy essentially routes all types of traffic unlike a normal proxy which routes http traffic only. 
- I'll use this to route all traffic via the SSH connection I've established to the firewall. The command I'll use is `ssh -D 9050 root@<IP>`.
	- `-D 9050` opens a local SOCKS proxy on port 9050, which forwards all traffic sent to it through the encrypted SSH tunnel to the remote host (pfSense).
	- The proxy listens on `localhost:9050`. I can route traffic through it two ways:
		1. **proxychains** — for non-Metasploit tools (e.g. nmap, ssh), prefix the command with `proxychains` to force its traffic through the proxy.
		2. **Metasploit** — set the module's `Proxies` option to `socks5:127.0.0.1:9050` instead.

I'm using this to route my attacks against the internal LAN — since pfSense sits on that internal network as the gateway, any traffic that reaches it through the tunnel gets forwarded onward to internal hosts automatically.
 