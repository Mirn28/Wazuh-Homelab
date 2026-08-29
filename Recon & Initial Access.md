I want to infiltrate my home lab network using a Kali Linux VM that is external to the network.

The firewall has a WAN interface that got its IP address from my ISP, so I'll try to figure that out 
by scanning my entire home subnet: 
![[Pasted image 20260829182412.png]]

Then after looking at the result, I noticed that there was an IP address that nmap assumed to be running a FreeBSD OS. So that's how i identified the firewall's WAN interface.



Now that I've found the IP, I'll use metasploit to exploit the vulnerability. 
Steps: I'll start by starting msfconsole and then finding an exploit that would be ideal for my use case.
- I