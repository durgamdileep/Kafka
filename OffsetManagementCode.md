# 🛠️ Step-by-Step Setup in Spring Boot

```bash

application.properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=my-group
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.enable-auto-commit=false

// This disables auto commit, which is necessary for manual offset management.


Create a Configuration Class

@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(consumerFactory);

        // 👇 Manual ack mode is required for manual commits
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);

        return factory;
    }
}

Use @KafkaListener with Acknowledgment

@KafkaListener(topics = "my-topic", containerFactory = "kafkaListenerContainerFactory")
public void listen(ConsumerRecord<String, String> record, Acknowledgment ack) {
    try {
        // Process message
        System.out.println("Received: " + record.value());

        // Business logic (e.g., DB write)

        // ✅ Commit the offset manually
        ack.acknowledge();
    } catch (Exception e) {
        // Optionally handle errors, retry, etc.
    }
}

/** 
Why You Need kafkaListenerContainerFactory

Spring Boot’s default Kafka listener container uses AckMode.BATCH or AckMode.RECORD, and auto-commit behavior, depending on the properties.
To use `MANUAL commit` or `MANUAL_IMMEDIATE` , you must override the container factory and set
*/
```

### AckMode.BATCH setup in Kafka

- 📦 `max.poll.records` controls batch size and enable `batchListener = true` in kafkaListenerContainerFactory
- 🎯 `@KafkaListener` with `List<T>` parameter receives messages as a batch.
- 🔒 `enable-auto-commit=false + AckMode.BATCH` ensures `offsets are committed automatically` after `each batch is processed`;
- the `developer does not call ack.acknowledge() manually`.
- ⚡ This is still streaming under the hood, but you are processing messages in batches.
- ❗ This is **not traditional batch processing**; it is just a batch in Kafka.

### Difference: Traditional Batch vs Kafka Batch

| ⚡ Aspect | 🏢 Traditional Batch | 🪶 Kafka Batch |
|-----------|--------------------|----------------|
| 🗂️ Processing | Entire dataset processed together | Messages processed continuously but delivered in small batches |
| ⏰ Timing | Scheduled or periodic | Streaming, happens as messages arrive |
| 🎛️ Developer Control | Usually controls when batch starts and ends | Kafka controls batch delivery based on `max.poll.records` |
| ✅ Acknowledgment | Typically done after batch completes | `AckMode.BATCH` commits offsets after each batch automatically; no manual ack needed |
| 🏗️ Nature | Blocking, all-or-nothing | Non-blocking, still streaming under the hood |
| 📌 Use Case | Reports, ETL, large offline jobs | High-throughput message processing with batching convenience |
| 💡 Examples | Can be done by Hadoop, Spark | Kafka only |


### 1. ❓ Can we use `AckMode.BATCH` for manual commit?

👉 **No**, `AckMode.BATCH` and `AckMode.RECORD` are used for **automatic offset commits** (either by `Kafka itself or Spring`, depending on settings).  
They are **not intended** for manual control of offset commits.

🛠️ If you want to manually commit, you must use one of these modes:

- ✋ `MANUAL`
- ⚡ `MANUAL_IMMEDIATE`

✅ So: You **cannot** use `BATCH` for manual offset commit.

---

| ⚙️ Config                                                                | 📝 Description                                                                                                                |
|--------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| `enable-auto-commit=true`                                                | 🔄 `Kafka auto-commits` offsets at intervals (`auto.commit.interval.ms`) — Spring's AckMode is ignored.                       |
| `enable-auto-commit=false` + `AckMode.BATCH`                             | 🔧 `Spring Kafka commits` offsets after each batch is processed — this is `not "manual commit"` but Spring-controlled commit. |
| `enable-auto-commit=false` + `AckMode.MANUAL`/`AckMode.MANUAL_IMMEDIATE` | ⚙️ you have to commit the offset yourself                                                                                     |
---

### 2. ❓ What happens if consumer crashes before batch size is reached (e.g. AckMode.BATCH)?

- ⚙️ If you're using `AckMode.BATCH` with auto-commit disabled , and:
    - 🛠️ You process some messages
    - 💥 A crash happens before the batch is acknowledged

- ⚠️ Then:
    - ❗ Offsets are **not committed** — those messages will be reprocessed after restart.

