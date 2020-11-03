# ElasticSearch 索引生命周期管理配置方法

## 概念理解

### Phase

在生命周期管理中每个索引都处于某个 Phase，如 Hot Phase 和 Delete Phase。

不同 Phase 之间的转换需要触发条件。

Hot Phase 代表该索引处于仍然被活跃地查询或写入中。

Delete Phase 代表该索引已不被需要。可以删除。

### Rollover

当索引处于 Hot Phase 时，可以开启 rollover 功能。

rollover 需要设置一定的触发条件，当 rollover 发生时，会停止写入当前索引并新建一个索引来写入；

如当前索引是 log-000001, rollover 发生时，会停止写入 log-000001 并自动新建 log-000002 来写入。

### Rollover Alias

因为在 rollover 的情况下，同一个写入源会对应多个索引，如 log-000001、log-000002 和 log-000003。

需要给所有的相关索引起一个统一的别名作为输入源（logstash）中配置的名称，即 rollover alias。

## 添加索引声明周期管理策略(ILM Policy)

进入 Kibana 左侧边栏-Management-ElasticSearch-Index Lifecycle Policy，点击 create policy：

![ElasticSearch 索引生命周期管理配置方法 - 图1](/Users/wenbo.zhao1/Documents/Blog/ElasticSearch 索引生命周期管理配置方法 - 图1.png)

在 Hot Phase 中 Enable rollover，并配置触发条件，如当索引达到 10g 或 1 天时触发 rollover

![ElasticSearch 索引生命周期管理配置方法 - 图2](/Users/wenbo.zhao1/Documents/Blog/ElasticSearch 索引生命周期管理配置方法 - 图2.png)

在 Delete Phase 中，配置 rollover 2 天后进入 Delete Phase 删除索引。

在该策略下，若索引没有在 1 天内超过 10g，则索引会在 3 天后自动删除，反之会在超过 10g 后 2 天被删除。

## 添加索引模板

Rollover 虽然会自动新建索引，但新建的索引却不会自动应用前面索引对应的生命周期管理策略，因此需要创建索引模板来统一将所有符合模板的索引都加入特定生命周期管理策略。

进入 Kibana 左侧边栏-Management-ElasticSearch-Index Management-Index Template，点击 create a template：

![ElasticSearch 索引生命周期管理配置方法 - 图3](/Users/wenbo.zhao1/Documents/Blog/ElasticSearch 索引生命周期管理配置方法 - 图3.png)

创建完成索引模板后，需要在 index settings 页面将索引生命周期管理策略与该模板绑定，并指定 rollover alias：

```json
{
  "index": {
    "lifecycle": {
      "name": "NameOfILMPolicy",
      "rollover_alias": "log-alias"
    }
  }
}
```

其中 name 为之前创建的生命周期管理策略名。

![ElasticSearch 索引生命周期管理配置方法 - 图4](/Users/wenbo.zhao1/Documents/Blog/ElasticSearch 索引生命周期管理配置方法 - 图4.png)

## 修改 Logstash

Logstash 需要修改为使用 rollover alias：

```yaml
    output {
      elasticsearch {
      hosts => ["http://elasticsearch:9200"]
      ilm_rollover_alias => "log-alias"
      }
    }
```

## 创建第一个索引

需要人工创建第一个索引让前述流程跑起来。

进入 Kibana 左侧边栏-Management-Dev Tools-Console，输入如下命令：

```json
PUT /log-000001
{
  "aliases": {
    "log-alias": {
      "is_write_index": true
    }
  }
}
```

创建一个名为 log-000001 的索引且指定该索引的 rollover alias 为 log-alias。

最后点击三角形图标执行即可：![ElasticSearch 索引生命周期管理配置方法 - 图5](/Users/wenbo.zhao1/Documents/Blog/ElasticSearch 索引生命周期管理配置方法 - 图5.png)

## 参考资料

https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html

https://medium.com/faun/elasticsearch-cluster-tuning-experience-with-indices-lifecycle-policies-386a78c94f96