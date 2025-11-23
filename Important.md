# 📝 Kafka Delivery, Producer, and Partition Notes

## 🚀 At-Most-Once Delivery
⚠️ `Main problems:`

- 🔹 `Consumer commits offset before processing → crash causes message loss`  
- 🔹 `Async commit fails → may re-fetch the same message`  

## ✅ Kafka At-Least-Once Delivery
⚠️ `Main issues:`

- 🔹 `Commit failure (sync or async) → message may be reprocessed`  
  - 🔸 `Because offset wasn’t recorded successfully, Kafka may resend it.`  
- 🔹 `Any side-effects (DB writes, API calls) cannot be rolled back unless you use transactions.`  
- 🔹 `Consumer crash after processing but before committing → duplicate processing`  
  - 🔸 `Kafka re-sends the last committed offset + 1.`  

---

## 🏭 Kafka Producer Notes: Acks, In-Flight Messages, and Ordering

### 1️⃣ Producer Acknowledgments (acks)

| acks | Meaning | Pros | Cons |
|------|---------|------|------|
| 0 | `Producer does not wait for broker acknowledgment` | ⚡ `Fastest throughput` | ❌ `Messages can be lost if broker fails` |
| 1 | `Wait for leader acknowledgment only` | ⚡ `Faster than acks=all` | ⚠️ `Less reliable; order can break with retries & in-flight messages` |
| all / -1 | `Wait for all in-sync replicas` | ✅ `Most reliable; preserves order` | 🐢 `Slower than acks=1 or 0` |

### 2️⃣ In-Flight Messages
- 🔹 `Definition: Messages that are sent by producer but not yet acknowledged by broker.`  
- 🔹 `Config: max.in.flight.requests.per.connection`  
- 🔹 `Default: 5`  
- 🔹 `Higher → more concurrency → higher throughput`  
- 🔹 `Too high + retries + acks<all → risk of out-of-order messages`  

### 3️⃣ Out-of-Order Messages Example
- 🔹 `Settings: acks=1, max.in.flight=5, retries>0`  
- 🔹 `Producer sends: Msg1, Msg2, Msg3, Msg4, Msg5`  
- 🔹 `Msg2 fails → retried`  
- 🔹 `Msg3–5 succeed first`  
- 🔹 `Broker stores: Msg1, Msg3, Msg4, Msg5, Msg2 ✅ out-of-order`  
- 🔹 `Cause: High in-flight + leader-only ack + retry → messages can overtake each other.`  

### 4️⃣ Ensuring Message Order
- 🔹 `Method 1: max.in.flight.requests.per.connection = 1`  
  - `Only 1 message at a time → no overtaking → order preserved`  
- 🔹 `Method 2: acks=all + max.in.flight>1`  
  - `Kafka uses sequence numbers per partition`  
  - `Retries maintain sequence → order preserved`  
  - `Allows higher throughput than max.in.flight=1`  

### 5️⃣ Summary Table: Order vs Reliability vs Throughput

| Setting | Order Guaranteed | Throughput | Reliability |
|---------|----------------|-----------|------------|
| `acks=0, any max.in.flight` | ❌ | `Highest` | `Low` |
| `acks=1, max.in.flight>1` | ⚠️ `Not guaranteed` | `High` | `Medium` |
| `acks=1, max.in.flight=1` | ✅ | `Lower` | `Medium` |
| `acks=all, max.in.flight>1` | ✅ | `High` | `Very High` |
| `acks=all, max.in.flight=1` | ✅ | `Lower` | `Very High` |

### 6️⃣ Quick Tips
- 🌟 `Use acks=all for safety and reliability.`  
- ⚡ `Use max.in.flight>1 for higher throughput if you don’t mind managing concurrency.`  
- ⏱️ `Use max.in.flight=1 for strict order when throughput is less important.`  
- 🔄 `Always enable retries for transient errors.`  

---

## 📊 Kafka Partition Sizing: 4-Pillar Method

### 1️⃣ Formula
Number of Partitions=max(Throughput per Partition,Latency per Partition,Consumer Parallelism,Future Growth Factor)


Where each pillar is calculated as:

**💨 Throughput per Partition:**

Throughput Pillar = Topic Load / Max throughput a partition can handle
	​



**⏱️ Latency per Partition:**

Latency Pillar = Topic Load / Max messages per partition to meet latency SLA
	​



**👥 Consumer Parallelism:**

Consumer Parallelism Pillar = Topic Load / Throughput per consumer
	​

**🌱 Future Growth Factor:**

Future Growth Pillar = Topic Load × Expected Growth Multiplier


### 2️⃣ Steps to Calculate Partitions

- 🔹 `Convert all units (messages/sec, messages/min, messages/day) to a common unit.`  
- 🔹 `Calculate each pillar using the formulas above.`  
- 🔹 `Take the maximum value among the four pillars.`  
- 🔹 `Round up to the nearest integer → this is the number of partitions needed.`  

### 3️⃣ Quick Tips

- 🌟 `Always consider future growth, because partition count is hard to reduce later.`  
- 👥 `Consumer parallelism ensures all consumers in a group can work without idle partitions.`  
- ⚡ `Throughput and latency pillars ensure partitions can handle traffic and meet SLA.`  
- 🔄 `If topic load changes frequently, recalculate periodically.`  

### 4️⃣ Example

`📌 Topic Load: 2 million msgs/hour`  

**Pillars:**

- 💨 `Throughput per Partition: 15,000 msgs/hour → 2,000,000 / 15,000 ≈ 134`  
- ⏱️ `Latency per Partition: 120,000 msgs/hour → 2,000,000 / 120,000 ≈ 17`  
- 👥 `Consumer Parallelism: 500 msgs/hour per consumer × 6 consumers → 2,000,000 / (500 ∗ 6) ≈ 667`  
- 🌱 `Future Growth: 4× → 2,000,000 × 4 = 8,000,000 → relative to throughput per partition? → 534 partitions`  

`✅ Number of Partitions = MAX(134, 17, 667, 534) = 667`

