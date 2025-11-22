# 🧭 Kafka Offset Management

- 📘 In Kafka, offset is the `position (unique number) of a message` in `a partition` — like a bookmark that `tells the consumer where it left off reading`. 
- 🔁 `Consumers must commit the offset` of messages `after they are successfully processed` to `avoid reading the same message again` `after a crash or restart.`

---

## 📦 Example:

Let’s say a Kafka topic: `Order-topic` and Partition: `P0` has the following messages:

| Offset | Message       |
|--------|---------------|
| 1      | "Order #1001" |
| 2      | "Order #1002" |
| 3      | "Order #1003" |

---

## 1. 1️⃣ Automatic Offset Commit (At-Most-Once Delivery) -> message is delivered 0 or 1 time

- ⚡ At-most-once delivery: `Message is delivered 0 or 1 time` — `no duplicates`, but `message loss is possible`.
- ⚡  `Fast delivery`
- ❌ `may lose messages`
- 📘 Kafka automatically `saves the offset at regular intervals` (default: every 5 seconds), whether `the message is processed or not`.

### ⚙️ How It Works

- ⚙️ **Property**: `enable.auto.commit = true` (default is true)
- **What happens:**
  - 🔄 Kafka's consumer automatically commits the latest offset returned by the `poll` at regular intervals (defined by `auto.commit.interval.ms`, default is 5000ms or 5 seconds).
  - 🚫 `No manual commit needed`.
  - 🧠 `Offsets are committed` `asynchronously` in the background.

### 📦 Example:

You read message at offset 3:  
🟢 `"Order #1003"`

- Kafka waits 5 seconds and automatically commits offset 3.
- If your `app crashes after reading` `but before 5 seconds`, on restart, Kafka sends offset 3 again.


### ⚙️ How It Works Internally:

- 📥 Consumer reads message
- 📊 Kafka tracks the offset
- ⏱️ Every 5 seconds (default), Kafka saves the current offset to a special Kafka topic called `__consumer_offsets`
- 🔁 On restart, Kafka starts from `last committed offset + 1`


### ❌ Problems:

- ⚠️ If the app crashes before Kafka auto-commits, the message is read again (`possible duplicates`)
- ❌ You can't control when the offset is saved
- 🔁 Risk of processing the same message twice


### ⚠️ Caveats

- ⏳ You don’t control exactly when the commit happens.
- 💥 If the consumer crashes after polling but before processing, the offset may be committed before the message is handled → `message loss` (lost messages if offsets are committed too early)


### 🕰️ When to Use:

- ⚡ When message `processing is fast` and `not critical`
- 🔄 You `don’t mind occasional duplicate processing`
- 🪄 Use for `logging`, `analytics`, or `simple data pipelines`


### ✅ Pros:

- 👍 Very easy to use
- 💡 No manual code needed to commit
- ⚙️ `Good for simple`, `fast-processing apps`

### ❌ Cons:

- 🚫 `Not reliable` for `critical data`
- 🔁 Risk of duplicates
- ❌ No control over when offset is saved

---

## 2. 2️⃣ Manual Offset Commit(At-Least-Once delivery) -> Message is delivered 1 or more times

- 🔁 At-least-once delivery: `Message is delivered 1 or more times` — `no loss`, but `duplicates are possible`.
- ✅ `Reliable delivery`, may have duplicates

- 📘 You (the developer) manually tell Kafka:
> “I’ve finished processing this message — now save the offset.”

- when the consumer commits offsets — typically **after successful processing** of messages.
- This helps ensure `at-least-once` delivery semantics, `reducing the risk of message loss` due to premature offset commits.


### 🔧 Manual Offset Commit – How It Works

**Step-by-step flow:**

1. 📥 Consumer polls messages from Kafka
2. 🛠️ Processes the messages (e.g., saves them to DB)
3. ✅ Commits the offsets manually (either synchronously or asynchronously)

💡 If the consumer fails before commit, the messages will be re-consumed.


### ✅ Enabling Manual Commit

- ⚙️ Set `enable.auto.commit = false`


### 🧠 Commit Types

#### 🔹 `commitSync()`

