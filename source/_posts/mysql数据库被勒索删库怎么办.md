title: 数据库被勒索了只能跑路？

categories:

- 填坑日志

tags:

- mysql
- 勒索软件

------

如果你的数据库被人扫描了端口，然后删掉了，并且让你交钱帮你恢复，不要相信！不要相信!不要相信！你的数据没那么珍贵，不会被人家保存的！

今天早晨，打开数据库一看，保存的数据全不见了，只剩下一个叫PLEASE_READ_ME_VVV的数据库。

<!-- more -->

里面写着

> To recover your lost Database and avoid leaking it: Send us 0.045 Bitcoin (BTC) to our Bitcoin address 1McksxpysJGSG9a9zHvan5f8Y1nfpDbVYF and contact us by Email with your Server IP or Domain name and a Proof of Payment. Your Database is downloaded and backed up on our servers. Backups that we have right now: *. Any email without your server IP Address or Domain Name and a Proof of Payment together will be ignored. If we dont receive your payment in the next 10 Days, we will make your database public or use them otherwise.

翻译过来就是：

> 要恢复丢失的数据库并避免泄漏：请将0.045比特币（BTC）发送到我们的比特币地址1Mcksxpysjgsg9a9zhvan5f8y1nfpdbvyf，并通过电子邮件与您的服务器IP或域名和付款证明联系。您的数据库已下载并备份到我们的服务器上。我们现在拥有的备份：*。任何没有您的服务器IP地址或域名和付款证明一起的电子邮件都将被忽略。如果我们在未来10天内没有收到您的付款，我们将公开您的数据库或使用它们。

被勒索了。

**第一步，打开谷歌寻找解决办法。看到这篇博文，遇到了同样的问题。**

数据库被人勒索比特币惨遭删库  https://www.printf520.com/single.html?id=53

以下内容，针对具体情况进行进一步处理。

**第二步，查看自已是否打开了mysql的binlog**

```sql
  SHOW VARIABLES LIKE 'log_bin%';
```

```
+---------------------------------+---------------------------------------+
| Variable_name                   | Value                                 |
+---------------------------------+---------------------------------------+
| log_bin                         | ON                                    |
| log_bin_basename                | /usr/local/var/mysql/mysql-bin       |
| log_bin_index                   | /usr/local/var/mysql/mysql-bin.index |
| log_bin_trust_function_creators | OFF                                   |
| log_bin_use_v1_row_events       | OFF                                   |
| sql_log_bin                     | ON                                    |
+---------------------------------+---------------------------------------+
6 rows in set (0.00 sec)
```

log_bin是ON说明mysql是开启了binlog的，找到了binlog的位置，在

/usr/local/var/mysql/mysql-bin

于是

```shell
cd /usr/local/var/mysql
```

![2C47299A-AF6B-496C-947F-84165E95F4AA](https://ws4.sinaimg.cn/large/007HYVVoly1g4kgxiwvjij30n703w751.jpg)

其中的mysql-bin.00000*就是binlog文件。

binlog是Mysql sever层维护的一种二进制日志，与innodb引擎中的redo/undo log是完全不同的日志；其主要是用来记录对mysql数据更新或潜在发生更新的SQL语句，并以"事务"的形式保存在磁盘中；

作用主要有：

- 复制：MySQL Replication在Master端开启binlog，Master把它的二进制日志传递给slaves并回放来达到master-slave数据一致的目的
- 数据恢复：通过mysqlbinlog工具恢复数据
- 增量备份

然后我们打开最新的binlog文件，在文件的最后几行，你会发现令人窒息的操作

![WechatIMG173](https://ws1.sinaimg.cn/large/007HYVVoly1g4kh28o60qj30ln09rtaw.jpg)

好了，这下我明白了，人家直接drop了。

**第三步：使用binlog恢复文件**

原理就是binlog保存了你所有的数据库操作，相当于你重新执行了在执行drop之前所有的增删改查。

那么我们就需要找到发生删除的时间点。如上图所示，# 190622 9:32:19

所以使用mysqlbinlog恢复数据库

在/usr/local/var/mysql 目录下执行

```shell
mysqlbinlog --start-datetime='2019-01-01 00:00:00' --stop-datetime='2019-06-24 10:30:00' mysql-bin.000001 |  mysql -h 123.45.678.9 -u root -p -P 3306
```

其中

start-datetime=为你想恢复的起始位置，一般为刚创建数据库的时候

stop-datetime=为被删库之前的时间,只要是被删库之前就可以

mysql-bin.000001为要恢复的binlog文件，将所有相关的mysql-bin都写上，以逗号隔开

mysql -h 123.45.678.9 -u root -p -P 3306 为你的远程数据库地址

123.45.678.9为远程ip

root为用户名

3306为端口号

**第四步：恢复完成**

如果第三步出错，一般都在于没有选择合适的起始和终止时间，这个需要自己去寻找

恢复过程视数据库大小时间长短不定

**参考资料：**

[数据库被人勒索比特币惨遭删库]: https://www.printf520.com/single.html?id=53

