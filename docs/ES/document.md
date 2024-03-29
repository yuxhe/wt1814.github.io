
<!-- TOC -->

- [1. 文档操作](#1-文档操作)
    - [1.1. 新建文档](#11-新建文档)
    - [1.2. 获取文档](#12-获取文档)
    - [1.3. 文档更新](#13-文档更新)
        - [1.3.1. 普通更新](#131-普通更新)
        - [1.3.2. 查询更新](#132-查询更新)
    - [1.4. 删除文档](#14-删除文档)
    - [1.5. 批量操作](#15-批量操作)

<!-- /TOC -->

# 1. 文档操作  

<!-- 
~~
https://mp.weixin.qq.com/s?__biz=MzI1NDY0MTkzNQ==&mid=2247490740&idx=1&sn=ba34fcdb40b82361aab3c738bcea2aa9&scene=21#wechat_redirect
https://mp.weixin.qq.com/s?__biz=MzI1NDY0MTkzNQ==&mid=2247490840&idx=2&sn=3bf45591fb8d383c06b49b16331482b3&scene=21#wechat_redirect
-->

## 1.1. 新建文档
&emsp; 首先新建一个索引。  
&emsp; 然后向索引中添加一个文档：  

```text
PUT blog/_doc/1
{
  "title":"6. ElasticSearch 文档基本操作",
  "date":"2020-11-05",
  "content":"微信公众号**江南一点雨**后台回复 **elasticsearch06** 下载本笔记。首先新建一个索引。"
}
```
&emsp; 1 表示新建文档的 id。  
&emsp; 添加成功后，响应的 json 如下：  

```text
{
  "_index" : "blog",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```
    _index 表示文档索引。
    _type 表示文档的类型。
    _id 表示文档的 id。
    _version 表示文档的版本（更新文档，版本会自动加 1，针对一个文档的）。
    result 表示执行结果。
    _shards 表示分片信息。
    _seq_no 和 _primary_term 这两个也是版本控制用的（针对当前 index）。

&emsp; 添加成功后，可以查看添加的文档：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/ES/es-27.png)  
&emsp; 当然，添加文档时，也可以不指定 id，此时系统会默认给出一个 id，如果不指定 id，则需要使用 POST 请求，而不能使用 PUT 请求。  

```text
POST blog/_doc
{
  "title":"666",
  "date":"2020-11-05",
  "content":"微信公众号**江南一点雨**后台回复 **elasticsearch06** 下载本笔记。首先新建一个索引。"
}
```

## 1.2. 获取文档
&emsp; Es 中提供了 GET API 来查看存储在 es 中的文档。使用方式如下：  
&emsp; GET blog/_doc/RuWrl3UByGJWB5WucKtP  
&emsp; 上面这个命令表示获取一个 id 为 RuWrl3UByGJWB5WucKtP 的文档。  
&emsp; 如果获取不存在的文档，会返回如下信息：  

```text
{
  "_index" : "blog",
  "_type" : "_doc",
  "_id" : "2",
  "found" : false
}
```
&emsp; 如果仅仅只是想探测某一个文档是否存在，可以使用 head 请求：  
&emsp; 如果文档不存在，响应如下：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/ES/es-28.png)  
&emsp; 如果文档存在，响应如下：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/ES/es-29.png)  
&emsp; 当然也可以批量获取文档。  

```text
GET blog/_mget
{
  "ids":["1","RuWrl3UByGJWB5WucKtP"]
}
```
&emsp; 这里可能有小伙伴有疑问，GET 请求竟然可以携带请求体？  
&emsp; 某些特定的语言，例如 JavaScript 的 HTTP 请求库是不允许 GET 请求有请求体的，实际上在 RFC7231 文档中，并没有规定 GET 请求的请求体该如何处理，这样造成了一定程度的混乱，有的 HTTP 服务器支持 GET 请求携带请求体，有的 HTTP 服务器则不支持。虽然 es 工程师倾向于使用 GET 做查询，但是为了保证兼容性，es 同时也支持使用 POST 查询。例如上面的批量查询案例，也可以使用 POST 请求。    

## 1.3. 文档更新
### 1.3.1. 普通更新
&emsp; 注意，文档更新一次，version 就会自增 1。  
&emsp; 可以直接更新整个文档：  

```text
PUT blog/_doc/RuWrl3UByGJWB5WucKtP
{
  "title":"666"
}
```
&emsp; 这种方式，更新的文档会覆盖掉原文档。  
&emsp; 大多数时候，我们只是想更新文档字段，这个可以通过脚本来实现。  

```text
POST blog/_update/1
{
  "script": {
    "lang": "painless",
    "source":"ctx._source.title=params.title",
    "params": {
      "title":"666666"
    }
  }
}
```
&emsp; 更新的请求格式：POST {index}/_update/{id}  
&emsp; 在脚本中，lang 表示脚本语言，painless 是 es 内置的一种脚本语言。source 表示具体执行的脚本，ctx 是一个上下文对象，通过 ctx 可以访问到_source、_title 等。   
&emsp; 也可以向文档中添加字段：    

```text
POST blog/_update/1
{
  "script": {
    "lang": "painless",
    "source":"ctx._source.tags=[\"java\",\"php\"]"
  }
}
```
&emsp; 添加成功后的文档如下：   
![image](https://gitee.com/wt1814/pic-host/raw/master/images/ES/es-30.png)  
&emsp; 通过脚本语言，也可以修改数组。例如再增加一个 tag：  

```text
POST blog/_update/1
{
  "script":{
    "lang": "painless",
    "source":"ctx._source.tags.add(\"js\")"
  }
}
```
&emsp; 当然，也可以使用 if else 构造稍微复杂一点的逻辑。  

```text
POST blog/_update/1
{
  "script": {
    "lang": "painless",
    "source": "if (ctx._source.tags.contains(\"java\")){ctx.op=\"delete\"}else{ctx.op=\"none\"}"
  }
}
```

### 1.3.2. 查询更新
&emsp; 通过条件查询找到文档，然后再去更新。  
&emsp; 例如将 title 中包含 666 的文档的 content 修改为 888。  

```text
POST blog/_update_by_query
{
  "script": {
    "source": "ctx._source.content=\"888\"",
    "lang": "painless"
  },
  "query": {
    "term": {
      "title":"666"
    }
  }
}
```

## 1.4. 删除文档
&emsp; 根据 id 删除  
&emsp; 从索引中删除一个文档。  
&emsp; 删除一个 id 为 TuUpmHUByGJWB5WuMasV 的文档。  

```text
DELETE blog/_doc/TuUpmHUByGJWB5WuMasV
```
&emsp; 如果在添加文档时指定了路由，则删除文档时也需要指定路由，否则删除失败。  

&emsp; 查询删除  
&emsp; 查询删除是 POST 请求。  
&emsp; 例如删除 title 中包含 666 的文档：  

```text
POST blog/_delete_by_query
{
  "query":{
    "term":{
      "title":"666"
    }
  }
}
```

&emsp; 也可以删除某一个索引下的所有文档：  

```text
POST blog/_delete_by_query
{
  "query":{
    "match_all":{
      
    }
  }
}
```

## 1.5. 批量操作
&emsp; es 中通过 Bulk API 可以执行批量索引、批量删除、批量更新等操作。 
&emsp; 首先需要将所有的批量操作写入一个 JSON 文件中，然后通过 POST 请求将该 JSON 文件上传并执行。  
&emsp; 例如新建一个名为 aaa.json 的文件，内容如下：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/ES/es-31.png)  
&emsp; 首先第一行：index 表示要执行一个索引操作（这个表示一个 action，其他的 action 还有 create，delete，update）。_index 定义了索引名称，这里表示要创建一个名为 user 的索引，_id 表示新建文档的 id 为 666。  
&emsp; 第二行是第一行操作的参数。  
&emsp; 第三行的 update 则表示要更新。  
&emsp; 第四行是第三行的参数。  
&emsp; 注意，结尾要空出一行。  

&emsp; aaa.json 文件创建成功后，在该目录下，执行请求命令，如下：  

```text
curl -XPOST "http://localhost:9200/user/_bulk" -H "content-type:application/json" --data-binary @aaa.json
```
&emsp; 执行完成后，就会创建一个名为 user 的索引，同时向该索引中添加一条记录，再修改该记录，最终结果如下：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/ES/es-32.png)  