- ⛔ `Blocking`
- 🔐 Waits for Kafka to confirm the offset is committed.
     -  If commit fails, the consumer knows immediately and can:
         - Log the error. 
         - Retry the commit.(retry logic can be automatic in your code because you get immediate failure notification).
- ✅ Ensures the offset is safely stored.
     - You can handle failures more reliably with commitSync() because you get a clear success/failure signal.
- 🐌 `Safer but slower`.

   ##### 📥 What happens:
     - 📤 Consumer sends a commit request to Kafka with the latest offset. 
     - 📦 `Kafka acknowledges the commit only` `when the offset is stored in its internal offsets topic` (`__consumer_offsets`). 
     - ⏳ `Consumer waits until Kafka replies` with **"OK, I saved it"**. 
     - ✅ Once that happens, everything is safe.
   - 🧱 If the `consumer crashes` `after Kafka stores the offset`, the `offset` is `safe inside Kafka`. 
   - ⚠️ If the `consumer didn’t receive the ACK` from `the broker`, it `may be unaware that the offset was successfully committed`, However, `since the offset is safely stored inside Kafka’s internal topic` (__consumer_offsets), this `does not cause re-fetching or duplicates`.
   - 🔁 When the consumer restarts, it will `continue` from the `last committed offset in __consumer_offsets.` 
   - 📡 `Kafka itself` `doesn’t “resend” the message` — the `consumer re-requests` it because it believes the previous commit failed.
   - 💡 Final verdict: `Whether the consumer receives the ACK or not`, `if the offset is stored inside __consumer_offsets`, it `does not lead to re-fetching or duplicates`.

#### 🔹 `commitAsync()`

- ✅ `Non-blocking`
- 🚀 Fire-and-forget: won’t wait for confirmation.
- ⚡  `Faster`, but `no guarantee commit` `was successful`.
   - No retry logic:
      - If the `commit fails` (e.g., network error, broker issue), the `consumer won't know` , unless you provide a callback — `it doesn’t wait or retry`. 
      - With `sync commit`, `you can catch the failure and retry`.
- 🧩 You can also provide a `callback` to handle failures and implement your own retry logic if needed.

  ##### ⚙️ What happens:
     - 🔄 Consumer `sends the offset to Kafka asynchronously` (in the background). 
     - ⏩ `Consumer does NOT wait` for `Kafka to confirm`. 
     - 📦 Kafka tries to store the offset in its internal topic, but:
          - ✅ If successful: Offset is saved. 
          - ❌ If failed: Offset might not be saved.
     - 💥 If the consumer crashes immediately after, Kafka might not have stored the offset yet. 
     - ➡️ **Result:** Kafka thinks the message was never processed, and sends it again when the consumer restarts → 🔁 duplicate processing.


### ⚠️ Key Considerations

| 🧩 Aspect        | 🔐 `commitSync()` | 🚀 `commitAsync()` |
|------------------|-------------------|---------------------|
| ⏱️ Blocking?     | ✅ Yes            | ❌ No              |
| 🛡️ Reliable?     | 🔒 High           | ⚠️ Medium          |
| ⚙️ Performance   | 🐢 Slower         | ⚡ Faster           |
| 🧪 Use Case      | 🧾 Critical processing | 📊 High-throughput scenarios |


## Kafka Offset Commit Comparison

### commitAsync()

- ⚠️ **No automatic retry:**
    - 📨 When you call `commitAsync()`, the Kafka client sends the commit request **asynchronously** and immediately returns.
    - ❌ If the commit fails (network error, broker down, etc.), the client **does not retry automatically**.
    - ℹ️ Kafka might succeed or fail in storing the offset, but the consumer has no guaranteed way to know unless a callback is used.

- 🔔 **Callback option:**
    - 📣 You can provide a callback to `commitAsync()` to get notified if the commit **succeeded or failed**.
    - 🔁 If it fails, you can **manually retry** inside the callback.

- ⏩ **Non-blocking:**
    - 🏃 The consumer continues processing messages **regardless of commit success**.
    - ⚡ This makes `commitAsync()` faster but **less safe for exactly-once guarantees** compared to `commitSync()`.

### commitSync()

