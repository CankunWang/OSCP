---
os: linux
service: AD
syntax: nxc ldap 10.80.131.149 -u '' -p '' --users
tags:
---
允许匿名以及空会话时，netexec是首选，可以显示更完整的信息，同时比ldapsearch更可视化