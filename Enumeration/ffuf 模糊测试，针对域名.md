---
os: linux
cmd_type: Recon
syntax: 'ffuf -w /usr/share/wordlists/dirb/common.txt -u http://10.81.156.97/ -H "Host: FUZZ.k2.thm" -fs <默认页面的字节数>'
---
测试可能的域名,因为大部分房间使用了虚拟主机，只接受域名访问，所以需要枚举可能的域名
```
ffuf -u http://<ip> -H "Host: FUZZ.thm" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

```