## Exporter数据导出
![爬虫框架](E:/坚果云同步/atom书籍/爬虫知识/res/Exporter导出数据.png)
### 1. 内置的数据导出器
**在Scrapy中，负责导出数据的组件被称为Exporter(导出器)，scrapy内部实现多个Exporter，每个Exporter实现一种数据格式的导出，总共支持6中数据格式，前4种是极为常见的文本数据格式，而后两种是python特有的，但是可以看出并没有txt和excel格式的导出器,在scrapy.settings.default_settings.py可以查看**
```python
  FEED_EXPORTERS_BASE = {
    'json': 'scrapy.exporters.JsonItemExporter',
    'jsonlines': 'scrapy.exporters.JsonLinesItemExporter',
    'jl': 'scrapy.exporters.JsonLinesItemExporter',
    'csv': 'scrapy.exporters.CsvItemExporter',
    'xml': 'scrapy.exporters.XmlItemExporter',
    'marshal': 'scrapy.exporters.MarshalItemExporter',
    'pickle': 'scrapy.exporters.PickleItemExporter',
}
```

### 2. 指定导出数据
在导出数据时，需向Scrapy爬虫提供以下信息:

  - **导出文件路径**
  - **导出数据格式(即选用哪个Exporter**)

  #### 2.1 命令行参数指定
  ```python
    scrapy crawl 爬虫名 -o 文件路径 -t 数据格式(json,xml)
    形如:
    scrapy crawl bookSpider -o book.json
  ```
  **上述中没有指定导出的文件格式，但是根据后缀可以推算出要使用哪个导出器，使用命令行参数指定如何导出数据很方便，但命令行参数只能指定导出文件和路径，并且每次都在命令行里输入很长的参数很麻烦，这时候可以考虑配置文件**

#### 2.2 配置文件指定

- **FEED_URL：导出文件路径**
```python
  FEED_URL ='export_data/%(name)s.data'
  # %(name)s：爬虫的名字，%(time)s：文件创建时间
```
- **FEED_FORMAT：导出数据格式**
```python
  FEED_FORMAT ='csv'
```
- **FEED_EXPORT_ENCODING：导出文件编码**
```python
  FEED_EXPORT_ENCODING ='utf-8'
```
- **FEED_EXPORT_FIELDS：导出数据包含的字段，并指定次序**
```python
  FEED_EXPORT_FIELDS=['name','author','price']
```
- **FEED_EXPORTERS：用户自定义Exporter字典，添加新的导出数据格式时使用。**
```python
  FEED_EXPORTERS ={'excel','my_project.exporters.ExcelItemExporter'}
```
### 3. 添加导出格式
- **源码参考: scrapy 中的exporters类**
```python
  class BaseItemExporter(object):

    def __init__(self, **kwargs):
        self._configure(kwargs)

    def _configure(self, options, dont_fail=False):
        """Configure the exporter by poping options from the ``options`` dict.
        If dont_fail is set, it won't raise an exception on unexpected options
        (useful for using with keyword arguments in subclasses constructors)
        """
        self.encoding = options.pop('encoding', None)
        self.fields_to_export = options.pop('fields_to_export', None)
        self.export_empty_fields = options.pop('export_empty_fields', False)
        self.indent = options.pop('indent', None)
        if not dont_fail and options:
            raise TypeError("Unexpected options: %s" % ', '.join(options.keys()))

    def export_item(self, item):
        raise NotImplementedError

    def serialize_field(self, field, name, value):
        serializer = field.get('serializer', lambda x: x)
        return serializer(value)

    def start_exporting(self):
        pass

    def finish_exporting(self):
        pass

    def _get_serialized_fields(self, item, default_value=None, include_empty=None):
        """Return the fields to export as an iterable of tuples
        (name, serialized_value)
        """
        if include_empty is None:
            include_empty = self.export_empty_fields
        if self.fields_to_export is None:
            if include_empty and not isinstance(item, dict):
                field_iter = six.iterkeys(item.fields)
            else:
                field_iter = six.iterkeys(item)
        else:
            if include_empty:
                field_iter = self.fields_to_export
            else:
                field_iter = (x for x in self.fields_to_export if x in item)

        for field_name in field_iter:
            if field_name in item:
                field = {} if isinstance(item, dict) else item.fields[field_name]
                value = self.serialize_field(field, field_name, item[field_name])
            else:
                value = default_value

            yield field_name, value

  class JsonItemExporter(BaseItemExporter):

    def __init__(self, file, **kwargs):
        self._configure(kwargs, dont_fail=True)
        self.file = file
        # there is a small difference between the behaviour or JsonItemExporter.indent
        # and ScrapyJSONEncoder.indent. ScrapyJSONEncoder.indent=None is needed to prevent
        # the addition of newlines everywhere
        json_indent = self.indent if self.indent is not None and self.indent > 0 else None
        kwargs.setdefault('indent', json_indent)
        kwargs.setdefault('ensure_ascii', not self.encoding)
        self.encoder = ScrapyJSONEncoder(**kwargs)
        self.first_item = True

    def _beautify_newline(self):
        if self.indent is not None:
            self.file.write(b'\n')

    def start_exporting(self):
        self.file.write(b"[")
        self._beautify_newline()

    def finish_exporting(self):
        self._beautify_newline()
        self.file.write(b"]")

    def export_item(self, item):
        if self.first_item:
            self.first_item = False
        else:
            self.file.write(b',')
            self._beautify_newline()
        itemdict = dict(self._get_serialized_fields(item))
        data = self.encoder.encode(itemdict)
        self.file.write(to_bytes(data, self.encoding))
```
- **def export_item(self, item): 负责导出爬取到的每一项数据，参数item为一项爬取到的数据，每个子类必须实现该方法。**
- **def start_exporting(self):导出开始时调用，用于初始化工作。**
- **def finish_exporting(self):导出结束时调用，用于清理工作。**
#### 3.1 实现excel导出器
 - **创建一个myexporter.py与spider文件夹同级，并编写如下代码**
```python
from scrapy.exporters import BaseItemExporter
import xlwt

class ExcelItemExporter(BaseItemExporter):
    def __init__(self,file,**kwargs):
        self._configure(kwargs)
        self.file = file
        self.wbook = xlwt.Workbook()
        self.wsheet = self.wbook.add_sheet('scrapy')
        self.row = 0
    def export_item(self, item):
        fields = self._get_serialized_fields(item)
        for col,v in enumerate(x for _,x in fields):
            self.wsheet.write(self.row,col,v)
        self.row += 1
    def finish_exporting(self):
        self.wbook.save(self.file)
```
- 然后开启
```python
  FEED_EXPORTERS ={'excel','my_project.exporters.ExcelItemExporter'}
```
