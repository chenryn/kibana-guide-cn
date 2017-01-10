# reindex

Elasticsearch 本身不提供对索引的 rename，mapping 的 alter 等操作。所以，如果有需要对全索引数据进行导出，或者修改某个已有字段的 mapping 设置等情况下，我们只能通过 scroll API 导出全部数据，然后重新做一次索引写入。这个过程，叫做 reindex。

之前完成这个过程只能自己写程序或者用 logstash。5.0 中，Elasticsearch 将这个过程内置为 reindex API，但是要注意：这个接口并没有什么黑科技，其本质仅仅是将这段相同逻辑的代码预置分发而已。如果有复杂的数据变更操作等细节需求，依然需要自己编程完成。

下面分别给出这三种方法的示例：

## Perl 客户端

Elastic 官方提供各种语言的客户端库，其中，Perl 库提供了对 reindex 比较方便的写法和示例。通过 `cpanm Search::Elasticsearch` 命令安装库完毕后，使用以下程序即可：

```
use Search::Elasticsearch;
 
my $es   = Search::Elasticsearch->new(
    nodes => ['192.168.0.2:9200']
);
my $bulk = $es->bulk_helper(
    index   => 'new_index',
);
 
$bulk->reindex(
    source  => {
        index       => 'old_index',
        size        => 500,         # default
        search_type => 'scan'       # default
    }
);
```

## Logstash 做 reindex

在最新版的 Logstash 中，对 logstash-input-elasticsearch 插件做了一定的修改，使得通过 logstash 完成 reindex 成为可能。

reindex 操作的 logstash 配置如下：

```
input {
  elasticsearch {
    hosts => [ "192.168.0.2" ]
    index => "old_index"
    size => 500
    scroll => "5m"
    docinfo => true
  }
}
output {
  elasticsearch {
    host => [ "192.168.0.3" ]
    index => "%{[@metadata][_index]}"
    document_type => "%{[@metadata][_type]}"
    document_id => "%{[@metadata][_id]}"
    protocol => "http"
  }
}
```

如果你做 reindex 的源索引并不是 logstash 记录的内容，也就是没有 `@timestamp`, `@version` 这两个 logstash 字段，那么可以在上面配置中添加一段 filter 配置，确保前后索引字段完全一致：

```
filter {
  mutate {
    remove_field => [ "@timestamp", "@version" ]
  }
}
```

## reindex API

简单的 reindex，可以很容易的完成：

```
curl -XPOST http://localhost:9200/_reindex -d '
{
  "source": {
    "index": "logstash-2016.10.29"
  },
  "dest": {
    "index": "logstash-new-2016.10.29"
  }
}'
```

复杂需求，也能通过配合其他 API，比如 script、pipeline 等来满足一些，下面举一个复杂的示例：

```
curl -XPOST http://localhost:9200/_reindex?requests_per_second=10000 -d '
{
  "source": {
    "remote": {
      "host": "http://192.168.0.2:9200",
    },
    "index": "metricbeat-*",
    "query": {
      "match": {
        "host": "webserver"
      }
    }
  },
  "dest": {
    "index": "metricbeat",
    "pipeline": "ingest-rule-1"
  },
  "script": {
    "lang": "painless",
    "inline": "ctx._index = 'metricbeat-' + (ctx._index.substring('metricbeat-'.length(), ctx._index.length())) + '-1'"
  }
}'
```

上面这个请求的作用，是将来自 192.168.0.2 集群的 metricbeat-2016.10.29 索引中，有关 `host:webserver` 的数据，读取出来以后，经过 localhost 集群的 `ingest-rule-1` 规则处理，在写入 localhost 集群的 metricbeat-2016.10.29-1 索引中。

注意：读取远端集群数据需要先配置对应的 `reindex.remote.whitelist:192.168.0.2:9200` 到 elasticsearch.yml 的白名单里。

通过 reindex 接口运行的任务可以通过同样是 5.0 新引入的任务管理接口进行取消、修改等操作。详细介绍见后续任务管理章节。
