
- I am going to make this be where I describe the current vulnerabilites that I will confugure on my lab endpoints and the services running on them. 
	- P.S I've love to attack right away with fully updated software and realistic secure configs but im not that cracked at off sec, but the plan is to get to that level for sure. So im just starting with this.
- I also plan to eventually place flags all over the lab and utilize Wazuh detection and eventially preventtive capabilites to protect them. And once that's in place I'll try to bypass them, then when I do, Ill harden, and then continue repeating that cycle.

### **pFsense Firewall** / Gateway
- The version I'm using (2.5) already has a built in vulnerability that allows me to establish that initial access to its LAN.
	
### **Windows Server** 
- Nothing yet

### **Windows Client**
- Nothing yet

### **Ubuntu Client**
- I plan to have a MariaDB app using Flask (python web framework) that will store info like credentials for each of the endpoitns, and I'll use a SQL injection to get that info

