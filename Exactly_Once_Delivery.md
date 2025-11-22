## ⚠️ Problem Without Exactly-Once Processing


### Setup:
🗂️ `Topic:` order-topic  
📊 `Partitions:` P0, P1

📤 `Producer sends orders like:`
- 🔹 P0: OrderID: 101, Amount: $50 (M1)
- 🔹 P1: OrderID: 102, Amount: $75 (M2)

📥 `Consumer reads orders and updates database or sends emails.`


### Problems that can happen:

#### 🔁 Duplicate Processing
- ⚠️ `Producer sends message` but `doesn’t get acknowledgment due to network issues`.
   - 📡 Internally
     - 📤  Producer sends a message to Kafka.
     - 🗄️ `Kafka stores` the `message safely.`
     - 📨 `Kafka sends back a message` called `an acknowledgment (ack)` saying, “Hey producer, I got your message!”
     - ⚠️  If the `producer does NOT receive` this ack because:
          - 🌐 `Network issue` `between Kafka and producer`, or
          - 🐢  Producer is `slow or busy` and `misses the ack,`
     - ❓  Then the `producer thinks` the `message didn’t get through.`
     - 🔄  So, the `producer sends the same message again` to `be safe.`

- 🔄 `Producer resends` the `same message`.
- 🗃️ `Kafka ends up with duplicate messages:`
    - 📍 P0: M1, M1
    - 📍 P1: M2, M2
- 🔁 `Consumer processes` the `same order twice.`

💥 `Result: Customer is charged twice or gets duplicate emails.`

#### 🚫 Message Loss
- 🕵️ `Consumer reads message` and `commits offset before processing`.
- 💥 `Consumer crashes before updating database.`
- ⏩ `After restart`, `consumer skips that message` (because offset was committed).

❌ `Result: The message is lost and the order is never processed.`

---

# 🎯 Why Exactly-Once matters?

To guarantee:

"`Every message` is `delivered and processed only once`, even with `failures`, `retries`, and `crashes`."

It Solves Two Big Problems:
- ✅ 🛑 No duplicates.  
- ✅ 📭 No message loss.

---

## ⚙️ How to Use Exactly-Once in Kafka

Kafka has a built-in mechanism for **exactly-once semantics (EOS)** using:

- ✅ **Idempotent Producers**
- 🔄 **Transactions**
- ⚙️ **Idempotent Consumers**


## 🛠️ You Need to:

1. ✅ **Enable Idempotence** on the producer.
2. 🔒 **Use Kafka Transactions** to group writes and commits.
3. 📥 **Use transactional-aware consumers.**

---

## ✔️ Pros and Cons

### ✅ Pros

- 🧼 No duplicates
- 🛡️ No lost messages
- ⚛️ Atomic writes to multiple topics
- 🔗 Clean integration with external systems (like DBs)

### ❌ Cons

- 🕒 More latency due to transactions
- ⚙️ Slightly more complex setup
- 🧠 Higher resource usage
- 🧬 Needs Kafka 0.11+ and Kafka client config

---

## ⚙️ When is Exactly-Once Useful?

### 💸 1. Payment Systems

**Example:** 💳 A user clicks “Pay Now” once.
- **Why Exactly-Once?**  
   - ✅ 🧾 To avoid double charging the customer.
   - ✅ 🧃 To make sure the payment is not lost.


### 🏦 2. Banking & Financial Transactions

**Example:** 🏧 Money is transferred between accounts.
- **Why Exactly-Once?**  
   - ✅ 🔁 To prevent duplicate transfers or missing funds.
   - ✅ 🧮 Guarantees account balances stay accurate.


### 🛒 3. Order Processing in E-commerce

**Example:** 📥 User places an order for a product.
- **Why Exactly-Once?**  
   - ✅ 📦 Avoid sending duplicate orders to the warehouse.
   - ✅ 🧺 Ensure the order is never missed.



### 📦 4. Inventory Management

**Example:** 🧾 Updating stock count after a sale.
- **Why Exactly-Once?**  
   - ✅ 🚫 Prevents over-selling or wrong stock levels.
   - ✅ 📊 Keeps inventory accurate and up-to-date.


