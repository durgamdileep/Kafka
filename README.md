# 🧩 Topic

A **Topic** is a `logical grouping` of messages/events.

- 📦 It `does not store data directly` — it `just organizes messages under a name`. 
- 🔄 Topics allow **multiple producers** to send and **multiple consumers** to read messages using the topic name. 
- ⚠️ A topic should contain `only one kind of logically related message` — messages from different domains (like orders, payments, inventory) should be `placed in separate topics`.

### ✅ Example:

🛒 In an e-commerce system:

- Use `order_topic` for all order-related messages (like:
    - 📬 order placed,
    - 🚚 order shipped)

❌ Do not send payment or inventory messages into `order_topic`.  
✅ Those should go into:
- `payment_topic` 💳 or
- `inventory_topic` 📦.

---

# 📦 Partition

A Partition is the `actual storage unit` of Kafka where messages are written.

- 📂 Each `Topic` is divided into `one or more Partitions`. 
- 1️⃣ By default, a topic has `one partition`. 
- 📝 Partitions are `append-only logs` — messages are written in sequence.
- ⏳ `Ordering` is guaranteed only within a single partition, `not across partitions`. 
- #️⃣ Each partition is identified by a `partition number (0, 1, 2, …)` under a topic.
- ⚙️ Partitions allow `parallelism` — multiple partitions can be read/written in parallel by different consumers/producers. 🔀
   - Broker-Level Parallelism
     - Kafka brokers store partitions on disk. 
     - `Multiple partitions` = `more disk I/O threads` `working in parallel` (especially with multiple disks or SSDs). 
     - Also, partitions can be spread across multiple brokers (horizontal scaling).
- 🔁 `Each partition` can have `replicas` for `fault tolerance`, though `only one replica` is the `leader` at a time. 🛡️
- 🔢 `Each message` within a partition has a `unique sequential ID` called an `offset`, which is important for `consumer tracking`. 📍


## 🗂️ Kafka Partition is a Log, Not a Queue

- 📥 In a **Queue**, once a message is read, it is removed. You cannot re-read it.

- 📜 In Kafka, a **Partition** acts like a log:
  - 📝 Messages are stored in sequential order.
  - 🕒 Even after reading, messages are not deleted immediately.
  - 🔄 Messages can be read multiple times based on log retention settings.

✅ That’s why **Kafka Partition is a Log, not a Queue**.

---

# 📬 What Does a Kafka Message Contain?

- 🔑 **key**: Used for partitioning logic. Not necessarily unique.
- 📦 **value**: Actual data (can be JSON, Avro, Protobuf, etc.).
- ⏰ **timestamp**: When the message was written to the partition.
- 🗂️ **partition**: Partition number where the message is stored.
- 🔢 **offset**: Position of the message in the partition. Unique within a partition.
- 🏷️ **headers**: Optional key-value pairs for metadata (e.g., tracing, filtering).
- Example Message:
``` 
    {
      "key": "user-123",
      "value": {
        "event": "user_signup",
        "userId": "123",
        "timestamp": "2025-10-12T15:45:00Z",
        "details": {
          "email": "user@example.com",
          "plan": "premium"
        }
      },
      "timestamp": 1697115900000,
      "partition": 3,
      "offset": 4567,
      "headers": {
        "traceId": "abc123xyz",
        "source": "web_app"
      }
    }
    // key: "user-123" — used for partitioning, e.g., all events for this user go to the same partition.
    // value: The actual payload of the message.
    // timestamp: Unix epoch milliseconds when the message was written.
    // partition: Partition number (3 in this case).
    // offset: Unique offset within the partition.
    // headers: Metadata like tracing info or source system.

```
---

## 🔑 Message/Event Without Key (Case 1)

- ❌ If we send a message **without a key**:
- 🔄 Kafka uses `round-robin` to distribute messages evenly across partitions.

- 📊 **Example**:
  - We have `3 partitions` for a topic.
  - Sending `3 messages` for `sensor_id: 42` without a key → each message may go to a **different partition**.

