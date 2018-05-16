## 机器分配

Ubuntu 14.04
<div> <table border="0">
<tr> <th>Server</th><th>Services</th><th>Description</th> </tr>
<tr> <td> 172.16.10.222 </td><td> zk-kafka </td><td> node-1 </td> </tr>
<tr> <td> 172.16.10.223 </td><td> zk-kafka </td><td> node-2 </td> </tr>
<tr> <td> 172.16.10.224 </td><td> zk-kafka </td><td> node-3 </td> </tr>
<tr> <td> 172.16.10.226 </td><td> logstash-elacticsearch </td><td> logrus->logstash->kafka </td> </tr>
<tr> <td> 172.16.10.228 </td><td> logstash-elasticsearch-kibana </td><td> kafka->logstash->es->kibana </td> </tr> 
</table></div>

## 数据流向

```text
logrus >> docker >> logspout >> logstash >> kafka >> logstash >> elasticsearch >> kibana 
```


## 前期准备

# 安装Java 环境

```text
Download jdk-8u111-linux-x64.tar.gz
tar zxf jdk-8u111-linux-x64.tar.gz
sudo mv jdk1.8.0_111 /usr/local/jdk
```
vim ~/.profile
```text
#Java
export JAVA_HOME=/usr/local/jdk
export JRE_HOME=$JAVA_HOME/jre
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib:$JRE_HOME/lib
```

#Reload the env
```text
source ~/.profile
```

修改 ulimit, elasticsearch 启动需求
```text
# elasticsearch can't run with root;
# max file descriptors for elasticsearch process must increase to at least [65536]
# max virtual memory areas vm.max_map_count must increase to at least [262144]
```

sudo vim /etc/sysctl.conf
```text
vm.max_map_count=262144
```

sudo vim /etc/security/limits.conf
```text
* hard nofile 65536
* soft nofile 65536
```
sudo reboot

# 修改ruby安装源，安装logstash plugin 需要

```text
$ gem source
//删除默认源
$ gem source -r https://rubygems.org/
//添加淘宝源
$ gem source -a https://ruby.taobao.org/
```

## 安装目录 

$HOME/elk


