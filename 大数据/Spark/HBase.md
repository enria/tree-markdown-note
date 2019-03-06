## Spark对HBase中的数据进行计算

Spark版本：2.3.2

HBase版本：2.1.0

### 项目中添加依赖

```xml
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-mapreduce</artifactId>
    <version>${hbase_version}</version>
</dependency>
```

### 驱动程序

对微博名进行字符统计

```scala
import java.util.Base64

import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.client.Scan
import org.apache.hadoop.hbase.mapreduce.TableInputFormat
import org.apache.hadoop.hbase.protobuf.ProtobufUtil
import org.apache.hadoop.hbase.util.Bytes
import org.apache.spark.{SparkConf, SparkContext}

object WeiboNameWordCount {
	val HBASE_QUORUM = "hbase-1,hbase-2,hbase-3"
	val DEFAULT_COLUMN_FAMILY = "info"

	def main(args: Array[String]): Unit = {
		/** 创建Spark环境 */
		val sparkConf = new SparkConf()
		sparkConf.setMaster("local[3]")
		sparkConf.setAppName("category-grouping")
		val sc = new SparkContext(sparkConf)

		/** 创建HBase连接参数 */
		val conf = HBaseConfiguration.create
		conf.set("hbase.zookeeper.quorum", HBASE_QUORUM); // zookeeper地址
		conf.set("hbase.zookeeper.property.clientPort", "2181"); // zookeeper端口
		val scanToString = {
			val scan = new Scan()
			scan.setCaching(1000)
			scan.addColumn(Bytes.toBytes(DEFAULT_COLUMN_FAMILY), Bytes.toBytes("weibo_name"))

			import java.io.IOException
			try {
				val proto = ProtobufUtil.toScan(scan)
				Base64.getEncoder.encodeToString(proto.toByteArray)
			} catch {
				case io: IOException =>
					System.exit(0)
					""
			}
		}
		val tableName = "base_weibo"
		conf.set(TableInputFormat.INPUT_TABLE, tableName)
		conf.set(TableInputFormat.SCAN, scanToString)

		/** 转换RDD */
		val hbaseRDD = sc.newAPIHadoopRDD(conf, classOf[TableInputFormat],
			classOf[org.apache.hadoop.hbase.io.ImmutableBytesWritable],
			classOf[org.apache.hadoop.hbase.client.Result])
			.map(record => {//微博名称
				Bytes.toString(record._2.getValue(Bytes.toBytes(DEFAULT_COLUMN_FAMILY), Bytes.toBytes("weibo_name")))
			})
			.flatMap(name => {//字符列表
				name.toCharArray
			})
			.map(c => (c, 1))//计数单元
			.reduceByKey((c1, c2) => {//按字符计数
				c1 + c2
			})
			.sortBy(r => r._2, false)//按字符出现次数反序输出

		/** 输出结果 */
		hbaseRDD.collect().foreach(println)
	}
}

```

### 结果

#### 扫描数据

35,379,898

#### 耗时

193s

#### 输出

