# zabbix

之前提到的都是 Elasticsearch 的 sites 类型插件，其实质是实时从浏览器读取 cluster stats 接口数据并渲染页面。这种方式直观，但不适合生产环境的自动化监控和报警处理。要达到这个目标，还是需要使用诸如 nagios、zabbix、ganglia、collectd 这类监控系统。

本节以 zabbix 为例，介绍如何使用监控系统完成 Elasticsearch 的监控报警。

github 上有好几个版本的 ESZabbix 仓库，都源自 Elastic 公司员工 untergeek 最早的贡献。但是当时 Elasticsearch 还没有官方 python 客户端，所以监控程序都是用的是 pyes 库。对于最新版的 ES 来说，已经不推荐使用了。

这里推荐一个修改使用了官方 `elasticsearch.py` 库的衍生版。GitHub 地址见：<https://github.com/Wprosdocimo/Elasticsearch-zabbix>。

## 安装配置

仓库中包括三个文件：

1. ESzabbix.py
2. ESzabbix.userparm
3. ESzabbix_templates.xml

其中，前两个文件需要分发到每个 ES 节点上。如果节点上运行的是 yum 安装的 zabbix，二者的默认位置应该分别是：

1. `/etc/zabbix/zabbix_externalscripts/ESzabbix.py`
2. `/etc/zabbix/agent_include/ESzabbix.userparm`

然后在各节点安装运行 ESzabbix.py 所需的 python 库依赖：

```
# yum install -y python-pbr python-pip python-urllib3 python-unittest2
# pip install elasticsearch
```

安装成功后，你可以试运行下面这行命令，看看命令输出是否正常：

```
# /etc/zabbix/zabbix_externalscripts/ESzabbix.py cluster status
```

最后一个文件是 zabbix server 上的模板文件，不过在导入模板之前，还需要先创建一个数值映射，因为在模板中，设置了集群状态的触发报警，没有映射的话，报警短信只有 0, 1, 2 数字不是很易懂。

创建数值映射，在浏览器登录 zabbix-web，菜单栏的 **Zabbix Administration** 中选择 **General** 子菜单，然后在右侧下拉框中点击 **Value Maping**。

选择 **create**， 新建表单中填写：

> name: ES Cluster State    
> 0 ⇒ Green
> 1 ⇒ Yellow
> 2 ⇒ Red

完成以后，即可在 **Templates** 页中通过 **import** 功能完成导入 `ESzabbix_templates.xml`。

在给 ES 各节点应用新模板之前，需要给每个节点定义一个 `{$NODENAME}` 宏，具体值为该节点 `elasticsearch.yml` 中的 `node.name` 值。从统一配管的角度，建议大家都设置为 ip 地址。

## 模板应用

导入完成后，zabbix 里多出来三个可用模板：

1. Elasticsearch Node & Cache
   其中包括两个 Application：ES Cache 和 ES Node。分别有 Node Field Cache Size, Node Filter Cache Size 和 Node Storage Size, Records indexed per second 共计 4 个 item 监控项。在完成上面说的宏定义后，就可以把这个模板应用到各节点(即监控主机)上了。
2. Elasticsearch Service
   只有一个监控项 Elasticsearch service status，做进程监控的，也应用到各节点上。
3. Elasticsearch Cluster
   包括 11 个监控项，如下列所示。其中，**ElasticSearch Cluster Status** 这个监控项连带有报警的触发器，并对应之前创建的那个 Value Map。
   * Cluster-wide records indexed per second
   * Cluster-wide storage size
   * ElasticSearch Cluster Status
   * Number of active primary shards
   * Number of active shards
   * Number of data nodes
   * Number of initializing shards
   * Number of nodes
   * Number of relocating shards
   * Number of unassigned shards
   * Total number of records
   这个模板下都是集群总体情况的监控项，所以，运用在一台有 ES 集群读取权限的主机上即可，比如 zabbix server。

## 其他

untergeek 最近刚更新了他的仓库，重构了一个 es_stats_zabbix 模块用于 Zabbix 监控，有兴趣的读者可以参考：<https://github.com/untergeek/zabbix-grab-bag/blob/master/Elasticsearch/es_stats_zabbix.README.md>


使用es_stats_zabbix模块监控 elasticsearch。并可扩展，比如添加 zabbix 的items，clusterstats，clusterstate，nodesstats，nodesinfo和API等等，形如：
health[status]
clusterstats[indices.docs.count]
clusterstate[master_node]
nodeinfo[YOUR_NODE_NAME,process.max_file_descriptors]
nodestats[YOUR_NODE_NAME,process.open_file_descriptors]

注意事项
你不能自定义监控项的字段为zabbix的一些关键名称；
Zabbix项目密钥长度限制为255个字节，所以这可能也限制了嵌套的监控项。

步骤如下：
下载该插件
1.git clone https://github.com/untergeek/zabbix-grab-bag.git

2.安装插件的py模块
(sudo) pip install es_stats_zabbix

3.添加Zabbix value maps
Go to: Administration （管理）→ General（常规）；
导航栏的右侧，下拉选择“Value mapping；
点击 Create value map (创建值映射)。

ES Cluster State
0 ⇒ Green
1 ⇒ Yellow
2 ⇒ Red

Exit Code
0 ⇒ SUCCESS
1 ⇒ FAIL

4.修改es_stats_zabbix.ini.sample配置文件，重命名为es_stats_zabbix.ini，并修改文件中的[elasticsearch] ，[logging] ， [thirty_seconds], [sixty_seconds],  [five_minutes], host , zabbix_host等字段为实际值。

5.修改es_stats_zabbix.userparm
将该文件放置到你的zabbix的conf.d下，并编辑agentd.conf，include该配置文件。编辑该文件，修改 es_stats _ zabbix路径，es_stats_zabbix.ini路径。重启Zabbix agent。

6.导入模板
Go to: Configuration （配置）→ Templates（模板）
导航栏的右侧，点击选择“Import”
点击 选择文件，导入es_stats_zabbix_template.xml文件。
因为官方提供的模板中，很多类型为trapper类型，但是，es_stats_zabbix.userparm定义的类型为agent，所以，需要把模板中得trapper类型修改为agent类型之后，才能正常显示数据。

故障排除
提供的ini文件的默认将日志输出到/dev/null。可以开启日志，使故障诊断更容易。
debug = True vs. loglevel = DEBUG，开启日志文件。

监控日志形如：
2015-10-09 14:47:19,210 DEBUG elasticsearch log_request_success:66 < {"cluster_name":"myelasticsearch","status":"green","timed_out":false,"number_of_nodes":6,"number_of_data_nodes":4,"active_primary_shards":174,"active_shards":348,"relocating_shards":0,"initializing_shards":0,"unassigned_shards":0,"number_of_pending_tasks":0,"number_of_in_flight_fetch":0}
