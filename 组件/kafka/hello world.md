两种模式：kraft && zookeeper
**注意**：两种模式不能同时使用，需要在启动前选定一种

以zk为例：

## 1. 启动
```
# 启动zk
./bin/zookeeper-server-start.sh ./config/zookeeper.properties
   
# 启动kafka
./bin/kafka-server-start.sh ./config/server.properties
```


## 2. 生产

* 创建主题
```
./bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092
```
* 发送消息
```
./bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
```
## 3. 消费

```
./bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
```


## 99. server.properties
路径：
./config/server.properties


关键参数：
- `broker.id=0`：当前节点在集群中的唯一 ID，单机部署设为 0 即可。
- `listeners=PLAINTEXT://:9092`：Broker 监听的地址和端口，默认是 9092。
- `log.dirs=/tmp/kafka-logs`：Kafka 存储消息数据的物理路径，最好改成一个有足够空间的目录。
- **KRaft 模式**：在 `config/kraft/server.properties` 中同样配置 `log.dirs` 即可



