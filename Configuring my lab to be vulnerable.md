
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
- I plan to have a MariaDB app using Flask (python web framework) that will store info like credentials for each of the endpoitns, and I'll use a SQL injection to get that info.
Below is the process I went through setting this up:

1. Installed MariaDB and then ran the security setup script, intentionally leaving it insecure for this phase of the lab:

```bash
sudo apt install mariadb-server -y
```
- Installs MariaDB fresh.

```bash
sudo mariadb-secure-installation
```
- Walks through initial hardening prompts (root password, anonymous users, remote root login, test database). For this lab, I left the root password blank and skipped the other hardening steps.

3. Created the database:

```sql
CREATE DATABASE homelab;
USE homelab;
```
- `CREATE DATABASE` makes a new container for tables. `USE` switches context so subsequent commands apply inside it.

4. Created the credentials table:

```sql
CREATE TABLE credentials (
    id INT AUTO_INCREMENT PRIMARY KEY,
    hostname VARCHAR(50),
    username VARCHAR(50),
    password VARCHAR(100),
    role VARCHAR(30),
    notes VARCHAR(100)
);
```
Defines the table schema: `id` auto-increments as a unique row identifier; the rest are text fields describing each account (which host it belongs to, its username/password, its privilege role, and free-text notes).

5. Populated it with lab credentials (I put the real ones in my database, but I won't post those here.):
```sql
INSERT INTO credentials (hostname, username, password, role, notes) VALUES
('DC01', 'user1', 'Password123!', 'domain user', ''),
('DC01', 'Administrator', 'Password123!', 'domain admin', 'Built-in domain root account'),
('UbuntuClient01', 'localuser', 'Password123!', 'local user', 'Also hosts MariaDB'),
('UbuntuClient01', 'root', 'Password123!', 'root', ''),
('WindowsClient01', 'localuser2', 'Password123!', 'local user', ''),
('firewall', 'root', 'firewallpass', 'firewall admin', 'Shared with webGUI admin');
```
