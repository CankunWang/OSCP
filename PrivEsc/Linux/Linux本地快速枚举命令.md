---
cmd_type: PrivEsc
os: Linux
tags:
  - cmd
syntax: sudo -l
---

拿到普通用户 shell 后，先用最短路径确认 sudo、用户目录和可疑文件。

```bash
sudo -l
ls -la /home
find /home -maxdepth 2 -type f 2>/dev/null | sort
file <file>
strings -n 8 <file> | head -n 20
xxd -l 96 <file>
find / -perm -4000 -type f 2>/dev/null
```

适合先筛出：
- 可 sudo 执行的二进制
- 用户目录下的 flag、凭据、提示文件
- 伪装成 data 的图片、压缩包、配置文件
```
cat /etc/os-release        # 查询系统发行版，例如 Ubuntu / Debian
lsb_release -a             # 查询发行版详细版本
hostnamectl                # 查询系统信息和内核信息
uname -a                   # 查询完整 kernel 信息
uname -r                   # 只查询 kernel 版本
cat /proc/version          # 查询 kernel 编译信息
```