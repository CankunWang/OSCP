---
cmd_type: Recon
service: General
os: General
risk: Low
syntax: nmap -Pn -p- --min-rate 1000 -oN nmap_full <target ip>
tags:
  - cmd
---

# Nmap 快速批量扫描
适用于目标禁止ICMP，但是在THM以外的环境下，大部分目标会启用IDS等各种防护措施，所以大概率被丢包
一般先指定端口80,443,445,3389等进行指定扫描，同时可以使用-f分片，-D RND:10部署decoy,同时使用-T0或T1慢速扫描，以及--randomize-hosts随机端口顺序




