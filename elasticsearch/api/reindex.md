# reindex

Elasticsearch 本身不提供对索引的 rename，mapping 的 alter 等操作。所以，如果有需要对全索引数据进行导出，或者修改某个已有字段的 mapping 设置等情况下，我们只能通过 scroll API 导出全部数据，然后重新做一次索引写入。这个过程，叫做 reindex。

既然没有直接的方式，那么自然只能使用其他工具了。这里介绍两个常用的方法，自己写程序和用 logstash。

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
