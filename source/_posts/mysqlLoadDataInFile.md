---
title: MySQL ERROR 1290 错误码原因及解决办法
tags: [MySQL]
categories:
  - MySQL
date: 2019-02-13 20:38:53
---
```sql
mysql> Load Data InFile 'C:/test.csv' Into Table `test` Lines Terminated By '\r\n';
ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement
```

## 报错原因
secure_file_priv 参数设置了指定目录，只能在指定的目录下进行数据导入导出。

<!-- more -->

```sql
mysql> show variables like '%secure%';
+--------------------------+------------------------------------------------+
| Variable_name            | Value                                          |
+--------------------------+------------------------------------------------+
| require_secure_transport | OFF                                            |
| secure_file_priv         | C:\ProgramData\MySQL\MySQL Server 8.0\Uploads\ |
+--------------------------+------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```

## 解决办法
- 可以设置 my.cnf 配置文件，增加以下配置：
```sql
secure_file_priv='/xx/xx'
```
需要保证 mysql 用户和当前执行命令的用户具有该目录的读写权限。

- 或者直接将需要导入的文件放在 secure_file_priv 指定的目录即可。导出同理。

<hr />