- 🔄 **Automatic retry:**
    - 🔁 `commitSync()` automatically retries internally for any retriable exceptions (like network issues, leader not available, or temporary broker failures).

- ⏳ **Blocking behavior:**
    - 🛑 It blocks the caller until either:
        - ✅ The offset is **successfully committed**, or
        - ❌ A **non-retriable exception** occurs.

- 🛠 **No extra retry logic needed:**
    - 💡 You don’t need to implement your own retry logic — the Kafka client handles it automatically.

- ⚠️ **Exception handling:**
    - 📝 The only thing you need is a `try-catch` around `commitSync()` to catch final exceptions if all internal retries fail.

- 🔁 **Guaranteed commit retry:**
    - 🔒 Automatic retry continues internally **until the commit succeeds or a fatal error happens**.



### 🔐 Why Manual Commit?

- ✅ To `ensure message processing` before committing.
- 🔁 To `implement retry mechanisms` or `transactional processing`.
- 🧷 To `prevent data loss` in case of consumer failure.

### 📦 Example:

You read message at offset 3:  
🟢 `"Order #1003"`  
You finish saving it to DB → then call:  
🔧 `commitSync()` or `commitAsync()`

✅ Now Kafka saves offset 3.

- 💥 If your app crashes before committing, Kafka will re-send offset 3. 
- 🛡️ You can make sure you don’t commit until processing is successful.

### ⚙️ How It Works Internally:

- 📥 Consumer reads message
- 🛠️ You process it (e.g., save to DB)
- 🔧 You manually call `commitSync()` or `commitAsync()`
- 🧠 Kafka stores the offset in `__consumer_offsets`
- 🔁 On restart, Kafka resumes from **last committed offset + 1**

### ❌ Problems:

- 👨‍💻 You must write extra code
- 🕳️ If `you forget to commit` → Kafka will `keep resending old messages`
- 💣 `Improper handling can lead` to `duplicate processing`, `not message loss`

### 🕰️ When to Use:

- 🛡️ For `critical applications` (e.g., orders, payments)
- For applications where `losing messages is unacceptable`
- 🧠 When you can handle duplicates safely (idempotent operations)
- ✅ For `critical data` where `full control over processing` is required


### ✅ Pros:

- 🔐 Safer and more reliable than auto-commit without processing guarantees
- 💼 Good for **critical business data**
- 💼 `Guarantees no message loss` if offset is committed after processing 
- ⚖️ Allows fine-grained control over processing logic


### ❌ Cons:

- 👨‍💻 More complex
- 🧾 Requires manual code to commit
- ❗ Mistakes can lead to `duplicate processing (reprocessing)`, but `not message loss`

---

Rebalancing

---
# 🚀 Kafka Consumer Lag

In Kafka, a consumer reads messages from a topic (think of a topic as a stream of messages).

**📝 Producer:** Sends messages to Kafka.  
**👤 Consumer:** Reads messages from Kafka.

## ⚡ Consumer Lag

The` number of messages` `in the topic` that `the consumer has not read yet`.

- ✅ If lag is 0, the consumer is caught up.
- ⏳ If lag is 100, the consumer is 100 messages behind.

### 📝 Simple Example with Data

Imagine a topic `orders`:

| Offset | Message |
|--------|---------|
| 0      | order1  |
| 1      | order2  |
| 2      | order3  |
| 3      | order4  |
| 4      | order5  |

Producer adds two more messages:

| Offset | Message |
|--------|---------|
| 5      | order6  |
| 6      | order7  |

Now, the consumer has not read offsets `3, 4, 5, 6`.

**Consumer lag = 4 messages.**

### Consumer lag can be calculated as:
`Consumer Lag = Latest Offset − Consumer’s Current Offset`

``` bash

From the example:  
   - Latest offset = 6  
   - Consumer offset = 2
   - Lag = 6 − 2 = 4
```



## ⚠️ The Problem: Consumer Lag

The `problem arises` when `your consumer is slower` than `the producer`, meaning it cannot keep up with the incoming messages.

Think of it like this:

- 📦 Messages are coming down a conveyor belt (Kafka).
- 👷 The consumer is a worker picking up packages. 
- If the worker is slow, packages pile up — that pile is your `lag`.