# zookeeper安装
[zookeeper](http://zookeeper.apache.org/doc/current/zookeeperStarted.html)

conf目录下增加文件zoo.cfg
```text
tickTime=2000
dataDir=/home/ubuntu/elk/data/zookeeper
clientPort=2181
initLimit=5
syncLimit=2
server.1=$host1:2888:3888
server.2=$host2:2888:3888
server.3=$host3:2888:3888
```

在配置的dataDir目录下创建一个myid文件，里面写入一个0-255之间的一个数字，每个zk上这个文件的数字要是不一样的，这些数字应该是从1开始，依次写每个服务器。

启动服务
```text
./bin/zkServer.sh start
```

查看状态 
```text
./bin/zkServer.sh status
```

# kafka
[kafka](http://kafka.apache.org/documentation)
```text
# The id of the broker. This must be set to a unique integer for each broker.
broker.id=1

# A comma seperated list of directories under which to store log files
log.dirs=/var/lib/kafka-logs

# The address the socket server listens on. It will get the value returned from 
# java.net.InetAddress.getCanonicalHostName() if not configured.
#   FORMAT:
#     listeners = security_protocol://host_name:port
#   EXAMPLE:
#     listeners = PLAINTEXT://your.host.name:9092
listeners=PLAINTEXT://$hostn:9092
zookeeper.connect=$host1:2181,$host2:2181,$host3:2181
```

启动服务
```text
nohup bin/kafka-server-start.sh config/server.properties &
```

Create a topic
```text
bin/kafka-topics.sh --create --zookeeper $host1:2181,$host2:2181,$host3:2181 --replication-factor 3 --partitions 1 --topic game-log
```

Okay but now that we have a cluster how can we know which broker is doing what? To see that run the "describe topics" command:
```text
bin/kafka-topics.sh --describe --zookeeper $host1:2181,$host2:2181,$host3:2181 --topic game-log
```

Let's publish a few messages to our new topic:
```text
bin/kafka-console-producer.sh --broker-list $host1:9092,$host2:9092,$host3:9092 --topic game-log
```

Now let's consume these messages:
```text
bin/kafka-console-consumer.sh --zookeeper $host1:2181,$host2:2181,$host3:2181  --from-beginning --topic game-log
```

# elacticsearch
启动服务
```text
bin/elasticsearch -d
```

修改配置
config/elasticsearch.yml
```text
cluster.name: es-cluster
node.name: node-1
path.data: /home/ubuntu/elk/data/elasticsearch/data
path.logs: /home/ubuntu/elk/data/elasticsearch/logs
#bootstrap.memory_lock: true
network.host: 172.16.10.228
http.port: 9200
discovery.zen.ping.unicast.hosts: [$host1, $host2, $host3]
discovery.zen.minimum_master_nodes: 2 
```

# logstash

kafka >> logstash >> elasticsearch

```text
input {
    kafka {
        bootstrap_servers => '$host1:9092,$host2:9092,$host3:9092'
        group_id => 'test_consumer-group'
        topics => ['game-log']
        consumer_threads => 5   
        decorate_events => true 
        auto_offset_reset="earliest" # automatically reset the offset to the earliest offset 
        codec => "plain"
    } 
}

filter {
    json {
        source => "message"
    }   
    kv {
        field_split => " " 
    }   

    #This does not ship with Logstash by default, but it is easy to install by running 
    #		bin/logstash-plugin install logstash-filter-de_dot
    #de_dot {
    #}  

    mutate {
        rename => { 
            "msg" => "message" 
            "time" => "logtime"
        }   
        
        #still not ok
        #gsub =>[ "appname", ".*", "111" ]
    }   
}

filter {
    geoip {
        source => "client_ip"
        target => "geoip"
        #database => "/usr/local/logstash/etc/GeoLiteCity.dat"
        add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
        add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
    }  
    mutate {
        convert => ["[geoip][coordinates]", "float"]
    }  
}

output {
    elasticsearch {
        hosts => [$host1, $host2, $host3]
        index => "logstash-log-%{+YYYY.MM.dd}"
        #workers => 5
        codec => "json"
    }

    #stdout {
    #     codec => rubydebug 
    #}   
}
```

logspout >> logstash >> kafka

```text
input {
    udp {
        port  => 5000
        codec => json
    }   
    tcp {
        port  => 5000
        codec => json
    }   
}


filter {
    mutate {
        add_field => { "app" => "%{[docker][name]}" }
    }
    mutate {
        gsub => [
            "app", "/", ""
        ]
    }
}


output {
    kafka {
        bootstrap_servers => '$host1:9092,$host2:9092,$host3:9092'
        topic_id => 'game-log'
#workers => 5
        codec => "plain"
    }   

    #stdout {
    #     codec => rubydebug 
    #}   
}
```

To enable automatic config reloading, start Logstash with the --config.reload.automatic (or -r) command-line option specified. For example:
```text 
bin/logstash –f logstash.config --config.reload.automatic
```

# kibana

config/kibana.yml
```text
server.host: "172.16.10.228"
elasticsearch.url: "http://172.16.10.228:9200"
```

启动服务
```text
nohup bin/kibana &
```
访问 
```text
http://172.16.10.228:5601
```

# logspout
```text
docker run -d --name="logspout" \
    --volume=/var/run/docker.sock:/var/run/docker.sock \
    -e ROUTE_URIS=logstash+tcp://172.16.10.226:5000 \
    bekt/logspout-logstash
```

# logrus
We can add fields for indexing,such as user's name ...
	
```go
package main

import (
  log "github.com/Sirupsen/logrus"
)

func main() {
  log.WithFields(log.Fields{
    "animal": "walrus",
  }).Info("A walrus appears")
}
```

# To use X-Pack for Elasticsearch and Kibana
[X-Pack plugin install](https://www.elastic.co/guide/en/x-pack/current/installing-xpack.html)

Run install from ES_HOME on each node in your cluster:
```text
bin/elasticsearch-plugin install x-pack
```
And restart
```text
jps & kill -2 pid
bin/elasticsearch -d
```

Install X-Pack into Kibana by running bin/kibana-plugin in your Kibana installation directory
```text
bin/kibana-plugin install x-pack
```
And restart
```text
ps aux and you will find some pid with (./bin/../node/bin/node --no-warnings ./bin/../src/cli)
kill -2 pid
nohup ./bin/kibana &
```
[logstash authentication](https://www.elastic.co/guide/en/x-pack/current/logstash.html)

Change logstash conf & it will reload if it started with -r
```text
output {
    elasticsearch {
    ...
    user => logstash_internal
    password => changeme
}
```
