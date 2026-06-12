# ffuf 参数与命令格式中文整理

## 1. 基本命令格式

### 目录 / 路径模糊测试

ffuf -u http://example.com/FUZZ -w wordlist.txt

### 子域名模糊测试

ffuf -u http://FUZZ.example.com -w subdomains.txt

### 虚拟主机 / Host 头模糊测试

ffuf -u http://目标IP/ -H "Host: FUZZ.example.com" -w subdomains.txt

### HTTPS 目标

ffuf -u https://FUZZ.example.com -w subdomains.txt

### 指定匹配状态码

ffuf -u https://FUZZ.example.com -w subdomains.txt -mc 200,301,302,403

### 过滤指定状态码

ffuf -u https://FUZZ.example.com -w subdomains.txt -fc 404

### 过滤指定响应大小

ffuf -u https://FUZZ.example.com -w subdomains.txt -fs 1234

### 自动校准过滤误报

ffuf -u https://FUZZ.example.com -w subdomains.txt -ac

### 通过代理发送到 Burp

ffuf -u https://example.com/FUZZ -w wordlist.txt -x http://127.0.0.1:8080

### 保存结果为 JSON

ffuf -u https://example.com/FUZZ -w wordlist.txt -o result.json -of json

### 保存结果为 HTML

ffuf -u https://example.com/FUZZ -w wordlist.txt -o result.html -of html

---

# 2. HTTP 请求相关参数

## -u

指定目标 URL。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt

说明：

FUZZ 是占位符，ffuf 会用字典里的每一行替换 FUZZ。

---

## -w

指定字典文件。

格式：

ffuf -u http://example.com/FUZZ -w /path/to/wordlist.txt

也可以指定关键字名称：

ffuf -u http://example.com/USER/PASS -w users.txt:USER -w passwords.txt:PASS

---

## -H

添加 HTTP 请求头。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -H "Name: Value"

常见用法：

ffuf -u http://目标IP/ -H "Host: FUZZ.example.com" -w subdomains.txt

ffuf -u http://example.com/FUZZ -w wordlist.txt -H "User-Agent: Mozilla/5.0"

ffuf -u http://example.com/FUZZ -w wordlist.txt -H "Authorization: Bearer TOKEN"

说明：

多个 Header 可以写多个 -H。

---

## -X

指定 HTTP 请求方法。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -X GET

ffuf -u http://example.com/login -w passwords.txt -X POST -d "username=admin&password=FUZZ"

常见方法：

GET
POST
PUT
DELETE
PATCH

---

## -d

指定 POST 请求体数据。

格式：

ffuf -u http://example.com/login -w passwords.txt -X POST -d "username=admin&password=FUZZ"

JSON 示例：

ffuf -u http://example.com/api/login -w passwords.txt -X POST -H "Content-Type: application/json" -d '{"username":"admin","password":"FUZZ"}'

---

## -b

添加 Cookie。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -b "PHPSESSID=xxxx; token=yyyy"

---

## -r

跟随重定向。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -r

---

## -raw

不对 URL 进行编码。

格式：

ffuf -u "http://example.com/FUZZ" -w payloads.txt -raw

适合测试特殊字符、路径穿越、注入 payload。

---

## -http2

使用 HTTP/2。

格式：

ffuf -u https://example.com/FUZZ -w wordlist.txt -http2

---

## -ignore-body

不读取响应体，只判断状态码等信息。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -ignore-body

适合大量扫描时提升速度。

---

## -sni

指定 TLS SNI。

格式：

ffuf -u https://目标IP/ -H "Host: FUZZ.example.com" -sni FUZZ.example.com -w subdomains.txt

注意：

-sni 不支持 FUZZ 关键字时，需要根据版本确认。

---

## -timeout

设置 HTTP 请求超时时间，单位是秒。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -timeout 5

---

## -x

设置代理。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -x http://127.0.0.1:8080

SOCKS5 示例：

ffuf -u http://example.com/FUZZ -w wordlist.txt -x socks5://127.0.0.1:1080

---

# 3. 通用参数

## -t

设置并发线程数。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -t 50

默认值一般是 40。

