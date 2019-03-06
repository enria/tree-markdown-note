### 安装Hive

1. 下载二进制文件

   https://hive.apache.org/downloads.html

   截止2019.03.04，最新版本为`3.1.1`

2. 解压放入文件夹

3. 修改配置文件

   + `hive-env.sh`

     1. 复制模板

        ```shell
        cp hive-env.sh.template hive-env.sh
        ```

     2. 修改HADOOP路径

        ```shell
        HADOOP_HOME=/home/zhangyd/soft/hadoop-2.8.5
        ```

   + `hive-site.xml`

     1. 复制模板

        ```shell
        cp hive-default.xml.template hive-site.xml
        ```

     2. 修改元数据存储方式

        ```xml
        <property>
            <name>javax.jdo.option.ConnectionURL</name>
            <value>jdbc:mysql://172.18.39.19:3306/hive?createDatabaseIfNotExist=true&amp;characterEncoding=UTF-8&amp;useSSL=false&amp;serverTimezone=UTC</value>
            <description>
              JDBC connect string for a JDBC metastore.
              To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
              For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
            </description>
          </property>
        <property>
            <name>javax.jdo.option.ConnectionDriverName</name>
            <value>com.mysql.cj.jdbc.Driver</value>
            <description>Driver class name for a JDBC metastore</description>
          </property>
          <property>
            <name>javax.jdo.option.ConnectionUserName</name>
            <value>admin</value>
            <description>Username to use against metastore database</description>
          </property>
         <property>
            <name>javax.jdo.option.ConnectionPassword</name>
            <value>admin</value>
            <description>password to use against metastore database</description>
         </property>
        ```

        > 注意：
        >
        > 1. `xml`文件中的`&`需要转化为实体`&amp;`。
        > 2. 使用较新版本的`MySQL`驱动，包名为`com.mysql.cj.jdbc.Driver`，连接参数也需要申明时区`serverTimezone`，否则会报错。

     3. 将`MySQL`驱动包放入`${HIVE_HOME}/lib`文件目录下

4. 初始化元数据

   ```shell
   ${HIVE_HOME}/bin/schematool -initSchema -dbType mysql
   ```

5. 测试

   进入`hive cli`

   ```shell
   ${HIVE_HOME}/bin/hive
   ```

   1. 建数据库

      ```sql
      create database sparktest;
      ```

   2. 建表

      ```sql
      create table sparktest.contact (tel INT);
      ```

   3. 插入数据

      ```sql
      insert into contact values(181);
      ```

   观察执行命令是否出现错误。

   验证持久化：

   1. 数据库中`dbs`表存在一条刚才创建库名的记录，`tbls`表存在一条刚才创建表名的记录。
   2. `HDFS`中有生成插入的记录的文件，位于`/user/hive/warehouse/sparktest.db/contact`目录下，下载该文件可以看到里面有一行`181`。

### 使用Spark管理Hive

自己更中意`Spark`的编程模型，而且对`MapReduce`使用较少。

有些教程里面说要使用支持`Hive`版本的`Spark`，我使用的版本为`2.3.2`，默认就支持，省去了一些麻烦。

1. 开启`metastore service`

   ```shell
   ${HIVE_HOME}/bin/hive --service metastore
   ```

2. 在`${SPARK_HOME}/conf`下创建文件`hive-site.xml`,内容如下

   ```xml
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
   <configuration>
    <property>
       <name>hive.metastore.uris</name>
       <value>thrift://172.18.31.107:9083</value>
       <description>Thrift URI for remote metastore.Used by metastore client to connect to                 remote metastore</description>
     </property>
   </configuration>
   ```

   > 记得修改服务器地址。

3. 进入`Spark Shell`进行测试

   ```shell
   ${SPARK_HOME}/bin/spark-shell
   ```

   按顺序输入以下命令：

   1. ```scala
      import org.apache.spark.sql.hive.HiveContext
      ```

   2. ```scala
      val hiveCtx = new HiveContext(sc)
      ```

   3. ```scala
      val contactRDD = hiveCtx.sql("select * from sparktest.contact").rdd
      ```

   4. ```scala
      contactRDD.collect.foreach(println)
      ```

   > `val hiveCtx = new HiveContext(sc)`这句会报`there was one deprecation warning`，这是因为较新的版本中会使用更简洁的方式创建`SQL`环境，可以参考下面的`HiveTest`代码文件。

   过程中没有出现错误，并且最后输出我们插入的那条记录，则测试成功。


### 在IDEA中，使用Spark连接Hive

`Spark`工作流一般是：

1. 写代码
2. 打成`jar`包
3. 提交到资源管理器中运行

第二步的时间都很长，第三步需要输入命令，如果是测试验证的时候，那会耗费许多时间，所以能在IDEA中直接运行，会极大地提高开发效率。

我们假设在`IDEA`中已经存在一个`Spark`项目，并且配置好了`Scala`环境以及`Maven`依赖。

1. `pom.xml`添加`Hive`依赖

   ```xml
   <dependency>
       <groupId>org.apache.spark</groupId>
       <artifactId>spark-sql_${scala.version}</artifactId>
       <version>${spark.version}</version>
       <scope>provided</scope>
   </dependency>
   <dependency>
       <groupId>org.apache.spark</groupId>
       <artifactId>spark-hive_${scala.version}</artifactId>
       <version>${spark.version}</version>
       <scope>provided</scope>
   </dependency>
   ```

2. 添加配置文件到项目的`src/main/resources`

   需要3个文件：

   1. `core-site.xml`（位于`${HADOOP_HOME}/etc/hadoop`）
   2. `hdsf-site.xml`（位于`${HADOOP_HOME}/etc/hadoop`）
   3. `hive-site.xml`（我们刚才创建于`${SPARK_HOME}/conf`的文件）

3. 创建测试代码文件

   ```scala
   import org.apache.spark.SparkConf
   import org.apache.spark.sql.SparkSession
   
   object HiveTest {
   	def main(args: Array[String]): Unit = {
   		val conf = new SparkConf()
   		conf.setAppName("TestHive").setMaster("local")
   		val spark = SparkSession.builder().config(conf).enableHiveSupport().getOrCreate()
   		spark.sql("select * from sparktest.contact").collect().foreach(println)
   	}
   }
   ```

   直接运行应该会报`java.lang.NoClassDefFoundError`，这是因为我们在添加依赖的时候作用域为`provided`，`IDEA`运行时不会加载这个`jar`包，解决方案如下：

   1. 点击运行按钮旁边的下拉框，点击`Edit Configurations`
   2. 勾选右侧中间位置的`Include dependencies with "Provided" scope`
   3. 点击应用

   再运行文件，如果没有报错，且输出我们插入的记录，则测试成功。

> 我在实现在`IDEA`中直接运行的时候，一直去修改`${SPARK_HOME}`的配置，但是根本不会生效。其实我们在`IDEA`运行的时候，`Spark`环境跟我们安装的`Spark`是独立的，也就是安装的`Spark`里面的配置跟在`IDEA`中运行没有任何关系，所以我们需要将这些关键配置放到`IDEA`项目中才可以。 
> 


> 在上面提到`Spark`获取`Hive metastore`的方式是使用`thrift`接口，其实也可以直接连接`MySQL`数据库，这种方式要注意`MySQL`的驱动包要能被加载。
> 


> 忽略掉了`Hadoop`安装、`Spark`安装、`Spark`开发环境搭建等流程，它们都是比较独立的单元，而且在网上很容易找到相关的教程，这里就不赘述了。
> 