- ⚠️ **Problem**:
  - Messages from the **same sensor** are stored in **different partitions**.
  - Kafka guarantees message **ordering only within a single partition**, but **not across partitions**.

- ⚡ **Consequence**:
  - We **lose the order** of messages when reading them, because messages from the **same source are scattered** across partitions.


## 🔑 Message/Event With Key (Case 2)

- ✅ If we send a message **with a key**:

- 🧮 Kafka hashes the key and assigns it to a partition using:  
  `partition = hash(key) % number_of_partitions`

- 🎯 All messages with the **same key** go to the **same partition** then Order of insertion and reading the message will be same.

  - 📊 Example:  
    10 messages with various sensor IDs as keys:
    ```
    Key: 42 → { sensor_id: 42, ... }
    Key: 83 → { sensor_id: 83, ... }
    Key: 42 → { sensor_id: 42, ... }
    Key: 37 → { sensor_id: 37, ... }
    Key: 55 → { sensor_id: 55, ... }
    Key: 42 → { sensor_id: 42, ... }
    Key: 77 → { sensor_id: 77, ... }
    Key: 100 → { sensor_id: 100, ... }
    Key: 88 → { sensor_id: 88, ... }
    Key: 99 → { sensor_id: 99, ... }
    Key: 100 → { sensor_id: 100, ... }
    ```

- 🔢 After hashing:
    ```
    Partition 1 → keys: 100, 99, 100, 88
    Partition 2 → keys: 77, 55
    Partition 3 → keys: 42, 42, 83, 42
    ```

- 🔒 Messages with the **same key** (e.g., 42, 100) go to the **same partition**, so:
- 🧭 **Order is preserved** for messages with the same key.
- ⚙️ Kafka guarantees **message order within a partition**.

---

# 🖥️ Apache Kafka Broker

A broker in Apache Kafka is `a server` that is part of a Kafka cluster. It is responsible for:

- 🚚 **Receiving messages** from producers,
- 🗄️ **Storing** those `messages on disk`,
- 📦 **Serving messages**  to consumers when they request them.

Each broker handles part of the data (topics/partitions), and Kafka can have `many brokers working together in a cluster` to provide `scalability` and `fault tolerance`.

---

# 🧩 Partition Distribution

📦 Suppose you create a topic `order_topic` with **5 partitions**.

Kafka assigns each partition to a broker based on:

- 🖥️ **Available brokers** in the cluster
- 📑 **Replication factor** (how many copies of each partition to create)
- 📈 Kafka’s **partition assignment strategy**:
  - 🔁 Typically **round-robin**
  - 🧠 Or **rack-aware balancing** if enabled


- `Partition and replica distribution` decisions are always made by the `Kafka controller` component (broker).
- In `KRaft` mode, the `Kafka controller` is part of the `KRaft controller`.
   - Kafka Controller = component responsible for partition/replica management. 
   - KRaft = new metadata mode where this controller is built into Kafka, replacing ZooKeeper.


### 🧍 If there is only **1 broker**:

   - ➡️ All partitions of the topic go to that **one broker**.

### 👥 If there are **multiple brokers**:

   - 🔄 Kafka spreads (distributes) partitions across brokers.
   - 1️⃣ First, it puts **1 partition on each broker**.
   - 🔁 If partitions are still left, it goes back to the **first broker** and continues.
   - 🔚 This goes on until **all partitions are assigned**.


### 📊 If `number of brokers > number of partitions`:
    
   - 🚫 Only **some brokers** will get partitions.
   - 🙅 Other brokers will **not have any partition** of that topic.

### ⚖️ Partitions are **not always evenly divided**:

   - 📉 All brokers may **not get the same number** of partitions.
   - 📈 Some brokers may have **more**, others **less**, depending on total partition count.

### 🛠️ Topic Creation:

   - 🧠 When you **create a topic**, Kafka normally decides how to spread partitions to brokers **automatically**.
   - 👤 You (the user) can **manually assign** which partition goes to which broker.
   - 👉 If you do this **manual assignment**, then **Kafka does not use its normal logic** (the round-robin distribution).

