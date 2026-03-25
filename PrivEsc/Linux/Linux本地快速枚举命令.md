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
```

适合先筛出：
- 可 sudo 执行的二进制
- 用户目录下的 flag、凭据、提示文件
- 伪装成 data 的图片、压缩包、配置文件
