## PipeLines数据处理
[![处理数据.png](https://i.loli.net/2018/05/02/5ae91e00e7e62.png)](https://i.loli.net/2018/05/02/5ae91e00e7e62.png)

### 1.实现Item PipeLine
  **在创建一个Scrapy项目时，会自动生成一个pipelines.py文件，它用来放置用户自定义的Item Pipeline。**

  - **一个Item Pipeline不需要继承特定基类，只需要实现某些特定方法，例如process_item、open_spider、close_spider**
```python
  class examplePipeline(object):
    def process_item(self, item, spider):
        return item

    def open_spider(self, item, spider):
        pass

    def close_spider(self, item, spider):
        pass
```
  - **一个Item Pipeline必须实现一个process_item(item,spider)方法，该方法用来处理每一项Spider爬取到的数据，其中的两个参数:**

    - Item  爬取到的一项数据(Item 或字典)
    - spider 爬取此项数据的Spider字典
  - **如果process_item在处理某项item时返回了一项数据(Item或字典)，返回的数据会递送给下一级Item Pipeline(如果有)继续处理。**
  - **如果process_item在处理某项item时抛出(raise)一个DropItem异常(scrapy.exceptions.DropItem),该项item便会被抛弃，不再递送给后面的Item PipeLine 继续处理，也不会导出到文件。通常，我们在检测到无效数据或想要过滤数据时，抛出DropItem异常。**
  - **open_spider(self,spider):Spider打开时(处理数据前)回调该方法,通常该方法用于在开始处理数据之前完成某些初始化工作，如连接数据库。**
  - **close_spider(self,spider):Spider关闭时(处理数据后)回调该方法,通常该方法用于在开始处理数据之后完成某些清理工作，如关闭数据库。**
  - **from_crawler(self,crawler):在创建Item PipeLine对象时回调该类方法，可以通过此方法获取配置设置的字段，如数据库地址**
### 2. 开启Item PipeLine
**通过settings.py进行设置开启,后面的数据1~1000,数字越大越晚执行**
```python
# Configure item pipelines
# See https://doc.scrapy.org/en/latest/topics/item-pipeline.html
ITEM_PIPELINES = {
#    'jdspider.pipelines.JdspiderPipeline': 300,
     'jdspider.pipelines.MongoDBPipeline': 500,
     'jdspider.pipelines.DuplicatesPipeline':50,
     'jdspider.pipelines.PriceConverterPipeline':20,
}
```

### 3. 典型应用
  - #### 过滤重复数据
  ```python
from scrapy.exceptions import DropItem
from scrapy.item import Item

    #过滤重复数据
    class DuplicatesPipeline(object):
    def __init__(self):
        self.set = set() # 定义set集合去重

    def process_item(self,item,spider):
        name = item['name'] #需要去重item字段
        if name in self.book_set:
            # 如果发现数据重复就抛出异常，等价于删除这一条数据
            raise DropItem('Duplicate book found:%s'% item)
        self.book_set.add(name) # 没有重复就加入set集合
        return item
  ```
  - #### 数据转换
  ```python
    class PriceConverterPipeline(object):
    # 英磅到到元的匯率
    exchange_rate = 8.2343
    def process_item(self, item, spider):
        price = float(item['price'][1:])*self.exchange_rate
        item['price'] = '%.2f元'%price
        return item
  ```
  - #### 数据库储存
  ```python
  from scrapy.item import Item
  import pymongo
    class MongoDBPipeline(object):

    @classmethod  # 类方法，读取设置中的字段
    def from_crawler(cls,crawler):
        cls.DB_URI=crawler.settings.get('MONGGO_DB_URI','mongodb://localhost:27017')
        cls.DB_NAME = crawler.settings.get('DB_NAME','Data')
        return cls()
    # 数据库初始化
    def open_spider(self,spider):
        self.client =pymongo.MongoClient(self.DB_URI)
        self.db = self.client[self.DB_NAME]
    # 数据储存
    def process_item(self,item,spider):
        collection = self.db[topic['path']]
        data = dict(item) if isinstance(item,Item) else item
        collection.insert_one(data)
    # 关闭数据库
    def close_spider(self,spider):
        self.client.close()
    # settings.py 中的配置数据
    MONGGO_DB_URI = 'mongodb://localhost:27017'
    DB_NAME = 'Data'
  ```