### ❌ Why It’s a Problem

- ⏱ **Delayed Processing**  
   - If the consumer is behind, messages are not processed in real-time.  
   - Example: In `an e-commerce system`, delayed order processing can lead to unhappy customers.

- 💾 **Memory & Storage Pressure**  
   - Kafka stores messages until they are read or expired.
   - `If consumers are slow`, the `topic may hold many unprocessed messages`, `increasing storage usage`.

- ⚡ **System Instability**  
  - Very high lag can indicate system problems:
    - 💥 Consumers crashed or slow
    - 🔢 Too few consumers for the number of partitions
    - 🌐 Network or resource bottlenecks

- 🗑 **Data Loss Risk (if retention is short)**  
  - Kafka deletes messages after a certain retention period.
  - If the consumer hasn’t read messages before they are deleted, data could be lost.


## 🔍 What Causes the Problem?

- 🐢 Slow consumer processing (e.g., heavy computation per message)
- 🌐 Network issues between consumer and Kafka
- 👥 Too few consumers for the number of partitions
- ⚡ High message production rate

Consumer lag is `a sign that your system` is `“falling behind”` and `might lead to delays` or `data loss`.


## 💡 Solutions

### 1️⃣ Make Consumers Faster
If your consumer is slow, it can’t keep up. Ways to make it faster:

- ⚙️ **Optimize processing logic:**  
  - Process messages faster or do less work per message.  
  - Example: Instead of saving to a database one by one, batch writes.

- 🔄 **Use asynchronous processing:**  
  - Don’t block the consumer while waiting for slow operations.

- 🖥 **Upgrade hardware or resources:**  
  - `More CPU, RAM`, or `faster disk` can help `your consumer keep up`.

### 2️⃣ Increase the Number of Consumers
Kafka allows multiple consumers in a consumer group to share partitions.

- 🧩 Topic has 4 partitions.
- 👤 Only 1 consumer → reads 1 partition at a time → lag builds.
- ➕ Add 3 more consumers → each reads a partition → lag decreases.

**Rule:** Number of consumers ≤ number of partitions.

### 3️⃣ Tune Kafka Configuration
Some Kafka settings can help reduce lag:

- 📥 `fetch.min.bytes / fetch.max.wait.ms`: Control how consumers fetch messages (batch more efficiently).
- 📄 `max.poll.records`: Increase the number of messages fetched in one poll to reduce lag.
- ⏲ `session.timeout.ms / heartbeat.interval.ms`: Ensure consumers stay connected properly to avoid unnecessary rebalancing.

### 4️⃣ Handle Backpressure
Sometimes producers are too fast. Options:

- 🐌 **Throttle producers:** Slow down message production if consumers can’t keep up.
- 🗂 **Buffer messages in a queue:** Temporarily store messages for consumers to catch up.

### 5️⃣ Monitor Lag
Always track consumer lag using tools like:

- 📊 Kafka’s Consumer Group command (`kafka-consumer-groups.sh`)
- 📈 Monitoring tools (Prometheus + Grafana)

**Alerts help you act before lag becomes a serious problem.**



---

# ⚙️ Auto Offset Reset

- When `a consumer starts reading a topic`, `it needs to know` `where to start reading messages from`. 
- `Kafka stores the offset` of the `last message a consumer read`. 
- `If the consumer has no saved offset` (first time reading) or the `saved offset is no longer valid` (for example, if the data has been deleted because it’s older than the retention period),  
- Kafka uses **auto.offset.reset** to decide what to do.

- By default, Kafka (and Spring Kafka) sets:
  - `🧭 auto.offset.reset = latest`

## 🧭 It has two main options:

- ⏮️ **earliest** → Start reading from the oldest message in the topic.
- ⏭️ **latest** → Start reading from the new messages arriving from now on.

### 📋 Setting Behavior

| ⚙️ Setting | 🧠 Behavior |
|------------|-------------|
| earliest | Reads all messages from the beginning of the topic if no offset exists |
| latest | Reads only new messages that arrive after the consumer starts |
| none | Throws an error if there is no previous offset (useful for strict systems) |

## ❓ Why Do We Need It?

