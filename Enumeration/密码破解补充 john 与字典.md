---
cmd_type: Cracking
service: Password
os: General
tags:
  - cmd
syntax: john --wordlist=<WORDLIST> --format=<FMT> hash.txt
risk: Low
---

# 密码破解补充：john 与字典

## 用途 / 适用场景

John the Ripper（`john`）是 CPU 友好的密码破解器，擅长 Linux `/etc/shadow`、zip/rar 等格式，与 GPU 为主的 hashcat 互补。本笔记补充 hash 识别、john 常用命令、定制字典（cewl/crunch）与规则库。

## 关键命令

### Hash 识别

```bash
# hashid
hashid '<HASH>'
hashid -m '<HASH>'        # 附带 hashcat -m 与 john --format 对照

# hash-identifier（交互式）
hash-identifier
```

### John 基础破解

```bash
# 指定字典 + 格式破解
john --wordlist=<WORDLIST> --format=<FMT> hash.txt

# 让 john 自动识别格式（不确定格式时）
john --wordlist=<WORDLIST> hash.txt

# 显示已破解结果
john --show hash.txt
john --show --format=<FMT> hash.txt

# 继续上次会话
john --restore

# 查看所有可用 format
john --list=formats | grep -i raw
```

### Linux shadow 破解

```bash
# 合并 /etc/passwd 与 /etc/shadow 成 john 可读格式
unshadow /etc/passwd /etc/shadow > unshadowed.txt

# 破解（crypt 格式，$6$ sha512crypt 通常自动识别）
john --wordlist=<WORDLIST> unshadowed.txt
```

### 常见 format 表

| format | 说明 |
| --- | --- |
| `raw-md5` | 裸 MD5 |
| `raw-sha1` | 裸 SHA1 |
| `raw-sha256` | 裸 SHA256 |
| `raw-sha512` | 裸 SHA512 |
| `nt` | Windows NTLM |
| `lm` | Windows LM |
| `crypt` | Linux /etc/shadow（自动识别 $1$ $5$ $6$） |
| `dynamic_n` | dynamic 格式族（dynamic_0 起） |

对应命令示例：

```bash
john --wordlist=<WORDLIST> --format=raw-md5 hash.txt
john --wordlist=<WORDLIST> --format=raw-sha1 hash.txt
john --wordlist=<WORDLIST> --format=nt hash.txt
john --wordlist=<WORDLIST> --format=crypt unshadowed.txt
```

## 定制字典

### cewl 从网站生成字典

```bash
# 抓取网站正文生成字典：深度2、最小长度5
cewl <URL> -w words.txt -d 2 -m 5

# 包含数字、邮箱
cewl <URL> -w words.txt -d 2 -m 5 --with-numbers --email
```

### crunch 掩码生成

```bash
# 生成 6-8 位纯数字
crunch 6 8 0123456789 -o list.txt

# 生成 8 位小写字母
crunch 8 8 abcdefghijklmnopqrstuvwxyz -o list.txt

# 用 charset 简写（@ = 小写, % = 数字）
crunch 8 8 -t @@@@@%%% -o list.txt

# 生成 word + 2位数字模式
crunch 6 6 -t pass%% -o list.txt
```

## 规则库

```bash
# best64 规则（小快）
john --wordlist=<WORDLIST> --rules=best64 --format=raw-md5 hash.txt

# OneRuleToRuleThemAll（大而全，需放入 john 的 rules 目录）
john --wordlist=<WORDLIST> --rules=OneRuleToRuleThemAll --format=raw-md5 hash.txt

# 查看 john 内置规则
john --list=rules
```

## 与 hashcat 的分工

- **hashcat**：GPU 加速，适合 MD5/SHA/NTLM 等大批量、高速度破解；掩码、规则、组合攻击都快，Windows 常用 `hashcat.exe`。
- **john**：CPU 通用性强，适合 Linux `crypt`、zip/rar/keepass，以及 hashcat 支持不佳或需特殊格式的算法；`unshadow` 直接处理 `/etc/shadow`。
- 实践：NTLM/MD5 先上 hashcat（`-m 1000` / `-m 0`），Linux `$6$` shadow 用 john + `--wordlist` + 规则；两者共用同一套字典（SecLists/rockyou）。

## 实战流程

```bash
# 1. 识别哈希类型
hashid -m '<HASH>'

# 2. 快速字典攻击
john --wordlist=/usr/share/wordlists/rockyou.txt --format=<FMT> hash.txt

# 3. 加规则再跑
john --wordlist=<WORDLIST> --rules=best64 --format=<FMT> hash.txt

# 4. 看结果
john --show hash.txt

# 5. 字典不够时定制
cewl <URL> -w words.txt -d 2 -m 5
crunch 6 8 0123456789 -o nums.txt
```

## 常见坑

- **format 不匹配**：raw-md5 与带 salt 的 md5 是不同 format，识别错直接跑不出；用 `hashid` 或 `john --list=formats` 确认。
- **`--format` 大小写**：john 的 format 名统一小写（`raw-md5`、`nt`），写错报错。
- **动态格式**：`dynamic` 系列（如 `dynamic_0`）对应各种自定义算法，识别到 `dynamic` 时需进一步看子类型。
- **shadow 权限**：`unshadow` 需要 root 才能读 `/etc/shadow`。
- **john 默认字典小**：自带 password.lst 很小，换成 rockyou/SecLists。
- **规则文件路径**：自定义规则要放 `john.conf` 同级或 `rules/` 目录，否则 `--rules=` 找不到。

## 参考开源项目

- John the Ripper — https://github.com/openwall/john
- CeWL — https://github.com/digininja/CeWL
- SecLists — https://github.com/danielmiessler/SecLists
- hashid — https://github.com/psypanda/hashID
