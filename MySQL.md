#### 索引无法使用

+ ##### 多表关联

  + [字符集不同](https://blog.csdn.net/xinghun61/article/details/5747344)

#### Innodb与Myisam引擎的区别

1. MyISAM是表级锁，而InnoDB是行级锁

2. + 如果执行大量的SELECT，MyISAM是更好的选择

   + InnoDB：如果你的数据执行大量的INSERT或UPDATE，出于性能方面的考虑，应该使用InnoDB表

#### 查询正在执行的进程

```sql
show processlist
```

