```
  
cd /d C:\你的\hashcat\目录  
  
查看 hashcat 帮助：  
hashcat.exe --help  
  
查看设备信息：  
hashcat.exe -I  
  
查看常见 hash 类型：  
hashcat.exe --help | findstr /i "MD5 SHA1 SHA256 SHA512 NTLM bcrypt"  
  
============================================================  
一、基础命令模型  
============================================================  
  
基础字典攻击模型：  
hashcat.exe -m 哈希类型 -a 0 hash.txt rockyou.txt  
  
查看破解结果模型：  
hashcat.exe -m 哈希类型 --show hash.txt  
  
输出破解结果到文件：  
hashcat.exe -m 哈希类型 -a 0 hash.txt rockyou.txt -o cracked.txt  
  
继续上一次任务：  
hashcat.exe --restore  
  
============================================================  
二、参数含义  
============================================================  
  
-m 哈希类型  
-a 攻击模式  
-a 0 字典攻击  
-a 1 组合攻击  
-a 3 掩码攻击  
-a 6 字典 + 后缀掩码  
-a 7 前缀掩码 + 字典  
  
hash.txt 存放哈希的文件  
rockyou.txt 字典文件  
-o cracked.txt 把破解结果输出到 cracked.txt  
--show 查看已经破解出来的结果  
--username 忽略 hash 文件里的用户名字段  
-r 使用规则文件  
-O 使用优化内核  
-w 设置工作负载  
-D 指定设备类型  
-d 指定具体设备编号  
  
============================================================  
三、常见 hash 类型对应 -m 参数  
============================================================  
  
MD5：  
-m 0  
  
SHA1：  
-m 100  
  
SHA256：  
-m 1400  
  
SHA512：  
-m 1700  
  
NTLM / Windows 本地 hash：  
-m 1000  
  
NetNTLMv1：  
-m 5500  
  
NetNTLMv2 / Responder 抓到的 hash：  
-m 5600  
  
bcrypt：  
-m 3200  
  
Linux sha512crypt / /etc/shadow 里的 $6$：  
-m 1800  
  
Apache htpasswd MD5：  
-m 1600  
  
WordPress / phpBB3 / Joomla phpass：  
-m 400  
  
MySQL 旧版本：  
-m 200  
  
MySQL 5：  
-m 300  
  
PDF：  
-m 10500  
  
ZIP：  
-m 13600  
  
RAR5：  
-m 13000  
  
7-Zip：  
-m 11600  
  
KeePass：  
-m 13400  
  
============================================================  
四、最常用字典攻击命令  
============================================================  
  
MD5：  
hashcat.exe -m 0 -a 0 hash.txt rockyou.txt  
  
SHA1：  
hashcat.exe -m 100 -a 0 hash.txt rockyou.txt  
  
SHA256：  
hashcat.exe -m 1400 -a 0 hash.txt rockyou.txt  
  
SHA512：  
hashcat.exe -m 1700 -a 0 hash.txt rockyou.txt  
  
NTLM / Windows 本地 hash：  
hashcat.exe -m 1000 -a 0 hash.txt rockyou.txt  
  
NetNTLMv2 / Responder：  
hashcat.exe -m 5600 -a 0 hash.txt rockyou.txt  
  
bcrypt：  
hashcat.exe -m 3200 -a 0 hash.txt rockyou.txt  
  
Linux sha512crypt：  
hashcat.exe -m 1800 -a 0 hash.txt rockyou.txt  
  
Apache htpasswd MD5：  
hashcat.exe -m 1600 -a 0 hash.txt rockyou.txt  
  
WordPress / phpass：  
hashcat.exe -m 400 -a 0 hash.txt rockyou.txt  
  
MySQL 旧版本：  
hashcat.exe -m 200 -a 0 hash.txt rockyou.txt  
  
MySQL 5：  
hashcat.exe -m 300 -a 0 hash.txt rockyou.txt  
  
PDF：  
hashcat.exe -m 10500 -a 0 hash.txt rockyou.txt  
  
ZIP：  
hashcat.exe -m 13600 -a 0 hash.txt rockyou.txt  
  
RAR5：  
hashcat.exe -m 13000 -a 0 hash.txt rockyou.txt  
  
7-Zip：  
hashcat.exe -m 11600 -a 0 hash.txt rockyou.txt  
  
KeePass：  
hashcat.exe -m 13400 -a 0 hash.txt rockyou.txt  
  
============================================================  
五、查看破解结果  
============================================================  
  
查看 MD5 结果：  
hashcat.exe -m 0 --show hash.txt  
  
查看 SHA1 结果：  
hashcat.exe -m 100 --show hash.txt  
  
查看 SHA256 结果：  
hashcat.exe -m 1400 --show hash.txt  
  
查看 SHA512 结果：  
hashcat.exe -m 1700 --show hash.txt  
  
查看 NTLM 结果：  
hashcat.exe -m 1000 --show hash.txt  
  
查看 NetNTLMv2 结果：  
hashcat.exe -m 5600 --show hash.txt  
  
查看 bcrypt 结果：  
hashcat.exe -m 3200 --show hash.txt  
  
查看 Linux sha512crypt 结果：  
hashcat.exe -m 1800 --show hash.txt  
  
============================================================  
六、攻击模式示例  
============================================================  
  
字典攻击：  
hashcat.exe -m 0 -a 0 hash.txt rockyou.txt  
  
组合攻击，两个字典拼接：  
hashcat.exe -m 0 -a 1 hash.txt dict1.txt dict2.txt  
  
掩码攻击，6 位纯数字：  
hashcat.exe -m 0 -a 3 hash.txt ?d?d?d?d?d?d  
  
掩码攻击，8 位纯数字：  
hashcat.exe -m 0 -a 3 hash.txt ?d?d?d?d?d?d?d?d  
  
掩码攻击，6 位小写字母：  
hashcat.exe -m 0 -a 3 hash.txt ?l?l?l?l?l?l  
  
掩码攻击，8 位小写字母或数字：  
hashcat.exe -m 0 -a 3 hash.txt ?1?1?1?1?1?1?1?1 -1 ?l?d  
  
字典后追加 1 位数字：  
hashcat.exe -m 0 -a 6 hash.txt rockyou.txt ?d  
  
字典后追加 2 位数字：  
hashcat.exe -m 0 -a 6 hash.txt rockyou.txt ?d?d  
  
字典后追加 4 位数字：  
hashcat.exe -m 0 -a 6 hash.txt rockyou.txt ?d?d?d?d  
  
前面加 2 位数字再拼接字典：  
hashcat.exe -m 0 -a 7 hash.txt ?d?d rockyou.txt  
  
============================================================  
七、掩码字符含义  
============================================================  
  
?l 小写字母，a-z  
?u 大写字母，A-Z  
?d 数字，0-9  
?s 特殊符号  
?a 所有字符，包括大小写、数字、符号  
?1 自定义字符集 1  
?2 自定义字符集 2  
?3 自定义字符集 3  
?4 自定义字符集 4  
  
自定义小写字母 + 数字：  
-1 ?l?d  
  
使用自定义字符集：  
hashcat.exe -m 0 -a 3 hash.txt ?1?1?1?1?1?1 -1 ?l?d  
  
============================================================  
八、规则攻击  
============================================================  
  
使用 best64 规则：  
hashcat.exe -m 0 -a 0 hash.txt rockyou.txt -r rules\best64.rule  
  
使用 rockyou-30000 规则：  
hashcat.exe -m 0 -a 0 hash.txt rockyou.txt -r rules\rockyou-30000.rule  
  
使用 OneRuleToRuleThemAll 规则：  
hashcat.exe -m 0 -a 0 hash.txt rockyou.txt -r rules\OneRuleToRuleThemAll.rule  
  
规则攻击含义：  
在 rockyou.txt 的基础上自动变形密码  
  
例如：  
password  
Password  
password1  
password123  
Password123  
p@ssword  
  
============================================================  
九、输出结果  
============================================================  
  
输出到 cracked.txt：  
hashcat.exe -m 0 -a 0 hash.txt rockyou.txt -o cracked.txt  
  
只输出明文密码：  
hashcat.exe -m 0 -a 0 hash.txt rockyou.txt -o cracked.txt --outfile-format 2  
  
输出 hash:password：  
hashcat.exe -m 0 -a 0 hash.txt rockyou.txt -o cracked.txt --outfile-format 3  
  
查看 potfile 中已破解结果：  
hashcat.exe -m 0 --show hash.txt
```