- 🆕 When `a new consumer joins a group`, there is no previous offset.
- 🗑️ Or `if offsets are deleted` (due to retention policies).
- 🤔 Kafka needs to know where to start; otherwise, it will throw an error.


## 🕐 When to Use

| 📌 Situation | ✅ Recommended Setting |
|--------------|------------------------|
| First time reading a topic and `want all messages` | earliest |
| First time reading a topic and `only want new messages `| latest |
| You don’t know if old messages are relevant | Usually latest |

## ⚠️ Problems with Auto Offset Reset

- ⚡ **Data loss risk**  
   - If `latest` is used and `consumer starts after messages are produced` → old messages are missed.

- 🔁 **Reprocessing old data**  
  - If `earliest` is used → consumer may `re-read old messages`, possibly `causing duplicate processing`.

- 😕 **Confusion in production**  
   - Choosing the wrong reset policy can lead to unexpected behavior in pipelines.

## 🧩 Solutions / Best Practices

- 🎯 **Pick the right policy for your use case:**
  - `earliest` → when `you need all historical data`
  - `latest` → when you `only care about new incoming data`
- 🧠 **Manually manage offsets if you need fine control.**
- 📊 **Combine with monitoring** → check consumer lag to ensure no messages are skipped or duplicated.
- 🗃 **Use compacted topics** if reprocessing old messages frequently causes problems.

## Example

Imagine a topic `order-topic`:

| 🧾 Offset | 📦 Message |
|------------|------------|
| 0 | order1 |
| 1 | order2 |
| 2 | order3 |
| 3 | order4 |
| 4 | order5 |

### 🔹 Scenario 1: Consumer has no offset

- ⚙️ **auto.offset.reset = earliest** → starts at offset **0** (order1)
- ⚙️ **auto.offset.reset = latest** → starts at offset **5** (new messages only)

### 🔹 Scenario 2: Consumer has offset 2

- ▶️ It continues reading from offset **3** (normal behavior, auto offset reset is not used here)

### ⚙️ When is `auto.offset.reset` Used?

`auto.offset.reset` is used when Kafka **cannot find a valid committed offset** for a consumer group and partition  

for example:
- 🆕 A new consumer group starts reading a topic for the first time (no offset yet).
- 🗑️ The stored offset in **`__consumer_offsets`** has expired or been deleted due to offset retention settings or log cleanup.

### 📌 Otherwise

- If a valid offset exists in **`__consumer_offsets`**,  
- Kafka uses that offset and **does not apply** `auto.offset.reset`.


## ⚙️ Kafka Consumer Configuration

``` bash

spring:
  kafka:
    consumer:
      # 🔁 Auto offset reset (decide where to start if no offset is found)
      # 🧭 Options: earliest | latest | none
      auto-offset-reset: earliest

```

## 🧾 Summary

- ⚙️ Auto Offset Reset decides where a consumer starts reading if no valid offset exists.
- 🧭 Options: earliest (oldest) / latest (new).
- ⚖️ Use it carefully depending on whether you want all past messages or only new messages.
- ⚠️ Problems: data loss or duplicate processing.
- 💡 Solution: choose policy wisely, monitor consumers, or manage offsets manually.


---

# 🚀 Producer-Side ACKs in Apache Kafka

When a producer sends a message to Kafka, it can ask for different levels of acknowledgment (confirmation) from the broker (Kafka server).  
This setting is controlled by the **`acks`** parameter.

There are 3 main types:

## 1️⃣ acks = 0
- Producer `doesn’t wait for any acknowledgment`.  
- It just `sends the message` and `moves on`.

**✅ Advantage:**
- Super fast (low latency).

**❌ Problem:**
- If `the broker crashes` or `network fails`, `messages can be lost` because the `producer never checks` if they arrived.

**💡 Solution:**
- Use only when you can tolerate message loss (e.g., logs, metrics).
- Otherwise, use **acks=1** or **acks=all** for reliability.

**📘 Real-life Use Cases:**
- 📝 `Collecting application logs` where losing a few entries is fine.
- 🌐 `Sending IoT sensor data continuously`, where speed matters more than 100% delivery.
- 📊 `Website analytics hits` (e.g., page view counts) that `don’t need perfect accuracy`.

