---
cmd_type: Recon
service: Web
os: General
tags:
  - cmd
syntax: ffuf -u 'http://<IP>/?FUZZ=test' -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs <baseline_size> -mc all -t 40
risk: Low
---

当页面没有明显参数时，用常见参数名字典做 GET 参数枚举。
先拿到基线页面大小，再用 -fs 过滤噪音。