⚙️ This is **by design** — batching delays offset commits to improve performance, but at the cost of reliability if the consumer fails mid-batch.

---

| ⚙️ **enable-auto-commit** | 🛠️ **AckMode**               | 👤 **Who commits the offset?**       | ⏰ **When?**                                    |
|--------------------------|------------------------------|-------------------------------------|------------------------------------------------|
| ✅ `true`                 | 🚫 Ignored                   | 🤖 Kafka client                    | ⏲️ Every interval (`auto.commit.interval.ms`)  |
| ❌ `false`                | 📦 `AckMode.BATCH`            | 🌱 Spring                          | 📄 After each batch is processed                |
| ❌ `false`                | 📃 `AckMode.RECORD`           | 🌱 Spring                          | 📝 After each record is processed               |
| ❌ `false`                | ✋ `MANUAL`                   | 👨‍💻 You (your code)                | 🖱️ When you call `ack.acknowledge()`           |

- ✅ So `AckMode.BATCH` / `AckMode.RECORD` works **only** when `enable-auto-commit = false`, Spring Kafka will commit the offset.
- ❌ If `enable-auto-commit = true`, then `AckMode` is ignored — Kafka client commits the offset.
- ⚙️ `enable-auto-commit=false`, and `AckMode.MANUAL` / `AckMode.MANUAL_IMMEDIATE`, you have to commit the offset yourself.


---

### 3.❓ If I use manual offset commit (`AckMode.MANUAL`), will it lead to more network calls to Kafka, and is that a performance issue?

🔹 **Yes** — if you commit after every message, manual commit causes more network calls to Kafka.

Because:

- 📞 Each call to `ack.acknowledge()` triggers an offset commit request to Kafka.
- 🌐 That’s a separate network call.
- ⚡ So if you process 1,000 messages per second and acknowledge every message → that’s up to **1,000 network calls per second** just for commits.

---

### 🧠 Why It’s a Problem

- 📡 **Network Overhead:** Too many small commit requests.
- 🏋️ **Broker Load:** Kafka brokers handle more commit requests.
- ⏳ **Throughput Impact:** Your app spends more time waiting on network.

---

### ✅ Solution: Manual + Batching

Even with `AckMode.MANUAL`, you can reduce network calls by:

👉 **Manually committing in batches:**
```bash

    int count = 0;
    List<String> buffer = new ArrayList<>();
    
    @KafkaListener(...)
    public void listen(String message, Acknowledgment ack) {
        buffer.add(message);
        count++;
    
        if (count >= 10) {
            process(buffer);         // Your logic
            ack.acknowledge();       // One commit for 10 messages
            buffer.clear();
            count = 0;
        }
    }
```

✅ This gives you:

- 🎛️ Full control
- 📉 Fewer network calls
- ⚖️ Balanced performance

---
### 4. In Kafka 🟡, if a consumer 👤 is using manual offset management 📊 with batch commits 🗂️, how does it fetch new messages 📥 without committing the current offset 🔒, and what happens to offsets if the consumer crashes 🛑 or stays active ▶️?

**Answer:**

- The `consumer keeps track of` `offsets` `in memory` 🧠, so it `can fetch new messages` 📥 even without committing. 
- If the consumer crashes 🛑, it uses the `last committed offset` 🔒 `from the broker`. 
- If the consumer is still running ▶️, it `uses the in-memory offset` 🧠.
- `Developers don’t need to create` the `in-memory storage`; Kafka does it automatically ⚡.

---
### ✅ Final Summary

| 🧾 Commit Style         | 📶 Network Calls | ✔️ Reliable? | 🎛️ Control    |
|------------------------|------------------|--------------|---------------|
| Manual (per message)   | 🔺 High          | ✅ Yes       | ✅ Full       |
| Manual (batched)       | 🔻 Low           | ✅ Yes       | ✅ Full       |
| Spring auto (`BATCH`)  | 🔻 Low           | ✅ Yes       | ❌ No        |
| Kafka auto-commit      | 🔻 Low           | ❌ Risky     | ❌ None      |

---

### Kafka Consumer Offset Commit Behavior