线程越高速度越快，但也更容易触发 WAF 或导致丢包。

---

## -rate

限制每秒请求数。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -rate 100

说明：

每秒最多发送 100 个请求。

---

## -p

设置请求延迟。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -p 0.1

随机延迟：

ffuf -u http://example.com/FUZZ -w wordlist.txt -p 0.1-2.0

说明：

每个请求之间随机等待 0.1 到 2 秒。

---

## -maxtime

设置整个扫描最大运行时间，单位是秒。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -maxtime 300

---

## -maxtime-job

设置每个任务最大运行时间，单位是秒。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -maxtime-job 60

---

## -s

静默模式，只输出结果。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -s

---

## -v

详细模式，输出完整 URL 和重定向地址。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -v

---

## -c

彩色输出。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -c

---

## -json

以 JSON 行格式输出。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -json

---

## -noninteractive

关闭交互式控制台功能。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -noninteractive

---

## -V

查看版本信息。

格式：

ffuf -V

---

# 4. 自动校准参数

## -ac

开启自动校准过滤。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -ac

说明：

ffuf 会自动请求一些随机不存在的路径，用来判断哪些响应是误报。

---

## -acc

指定自动校准字符串。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -ac -acc teststring

---

## -ach

对每个 Host 单独自动校准。

格式：

ffuf -u http://目标IP/ -H "Host: FUZZ.example.com" -w subdomains.txt -ac -ach

适合虚拟主机扫描。

---

## -ack

指定自动校准关键字。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -ac -ack FUZZ

默认关键字是 FUZZ。

---

## -acs

指定自定义自动校准策略。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -ac -acs strategy

---

# 5. 匹配参数 Matcher

匹配参数用于指定“哪些响应算作结果”。

---

## -mc

匹配 HTTP 状态码。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -mc 200

多个状态码：

ffuf -u http://example.com/FUZZ -w wordlist.txt -mc 200,301,302,403

匹配所有状态码：

ffuf -u http://example.com/FUZZ -w wordlist.txt -mc all

默认匹配：

200-299,301,302,307,401,403,405,500

---

## -ml

匹配响应行数。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -ml 10

---

## -mr

匹配响应中的正则内容。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -mr "admin"

---

## -ms

匹配响应大小。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -ms 1234

---

## -mt

匹配响应时间。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -mt ">100"

说明：

匹配首字节响应时间大于 100 毫秒的结果。

---

## -mw

匹配响应单词数。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -mw 50

---

## -mmode

设置多个匹配器之间的关系。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -mc 200 -mw 50 -mmode and

可选值：

and
or

默认是 or。

---

# 6. 过滤参数 Filter

过滤参数用于排除误报结果。

---

## -fc

过滤 HTTP 状态码。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -fc 404

多个状态码：

ffuf -u http://example.com/FUZZ -w wordlist.txt -fc 404,403

---

## -fl

过滤响应行数。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -fl 12

---

## -fr

过滤响应中的正则内容。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -fr "Not Found"

---

## -fs

过滤响应大小。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -fs 1234

多个大小：

ffuf -u http://example.com/FUZZ -w wordlist.txt -fs 1234,5678

范围：

ffuf -u http://example.com/FUZZ -w wordlist.txt -fs 1000-2000

---

## -ft

过滤响应时间。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -ft "<100"

说明：

过滤首字节响应时间小于 100 毫秒的结果。

---

## -fw

过滤响应单词数。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -fw 50

---

## -fmode

设置多个过滤器之间的关系。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -fc 404 -fs 1234 -fmode and

可选值：

and
or

默认是 or。

---

# 7. 输入参数

## -e

添加扩展名。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -e .php,.txt,.bak

说明：

会把 FUZZ 扩展成：

admin
admin.php
admin.txt
admin.bak

---

## -D

兼容 DirSearch 字典模式，需要和 -e 一起使用。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -e .php,.txt -D

---

## -enc

对关键字进行编码。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -enc FUZZ:urlencode

常见编码：

urlencode
b64encode
b64decode

---

## -ic

忽略字典中的注释行。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -ic

---

## -mode

多字典模式。

可选值：

clusterbomb
pitchfork
sniper

