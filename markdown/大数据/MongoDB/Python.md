#### 连接

```python
import pymongo
client = pymongo.MongoClient("mongodb://{}:{}@{}:{}/{}".format('root', 'mysdpico', '172.18.31.106', 27017, 'admin'))
zhangyd_db = client.zhangyd
content_company_c = zhangyd_db.content_company
```

#### 游标

```python
import bson
mg_cursor = content_company_c.find_raw_batches()
for batch in mg_cursor:
    for item in bson.decode_all(batch):
        print(item)
```

#### 统计

```python
content_company_c.count()
```

