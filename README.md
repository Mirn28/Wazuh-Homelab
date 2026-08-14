# Wazuh-Homelab

## Hi there!

This project is essentially a home lab that I'll be creating and my goal is to further develop my skills in cybersecurity such as: detection engineering, offSec, etc.


### Lab network architecture:

**pFsense Firewall** version 2.5.0 (also acting as gateway and DHCP server)
- IP Address: 10.10.10.1

**Windows Server** (Running Active Directory and DNS server)
- IP Address: 10.10.10.10

**Windows Client**
- IP Address: Dynamic

**Ubuntu Client**
- IP Address: Dynamic

**Security Server** (Running Wazuh)
- IP Address: 10.10.10.20
^Might replace with a picture from threat model canvas app^

### **Context**: 
- This subnet is currently protected by that firewall which has a LAN interface and WAN Interface. The WAN got its IP from my home ISP's router so its essentially a part of my network. I also have a kali VM that's also running on my home subnet, and I plan to use it to attack the internal subnet that my homelab environment is on.
- From there I plan to also use Wazuh to initially just detect and see these attacks from the defensive side. This where I'll be getting into detection engineering. 
- I eventually plan to harden each of the endpoints on the LAN, and implement preventive controls to deal with those attacks and from there I'll continue to adjust my attacks accordingly.
- That said, the main goal of this lab is to get better at detection engineering not OffSec, but the skills I develop from the OffSec work will help me to become a better defender.






# <Insert NetTest Python Script>
When setting this lab up, I ran into some issues that I didn't feel like documenting cause they weren't relevant to the actual goals of this project, but I may or may not include them below. They were mostly networking issues between the gateway and one of my clients, and some other system level issues. But one thing I did automate was the process of checking network connectivity within my lab. I used 2 scripts, a PowerShell script (made by AI) and another python script. the PS script is running on windows and the python script was for the ubuntu client but i think ima run the python script on everything since i made it myself.





## The actual vulnerability I'll be exploiting, and the context behind it
- As mentioned, the way I plan to attack the network from an external Kali linux machine, is using exploiting `CVE-2021-41282`. It is specifc to versions of pFsense <2.5.2. At a high level, it will allow me create a file on the machine running the firewall, and put any type of code/text into that file.
- I plan to create a webshell that will allow me to run any type of commands on the system, granting me that initial foothold into the LAN.
- As for the specific attacks I use beyond this point, they will vary and I'll document them as I go. But this initial exploit is the foundation of it all, that allows me to breach this network. Detailed breakdown of the CVE at the bottom.














# Brief breakdown of CVE-2021-41282
Sed is a stream text editor.

A stream text editor doesn't open or close a file for you to type into — instead, it processes text that flows through it (usually piped in), and you tell it what transformation to apply, and it does that on your behalf.

Sed is great for automation.

Now I'm currently using pfSense firewall 2.5.0, which runs on the FreeBSD Operating System. This OS has Sed installed as a stream editing utility by default.

The vulnerability I'm exploiting is called CVE-2021-41282. This is how it works:

- The pfSense web app GUI that allows you to configure your firewall has a specific input field in the following location. (Diagnostics —> Routes)
  - In that location, there is an input field called "Filter" and this is where the vulnerability lies.
  - Normally, this field is used to filter for certain routing info in this firewall's configuration. When given input, it calls on the Sed utility to filter the output of the netstat command, which pulls the live routing table from the server.
  - However, it does not properly sanitize input. It only checks your input for shell syntax (characters that would let you break out and run a new shell command), but it never checks for Sed's own command syntax. So you can smuggle in one of Sed's own instructions — the w (write) command — which tells Sed to write arbitrary content into a file of your choosing on that machine. This isn't disguising anything past the filter; it's using an entirely different Sed feature that the filter was never built to catch in the first place.
  - We're going to take advantage of this by using Metasploit. There's an exploit called exploit/unix/http/pfsense_diag_routes_webshell and it essentially uses Sed's write command to create a PHP file (PHP, because pfSense's web server is only configured to execute PHP files when requested through the browser) that acts as a webshell. That will allow us to run commands on the server, which will essentially allow us to get initial access to the LAN.