| # | Offset Commit Timing | Message Processing Status | Commit Stored in Broker? | Commit ACK Received by Consumer? | Consumer Restart Behavior | Outcome on Message | Delivery Guarantee / Behavior |
|---|--------------------|--------------------------|-------------------------|---------------------------------|--------------------------|------------------|------------------------------|
| 1 | After processing   | ✅ Success               | 🔒 Yes                  | ✅ Yes                          | `Starts from next offset → last committed offset = 4 → starts from 5` | Processed once, no reprocess 🔄 | At-least-once ☑️ |
| 2 | After processing   | ✅ Success               | 🔒 Yes                  | ❌ No                           | `Starts from next offset → last committed offset = 4 → starts from 5` | Processed once, no reprocess 🔄 | At-least-once ☑️ |
| 3 | After processing   | ❌ Failed               | ❌ No                   | —                               | `Starts from next offset → last committed offset = 3 → starts from 3` | Message reprocessed 🔄 | At-least-once ☑️ |
| 4 | After processing   | ❌ Failed               | ❌ No                   | —                               | `Starts from next offset → last committed offset = 3 → starts from 3` | Message reprocessed 🔄 | At-least-once ☑️ |
| 5 | Before processing  | ✅ Success               | 🔒 Yes                  | ✅ Yes                          | `Starts from next offset → last committed offset = 4 → starts from 5` | Message processed 📬 | At-most-once ❌ |
| 6 | Before processing  | ✅ Success               | 🔒 Yes                  | ❌ No                           | `Starts from next offset → last committed offset = 4 → starts from 5` | Message processed 📬 | At-most-once ❌ |
| 7 | Before processing  | 🛑 Crash before processing | 🔒 Yes                 | —                               | `Starts from next offset → last committed offset = 3 → starts from 4` | Message lost ⚠️ | At-most-once ❌ |
| 8 | After processing   | ✅ Success               | ❌ No                   | ❌ No                           | `Starts from next offset → last committed offset = 3 → starts from 3` | Message reprocessed 🔄 | At-least-once ☑️ (duplicates possible ⚠️) |
| 9 | Commit + message output in transaction (EOS) | ✅ Success | 🔒 Yes | — | `Starts from next offset → transaction committed → next offset = 5` | No duplicates, no loss 💎 | Exactly-once 💎 |

---

# 🧑‍💻 Offset Commit Control in Spring Kafka 

🧑‍💻 In Spring Boot with Spring Kafka, offset commits are usually handled via `Acknowledgment.acknowledge()`, which is a synchronous commit behind the scenes. But if you want more control to explicitly choose between `commitSync()` and `commitAsync()` like in the raw Kafka consumer:

## 1. 🟢 Default in Spring Kafka (`ack.acknowledge()`)

- Calling `acknowledge()` triggers a synchronous commit — Spring waits for the commit to complete (similar to `commitSync()`).
- ⚙️ It’s simple and reliable for most use cases.

## 2. ⚡ How to use `commitAsync()` in Spring Kafka?


- Spring Kafka doesn’t expose `commitAsync()` directly on the `Acknowledgment` interface, but you can do it manually by accessing the underlying Kafka consumer.
- Using `commitAsync()` manually inside `@KafkaListener`
   - To get the `Consumer<?, ?>` injected, your listener method should include it as a parameter.
   - 🧮 You must calculate offsets manually and call `commitAsync()` on the consumer instance.
   - ⚠️ This approach is lower-level, bypassing Spring Kafka’s `Acknowledgment` abstraction.
   - 🚫 You lose some container-managed features like retries and error handling integration.

### 📋 Best practice in Spring Kafka projects

- ✅ Use safe synchronous commit (`acknowledge()`).
- ⚡ Use manual `commitAsync()` on the `Kafka consumer only` if you need high throughput and can tolerate potential re-processing.
- ⚠️ Always handle commit failures in the callback when using `commitAsync()`.

---

## Example in Java
```bash

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("my-topic"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        // 1. Process the record
        System.out.printf("Consumed record: key = %s, value = %s%n", record.key(), record.value());
        
        // 2. Do something meaningful (e.g., write to DB)
    }
    
    // 3. Commit offsets manually AFTER processing
    consumer.commitSync();  // or commitAsync() 
    
    // or commitAsync with callback handle failure
      consumer.commitAsync((offsets, exception) -> {
            if (exception != null) {
                System.err.println("Commit failed: " + exception.getMessage());
            }
        });
}

```