---
cmd_type: Recon
service: Directory
tags: cmd
syntax: gobuster dir -u http://ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x php,txt,html
risk: Low
---
使用gobuster对目标进行目录枚举，并发线程50，搜寻.php .txt .html
如果目标相应速度慢，--timeout 30s调整速度