## 🔄 Partition Assignment Strategy:

- ✅ **If you let Kafka decide** → it uses the **round-robin** strategy.
- ✳️ **If you assign manually** → your **custom setup** is used.

## 📝 Partition  Placement: Who Decides & Spring Boot Ease

| Case                   | Who Decides Partition Placement?           | Easy to Do in Spring Boot?              |
|------------------------|---------------------------------------------|----------------------------------------|
| ⚙️ Auto Assignment       | Kafka (default behavior)                     | ✅ Yes                                 |
| 🛠️ Manual Assignment (Advanced) | You (custom broker/partition mapping)         | ⚠️ Needs Kafka AdminClient              |


---

# 🧭 Kafka Partition Leadership & Replication

## 👑 Each Partition Has:

- 🟢 **One Leader**
  - 🧠 Responsible for `handling all client requests` (both reads and writes)
  - 📤 Producers send data to the leader
  - 📥 Consumers fetch data from the leader

- 🔄 **Zero or More Followers**
  - 📚 Followers `replicate data` from the leader
  - 🚫 `Do not serve client requests`
  - 🔁 Keep in sync with the leader (used for failover)
  - 🔄 Only replicate data from the leader 
  - 🧍 Do not handle any client operations
       - ❌ No writes from producers 
       - ❌ No reads from consumers
  - 🧯 Serve as backup in case the leader fails 
  - 🪄 Can be promoted to leader if the current leader goes down

## 🚨 Replicas Read/Write Behavior

### ⚙️ Default Behavior
- ❌ By default, `replicas do not handle read/write operations` from clients (producers and consumers).
- 🔑 The leader handles all those requests.

### 🚀 Performance Optimization
- ⚙️ Some systems allow `configuring replicas` to `serve read-only` operations to `clients`.
- 💡 This `reduces load on the leader` and `improves latency`.
- 🖥️ So replicas can be used for **read operations only**.

### Summary:
| Mode                  | Read from Replicas      | Write from Replicas    |
|-----------------------|------------------------|-----------------------|
| **Default**            | ❌ No                  | ❌ No                 |
| **With Configuration** | ✅ Yes                 | ❌ No                 |


---

## 🔄 Kafka Replication Factor 

When a topic is created with a **replication factor > 1**  
Kafka assigns additional replicas for **high availability**.

Kafka tries to:

- ❌ **Avoid placing the leader and its replicas on the same broker**
- ⚖️ **Balance replicas evenly across all brokers**



### 1️⃣ Replication Factor = 1

- 🟢 Only **1 copy** of each partition (the **leader**)
- ⚠️ **No backup**
- 💥 If that broker goes down, **data is lost**

### 2️⃣ Replication Factor > 1

Each partition will have:

- 👑 **1 Leader**
- 🔁 **(replication factor - 1) Replicas** (backups on other brokers)

---

## 🤔 How Does Kafka Distribute Replicas When Replication Factor > 1?

- 🤖 Kafka **automatically** gets the list of brokers **except the leader** and distributes the replicas among them.
- ⚖️ Tries to **distribute replicas evenly** across brokers, similar to how partitions are distributed.
- 🚫 Tries to **avoid placing a replica on the same broker as the leader**.

---

## ⚠️ Notes

- ❌ Kafka does **not guarantee perfectly equal** replica distribution.
- 🛠️ You can **customize** replica placement using a **manual `ReplicaAssignment`** if needed.



---

## 🧪 Example:

🎯 Topic: `order_topic`  
📂 Partitions: 3  
🔁 Replication Factor: 2  
🖥️ Brokers: 3

| Partition | Leader Broker | Replica Brokers |
|-----------|----------------|------------------|
| P0        | Broker 1       | Broker 2         |
| P1        | Broker 2       | Broker 3         |
| P2        | Broker 3       | Broker 1         |

✅ Clients talk only to leaders.  
🔁 Followers passively sync and take over if a leader fails.

