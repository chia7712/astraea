Astraea Partitioner 中文文件
===
Astraea Partitioner 是強大且高效率的 Kafka Partitioner 實作，提供豐富且彈性的叢集負載選項，從寫入端動態維持使用者定義後的負載平衡.

### 通過gradle引入Astraea
在build.gradle中添加以下內容
```gradle
repositories {
    maven {
        url = "https://maven.pkg.github.com/opensource4you/astraea"
        credentials {
            username = System.getenv("GITHUB_ACTOR")
            password = System.getenv("GITHUB_TOKEN")
        }
    }
    
    dependencies {
        implementation 'org.astraea:astraea-common:0.1.0-SNAPSHOT'
    }
}
```

### Astraea Partitioner 使用

為自己的 Kafka Producer 配置使用 Astraea Partitioner

```java
var props = new Properties();

/* ... Some Kafka properties to put into ... */

/* Set Astraea Partitioner as partitioner. For example, Strict Cost Partitioner*/
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, "org.astraea.common.partitioner.StrictCostPartitioner");

var producer = new KafkaProducer<String, String>(props);
```

為自己的 Astraea Producer 配置使用 Astraea Partitioner

```java
var producer =
    Producer.builder()
        .bootstrapServers("localhost:9092")
        /* ... Some producer configs to put into ... */
        .partitionClassName("org.astraea.common.partitioner.StrictCostPartitioner")
        .build();
```



### Astraea Partitioner "Interdependent Message" 功能

應用或許會需要把 "某幾筆" 訊息 (record) 發送到同一個 partition (可能是想要確保那 "某幾筆" 訊息的讀取相對順序) ， Astraea Partitioner 提供了"Interdependent Message" 的功能，對於使用 Astraea Partitioner 的 producer 會保證 "指定時間" 內 **所有同 topic 的訊息 (record)** 都發到 **同一個 partition** 上。使用範例

使用 Kafka Producer

```java
var props = new Properties();

/* ... Some Kafka properties to put into ... */
/* ... Some Astraea Partitioner properties to put into ... */

var producer = new KafkaProducer<String, String>(props);

/* 開始使用 Interdependent Message 功能，以下的 record 都會被發送到同一個 partition 上 */
Partitioner.beginInterdependent(producer);
producer.send(new ProducerRecord<>("topicName", "These"));
producer.send(new ProducerRecord<>("topicName", "should"));
producer.send(new ProducerRecord<>("topicName", "be"));
producer.send(new ProducerRecord<>("topicName", "in"));
producer.send(new ProducerRecord<>("topicName", "the"));
producer.send(new ProducerRecord<>("topicName", "same"));
producer.send(new ProducerRecord<>("topicName", "partition."));
/* 結束 Interdependent Message 功能 */
Partitioner.endInterdependent(producer);
```

使用 Astraea Producer

```java
var producer =
    Producer.builder()
        .bootstrapServers("localhost:9092")
        /* ... Some producer configs to put into ... */
        .partitionClassName("org.astraea.common.partitioner.StrictCostPartitioner")
        .build();

/* 開始使用 Interdependent Message 功能，以下的 record 都會被發送到同一個 partition 上 */
Partitioner.beginInterdependent(producer);
producer.sender().topic("topicName").value("These".getBytes()).run();
producer.sender().topic("topicName").value("should".getBytes()).run();
producer.sender().topic("topicName").value("be".getBytes()).run();
producer.sender().topic("topicName").value("in".getBytes()).run();
producer.sender().topic("topicName").value("the".getBytes()).run();
producer.sender().topic("topicName").value("same".getBytes()).run();
producer.sender().topic("topicName").value("partition".getBytes()).run();
/* 結束 Interdependent Message 功能 */
Partitioner.endInterdependent(producer);
```

**注意： Interdependent 內，不可發送不同 topic 的 record，因為 topic 間的 partition 數量不一定相同。**

### Astraea Partitioner 實作

1. [Smooth Partitioner](smooth_partitioner.md):  通過收集多metrics數據，結合熵權法與AHP進行節點狀況評估。再根據評估結果，使用 smooth weight round-robin 進行資料的調配。
1. [Strict Cost Partitioner](./strict_cost_partitioner.md): 收集使用者自定義的效能指標，使用效能指標為節點打分。再根據加權分數，使用 smooth weight round-robin 進行資料調配。

### Astraea Partitioner 實驗

experiments 資料夾中收錄不同版本的實驗紀錄，主要使用 [performance tool](../performance_benchmark.md) 測試並紀錄數據。

* [2022 Aug28](experiments/StrictCostDispatcher_1.md), 測試 [Strict Cost Partitioner](./strict_cost_partitioner.md) (Astraea revision: [75bcc3faa39864d5ec5f5ed530346184e79fc0c9](https://github.com/opensource4you/astraea/tree/75bcc3faa39864d5ec5f5ed530346184e79fc0c9))
* [2022 Oct1](experiments/StrictCostDispatcher_2.md), 測試資源充足下 [interdependent message](#astraea-partitioner--interdependent-message--功能) 對 [Strict Cost Partitioner](./strict_cost_partitioner.md) 的影響 (Astraea revision: [08b4e32f31091a3de69775db5442eb631deca550](https://github.com/opensource4you/astraea/tree/08b4e32f31091a3de69775db5442eb631deca550))
* [2022 Oct1](experiments/StrictCostDispatcher_3.md), 測試高壓下 [interdependent message](#astraea-partitioner--interdependent-message--功能) 對 [Strict Cost Partitioner](./strict_cost_partitioner.md) 的影響 (Astraea revision: [08b4e32f31091a3de69775db5442eb631deca550](https://github.com/opensource4you/astraea/tree/08b4e32f31091a3de69775db5442eb631deca550))
* [2022 Dec17](experiments/StrictCostDispatcher_4.md), 測試多台 producer 同時發送訊息時，[Strict Cost Partitioner](./strict_cost_partitioner.md) 的表現，並比較 [Strict Cost Partitioner](./strict_cost_partitioner.md) 、 Kafka Built-In Partitioner 和 Default Partitioner 的吞吐量、發送延遲。(Astraea revision: [5003f424e9cef4d27bbc56621d6e42f19a687a61](https://github.com/opensource4you/astraea/tree/5003f424e9cef4d27bbc56621d6e42f19a687a61)) 