## 2️⃣ acks = 1

- Producer waits for **leader broker** (the main broker for that partition) to confirm the message is written.

**✅ Advantage:**
- Balance between speed and safety.
- The message is written **at least once** to the leader.

**❌ Problem:**
- If the `leader crashes before followers replicate the data`, that `message can be lost` (since only the leader had it).

**💡 Solution:**
- Use this when small data loss is acceptable, or increase replication and use **acks=all** for higher durability.

**📘 Real-life Use Cases:**
- 👤 `Sending user activity events` (likes, clicks, scrolls) on `a website` or app or `Social Media App`.
- 📂 `Log aggregation systems` where most data is retained but occasional loss is acceptable.
- 📈 `Streaming metrics` (like CPU usage, app performance) where timeliness > perfection.

## 3️⃣ acks = all (or -1)
 
- Producer waits until `all in-sync replicas (ISR)` confirm they got the message.

**✅ Advantage:**
- Most reliable — message is safe even if leader crashes.

**❌ Problem:**
- `Slower` because `producer must wait for all replicas to confirm`.
- If `replicas are slow`, `throughput drops`.

**💡 Solution:**
- Use this when **data must never be lost** (e.g., financial transactions).
- To improve performance, ensure replicas are healthy and close in the network.

**📘 Real-life Use Cases:**
- 💳 `Bank transactions` or payment messages — every record must be stored safely.
- 🛒 `E-commerce orders` — you can’t lose an order even if servers crash.
- 📦 `Inventory or stock updates` — consistency is critical across systems.

### Kafka Producer Configuration ⚡

Configure Kafka Producer Properties in `application.yml`.
- `acks` can be set to `0`, `1`, or `all`:

```bash

application.yaml (only acks)

spring:
  kafka:
    producer:
      acks: all
```

| ⚡ ACK Value | 📝 Description                       |
|-------------|-------------------------------------|
| 0           | 🚀 No wait, fastest but risky       |
| 1           | ⏳ Wait for leader only              |
| all         | 🔒 Wait for all in-sync replicas, safest |


```bash

application.yaml ( acks + Retries and Idempotence 🔄)

spring:
  kafka:
    producer:
      acks: all
      retries: 3
      enable-idempotence: true
```

## Key Notes 💡

- ⚙️ `acks is set in producer configuration`; Spring Kafka passes it to the Kafka client.
- 🔄 `Retries + Idempotence` help `avoid duplicate messages` when using acks=all or acks=1.
- 🛡️ `Critical Data`: Always use acks=all + retries>0 + enable-idempotence=true.
- ⚡ `Non-critical Logs`: acks=0 is enough for speed.
- 🔄 `Retries + multiple in-flight messages` = possible reordering.
  - ⚙️ Set `max.in.flight.requests.per.connection=1` → `ensures strict ordering`.
    - 📝 This setting tells the producer: `only send 1 message` at a time per connection.
    - ⏳ That way, Kafka won’t send the next message until the previous one is fully acknowledged.
    - ✅ Result: order is preserved, even with retries.
    - ⚠️ Effect:
      - 🕒 If the broker is slow or there’s a network delay, all following messages are blocked, increasing latency.
    - ❓ Why this matters
      - 💰 Strict ordering is often required for financial transactions, logs, or inventory updates.
      - ⚡ But if you have high-throughput use cases, like metrics or clickstream events, this can significantly reduce performance.

### Tradeoff ⚖️:

| ⚙️ Setting | 👍 Pros | ⚠️ Cons | 🏷️ Real-life Use Cases | 
|------------|---------|---------|-----------------------| 
| `max.in.flight.requests.per.connection=1` | ✅ Guarantees ordering, safe with retries | ⚠️ Slower throughput, higher latency | 1. Banking transactions – debit/credit must happen in exact order<br>2. Inventory updates – prevent overselling products<br>3. Financial trade processing – orders executed in sequence | 
| `max.in.flight.requests.per.connection > 1` | ⚡ Higher throughput | 🔄 Messages may be reordered on retries | 1. Clickstream analytics – occasional reordering doesn’t break insights<br>2. Metrics collection – minor ordering issues are acceptable<br>3. Log aggregation – order of log entries isn’t critical |


