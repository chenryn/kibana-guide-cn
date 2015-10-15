##基础介绍
Graphite是一个Python写的，采用django框架的画图工具，Graphite将数据以图形的方式展现出来。它主要做两件事：<br />
存储时间序列数据<br />
根据需要呈现数据的图形<br />

Graphite由三个软件组件组成：<br />
简单架构<br />
carbon是一个Twisted守护进程，组成一个Graphite安装的存储后端，监听时间序列数据；<br />
whisper是一个简单的数据库，用来存储时间序列数据，类似于RRD；<br />
graphite webapp是一个Django webapp；<br />

statsd:<br />
一个用于统计汇聚数据的（默认udp，看官方github，也有tcp，见https://github.com/etsy/statsd/blob/master/docs/server.md ）服务，可以将数据发送到graphite等；<br />
Graphite以树状结构存储监控数据，所以statsd的数据的key也一定得是 "namespace.sender.metric" 这样的形式。而在 outputs/statsd 插件中，就会以三个配置参数来拼接成这种形式。<br />
最基本的做法是：把statsd计数或延迟数据每隔几秒钟就会发出的聚集值到后台。例如总量，最大，最小，平均，标准偏差，等等。<br />

statsd metrics简单解释：<br />
count:对数字的计数，比如，每秒接收一个数字，一个计量周期内，所有数字的和，比如nginx的body_bytes_sent；<br />
increment：增量，一个计量周期内，某个数字接收了多少次，比如nginx的status状态码；<br />
timing：时间范围内，某种数字的最大值，最小值，平均值，比如nginx的响应时间request_time。<br />

##配置graphite和statsd
1.安装cairo和pycairo<br />
yum -y install cairo pycairo

2.pip安装方式
pip是python的一个组件，安装pip的方法可以参考pip安装和使用教程。<br />
yum install python-devel<br />
pip install django django-tagging carbon whisper graphite-web uwsgi

3.配置graphite
cd /opt/graphite/webapp/graphite<br />
cp local_settings.py.example local_settings.py<br />
python manage.py syncdb<br />
修改local_settings.py中的DATABASE为设置的db信息

4.启动cabon
cd /opt/graphite/conf/<br />
cp carbon.conf.example carbon.conf<br />
cp storage-schemas.conf.example storage-schemas.conf<br />
cd /opt/graphite/<br />
./bin/carbon-cache.py start<br />

Starting carbon-cache (instance a)

5.安装statsd
cd /opt/<br />
git clone git://github.com/etsy/statsd.git<br />
cd /opt/statsd<br />
cp exampleConfig.js Config.js<br />
修改Config.js中的配置
```
{
  graphitePort: 2003
, graphiteHost: "10.10.10.124"
, port: 8125
, backends: [ "./backends/graphite" ]
}
```

6.nginx和uwsgi配置
cd /opt/graphite/webapp/graphite
新建配置文件：
more wsgi_graphite.xml
```
<uwsgi>
    <socket>0.0.0.0:8630</socket>
    <workers>2</workers>
    <processes>2</processes>
    <listen>100</listen>
    <chdir>/opt/graphite/webapp/graphite</chdir>
    <pythonpath>..</pythonpath>
    <module>wsgi</module>
    <pidfile>graphite.pid</pidfile>
    <master>true</master>
    <enable-threads>true</enable-threads>
    <logdate>true</logdate>
    <daemonize>/var/log/uwsgi_graphite.log</daemonize>
</uwsgi>
```

cp /opt/graphite/conf/graphite.wsgi /opt/graphite/webapp/graphite/wsgi.py

nginx的uwsgi配置：
cat /usr/local/nginx/conf/conf.d/graphite.conf
```
server {
    listen 8081;
    server_name graphite;

    access_log /opt/graphite/storage/log/webapp/access.log ;
    error_log /opt/graphite/storage/log/webapp/error.log ;

    location / {
        uwsgi_pass 0.0.0.0:8630;
        include uwsgi_params;
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
    }
}
```
7.启动uwsgi和nginx
uwsgi -x /opt/graphite/webapp/graphite/wsgi_graphite.xml
systemctl nginx reload

8.数据测试：
可以做个小测试：echo "test.logstash.num:100|c" | nc -w 1 -u $IP $port 如果安装配置是正常的，在graphite的左侧metrics->stats->test->logstash->num的表，statsd里面多了numStats等数据。

9.logstash output到statsd配置详解
```
output {
	if [type] == "nginxapiaccess" {  #类型判断，避免重复数据
	statsd {
	host => "10.10.10.124" #statsd 主机
	port => 8125 #statsd 端口，默认为8125
	sender => "%{host}" #默认为主机名，数据发送者的名称，点.将被下划线_替代；
	namespace => "logstash" #数据在statsd的名字空间，默认为logstash
	count => [ "nx_body_bytes_sent", "%{body_bytes_sent}" ] #对body_bytes_sent流量统计
	increment => [ "nx_status.%{status}" ] #状态码进行增量计数
	timing => [ "nx_request_time", "%{request_time}" ] #请求时间统计分析
	}
}
}
```

其他解释：
* decrement：递减运算，没用过，用法同increment;
* gauge：代表一个度量的即时值,用法同count，A gauge metric. metric_name => gauge as hash；
* set：statsd 支持在两个刷新间隔的独立事件的计数，set存储所有发送的events，A set metric. metric_name => "string" to append as hash。

##参考文档：
* [statsd github](https://github.com/etsy/statsd/tree/master/docs)<br />
* [logstash 官方文档](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-statsd.html#plugins-outputs-statsd-decrement)<br />
