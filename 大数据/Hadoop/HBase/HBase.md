#### 启动

1. 开启HDFS：在主节点上运行 start-dfs.sh
2. 开启Zookeeper：在各节点上运行 zkServer.sh start
3. 开启HBase：在主节点上运行 start-hbase.sh

#### 创建表

进入hbase shell

```shell
create '${table_name}', '${cf}'
```