### 📊 Summary Table

| ⚙️ acks | ⏱ What it waits for | ✅ Pros | ❌ Cons | 📝 When to use | 📘 Real-life Examples |
|---------|------------------|---------|---------|----------------|---------------------|
| 0       | No one            | Fastest | Data loss possible | Non-critical logs, metrics | 1. App logs<br>2. IoT data<br>3. Page views |
| 1       | Leader only       | Balanced | Some data loss possible | Most common use | 1. User clicks<br>2. Aggregated logs<br>3. Metrics |
| all / -1 | All replicas     | Most reliable | Slowest | Critical data | 1. Bank payments<br>2. Orders<br>3. Inventory updates |

---

### 1️⃣ Role of the key in Kafka

- 🗝️ Kafka partitions messages using the key.
- 📦 All messages with the same key always go to the same partition.
- 📝 Within a single partition, Kafka stores messages in the order they arrive.
- ✅ So yes, sending a key ensures messages for that key stay in the same partition, which is required for ordering per key.

### 2️⃣ Why key alone doesn’t guarantee order with retries

- ⚠️ Even with the key:
  - If retries are enabled and multiple messages are in-flight:
    - 🔄 Message 1 fails and is retried
    - ⏩ Message 2 succeeds immediately
      → Kafka may store them in the wrong order within the partition.

- 🛡️ This is why `max.in.flight.requests.per.connection=1` is needed to strictly preserve order, even with retries, for messages with the same key.


---

## What is ISR? ⚡

- 🗝️ `ISR = In-Sync Replicas`
- 📦 Kafka replicates data across multiple brokers for reliability.
- 📝 A topic partition has one leader and one or more followers.
- ✅ `ISR` is the set of replicas that are fully caught up with the leader (i.e., have all messages the leader has).

### Types of ISR 🏷️

While `ISR` is essentially the list of in-sync replicas, we can think in terms of replica states:

- 👑 **Leader** – The replica handling all read/write requests.
- ✔️ **Follower in ISR** – Fully caught up, considered safe.
- ⚠️ **Follower out of ISR** – Lagging behind, not fully synced; may be temporarily removed from ISR until it catches up.


### Problem: How to ensure data reliability and consistency when brokers fail? ❓

- Without `ISR`, a leader could fail, and followers may not have all data, leading to data loss.
- `ISR` ensures Kafka only considers replicas fully in sync when electing a new leader or committing messages.


### How ISR Solves the Problem 🛡️

- Kafka uses `ACKs` from ISR replicas for message durability.
- `acks=all` waits for all ISR replicas to confirm before considering a message committed.
- If a leader fails, only a replica in ISR can become the new leader → ensures no committed data is lost.
- Followers lagging behind are temporarily removed from ISR until they catch up.

**Example:**

- Partition `R1` (leader), `R2` & `R3` (followers)
- `R2` and `R3` are in `ISR`
- Leader receives a write → waits for ISR replicas to acknowledge → commits
- If `R1` crashes, only `R2` or `R3` (in ISR) can become the new leader → no committed data loss


### Real-life Use Case 💡

- **E-commerce orders processing**:  
  - Partition for orders topic replicated across 3 brokers.  
  - `ISR` ensures all replicas have the same order data.  
  - If one broker fails, Kafka can elect a new leader from ISR → no orders are lost.

- **Financial transactions**:  
  - `ISR` ensures committed trades are never lost even if a broker fails.

- **Logging / Metrics pipelines**:  
  - Guarantees log events are replicated and available across multiple brokers.

### Problem: Follower Out of ISR ⚠️

In Kafka, a follower replica is supposed to stay in sync with the leader.  
Sometimes, a follower lags behind due to:

- 🌐 Slow network
- 🖥️ High load on the broker
- 💾 Disk or CPU bottlenecks

**Effect:**

1. ⏳ **Temporary removal from ISR**:  
   - The leader of the partition monitors followers. If a follower falls behind beyond the allowed timeout (`replica.lag.time.max.ms`), the leader removes it from ISR until it catches up.