默认是 clusterbomb。

---

### clusterbomb 模式

所有字典进行笛卡尔积组合。

格式：

ffuf -u http://example.com/USER/PASS -w users.txt:USER -w passwords.txt:PASS -mode clusterbomb

说明：

假设 users 有 10 行，passwords 有 100 行，总请求数是 10 x 100 = 1000。

---

### pitchfork 模式

多个字典按行一一对应。

格式：

ffuf -u http://example.com/USER/PASS -w users.txt:USER -w passwords.txt:PASS -mode pitchfork

说明：

users 第 1 行配 passwords 第 1 行，users 第 2 行配 passwords 第 2 行。

---

### sniper 模式

一次只替换一个关键字。

格式：

ffuf -u http://example.com/P1/P2 -w payloads.txt -mode sniper

---

## -input-cmd

使用命令生成输入。

格式：

ffuf -u http://example.com/FUZZ -input-cmd "seq 1 100"

说明：

使用命令输出作为字典内容。

---

## -input-num

指定 input-cmd 生成的输入数量。

格式：

ffuf -u http://example.com/FUZZ -input-cmd "seq 1 100" -input-num 100

---

## -input-shell

指定运行 input-cmd 的 shell。

格式：

ffuf -u http://example.com/FUZZ -input-cmd "seq 1 100" -input-shell /bin/bash

---

## -request

使用原始 HTTP 请求文件。

格式：

ffuf -request request.txt -w wordlist.txt

适合从 Burp 复制请求后进行模糊测试。

---

## -request-proto

指定原始请求使用的协议。

格式：

ffuf -request request.txt -request-proto http -w wordlist.txt

或者：

ffuf -request request.txt -request-proto https -w wordlist.txt

---

# 8. 递归扫描参数

## -recursion

开启递归扫描。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -recursion

注意：

URL 必须以 FUZZ 结尾，递归只支持 FUZZ 关键字。

---

## -recursion-depth

设置递归深度。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -recursion -recursion-depth 2

默认深度是 0。

---

## -recursion-strategy

设置递归策略。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -recursion -recursion-strategy default

可选值：

default
greedy

说明：

default：基于重定向进行递归。
greedy：对所有匹配结果都进行递归。

---

# 9. 输出参数

## -o

输出结果到文件。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -o result.json

---

## -of

指定输出格式。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -o result.html -of html

可选格式：

json
ejson
html
md
csv
ecsv
all

---

## -od

指定保存匹配结果的目录。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -od results/

---

## -or

没有结果时不创建输出文件。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -o result.json -or

---

## -debug-log

保存调试日志。

格式：

ffuf -u http://example.com/FUZZ -w wordlist.txt -debug-log debug.log

---

# 10. 常用实战命令

## 10.1 目录扫描

ffuf -u http://example.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -mc 200,301,302,403

---

## 10.2 目录扫描并添加扩展名

ffuf -u http://example.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e .php,.txt,.bak,.zip -mc 200,301,302,403

---

## 10.3 子域名扫描

ffuf -u http://FUZZ.example.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc 200,301,302,403

---

## 10.4 HTTPS 子域名扫描

ffuf -u https://FUZZ.example.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc 200,301,302,403

---

## 10.5 虚拟主机扫描

ffuf -u http://10.10.10.10/ -H "Host: FUZZ.example.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc 200,301,302,403

---

## 10.6 虚拟主机扫描并过滤误报大小

ffuf -u http://10.10.10.10/ -H "Host: FUZZ.example.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 1234

---

## 10.7 自动校准虚拟主机扫描

ffuf -u http://10.10.10.10/ -H "Host: FUZZ.example.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -ac -ach

---

## 10.8 POST 表单爆破

ffuf -u http://example.com/login -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "username=admin&password=FUZZ" -w passwords.txt -mc 200,302

---

## 10.9 POST JSON 爆破

ffuf -u http://example.com/api/login -X POST -H "Content-Type: application/json" -d '{"username":"admin","password":"FUZZ"}' -w passwords.txt -mc 200,302

---

## 10.10 参数名模糊测试

ffuf -u "http://example.com/index.php?FUZZ=test" -w parameters.txt -mc 200

