
本文主要介绍Elasticsearch/Logstash/Kibana/Filebeat在k8s上的分布式部署。

部署使用的docker镜像可以从[此处](https://github.com/leizhu/elastic-stack-docker)自行build

# Elasticsearch

Elasticsearch各节点在k8s上的通信可以通过单播(unicast)发现机制，这里推荐[elasticsearch-cloud-kubernetes](https://github.com/fabric8io/elasticsearch-cloud-kubernetes)插件，使用方式十分简单，只需要在elasticsearch.yml中配置如下部分即可
```
cloud:
  kubernetes:
    service: ${SERVICE}
    namespace: ${NAMESPACE}
discovery:
  type: kubernetes
```

## Elasticsearch不同功能节点的部署方案

Elasticsearch的节点类型分为client/master/data，为了保证ES集群的HA，我们采取如下部署方案：
1. 所有节点以k8s pod的形式进行部署，一个节点对应一个pod，通过k8s ReplicationController来控制pod的自愈
1. client部署2个节点，采用k8s service来控制负载均衡
1. master部署3个节点来保证HA，注意discovery.zen.minimum_master_nodes要设置为2以防止split brain
1. data部署3个节点，需要进行数据持久化以保证data节点的pod被重建时不会丢失数据

[elasticsearch.yml](https://github.com/leizhu/elastic-stack-docker/blob/master/elasticsearch/config/elasticsearch.yml)的部分配置如下，用户可以加以修改再进行镜像编译。
```
cluster:
  name: ${CLUSTER_NAME}
node:
  name: ${HOSTNAME}
  master: ${NODE_MASTER}
  data: ${NODE_DATA}
  ingest: ${NODE_INGEST}
network.host: ${NETWORK_HOST}
cloud:
  kubernetes:
    service: ${DISCOVERY_SERVICE}
    namespace: ${NAMESPACE}
discovery:
    type: kubernetes
    zen.hosts_provider: kubernetes
    zen.minimum_master_nodes: ${MINIMUM_MASTER_NODES}
    zen.master_election.ignore_non_master_pings: true
bootstrap:
    memory_lock: true
gateway.recover_after_time: 5m
gateway.recover_after_data_nodes: 2
cluster.routing.allocation.same_shard.host: true
```

elasticsearch.yml的配置支持系统环境变量，可以在k8s部署时再传入相应参数，这样做十分灵活，例如可以指定${NODE_MASTER}${NODE_DATA}来指定pod所启动的es节点的类型

## k8s部署

部署文件可参看[这里](https://github.com/leizhu/elastic-stack-docker/tree/master/k8s/elasticsearch)
，执行以下命令即可
```
kubectl create -f es-discovery-svc.yaml
kubectl create -f es-svc.yaml
kubectl create -f es-master.yaml
kubectl create -f es-data.yaml
kubectl create -f es-client.yaml
```
注意，因为es的data节点需要数据持久化，这里采用了k8s的hostpath volume，需要在每个k8s minion节点上创建数据目录(/data/es-data /data/es-backup)

部署完毕后，执行以下命令才看pod运行状态
```
root@QACloud-Master1:/cloud/kubernetes-elk# kubectl get pods -o wide | grep es-
es-client-1-s7m1l                    1/1       Running   0          30d       172.16.41.6    172.22.78.73
es-client-2-6kjh5                    1/1       Running   0          30d       172.16.46.6    172.22.78.72
es-data-1-b0mj0                      1/1       Running   0          30d       172.16.45.3    172.22.78.71
es-data-2-2w4d8                      1/1       Running   0          30d       172.16.46.5    172.22.78.72
es-data-3-6lh73                      1/1       Running   0          30d       172.16.41.5    172.22.78.73
es-master-1-fjrnw                    1/1       Running   0          30d       172.16.41.4    172.22.78.73
es-master-2-hcmv9                    1/1       Running   0          30d       172.16.45.2    172.22.78.71
es-master-3-66z4f                    1/1       Running   0          30d       172.16.46.4    172.22.78.72

root@QACloud-Master1:/cloud/kubernetes-elk# curl http://172.16.41.6:9200/_cluster/health?pretty
{
  "cluster_name" : "es-sg",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 8,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 34,
  "active_shards" : 56,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

# Logstash

部署文件可参看[这里](https://github.com/leizhu/elastic-stack-docker/tree/master/k8s/logstash)

该例子从kafka中读取日志消息，进行相应的处理后送入ES

# Kibana

部署文件可参看[这里](https://github.com/leizhu/elastic-stack-docker/tree/master/k8s/kibana)

注意：
1. ana-controller.yaml中的ELASTICSEARCH_URL需要指定为es-svc.yaml定义的service名称
1. kibana svc需要指定为NodePort类型，这样会暴露一个NodePort给外部访问者，在浏览器访问"http://<K8S_NODE_IP>:PORT"可进入kibana UI

# Filebeat

部署文件可参看[这里](https://github.com/leizhu/elastic-stack-docker/tree/master/k8s/filebeat)

Filebeat采用k8s DaemonSet部署，这样每个k8s minion节点都会启动一个filebeat pod，这些pods用于收集k8s上运行的容器的log，然后送入kafka


# Tips
1. Elasticsearch集群比较耗费资源(特别是内存)，如果k8s集群资源充沛，可以通过部署文件中的ES_JAVA_OPTS和resources.request/resources.limits等参数进行资源的合理配置
1. ES data节点的数据持久化可以采用k8s的多种类型的volume，但是我们经过测试，如果volume对应的底层存储介质只是普通硬盘的话（非ssd），建议采用hostpath volume，相当于数据持久化到本地disk，省去了网络开销。当然，如果不差钱的话，尽量用ssd硬盘