---

## 🚨 Important Notes:

- 🛑 If the **leader fails**, Kafka controller promotes a **follower to leader**.
- ✅ Kafka ensures that at least one replica is **in-sync (ISR)** to maintain consistency.

---

## ⚠️ Important Constraints

- 🧠 `Replication Factor` ≤ `Number of Brokers` 
   - Kafka cannot place more replicas than the number of brokers.
- 🚫 → `No broker` can `host multiple replicas` of the `same partition`. 
   - Kafka `does not place more than one replica` of the `same partition` on the `same broker`.

### 📌 Example

📦 **Topic**: `order_topic`  
🧩 **Partitions**: `3`  
🖥️ **Brokers**: `3`  
📑 **Replication Factor**: `5`

❌ **Invalid Scenario**: RF = 5, Brokers = 3

⚠️ Kafka throws an error like:

```bash

InvalidReplicationFactorException: Replication factor: 5 larger than available brokers: 3
```

---

# 🤔 What is a Kafka Consumer?

- 📩 A consumer `reads messages from Kafka topics`.
- ⬇️ It pulls data (Kafka doesn’t push). 
- 👥⚖️ `Consumers are grouped` into `consumer groups` to `share the load`.

---

### 🧪 Example Without Consumer Group

📌 Let's say you have a Kafka topic called orders, and it contains these 4 messages:  
📦 Message 1: Order#101  
📦 Message 2: Order#102  
📦 Message 3: Order#103  
📦 Message 4: Order#104

👥 You have 2 consumers (Consumer A and Consumer B), but they are `not part of a consumer group` (i.e., each has a unique group ID or no group at all).

❓ What happens?

- 👤 Consumer A reads: Order#101, Order#102, Order#103, Order#104 
- 👤 Consumer B reads: Order#101, Order#102, Order#103, Order#104


🔁 So, `instead of splitting the work`, `both consumers process` the `same messages`.

---

### 🔍 Problem Without Consumer Group

- ❌ `Duplicate processing`: Every consumer gets the full data set. 
- ❌ `No load sharing`: No performance benefit from adding more consumers. 
- ❌ `Wasted resources`: You’re using multiple consumers without gaining parallelism.

---

### 🧠 What is a Consumer Group?

- 👥 A `consumer group` is a `way to split the work` of `reading messages` from a Kafka topic `among multiple consumers`.
- 🆔 All consumers in a group `share a group ID`. 
- ✅ Kafka makes sure that `each message is read by only one consumer` `within a group`.

👉 Think of it like a team of workers sharing a task.

```bash
   if Consumer C1 reads Message 1, then no other consumer in the same group will read Message 1.
```

---

### 👀 When to Use a Consumer Group?

- ⚡ Use a consumer group when you want to `process messages in parallel` to `make your system faster` and `more scalable`. 
- ✅ Use consumer groups to `distribute the load` across `multiple consumers`.

---

### 🧪 Example

📌 You have a Kafka topic called `order-topic`,  
📌 Assume the topic has 2 partitions, and 4 messages across them:

📁 Topic: order-topic  
📂 Partition 0: Order#101, Order#102  
📂 Partition 1: Order#103, Order#104

👤 Kafka assigns partitions like this:

- 👤 Consumer A reads: Order#101, Order#102 (from Partition 0)
- 👤 Consumer B reads: Order#103, Order#104 (from Partition 1)

✅ `Each message` is `read only once` within the `same group`, and the work is shared between consumers — thanks to `consumer assignment`.

---

### 🔄 What happens?

🧑‍💻 `Kafka Controller` `divides the partitions` to `consumers in a consumer group`
- 📊 Kafka controller (via the group coordinator) assigns partitions to consumers in the group — this is called `consumer assignment`.
   - ⚙️ By default, it uses built-in strategies like `Range`, `RoundRobin`, or `CooperativeSticky`. 
   - 🛠️ If needed, you can use a `custom consumer assignment strategy`.

---

