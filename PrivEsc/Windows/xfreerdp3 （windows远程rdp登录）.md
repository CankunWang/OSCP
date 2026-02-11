---
cmd_type: Initial access
service: RDP
protocol: TCP
port: "3389"
tags: cmd
syntax: xfreerdp3 /v:ip /u:用户名 /p:密码 /dynamic-resolution +clipboard
OS: windows
---
使用xfreerdp进行远程windows登录，开启剪贴板；
前提是目标开放了rdp端口