### 📊 5. Analytics / Event Tracking

**Example:** 🖱️ Logging user actions or clicks.
- **Why Exactly-Once?**  
   - ✅ 📈 Avoid inflated numbers from duplicates.
   - ✅ 💥 Prevent missing data from crashes.


### 🔁 6. Data Pipelines / ETL Jobs

**Example:** 🔄 Moving data between systems (Kafka → Database).
- **Why Exactly-Once?**  
   - ✅ 🧷 Ensures data is not duplicated or lost during transfers.
   - ✅ 🗃️ Keeps data consistency across systems.

---


# 🚀 EOS (Exactly Once Semantics) 

**What it means:**  
Kafka guarantees that `each message is processed` `only once`, even with:

- 🔄 Producer retries
- 🔁 Consumer restarts
- 💥 Crashes

It combines all of the above:
- ✅ Idempotent Producers
- ✅ Kafka Transactions
- ✅ Idempotent Consumers  

---


# 🔄 Kafka Idempotent Producer

## ❓ What is an Idempotent Producer?

A Kafka producer sends messages to Kafka topics.

Sometimes, `due to network issues or failures`, it `may retry sending` the `same message`.

This can cause duplicate messages in Kafka.

💡 **Idempotent Producer** makes sure even if the producer retries, `Kafka stores` the `message only once`.

## 🧠 Why is it Needed?

Imagine:

🛒 You place an order → "`order-123`"  
📤 Producer sends the message to Kafka  
💾 Kafka stores it, but the producer doesn’t get an acknowledgment (ack)  
🔁 Producer thinks the message failed and sends it again

❌ Without idempotence → Duplicate message in Kafka  
✅ With idempotence → Only one message is stored


## ⚙️ How Does It Work Internally?

Kafka uses two things to detect duplicates:

### 1️⃣ PID (Producer ID)

🆔 A `unique ID` `given by Kafka broker` to `each producer instance` `when it starts`.

`Identifies` who is sending the message.

### 2️⃣ Sequence Number

🔢 `Assigned by the producer` itself (not Kafka).  
It’s like a message counter per partition.

For example:

```bash

toipc : order-topic
Partition 0:  
            M1 -> Seq 1
            M2 -> Seq 2
            M3 -> Seq 3
Partition 1:
            M4 -> Seq 1
            M5 -> Seq 2
Partition 2:
            M6 -> Seq 1
            M7 -> Seq 2
            M8 -> Seq 3            
```


🧩 Sequence numbers are maintained independently per partition  
🔹 The producer assigns these sequence numbers when sending

## Kafka tracks:  
🧾 `Producer ID + Partition + Sequence Number`
- If it `sees the same combination again` → `it's a duplicate` → `Kafka ignores it`.


## 🔁 How Retries Work (with idempotence)

1. Producer sends message with seq number = 3 of Message 8 in Partition 2
2. Kafka stores it and sends ack
3. If ack is lost, producer retries
4. Retry is sent with the same sequence number (3)
5. Kafka sees it’s already stored seq 3 from this producer → skips it

✅ No duplicate in the topic.

## 🧠 Key Points

- 📦 `Each partition` has `its own sequence number counter`
- ⏱️ Sequence number starts from 1 (or 0) and `increments only` `after receiving` an acknowledgment `ack` from the `Kafka broker`.
- 🔁 If `a message fails` and is `retried`, the `same sequence number` is `used again`
- 🔍 Kafka checks (PID + Partition + Seq) to detect duplicates.


## ❓ Who adds the sequence number in Kafka producer?

🛠️ **Answer:**

- The `Kafka client library` (the Kafka Producer inside your app) `automatically assigns` the sequence number for each message you send — you don’t add it manually. 
- 🧰 When you call `kafkaTemplate.send(...)`, the KafkaTemplate uses Kafka's producer client under the hood. 
- ⚙️ The Kafka producer client handles sequence numbers internally.

## ❓ Do I need to manually add PID to my producer?

