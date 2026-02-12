---
os: General
service: Enumeration
syntax: gobuster vhost -u http://10.80.171.120 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
---
可以配合ffuf一起用