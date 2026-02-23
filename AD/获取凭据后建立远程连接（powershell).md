---
os: windows
syntax: impacket-wmiexec k2.thm/r.bud:'password'@ip
---
使用evil-winrm也可以
evil-winrm -i ip -u r.bud -p 'password'