- 🚫 No, `Kafka broker handles` this `automatically` if you enable idempotence.
- ✅ You just turn on a setting, and Kafka does the rest.  
- 🔐 You don’t manually set PID.


## ⚙️ How to enable idempotent producer in Spring Boot?

If you're using KafkaTemplate, you just need to set this property in your `application.yml` or `application.properties`.

```bash

spring:
  kafka:
    producer:
      properties:
        acks: all
        retries : 3 
        enable.idempotence: true

```

---

## 🔄 Quick Recap of Your Scenario:

- 🧑‍💻 Producer PID = 123
- 📤 It sends a message → Kafka assigns sequence number = 3 of message 8 in Partition 2
- 📬 Kafka stores the message and sends an ack
- ❌ But the ack is lost (network issue)
- 🔁 Producer retries sending the same message (because it didn't get ack)

👉 **Your question:**

❓ Won’t the producer assign sequence number = 4 now and Kafka think it’s a new message?

🧰 kafka producer client library handles:
- Sequence number for each partition

📡 Kafka broker handles only:
- PID
- Receives messages
- Tracks the producer ID + sequence number for each partition
- Detects duplicates based on those


## 🧑‍💻 On Producer Side (PID 123):

- 🧮 Keeps a sequence number counter per partition
- For partition 3, sends message 8 (M8) with seq = 3
- 🕐 Waits for ack
- ❌ Now, if the ack is lost, the producer does not increase the sequence number.
- 🔁 Instead, it retries sending the same message with the same seq = 3.

📡 Kafka sees:

- Producer ID = 123
- Partition = 2
- Sequence number = 3 (same as before)

🧠 Kafka checks its internal state:

🗂️ "I've already received a message from PID 123 for partition 2 with sequence 3"

👉 Duplicate detected → Kafka ignores it

✅ **Result: No duplicate message in topic**


## 🔁 And when does the producer increase the sequence number?

⏩ **Only after Kafka sends a successful ack.**


## 💥 What if Producer crashes and restarts?

- ♻️ Kafka gives it a new PID
- 🔢 Sequence number starts from 0
- 🔍 Kafka can detect this as a new session, and handle it properly


## 📊 Step Table

| 🪜 Step                          | 👷 Who Does It?                 | ❗ Why it matters                                 |
|-------------------------------|-------------------------------|--------------------------------------------------|
| 🆔 Assign producer ID (PID)     | 🖥️ Kafka broker                | Identifies unique producer instance              |
| 🔢 Assign sequence numbers      | 🧑‍💻 Kafka producer client (library) | Helps Kafka detect duplicates                    |
| 🧾 Detect duplicates            | 🖥️ Kafka broker                | Ensures exactly-once delivery if idempotence is enabled |
| ✍️ You add messages to topic     | 💻 Your Spring Boot app (KafkaTemplate) |                                                  |


---


# 🧾 Kafka Transactions

## ❓ What is a Transaction?

- 🧩 A transaction `groups multiple Kafka operations` to be `treated` as `one atomic unit`. 
- ✅ Either `all messages succeed` or `none are saved`. 
- 🎯 Helps `achieve exactly-once delivery` `when producing multiple messages`.

## 🤔 Why do we need Transactions?

📦 **Example:**

You have an order system that:

- 📝 Writes a message to the `order-topic` (new order placed)
- 💳 Writes another message to the `payment-topic` (payment info)

❌ If one message is stored but the other fails, your system gets out of sync!

🔐 **Transactions help avoid this problem** by making sure both messages are saved atomically — together or none at all.


## 🔑 How Transactions Work in Kafka

1. ▶️ Producer starts a transaction
2. ✉️ Producer sends multiple messages (to one or more topics/partitions)
3. ✅ Producer commits the transaction — all messages become visible atomically
4. ❌ Or aborts the transaction — none of the messages are saved

```bash

Topics:
order-topic -> Partition 0 -> Order message -1001
payment-topic -> Partition 1 -> Payment message -1001
```


### 🧪 Scenario:

- 🟢 Producer starts transaction
- 📤 Sends `order-1001` to `order-topic` topic partition 0
- 📤 Sends `payment-1001` to `payment-topic` topic partition 1
- ✅ Producer commits transaction

📌 **Result:** Both messages appear in their topics at the same time.

❌ **If failure happens before commit:**

- 🚫 Producer aborts transaction
- 🕳️ Neither message appears in Kafka

## 🧰 Use Cases for Kafka Transactions

- 🔗 **Atomic writes across multiple topics**  
   - Ensuring messages sent to different topics succeed or fail together (e.g., orders and payments topics).

- 🧵 **Atomic writes across multiple partitions within the same topic**  
   - Ensuring messages to different partitions of one topic are committed together to keep data consistent.

- ♻️ **Exactly-once stream processing**  
  - **Example:**  
    - A stream processing app reads from input topics, processes data, and writes to output topics.  
  - **Problem:** 
    - Duplicates or partial writes cause inconsistent downstream data.  
  - **Solution:** 
    - Wrap the entire processing and output writes inside a transaction so that results are committed atomically.

- 📡 **Distributed event coordination**  
   - Ensuring multiple related events sent to different topics or partitions are published together or not at all, maintaining system consistency.


## 🛠️ How to Use Transactions in Kafka Producer

| 🪜 Step             | 🧑‍💻 Kafka Producer (API)         |
|-------------------|------------------------------|
| ▶️ Start           | `producer.beginTransaction()` |
| ✉️ Send Messages   | `producer.send(record)`       |
| ✅ Commit          | `producer.commitTransaction()` |
| ❌ Abort (on failure) | `producer.abortTransaction()`  |

## 🛡️ Kafka Consumer Isolation Level: `read_committed`

- **🎯 Purpose**
   - Ensure that Kafka consumers only read `committed messages`, even if a producer fails or crashes mid-transaction.
- **🧠 Concept**
   - Kafka supports `transactions` to `write multiple records atomically`. If `a producer fails` `before completing a transaction`, `those partial records` `should not` be `visible to consumers`.
- By default, `consumers might read` `these uncommitted messages` `unless configured properly`.


## 🔒 Solution: `read_committed` Isolation

### ✅ With `read_committed`
- Consumers `only read committed messages`
- Uncommitted or in-progress messages are `hidden`
- Ensures `exactly-once semantics` (EOS)

### ⚠️ Without it (default: `read_uncommitted`)
- Consumers might read `uncommitted` or `partial` transactions
- Risk of `dirty reads`, duplicates, or `inconsistent state`

## ⚙️ How to Configure
- In your **consumer application**, set the following in `application.properties` (or YAML equivalent):

```properties
spring.kafka.consumer.properties.isolation.level=read_committed

```

## 🧪 How to Use Transactions in Spring Boot KafkaTemplate


- 🧪 Setting `transaction-id-prefix` `automatically enables transaction support` in KafkaTemplate. 
- 🔁 Use `KafkaTemplate.executeInTransaction(...)` to `send messages transactionally`.
- add isolation level in `consumer side` as `read_committed`


🧰 Enable Transactions in application.yml or application.properties
```bash


spring:
  kafka:
    producer:
      transaction-id-prefix: txn-       # 🛡️ Enables transactional support for KafkaTemplate
    listener:
      ack-mode: RECORD                  # 📝 Ensures proper message acknowledgment for transactions
    consumer:
      isolation-level: read_committed   # 🔒 Reads only committed (non-aborted) messages
      
```
🧰 Spring Kafka simplifies transactions with `executeInTransaction` method.
```bash

kafkaTemplate.executeInTransaction(kt -> {
    kt.send("orders", orderKey, orderValue);
    kt.send("payments", paymentKey, paymentValue);
    return true; // commits transaction if no exceptions
});
```

- ▶️ `executeInTransaction` starts the transaction
- 🧾 Runs your send logic
- ✅ Commits automatically if no exception
- ❌ If an exception happens, transaction aborts automatically

### 🔁 Return Value Matrix

| 🧾 Return Value | ✅ No Exception               | ❌ Exception Thrown              |
|----------------|------------------------------|----------------------------------|
| `true`         | Transaction committed automatically | Transaction rolled back automatically |
| `false`        | Transaction rolled back automatically | Transaction rolled back automatically |



## 🔄 Step Comparison Table

| 🪜 Step               | 🧑‍💻 KafkaProducer       | 💻 KafkaTemplate (Spring Kafka)   |
|----------------------|-------------------------|----------------------------------|
| ▶️ Start transaction  | `beginTransaction()`     | `executeInTransaction()`         |
| ✉️ Send messages      | `send(...)`              | `send(...)` inside lambda        |
| ✅ Commit transaction | `commitTransaction()`    | Automatic if no exception        |
| ❌ Abort transaction  | `abortTransaction()`     | Automatic if exception thrown    |


## 🧠 Key Points

- 🧱 Transactions ensure **atomicity**: all messages inside either get committed or none.
- 🧩 Used to avoid partial updates when producing messages to multiple topics/partitions.
- ⚙️ Requires producer config `transactional.id` or `transaction-id-prefix` in Spring.
- 🔄 Transactions work together with **idempotent producers** for exactly-once semantics.

---

## 🔄 Kafka Transactions and Idempotent Producer

### 1. Idempotent Producer and Transactions Relationship

- ⚙️ `Transactions` `require` `idempotent producer to work`. 
- Setting `transactional.id` (or `transaction-id-prefix` in Spring Boot) enables transactions and automatically enables idempotence (`enable.idempotence=true`).


### 📌 What happens under the hood:

When you set `transactional.id` or `transaction-id-prefix`
  - ➡️ Kafka automatically enables idempotence (`enable.idempotence=true`).

### 🔐 Why is this required?

Because transactions and idempotence work together to achieve exactly-once semantics:

| 🏷️ Feature       | 🎯 Purpose                             |
|------------------|--------------------------------------|
| 🔁 Idempotence    | Prevents duplicate messages on retries |
| 🔒 Transactions   | Groups messages atomically (all or none) |



---


# 🔁 Retryable & Idempotent Consumers

## 1️⃣ What is the Problem?

- 🧠 When a Kafka consumer reads a message and starts processing it, failures can happen — due to:
   - ❌ Errors 
   - 💥 App crashes 
   - 💤 Downtime 
   - 🐞 Bugs
- 🔁 If the consumer retries the same message after a failure, it might process it more than once — which can lead to:
   - 📛 Duplicate operations 
   - ⚠️ Inconsistent state


## 🧵 Scenario Breakdown

📦 **Topic:** `order-topic`  
🧾 **Message:** `order-123`

### ✅ Step-by-step:

1. 1️⃣ A consumer reads the message `order-123` from `order-topic`. 
2. 2️⃣ It starts processing the message:
    - 🗃️ ✅ Updates the inventory database
    - 🌐 ✅ Calls an external API
3. 3️⃣ 💥 But before it can commit the offset back to Kafka, the consumer crashes or fails.


## ⚠️ The Core Issue

- 🕳️ Since Kafka didn’t get the offset commit, it assumes the message wasn’t processed —  
- 🧠 But in reality, it was — just not acknowledged.

📉 This leads to the same message being retried, and unless the processing is idempotent, it causes:
   - 🔁 Duplicate executions 
   - 🧟 Data inconsistencies 
   - 🚨 Business logic failures

## 🔁 What Happens Next:

- ♻️ The consumer restarts and reads the same message (`order-123`) again from Kafka.
- But:
   - 🗃️ The DB was already updated 
   - 🌐 The API was already called
- 📛 So, the message gets processed again, causing problems like:
   - ❌ Inventory reduced twice 
   - ❌ Duplicate API calls 
   - ❌ Wrong or inconsistent state

---

# 🔄 What is a Retryable Consumer?

A `Retryable Consumer` is a Kafka consumer that can `automatically retry processing a message` when `it fails`, without `crashing or re-reading the message from Kafka` from the beginning.

## 📚 Example:

- 🗂️ Topic: `order-topic`
- 📌 Partition 0: `{ "orderId": "123", "item": "Laptop", "price": 1000 }`

Consumer reads the message from `order-topic` and tries to process it (e.g., store it in DB), but the database is down.

#### 💥 What happens with a normal consumer?

- Crash 💥
- 🛑 Block / Stop consuming
   - The consumer keeps trying to process the same failed message again and again. 
   - It does not move on to the next message, because the offset is not committed. 
   - So it blocks the partition → no progress.

## 🤖 What a Retryable Consumer does smarter:

- 🔄 Tries to process the message
- ❌ If it fails, it moves the message to a retry topic
- ⏳ Waits (e.g., 10s, 1m, etc.) based on the delay time that we mentioned
- 🔁 Tries again later
- 🧾 Keeps track of retries
- 🚫 Gives up and moves it to a dead-letter-topic (DLT) after too many failures


## 💡 Why use a Retryable Consumer?

- Because things break:
   - 🌐 Network glitches
   - 💾 Temporary DB failure 
   - ⏳ Third-party API timeout

- A retryable consumer helps:
   - 📈 Improve reliability 
   - ⚙️ Keep processing other messages 
   - 🧯 Handle errors gracefully 
   - 🛡️ Handle temporary errors without crashing or blocking processing

## 📅 When to use it?

- Use a Retryable Consumer when:
  - ⚠️ `Your message processing` is `not 100% reliable`
  - 🔄 `You want to retry on transient errors` (e.g., DB down, API timeout)
  - 🚫 You don’t want to crash or block your Kafka consumer

## ⚠️ The duplicate update problem on retries

- When `your consumer retries` processing the `same message multiple times` (because the first attempts failed), and the processing involves something like:
  - 🗃️ Updating a database 
  - 🧾 Creating an order record 
  - 📧 Sending an email 
  - 💳 Charging a payment

- then `each retry` might do the same action again, `causing duplicate effects` like:
  - 🛑 Duplicate rows in DB 
  - 💸 Duplicate charges 
  - 📨 Multiple emails sent

## ❓ Why does this happen?

- Because Kafka retries mean:
   - ⛔ Message processing fails → retry → process the same message again
- If `your processing logic is not idempotent`, it `repeats side effects`.

## 🔁 Two Types of Retries

### 1. 1️⃣ Synchronous (in-memory) retries
   - ⏱️ `Happens immediately` `inside the consumer`
   - 🔄 Retries a few times (e.g. 3 attempts with 5s backoff)
   - ⚠️ `If it still fails` → `escalate` to `async retries`
- ❌ **Problem:** Blocks that Kafka partition while retrying!

---

### 2. 2️⃣ Asynchronous retries using retry topic (e.g. `order-topic-retry`)
   - 📤 `Pushes failed message` to a `retry topic`
   - 👥 `A separate consumer` processes it later (after delay)
   - 🆓 `Original consumer` is `free` to continue
- ✅ **Scales better, doesn't block partitions**

## 🔄 Retry topic flow

```bash

1. order-topic  
    ↓  
[Main Consumer tries to process]  
    ↓ (fails after in-memory retries)  
2. order-topic-retry  
    ↓ (processed by separate retry consumer)  
[Retry Consumer tries again]  
    ↓ (fails again)  
3. order-topic-dlt (Dead Letter Topic)

```
## ⚙️ Two Ways to Implement Retry Mechanism


### 1️⃣ Manual Implementation (Recommended Approach)

- 🛠️ `Manually send message` `after in-memory retries fail` to `a separate retry consumer class`
- 🔄 `Retry consumer` tries processing the message
- 🚨 If `retry consumer fails again`, `send message` to `dead letter consumer class` for handling

- How to handle these manually

   - 🔄 Implement `in-memory retry loop` `inside the consumer method`. 
   - 📤 If retries fail, `send a custom error message` to a `retry topic` with exception info. 
   - 👥 Have `a separate retry consumer` that reads the retry topic, processes the message, and if `it fails again`, `sends it to the Dead Letter Topic (DLT)` with error details. 
   - 📝 Use `manual offset commit` with `Acknowledgment.acknowledge()` after successful processing to control message acknowledgment precisely. 
   - ⚙️ Provides **fine-grained control** over retry logic, error handling, and offset management. 
   - 🛡️ Enables **integration with Kafka transactions** for exactly-once semantics if needed.

``` bash


`Main Consumer` (in-memory retries) (order-topic) :
Reads from order-topic, retries 3 times in memory, then pushes to order-topic-retry.

@Service
public class OrderConsumer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    private static final int MAX_RETRIES = 3;

    @KafkaListener(topics = "order-topic", groupId = "order-group")
    public void consume(
        @Payload String message,
        Acknowledgment acknowledgment
    ) {
        int attempts = 0;
        boolean success = false;

        while (attempts < MAX_RETRIES) {
            try {
                System.out.println("Processing order: " + message);
                // Simulate processing
                processOrder(message);
                acknowledgment.acknowledge(); // ✅ Commit offset only after success
                success = true;
                break;
            } catch (Exception e) {
                attempts++;
                System.out.println("Failed attempt " + attempts + " for message: " + message);
                try {
                    Thread.sleep(2000); // Wait before retry
                } catch (InterruptedException ignored) {}
            }
        }

        if (!success) {
            // Send to retry topic
            kafkaTemplate.send("order-topic-retry", message);
        }
    }

    private void processOrder(String message) {
        if (message.contains("fail")) {
            throw new RuntimeException("Simulated failure");
        }
        // Simulate order DB save
        System.out.println("✅ Order processed: " + message);
    }
}


`Retry Consumer` (order-topic-retry) :
If the message still fails here → send it to order-topic-dlt.

@Service
public class RetryConsumer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    private static final int MAX_RETRY_ATTEMPTS = 2;

    @KafkaListener(topics = "order-topic-retry", groupId = "order-retry-group")
    public void retryConsume(
        @Payload String message,
        Acknowledgment acknowledgment
    ) {
        int attempts = 0;
        boolean success = false;

        while (attempts < MAX_RETRY_ATTEMPTS) {
            try {
                System.out.println("🔁 Retrying order: " + message);
                processOrder(message);
                acknowledgment.acknowledge();
                success = true;
                break;
            } catch (Exception e) {
                attempts++;
                System.out.println("❌ Retry attempt " + attempts + " failed for: " + message);
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException ignored) {}
            }
        }

        if (!success) {
            // Send to dead letter topic
            kafkaTemplate.send("order-topic-dlt", message);
        }
    }

    private void processOrder(String message) {
        if (message.contains("fail")) {
            throw new RuntimeException("Still failing in retry");
        }
        System.out.println("✅ Retry processed: " + message);
    }
}

`Dead Letter Consumer` (DLT) (order-topic-dlt)

This consumer handles messages that failed all retries

@Service
public class DeadLetterConsumer {

    @KafkaListener(topics = "order-topic-dlt", groupId = "dlt-group")
    public void handleDeadLetter(@Payload String message) {
        System.out.println("🚨 Dead letter message received: " + message);
        // Notify team, alert, save to DB, etc.
    }
}



```


## 2️⃣ Using `@RetryableTopic` Annotation
   
   - 🚨 Your `@KafkaListener` method throws an exception if processing fails. \
   - 🔄 Spring Kafka’s `@RetryableTopic` infrastructure catches the exception and:

      - 🔁 Retries **in-memory first** (retry attempts happen inside the listener container)
     
      - 📤 If all in-memory retries fail, it **sends the message to the retry topic automatically** (like `order-topic-retry-0`)
     
      - 👂 Spring Kafka automatically creates a listener on the retry topic and processes the message again. 
      - ❌ If the retry topic processing fails again, Spring Kafka sends the message to the **dead-letter topic automatically**.

- 🚫 **No** `kafkaTemplate.send()` to retry topic or DLT
- 🚫 **No** manual offset commits or error handlers for retries
- 🛑 Just **throw exceptions for failures** — Spring Kafka handles routing and retries!

```bash

@RetryableTopic(
    attempts = 4, // total attempts: 1 main + 3 retries (2 in-memory + 1 async)
    backoff = @Backoff(delay = 5000), // retry delay = 5 seconds
    retryTopicSuffix = "-retry",
    dltTopicSuffix = "-dlt",
    autoCreateTopics = false
)
  
/** If you want to use your own retry topic names, disable auto-creation and create the topics yourself
Then you must create:

order-topic-retry
order-topic-dlt

You can use Kafka CLI or Spring Kafka NewTopic bean.

if not enable  auto-create-topic as true
*/
```

---

### ⚠️ Limitations of `@RetryableTopic` for Exactly-Once Semantics (EOS):

- 🔄 It uses **automatic offset commits**, which align with **at-least-once** delivery.
- ❌ It **doesn’t handle Kafka transactions** or atomic commits of offsets and side effects.
- 🛠️ You **lose fine-grained control** needed for EOS guarantees.



---

# 🧠 Idempotent Consumer in Kafka

An **Idempotent Consumer** is a system that can `receive the same message multiple times` but `only processes it once`.

> 🧮 In math:  
> `f(x) = f(f(x))` → Applying it once or multiple times = same result.

## 🔁 When does Kafka send the same message again?

Kafka can **redeliver a message** when:

- 💥 The consumer crashes
- 🔄 There's a rebalance
- ❌ Offset is not committed
- 🌐 Network issues


## 🕒 When to use Idempotent Consumers?

Use them whenever:

- 🔂 Messages `might be delivered more than once` (which happens often in real-world systems)
- ⚠️ You `can’t afford to do the same operation multiple times`  
  (like charging money, shipping products, sending emails, etc.)


## 🆔 ID Tracking Mechanism

Yes, the consumer needs to `create` and `manage` the `ID tracking` mechanism.

Because `only the consumer knows` which messages it has received and processed — so it’s the **consumer’s job** to:

- 🕵️‍♂️ Track processed message IDs (e.g., `orderId`)
- 🗂 Store them in a `reliable place` (like a database, cache, or external system)
- ✅ Check before processing: If already processed, **skip it**

## 🧑‍💻 Who creates this?

👉 **You (as the developer)** need to implement it.

- ❌ Spring Kafka does `not provide automatic idempotency` at the consumer level.
- ✅ Kafka **allows duplicate delivery** (especially in case of retries, crashes, network issues,rebalances, etc.)
- 🔧 So, **your code in the consumer** must manage message **deduplication**.


## 🛠 Options for Storing Processed IDs

| Option           | Description                              | ✅ Pros                   | ⚠️ Cons                        |
|------------------|------------------------------------------|---------------------------|--------------------------------|
| 🧠 In-memory Set  | Simple Java `Set` or `Map`               | Fast, easy                | Lost on restart; not for prod |
| 🗃 Database        | Store `orderId` in a SQL/NoSQL table     | Persistent, reliable      | Slower, adds DB dependency    |
| 🚀 Redis          | Fast, distributed key-value store        | Fast + persistent         | Requires Redis server         |
| 📦 Kafka log compaction | Use a compacted topic to track IDs | Native to Kafka           | More complex, advanced use    |


## 🔁 Idempotent Consumer Flow (with manual offset commit)

1. 📥 Kafka delivers a message to the consumer
2. 🔍 Consumer extracts unique ID (like `orderId`)
3. 🧾 Consumer checks a DB or Redis if `orderId` is already processed
4. ✅ If **not processed**:
  - 🔧 Process the message
  - 💾 Save `orderId` to DB
  - 📝 Commit the offset (manually if `auto-commit = false`)
5. 🚫 If **already processed**:
  - ⏭ Skip the message
  - 📝 Still commit offset (to avoid re-delivery)

```bash

Kafka -> Spring Consumer -> Extract orderId (Tracking ID/Message ID)
                        |
                        v
               Is orderId already in DB?
                    /         \
                Yes            No
                /               \
           Skip it         Process it
                              |
                              v
                      Save orderId to DB

```

---