2. 🚫 **Cannot serve as leader**:  
   - Followers outside ISR cannot be elected as leader, ensuring no data loss.

3. 🐢 **May delay message commits if `acks=all`**:  
   - If your producer is configured with `acks=all` (wait for all ISR replicas to confirm a message), a smaller ISR may slow down message commits because the leader waits for fewer replicas.  
   - If too many followers are out of ISR, this can reduce redundancy and affect durability guarantees.


### How to Overcome Lagging Followers 🛠️

**Practical solutions:**

- 🌐 **Check Network / Broker Performance**  
  - Ensure fast and reliable network between brokers.  
  - Avoid overloaded brokers or slow disks.

- ⚡ **Increase Replication Performance**  
  - Tune Kafka configs for followers:
    - `replica.fetch.max.bytes` → allow followers to fetch more messages per request
    - `replica.fetch.wait.max.ms` → reduce wait time for fetch

- 🖥️ **Add More Resources**  
  - Give lagging brokers more CPU, RAM, or disk speed.  
  - Sometimes followers lag because they are resource-starved.

- 📊 **Reduce Leader Load**  
  - If leader is too busy, followers cannot keep up.  
  - Consider adding partitions or balancing data across brokers.

- 🔍 **Monitor ISR**  
  - Use Kafka tools or monitoring (Prometheus/Grafana) to detect lagging replicas early.  
  - React before followers fall far behind.

### Implementation ⚙️

1. 1️⃣ **Automatic Management**

   - 🤖 Kafka automatically maintains `ISR` for each partition. 
   - 👀 Leader monitors followers and removes lagging replicas temporarily.

2. 2️⃣ **What You Control**

   - 🏷️ `Replication factor`: Number of replicas for a topic. 
   - 📩 `Producer acks`: Determines how many ISR replicas must confirm a message.

      - `acks=0` → no wait
      - `acks=1` → wait for leader only
      - `acks=all` → wait for all ISR replicas

3. 3️⃣ **What You Don’t Need to Do**

   - 🚫 No manual adding/removing replicas from `ISR`. 
   - 🔄 Kafka handles catching up and rejoining lagging replicas automatically.


### Pros ✅

- 🛡️ **Data durability**: Only replicas that are fully in sync with the leader are considered part of ISR, ensuring that committed messages are not lost.
- 👑 **Safe leader election**: If the leader fails, only in-sync replicas can become the new leader, which prevents data loss.
- ⚡ **High availability**: Even if some followers fall behind, Kafka can continue processing messages using the remaining ISR replicas.
- 🔄 **Automatic recovery**: Lagging followers can rejoin the ISR once they catch up, making replication resilient and self-healing.

### Cons ❌

- 🐌 **Lagging followers can reduce redundancy**: When replicas fall out of ISR, fewer replicas store the committed messages, slightly increasing risk if the leader fails.
- 🚫 **Temporary unavailability for leader election**: Lagging followers cannot become leader until fully synced, which may limit options in case of leader failure.
- ⏳ **Potential commit delays**: In some cases (with `acks=all`), if ISR shrinks due to slow followers, it may affect how replication guarantees behave.
- 🖥️ **Resource sensitivity**: Maintaining ISR requires network, CPU, and disk performance; if brokers are slow, replicas may fall out of ISR frequently.



---
Exactly-Once Delivery  (Kafka Transactions)

---
Exactly-once vs at-least-once

---

Replica reads and consistency tradeoffs

---

Idempotent & Retryable Processing

---
Dead Letter Queues (DLQ)

When a message fails processing repeatedly, you can:

Skip it temporarily (log and retry later)

Send it to a DLQ topic to analyze and fix later

This prevents a bad message from blocking the consumer

---
Security in Consumers

Kafka supports:

TLS encryption

SASL authentication (e.g., SCRAM, GSSAPI)

ACLs for topic-level access control

Configure in consumer with:

---
Kafka Consumer Metrics

Kafka consumer exposes metrics you can monitor:

records-consumed-rate

fetch-latency-avg

commit-latency-avg

records-lag

records-lag-max

Use Prometheus + Grafana, or JMX exporters to track these.

---