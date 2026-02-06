# Reminder
```
Deep Enumeration!
如果发现很多攻击面都被很好的防护住了，完全没有入口，重新做一遍Enumeration，更详细更完善的做一遍,像UDP这些都不能错过
```
```
cat/type flag from its original location.
不接受其他任何形式的获得flag,包括web shell
```

AD域 checklists
  - [ ] Enumerate machines **until the end**, even if you already have **local admin**
	  - [ ] PEAS脚本
	  - [ ] 检查各种可能性，包括证书等
  - [ ] Check **PowerShell history** for **every local user**
	  - [ ] Look for hardcoded creds, scripts, tokens
	  - [ ] Remember: local users may actually be **domain users** 反过来也成立，domain users也可能控制一台remote desktop
  - [ ] Try **username = password** (domain & local)
	  - [ ] Use NXC
  - [ ] Perform **local authentication / RID brute**
	  - [ ]  Enumerate local users/尝试匿名访问与空进程，用impacket或nxc的脚本尝试拉取用户名
	  - [ ] Enumerate domain users from authenticated hosts
	  - [ ] 在获得了可用凭据后，可以进一步使用bloodhound 进行完整枚举
  - [ ] Check **SMB shares**
	  - [ ] Look for readable shares
	  - [ ] Look for scripts, configs, backups
  - [ ] Run **Responder**
	  - [ ] Monitor for hashes / auth attempts
  - [ ] If a **mail server** is present
	  - [ ] Send an email with `config.Library-ms`
  - [ ] Check **SYSVOL** share from the Domain Controller
	  - [ ]  Look for GPP passwords
	  - [ ] Look for login scripts
  - [ ] Reference **Active Directory Enumeration – Pentest Everything**
	- [ ] 尝试所有可能的路径
	