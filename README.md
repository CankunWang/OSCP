# OSCP_Study_Vault

OSCP / HTB / THM 学习知识库（Obsidian）。目标：把「单点命令速查」补成「可迁移的技法体系」，覆盖从侦察到提权、从 Web 到 AD 的完整攻击链。

## 目录结构

| 目录 | 内容 |
|---|---|
| `Enumeration/` | 侦察与信息收集（nmap / ffuf / gobuster / curl / netexec / 子域 / 密码破解） |
| `Exploitation/` | 漏洞利用（SQLi / SSTI / XSS / LFI / 命令注入 / XXE / SSRF / JWT / 反序列化 / Burp / sqlmap / Metasploit） |
| `PrivEsc/Linux/` | Linux 提权（kernel / GTFOBins / cron / capabilities） |
| `PrivEsc/Windows/` | Windows 提权（服务 / Token / SeBackup / LOLBAS / UAC） |
| `AD/` | 域渗透（枚举 / Kerberoast / AS-REP / RBCD / NTLM Relay / DCSync / ACL / ADCS / 委派 / impacket） |
| `Lateral Movement/` | 凭据导出与横向移动（mimikatz / secretsdump / PTH） |
| `Pivoting/` | 内网隧道与端口转发（ligolo-ng / chisel / sshuttle） |
| `Reverse shell/` | 反弹 shell / TTY 升级 / msfvenom |
| `Post-Exploitation/` | AV 绕过 / 后渗透 / 取证 |
| `Checklists/` | Linux / Windows / Web 提权与漏洞 checklist、OSCP 报告规范 |
| `HTB_Writeups/` | HTB 靶机 writeup |
| `Labs/THM/` | THM 房间 writeup |
| `Templates/` | writeup 模板 |

## 方法论（详见 CLAUDE.md）

- 侦察默认：`nmap -Pn -T4 --min-rate 1000 -p- <ip>` → `-sV -sC`。
- 发现 vhost 后写入 `/etc/hosts`。
- 反弹 shell 回调用 Kali `tun0` IP。
- Ubuntu 目标 `/bin/sh`→dash，无 `/dev/tcp`，反弹 shell 用 `bash -c "..."` 包装。
- 本地 Agent Skills：`D:/hack-skills/`（yaklang/hack-skills），三层路由懒加载。