| 字符 | 数量 | 字符 | 数量 | 字符 | 数量 |
| ---- | ---- | ---- | ---- | ---- | ---- |
|_|3912877|的|3670318|1|3422424|
|小|3361823|-|2895002|0|2682679|
|2|2480696|e|2477739|a|2393230|
|i|2341248|n|2204514|9|2056759|
|8|1968276|o|1770101|6|1695881|
|3|1693724|大|1634844|5|1600575|
|7|1529272|l|1445567|一|1406578|
|y|1366262|子|1347075|r|1284040|
|4|1261470|天|1157501|s|1156777|
|人|1006324|爱|999964|u|996038|
|我|990024|是|902820|美|872080|
|不|862974|h|861916|c|847083|
|海|782100|有|774657|g|772641|
|王|766665|生|743914|t|739075|
|A|733062|m|732763|家|701067|
|L|677402|心|671736|宝|639846|
|金|619846|中|616843|花|614779|
|S|588609|文|585040|乐|579944|
|d|575398|M|570729|李|569388|
|华|557452|C|550997|张|539466|
|学|537339|公|537114|儿|535908|
|林|530504|你|529935|阿|525937|
|安|519313|b|504863|星|497784|
|南|483976|山|483279|西|478793|
|新|478441|k|473479|明|470659|
|风|468955|东|467040|上|463931|
|博|448568|水|447404|阳|444708|
|Y|444502|女|441960|v|441011|
|w|434189|好|426508|z|424662|
|三|423889|月|422958|光|419305|
|青|419261|木|419225|老|415380|
|云|415178|龙|411396|微|408582|
|陈|407690|梦|398555|p|396399|
|店|395824|司|393189|飞|393076|
|T|392330|国|383426|之|382644|
|O|381702|H|380854|E|380044|
|红|379598|J|377224|在|374829|
|高|371108|刘|367153|D|365186|
|雨|362827|北|362396|N|361554|
|宇|360051|哥|359313|长|358536|
|佳|357779|可|354505|丽|347719|
|和|346306|成|345944|白|345010|
|I|344290|了|341214|会|337358|
|马|333808|城|332493|个|329608|
|江|327605|年|326353|杨|321194|
|啊|314576|行|314248|州|312465|
|来|312373|限|311559|二|310493|
|方|308140|福|308022|丶|305299|
|达|304527|品|304018|x|303889|
|德|299072|思|298227|里|297735|

------

| 字符 | 数量 | 字符 | 数量 | 字符 | 数量 |
| ---- | ---- | ---- | ---- | ---- | ---- |
|趧|1|脋|1|ａ|1|
|邿|1|巗|1|醩|1|
|趡|1|憻|1|齇|1|
|炋|1|膉|1|廱|1|
|穯|1|馛|1|貙|1|
|甅|1|俇|1|嶿|1|
|揝|1|豻|1|氝|1|
|籉|1|馉|1|瑻|1|
|鷁|1|蠙|1|蔇|1|
|墑|1|籡|1|穣|1|
|傇|1|輳|1|欕|1|
|鰋|1|緳|1|屩|1|
|∫|1|鵳|1|鏣|1|
|靹|1|衏|1|褳|1|
|歝|1|冹|1|尡|1|
|騥|1|僉|1|螹|1|
|觭|1|趏|1|趛|1|
|舓|1|鄁|1|貱|1|
|鍽|1|蚇|1|魩|1|
|爣|1|斍|1|剳|1|
|綇|1|諅|1|摡|1|
|㈡|1|錗|1|搱|1|
|覫|1|橕|1|櫙|1|
|é|1|鯧|1|踇|1|
|輹|1|灉|1|鑧|1|
|鵹|1|鉵|1|蘛|1|
|痝|1|稉|1|陙|1|
|尳|1|齱|1|頧|1|
|憣|1|饇|1|笝|1|
|廇|1|珁|1|鵛|1|
|鈧|1|覟|1|珇|1|
|诇|1|鐱|1|稡|1|
|筓|1|齓|1|膧|1|
|糓|1|嚙|1|偯|1|
|珓|1|♩|1|瞁|1|
|鸃|1|軷|1|膕|1|
|剕|1|觕|1|褉|1|
|戃|1|躝|1|槣|1|
|疭|1|鼵|1|髇|1|
|陫|1|傽|1|褏|1|
|籕|1|鐇|1|驏|1|
|弳|1|磩|1|磵|1|
|皗|1|靭|1|扅|1|
|蟅|1|趃|1|楇|1|
|榏|1|焿|1|♣|1|
|觡|1|袑|1|鲿|1|
|乥|1|鐍|1|幭|1|
|歑|1|鋏|1|赥|1|
|躋|1|瓭|1|錣|1|
|腇|1|嶃|1|靧|1|

