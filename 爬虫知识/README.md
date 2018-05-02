# 爬虫知识框架
[![爬虫知识.png](https://i.loli.net/2018/05/02/5ae91e01002f3.png)](https://i.loli.net/2018/05/02/5ae91e01002f3.png)

# 一般爬虫的流程图
```flow
st=>start: 开始爬虫
e=>end: 结束爬虫
op=>operation: 创建项目并
确定Item字段
op1=>operation: 对网页发起请求
sub1=>subroutine: 添加请求头部:
  用户代理
  cookie
  http代理
  验证码
op3=>operation: 网页提取(Selector)数据
io=>inputoutput: scrapy数据获取测试、PipeLines数据清洗
op4=>operation: 数据库的储存、数据直接导出
cond=>condition: 请求成功
or 请求失败?
cond1=>condition: 数据是否处理?
st->op->op1->cond
cond(yes)->op3->cond1(no)->io->op4
cond(no)->sub1->op1
cond1(yes)->op4->e
