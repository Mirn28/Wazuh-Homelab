# Wazuh-Homelab

Hi there!

This project is essentially a home lab that I'll be creating with the following network architecture:

Firewall (also acting as gateway and DHCP server)
- IP Address: 10.10.10.1
Windows Server (Running Active Directory and DNS server)
- IP Address: 10.10.10.10
Windows Client
- IP Address: Dynamic
Ubuntu Client
- IP Address: Dynamic
Security Server (Running Wazuh)
- IP Address: 10.10.10.20

When setting this lab up, I ran into some issues that I didn't feel like documenting cause they weren't relevant to the actual goals of this project, but I may or may not include them below. They were mostly networking issues between the gateway and one of my clients, and some other system level issues. But one thing I did automate was the process of checking network connectivity within my lab. I used 2 scripts, a PowerShell script (made by AI) and another python script. the PS script is running on windows and the python script was for the ubuntu client but i think ima run the python script on everything since i made it myself.

# <Insert NetTest Python Script>

### I'll add a visual diagram to make more clear the way data flows through this network later.