## ⚙️ How Message Consumption Works

### ➡️ Within a single partition:
- ➡️ Messages are consumed one at a time in order (sequentially).

### ⚡ Across different partitions:
- ⚡ `A consumer` can `consume messages` in `parallel from multiple partitions assigned to it`.

📝 Topic : 📦 `order-topic`  
👥 Consumer Group : 👥 `G1`  
🗂️ Partition Assignment in Consumer Group
   - Consumer C1 → `Partitions P0` and `P1`
   - Consumer C2 → `Partition P2`

### 📌 Example with Consumer C1

Since C1 is assigned partitions P0 and P1, it can:

- ✅ Process `a message from P0` and `a message from P1` at the same time.
- ❌ But it `cannot process multiple messages from P0 simultaneously`, `nor multiple messages from P1 simultaneously`.

---

# 📦 Kafka Consumer Group Partition Assignment

## 🔄 Partition assigning to consumer group (Consumer Assignment)

### ➤ Rules:
In a consumer group:

- ✅ Each partition is assigned to only one consumer.
- 🔁 A consumer can be assigned multiple partitions.
- ⚖️ Kafka by default uses `Range`, `RoundRobin`, or `CooperativeSticky` strategies for distribution.
     - 📦 `RangeAssignor` (**default**): Contiguous blocks of partitions 
     - 🔄 `RoundRobinAssignor`: Evenly round-robins across consumers 
     - 🧷 `StickyAssignor`: Tries to avoid moving partitions during rebalances (preferred for stability)
     - 🤝 `CooperativeStickyAssignor`: Allows smooth rebalancing without stopping all consumers

- 🛠️ If needed, you can use a `custom consumer assignment strategy`.

---

### 💡 For Suppose:

In Kafka, consumer groups are used to distribute partitions among consumers. However:

- 🔢 If we have `no. of consumers in a group > no. of partitions`
   - then some consumers will not have any partition, they will be `idle consumers` (i.e., not assigned any partition).
- ➗ If we have `no. of consumers < no. of partitions`
   - then each consumer will be assigned `multiple partitions`.
- 🟰 If we have `no. of consumers == no. of partitions`
   - then each consumer will have `one partition`.

```bash

Topic: order-topic
Partitions: 3
Consumers: C1, C2
Group: group-A
   - C1 → P0, P2  
   - C2 → P1
```

### 📝 Summary:

- ✅ Kafka guarantees that `each partition is assigned to only one consumer in a group`.
- ⚠️ But it does `not guarantee` that `each consumer will get a partition`.

---

## 📚 Multiple Consumers Reading and Assigning Same Partition

- ❌ Not allowed within the same consumer group.
- ✅ Allowed if consumers are in different consumer groups.

```bash

Invalid
Group-A 
  - C1 → P0 
  - C2 → P0

Valid
Group-A → C1 → P0  
Group-B → C2 → P0
```

---

## 🔍 Can a Consumer Read from Multiple Partitions?

🧑‍💻 `A single consumer` can be assigned `multiple partitions` and pull messages from each.
```bash

Group: group-A
   - C1 → P0, P2  
   - C2 → P1
```
---

## ❓ Can the same messages be consumed by different consumer groups?

```bash

Topic: order-topic
Partitions: 3
Consumers: C1, C2
Group: 
  group-A(G1)
     - C1 → P0, P2  
     - C2 → P1
  group-B(G2)
     - C1 → P0,P2
     - C2 → P1
```

### 👥 In Consumer Group G1:

- 👤 Consumer C1 reads Partition P0 → Message M1
- 👤 Consumer C1 also reads Partition P2 → Message M2  
  ✅ C1 can consume messages from both partitions at the same time **within G1**.

### 👥 In Consumer Group G2:

- 👤 Consumer C1 (a different consumer instance in G2) reads Partition P0 → Message M1
- 👤 Consumer C1 reads Partition P2 → Message M2  
  ✅ Since G2 is a **different group**, it can consume the **same partitions and messages** independently from G1.

---



