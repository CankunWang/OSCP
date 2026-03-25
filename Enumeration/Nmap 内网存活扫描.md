---
cmd_type: Recon
service: General
os: Linux
tags:
  - cmd
syntax: sudo nmap -sn <subnet>
risk: Low
---

当目标机器可以访问内网但攻击机不直接可达时，先在目标上做存活扫描。
适合 sudo 可执行 nmap 或已经拿到主机 shell 的场景。

