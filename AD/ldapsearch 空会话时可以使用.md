---
os: linux
service: AD
syntax: ldapsearch -x -H ldap://10.80.131.149 -b "dc=thm,dc=local" "(sAMAccountName=*)"
---
nxc ldap 10.80.131.149 -u '' -p '' --users 可以直接用nxc的ldap
当目标ad域允许匿名访问，空会话时，使用ldapsearch进行枚举；查看包括description在内的所有信息，同时可以生成users.txt
和GetADUsers.py以及GetNPUsers.py相比，能显示更多信息，不过比另外两个繁杂很多
