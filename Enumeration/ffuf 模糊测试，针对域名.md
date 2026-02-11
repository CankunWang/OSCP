---
os: linux
cmd_type: Recon
syntax: 'ffuf -w /usr/share/wordlists/dirb/common.txt -u http://10.81.156.97/ -H "Host: FUZZ.k2.thm" -fs <默认页面的字节数>'
---
测试可能的域名