---
banner: "[[pixel-banner-image.png]]"
title: ElasticSearch分布式搜索引擎
tags:
  - ElasticSearch
categories: 编程
date: 2025-02-24T18:46:00
---

# ElasticSearch分布式搜索引擎
[参考文献](https://www.cnblogs.com/buchizicai/p/17093719.html)
## 概念理解
- 当你在搜索栏输入'显卡'的时候, 发送请求到后端, 请求携带'显卡'这个字符串到es的分词器进行分词, 发现不可拆分, 以'显卡'为词条然后查询es索引库得到在MySQL中ids(ES 根据倒排索引查出匹配的文档), 根据ids到MySQL中去查找数据getByIds, 这就是es充当的角色 -> 中间件
- ElasticSearch是一个分布式搜索引擎中间件, 用来过滤查找符合条件的数据, 也可以结合Logstash做日志统计、分析、系统监控等功能
![17584443366841758444336429.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17584443366841758444336429.png)
- Kibana可以给我们提供一个elasticsearch的可视化界面: 9200 -> 5601
## 倒排索引
- 核心原理：通过单词一文档ID的映射实现高速搜索（如搜索“手机”返回所有含该词的文档）
- 倒排索引的概念是基于MySQL这样的正向索引而言的, **倒排只能给词条创建索引**，而不是字段
- **文档**（`Document`）：其中的每一条数据就是一个文档, 具有文档唯一id(自动生成)
- **词条**（`Term`）：对文档数据或用户搜索数据，利用某种算法分词，得到的具备含义的词语就是词条。例如：我是中国人，就可以分为：我、是、中国人、中国、国人这样的几个词条
![17403951668551740395166641.png|706x449](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17403951668551740395166641.png)
![17403952738551740395273706.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17403952738551740395273706.png)
## 分词器
- 分词器的选用是es搜索的精髓
- 创建倒排索引时对文档分词,根据自定义分词规则进行字符串分词
- 默认分词器为 Standard分词器, 但是对中文分词不友好
- 常用的分词器: IK分词器 有两种模式: 例如我爱你
	- `ik_smart`：最少切分 -> 我爱你
	- `ik_max_word`：最多切分 -> 我爱你 爱你
- ”analyzer“模式: 选择分词器的开关
- 停用词: 所以很多语言是不允许在网络上传递的, 在索引时就直接忽略当前的停用词汇表中的内容
## 客户端
- JavaRestClient操作ElasticSearch的流程基本类似
- 索引库操作的基本步骤
	- 初始化RestHighLevelClient客户端 client
	- 创建XxxRequest
	- 准备DSL语句
	- 发送请求
```Java
/*索引表CRUD*/  
@Test  
public void createIndex() throws IOException {  
    // 第一步: 创建request对象 index为表名  
    CreateIndexRequest request = new CreateIndexRequest("hotel");  
  
    // 第二步: 封装请求参数: 创建表 PUT/hotel  -->  DSL语句  
    request.source(esConstants.CREATE_DSL, XContentType.JSON);  
  
    // 第三步: 发起请求  
    client.indices().create(request, RequestOptions.DEFAULT);  
}
```
```Java
/*文档CRUD*/  
@Test  
public void matchAllDocument() throws IOException {  
    // 第一步: 创建request对象  
    SearchRequest request = new SearchRequest("user");  
  
    // 第二步: DSL语句  
    request.source().query(QueryBuilders.matchAllQuery());  
  
    // 第三步: 发起请求  
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);  
  
    // 第四步: 拿到数据  
    SearchHits searchHits = response.getHits();  
    TotalHits totalHits = searchHits.getTotalHits();  
    for (SearchHit hit : searchHits.getHits()) {  
        System.out.println(hit.getSourceAsString());  
    }  
}
```
## 可视化界面条件检索
- match_all: 查询所有
- multi_match：多字段查询，任意一个字段符合条件就算符合查询条
- term：根据词条精确值查询
- range：根据值的范围查询
- geo_distance: 根据地理位置查询(经纬度)
	- `took` – Elasticsearch运行查询所花费的时间（以毫秒为单位）
	- `timed_out` –搜索请求是否超时
	- `_shards` - 搜索了多少个碎片，以及成功，失败或跳过了多少个碎片的细目分类。
	- `max_score` – 找到的最相关文档的分数
	- `hits.total.value` - 找到了多少个匹配的文档
	- `hits.sort` - 文档的排序位置（不按相关性得分排序时）
	- `hits._score` - 文档的相关性得分（使用match_all时不适用）
```bash
GET /表名/_search(类型: 文档/搜索)
{
	# 搜索规则
    "query": {
      "查询条件": "条件值"
    },
    # 排序规则
    "sort": [
    { "account_number": "asc" }
   ],
   # 分页查询
    "from": 10,
    "to": 10 
}

PUT /索引表名/_doc(类型: 文档/搜索)/文档id
{
# 存放内容
  "id": "1",
  "name" : "唐伯虎点秋香",
  "realName": "唐伯虎"
}
```
## 问题
- 创建索引库，最关键的是mapping映射，而mapping映射要考虑的信息包括：
	- 字段名是什么? 字段数据类型是什么?  字段名、字段数据类型，可以参考数据表结构的名称和类型
	- 是否参与搜索?  是否参与搜索要分析业务来判断，例如图片地址，就无需参与搜索
	- 是否需要分词?  是否分词呢要看内容，内容如果是一个整体就无需分词，反之则要分词 -> 现有的分词库
	- 如果分词，分词器是什么？ 分词器，我们可以统一使用ik_max_word类型
- ES与Mysql数据同步?  同步更新 vs 异步更新(mq+定时任务)
 ![17584364152201758436414246.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17584364152201758436414246.png)
- TF-IDF相关性算法?  score = TF * IDF
	- TF = 词条出现的次数/词条总数  
	- IDF = log(文档总数/出现词条的文档数)
- es和mysql的like的不同
	- 比如搜索 显卡9060和9060显卡 es两种都能搜索到, 而like匹配只能匹配一种 
	- es还能进行范围geo搜索, 拼音搜索和相关性搜索
## 总结
- 索引的CRUD
	- **创建索引** (`PUT/user`): 用于创建索引。
	- **查看索引** (`GET/user`): 获取索引的配置和信息。
	- **删除索引** (`DELETE/user`): 删除指定的索引。
- 文档的CRUD
	- **创建文档** (`PUT/user/_doc/1001`): 向索引中插入一个新的文档。
	- **查看文档** (`GET/user/_doc/1001`): 根据文档 ID 查看文档的内容。
	- **更新文档** (`POST/user/_update/1001`): 更新文档的某些字段，不会替换整个文档。
	- **删除文档** (`DELETE/user/_doc/1001`): 删除指定 ID 的文档。
	- **查询文档** (`GET/user/_search`): 根据查询条件检索符合条件的文档。
- 什么是ELK
	- ELK = Elasticsearch + Logstash + Kibana
	- Logstash负责日志收集, 解析以及转发到es, es负责存储数据, Kibanna负责可视化工具，提供图表、仪表盘、搜索分析界面