---

## 10.11 参数值模糊测试

ffuf -u "http://example.com/index.php?id=FUZZ" -w numbers.txt -mc 200

---

## 10.12 LFI 路径穿越测试

ffuf -u "http://example.com/index.php?file=FUZZ" -w lfi-payloads.txt -mc 200 -mr "root:"

---

## 10.13 通过 Burp 代理测试

ffuf -u http://example.com/FUZZ -w wordlist.txt -x http://127.0.0.1:8080

---

## 10.14 使用原始请求文件测试

ffuf -request request.txt -w wordlist.txt -mc 200,302,403

---

## 10.15 匹配包含 admin 的响应

ffuf -u http://example.com/FUZZ -w wordlist.txt -mr "admin"

---

## 10.16 过滤包含 Not Found 的响应

ffuf -u http://example.com/FUZZ -w wordlist.txt -fr "Not Found"

---

## 10.17 限速扫描

ffuf -u http://example.com/FUZZ -w wordlist.txt -rate 50 -p 0.1-1.0

---

## 10.18 保存 Markdown 结果

ffuf -u http://example.com/FUZZ -w wordlist.txt -o result.md -of md

---

# 11. HTB 常用流程

## 第一步：先扫虚拟主机

ffuf -u http://目标IP/ -H "Host: FUZZ.域名.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc 200,301,302,403

---

## 第二步：如果大量误报，观察 Size

示例输出：

admin   Status: 200, Size: 1543
dev     Status: 200, Size: 1543
test    Status: 200, Size: 1543
random  Status: 200, Size: 1543

如果不存在的子域名都是 Size 1543，就过滤它：

ffuf -u http://目标IP/ -H "Host: FUZZ.域名.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 1543

---

## 第三步：发现子域名后写入 /etc/hosts

echo "目标IP dev.域名.htb admin.域名.htb test.域名.htb" | sudo tee -a /etc/hosts

---

## 第四步：浏览器访问

http://dev.域名.htb
http://admin.域名.htb
http://test.域名.htb

---

# 12. 参数速查表

-u              指定目标 URL
-w              指定字典
-H              添加 Header
-X              指定请求方法
-d              指定 POST 数据
-b              添加 Cookie
-r              跟随重定向
-raw            不进行 URL 编码
-http2          使用 HTTP/2
-ignore-body    不读取响应体
-timeout        设置请求超时时间
-x              设置代理
-t              设置线程数
-rate           限制每秒请求数
-p              设置请求延迟
-s              静默输出
-v              详细输出
-c              彩色输出
-json           JSON 行格式输出
-ac             自动校准过滤
-ach            每个 Host 单独自动校准
-mc             匹配状态码
-ml             匹配响应行数
-mr             匹配响应正则
-ms             匹配响应大小
-mt             匹配响应时间
-mw             匹配响应单词数
-fc             过滤状态码
-fl             过滤响应行数
-fr             过滤响应正则
-fs             过滤响应大小
-ft             过滤响应时间
-fw             过滤响应单词数
-e              添加扩展名
-enc            编码关键字
-mode           多字典模式
-request        使用原始 HTTP 请求
-request-proto  指定原始请求协议
-recursion      开启递归扫描
-recursion-depth 设置递归深度
-o              输出到文件
-of             指定输出格式
-od             指定输出目录
-debug-log      保存调试日志

---

# 13. 最常用参数组合

## 子域名 / 虚拟主机

ffuf -u http://目标IP/ -H "Host: FUZZ.example.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc 200,301,302,403 -fs 1234

## 目录扫描

ffuf -u http://example.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -mc 200,301,302,403 -e .php,.txt,.bak

## POST 爆破

ffuf -u http://example.com/login -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "username=admin&password=FUZZ" -w passwords.txt -mc 200,302

## 代理到 Burp

ffuf -u http://example.com/FUZZ -w wordlist.txt -x http://127.0.0.1:8080

## 自动校准

ffuf -u http://example.com/FUZZ -w wordlist.txt -ac

## 输出结果

ffuf -u http://example.com/FUZZ -w wordlist.txt -o result.html -of html