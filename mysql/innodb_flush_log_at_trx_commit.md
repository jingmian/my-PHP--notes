
##[看下mysql官方文档解释](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit)
>innodb_flush_log_at_trx_commit
>>Controls the balance between strict ACID compliance for commit operations and higher performance that is possible when commit-related I/O operations are rearranged and done in batches. You can achieve better performance by changing the default value but then you can lose transactions in a crash.

* The default setting of 1 is required for full ACID compliance. Logs are written and flushed to disk at each transaction commit.
 -  在每个事务提交时，日志被写入并刷新到磁盘。

* With a setting of 0, logs are written and flushed to disk once per second. Transactions for which logs have not been flushed can be lost in a crash.

* With a setting of 2, logs are written after each transaction commit and flushed to disk once per second. Transactions for which logs have not been flushed can be lost in a crash.

* For settings 0 and 2, once-per-second flushing is not 100% guaranteed. Flushing may occur more frequently due to DDL changes and other internal InnoDB activities that cause logs to be flushed independently of the innodb_flush_log_at_trx_commit setting, and sometimes less frequently due to scheduling issues. If logs are flushed once per second, up to one second of transactions can be lost in a crash. If logs are flushed more or less frequently than once per second, the amount of transactions that can be lost varies accordingly.:

* Log flushing frequency is controlled by innodb_flush_log_at_timeout, which allows you to set log flushing frequency to N seconds (where N is 1 ... 2700, with a default value of 1). However, any unexpected mysqld process exit can erase up to N seconds of transactions.
 - 它允许你每N秒就刷新日志
* DDL changes and other internal InnoDB activities flush the log independently of the innodb_flush_log_at_trx_commit setting.

* InnoDB crash recovery works regardless of the innodb_flush_log_at_trx_commit setting. Transactions are either applied entirely or erased entirely.

For durability and consistency in a replication setup that uses InnoDB with transactions:
* If binary logging is enabled, set sync_binlog=1.
  
* Always set innodb_flush_log_at_trx_commit=1.

>Caution
Many operating systems and some disk hardware fool the flush-to-disk operation. They may tell mysqld that the flush has taken place, even though it has not. In this case, the durability of transactions is not guaranteed even with the recommended settings, and in the worst case, a power outage can corrupt InnoDB data. Using a battery-backed disk cache in the SCSI disk controller or in the disk itself speeds up file flushes, and makes the operation safer. You can also try to disable the caching of disk writes in hardware caches.

```
innodb_flush_log_at_trx_commit=1
sync_binlog=1
```
上面的解释即使使用`双1`方案不能保证100%安全,可以使用`SCSI`方案
| 
Command-Line Format | --innodb-flush-log-at-trx-commit=# | 居中对齐 |
| :-----| ----: | :----: |
| System Variable | innodb_flush_log_at_trx_commit | 单元格 |
| Scope | Global | 单元格 |
| Dynamic | Yes | 单元格 |
| SET_VAR Hint Applies | Global | 单元格 |
| Scope | Global | 单元格 |
| Type | Enumeration | 单元格 |
| Default Value | 	1 |单元格  |
| Valid Values | 	0 1 2 |单元格  |