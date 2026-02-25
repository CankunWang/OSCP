---
os: General
syntax: nmap -p 88 --script krb5-enum-users --script-args krb5-enum-users.realm='K2.THM',userdb=/usr/share/wordlists/seclists/Usernames/Names/names.txt 10.81.145.106
---
可以替代一下kerbrute,但是还是建议使用kerbrute,kerbrute可以很好的枚举用户名

```
./kerbrute_linux_amd64 userenum --dc 10.81.145.106 -d k2.thm /usr/share/wordlists/seclists/Usernames/Names/names.txt
```