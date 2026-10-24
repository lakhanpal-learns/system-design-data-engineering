# pratical example 

# Our Company: QuickCart

Let's create a fictional e-commerce company similar to Amazon/Flipkart.

**Business**

**QuickCart has:**

Customers placing orders
Products and inventory
Sellers
Payments
Deliveries

**Suppose we grow to:**

10 million customers
1 million orders/day
100+ million events/day

**The company wants a Data Platform for:**

Business analytics
Fraud detection
ML
Supply-chain analytics
Dashboards

# Step 1 — Start with the actual business problem

Imagine QuickCart's CEO says:

"I want to know today's sales, inventory levels, failed payments, delivery performance, and customer behavior. The data should be available within 15 minutes."

Immediately, we have a system-design problem.

## Where does the data originate?
Customer Service
       ↓
Orders DB
       ↓
Payments DB
       ↓
Inventory DB
       ↓
Delivery DB

**So our first architectural decision is:**

Separate operational workloads from analytical workloads.

# step 2 Our first requirement

We need to take data from operational systems and move it into our analytical data platform.

Operational Systems
       ↓
   Data Platform
       ↓
 Analytics / ML / BI

 This is where our Data Engineering system begins.

## Our first question is simply:

How do we **get data OUT** of the production systems?

That's the Ingestion problem.

## Step 1A — We will solve ingestion first

Orders DB
    ↓
 ????
    ↓
Data Lake

**And we'll answer one question at a time:**

> What is our source?
> Batch or streaming?
> How do we capture inserts?
> How do we capture updates?
> How do we capture deletes?
> What happens if the pipeline crashes?
> How do we avoid duplicate data?
> How do we handle late data?
> Where do we store raw data?
> Then finally: why do we need Hudi?

## task 1 Orders DB → Data Lake

What exactly does QuickCart's Orders DB look like?

Before building an ingestion pipeline, a Data Engineer needs to understand the source.

1. Our Orders DB

Let's assume QuickCart uses PostgreSQL for its production Orders DB.

orders

order_id        BIGINT
customer_id     BIGINT
product_id      BIGINT
quantity        INT
amount          DECIMAL
status          VARCHAR
created_at      TIMESTAMP
updated_at      TIMESTAMP

| order_id | customer_id | product_id | quantity | amount | status | updated_at |
| -------: | ----------: | ---------: | -------: | -----: | ------ | ---------- |
|      101 |         501 |       9001 |        2 |   1000 | PLACED | 10:01      |
|      102 |         502 |       9002 |        1 |    500 | PLACED | 10:03      |
|      103 |         503 |       9003 |        3 |   1500 | PLACED | 10:05      |

## So our first major requirement becomes:

Capture only new or changed orders.

## What changes can happen?

This is where real system design begins.

An order can be:

**INSERT**

New order

101 → PLACED

**UPDATE**

101 → PLACED
       ↓
101 → SHIPPED

**DELETE**

Potentially:

101 → deleted

So our ingestion system needs to understand:

INSERT
UPDATE
DELETE

This is called **Change Data Capture (CDC).**

## version 01 
                  ┌─────────────┐
                  │ PostgreSQL  │
                  │  Orders DB  │
                  └──────┬──────┘
                         │
                         ↓
                       CDC
                         │
                         ↓
                      Kafka
                         │
                         ↓
                    Data Lake

## How CDC actually works

PostgreSQL maintains something called a Write-Ahead Log (WAL).

Very simplified:

Application
    ↓
PostgreSQL
    ↓
WAL

When an order changes:

UPDATE orders
SET status = 'SHIPPED'
WHERE order_id = 101;

PostgreSQL records that change in its WAL.

**A CDC system can read those changes.**

Conceptually:

PostgreSQL WAL

INSERT order 101
UPDATE order 101
UPDATE order 101
DELETE order 101
       ↓
   CDC Reader

**The CDC reader converts those database changes into events.**

{
  "operation": "UPDATE",
  "order_id": 101,
  "status": "SHIPPED",
  "updated_at": "2026-09-15T10:30:00"
}

**we get:**

500 million existing rows
        +
1 million changes
        ↓
CDC captures the 1 million changes

**That's the fundamental reason we use CDC.**


## Our first mini-system

┌─────────────────┐
│   PostgreSQL    │
│                 │
│ orders table    │
└────────┬────────┘
         │
         │ WAL
         ↓
┌─────────────────┐
│   CDC Reader    │
└────────┬────────┘
         │
         ↓
   Change Events

# step 3  design cdc 

## Business situation

QuickCart has:

500 million orders
1 million new/updated orders per hour

We need data available in the Data Lake within 15 minutes.

------------------------------------------------------------

there is a two option we have 
1. access the production DB 
2. instant forward the event when there is something chnage in the wal 

------------------------------------------------------------

## now lets explore it 

now consider:

10:00 → order updated
10:01 → CDC job starts
10:02 → order updated again

Or:

10:00:59 → record updated
10:01:00 → pipeline checks timestamp

> **Timestamp-based extraction** can create duplicates, missed changes, clock/timing problems, etc.
> More importantly, it puts repeated query **load on the production DB.**

>So at larger scale, we choose CDC from the database's transaction log.

## what change it 

**Suppose order 101 starts as:**

order_id = 101
status = PLACED
amount = ₹1000

Customer places the order.

PostgreSQL transaction:

**INSERT order 101**

**The database writes the transaction to its WAL.**

Our CDC system reads the WAL.

**It produces an event:**

{
  "operation": "INSERT",
  "order_id": 101,
  "status": "PLACED",
  "amount": 1000
}

**Later the order ships:**

UPDATE order 101
SET status = 'SHIPPED'

WAL records the change.

CDC produces:

{
  "operation": "UPDATE",
  "order_id": 101,
  "status": "SHIPPED"
}

## One important LLD problem: CDC must remember its position

Database WAL
     │
     ↓
CDC Reader
     │
     ├── Event 1 ✓
     ├── Event 2 ✓
     ├── Event 3 ✓
     ├── Event 4 ✓
     └── Event 5 ✓
             ↑
          checkpoint

--------------------------------------------------------

CDC Reader
     ↓
resume from checkpoint
     ↓
Event 6

-------------------------------------------------------

## So our CDC design currently looks like:

                 QuickCart
                     │
              Orders PostgreSQL
                     │
                     ↓
                    WAL
                     │
                     ↓
                CDC Reader
                     │
          ┌──────────┴──────────┐
          │                     │
      Change Events          Checkpoint
          │
          ↓
        Kafka

# step 4  design kafka

> Our CDC layer is producing changes from PostgreSQL. Now we need a system that can reliably transport and **buffer those changes before processing them.**

> Why do we need Kafka?

Normally:

1,000 events/sec

During a sale:

50,000 events/sec

if CDC directly sends everything to Flink:

CDC ─────────→ Flink
                 ↓
            overloaded ❌


Kafka acts as a **durable buffer:**

If Flink is temporarily slow, the data doesn't have to disappear. It can remain in Kafka until the consumer catches up.

> Kafka stores these events in a topic.

{
  "operation": "UPDATE",
  "order_id": 101,
  "status": "SHIPPED",
  "updated_at": "10:10:00"
}

## Topic → Partition

A Kafka topic is divided into partitions.

Because partitions allow Kafka to distribute data and allow consumers to process data in parallel.

             Kafka Topic
                 │
       ┌─────────┼─────────┐
       ↓         ↓         ↓
      P0        P1        P2
       │         │         │
    Consumer   Consumer  Consumer
       1          2         3

Now we can process multiple streams simultaneously.

## Critical decision: partition key
How do we decide which partition an order goes to?

We'll use:

order_id

Example:

Order 101 → P1
Order 102 → P0
Order 103 → P2
Order 104 → P1


**Because we care about ordering for the same order.**
By using order_id as the partition key, events for the s**ame order go to the same partition**, **preserving their order within that partition.**

Order 101
   ↓
PLACED
   ↓
SHIPPED
   ↓
DELIVERED

## Kafka doesn't give global ordering

> Kafka guarantees ordering within a partition, not across the entire topic.
> Each partition is ordered.

But there is no guarantee that:

A < X < M < B < Y...

has any global ordering.

> We don't need global ordering for QuickCart.

We mainly need:

Correct ordering for events belonging to the same order.

## What if a consumer crashes?

Suppose Flink processes:

Event 1 ✓
Event 2 ✓
Event 3 ✓

**Kafka maintains a consumer offset** representing the consumer's position.

This is similar to the checkpoint idea we discussed for CDC, but now we're dealing with Kafka consumer offsets

## What if Kafka itself loses a machine?

Kafka partitions can **have replicas**.

Partition 0

Broker 1 → Leader
Broker 2 → Replica
Broker 3 → Replica

If Broker 1 fails:

Broker 1 ❌

Broker 2
   ↓
becomes leader

So Kafka provides fault tolerance through replication.

## Our Kafka architecture so far
                    PostgreSQL
                        │
                        ↓
                       WAL
                        │
                        ↓
                       CDC
                        │
                        ↓
              ┌───────────────────┐
              │       Kafka       │
              │ orders.cdc topic  │
              └─────────┬─────────┘
                        │
             ┌──────────┼──────────┐
             ↓          ↓          ↓
            P0         P1         P2
             │          │          │
             └──────────┼──────────┘
                        ↓
                  Flink / Spark

## What we've designed

| Decision          | Our choice                  |
| ----------------- | --------------------------- |
| Kafka topic       | `quickcart.orders.cdc`      |
| Partitioning      | 3 partitions initially      |
| Partition key     | `order_id`                  |
| Why key?          | Preserve per-order ordering |
| Buffer            | Kafka                       |
| Consumer position | Kafka offset                |
| Fault tolerance   | Partition replication       |

# step 5 Flink vs Spark

> who consumes Kafka and writes to Hudi — Flink or Spark?

1. Flink — real-time path

Use Flink when QuickCart needs data within minutes/seconds.

Kafka
  ↓
Flink Streaming
  ↓
Hudi

For example:

Order 101 changes from PLACED → SHIPPED

Flink **continuously consumes that Kafka event and updates Hudi.**

**Use cases:**

Real-time inventory
Fraud detection
Order tracking
Live dashboards
Near-real-time ML features

2. Spark — batch path

Use Spark when processing large amounts of data together.

Kafka / Files
      ↓
Spark Batch
      ↓
Hudi

Every night, QuickCart processes 500 TB of historical order data for analytics.

Spark is suitable for:

Large ETL jobs
Backfills
Historical processing
Complex transformations
ML pipelines

## QuickCart architecture

So we don't choose Flink OR Spark.

                    ┌──→ Flink Streaming ──→ Hudi
                    │
PostgreSQL → CDC → Kafka
                    │
                    └──→ Spark Batch ───────→ Hudi

# step 6 Flink Streaming Layer 

What does Flink actually do?

For QuickCart, Flink has several responsibilities:

Read events from Kafka
Deserialize/parse the CDC event
Validate the event
Handle duplicates
Handle ordering
Handle late events
Transform/enrich data if needed
Write/upsert the result into Hudi
Checkpoint its progress so failures can recover

>**Flink processes this** and **Hudi updates the existing record.**

## What happens if Flink crashes after processing an event but before writing it to Hudi?

Processing Kafka events
          ↓
   Flink checkpoint
          ↓
       💥 crash
          ↓
   restore checkpoint
          ↓
    continue processing

Flink checkpointing periodically saves the processing state and source progress, allowing the job to recover from a consistent point after failure instead of restarting from the beginning.

## Duplicate Events in Flink
Now we handle another real production problem.

**Suppose Kafka contains:**

Order 101 → PLACED
Order 101 → PAID
Order 101 → SHIPPED

Flink processes PAID.

**Then something fails during processing.**

After recovery, the same event may be read again:

Order 101 → PAID

Now Flink has processed the same event twice.

This is called duplicate processing

### Why duplicates are dangerous

For some operations, duplicates can produce wrong results.

Example:

Order amount = ₹1,000

If we calculate revenue:

₹1,000
+ ₹1,000   ← duplicate
-----------
₹2,000 ❌

Actual revenue is ₹1,000.

### 
How do we handle it?

We give every event a unique identifier.

event_id = abc123
order_id = 101
status = PAID

Flink can maintain/check processed event IDs.

Incoming event
      ↓
Have we processed event_id abc123?
      ↓
 ┌────┴────┐
 YES       NO
 ↓          ↓
Ignore    Process
           ↓
         Hudi

This is called **deduplication.**

### But there's an even more important idea

For our CDC pipeline, we don't always need to rely only on event_id.

We can use the record's identity:

order_id = 101

**and its ordering/version information, such as:**

updated_at

or **a CDC sequence/LSN.**

order_id | status  | version
---------|---------|--------
101      | PLACED  | 10
101      | PAID    | 11
101      | SHIPPED | 12

**Why LSN / Sequence Order is Critical?**

When ingesting this event stream into a Data Lakehouse (e.g., Apache Iceberg, Apache Hudi, or Snowflake), events can arrive out of order due to network retries, parallel Kafka partitions, or worker restarts.

Duplicate events can occur because of retries or failure recovery. We handle them using unique event IDs, stateful deduplication, and record/version information so the same logical change isn't applied incorrectly multiple times.


# step 7 Event Ordering in Flink
Now suppose Kafka receives these events for the same order:

Order 101 → PLACED
Order 101 → PAID
Order 101 → SHIPPED

But because of delays/retries, Flink might see:

PLACED
SHIPPED
PAID

That creates a problem

If we process SHIPPED first, Hudi could temporarily show:

101 | SHIPPED

and then later:

101 | PAID

Now the older state has overwritten the newer state. ❌

> **important note :-** 
Kafka guarantees ordering within that partition.

## solution LSN or event version / sequence

order_id | status  | sequence
---------|---------|---------
101      | PLACED  | 1
101      | PAID    | 2
101      | SHIPPED | 3

> important note :- 

Kafka ordering and business ordering aren't exactly the same thing.

Kafka can preserve the order of messages in a partition, but your system may still need event timestamps/sequence numbers to determine the correct business state

## Late-Arriving Events

Now imagine a customer places an order on Monday, but because of a network/device issue, the event reaches our system on Tuesday.

Event time:       Monday 10:00
Arrival time:     Tuesday 09:00
                         ↑
                   23 hours late

### Why is this a problem?

Suppose our pipeline already calculated Monday's sales:

Monday sales = ₹10,00,000

Then a delayed order worth ₹5,000 arrives:

Monday actual sales = ₹10,05,000

If our system only processes "today's" incoming data, **Monday's result stays wrong.**


## 
Flink's job

Order happened:   Monday 10:00
Arrived at Flink: Tuesday 09:00

event_time      = Monday 10:00
processing_time = Tuesday 09:00

### What does Flink do with late data?

Kafka
  ↓
Flink
  ↓
Check event time
  ↓
Is event late?
  ├── No → normal processing
  │
  └── Yes → handle late event
              ↓
             Hudi

For our orders table, the late CDC event can update/upsert the affected order in Hudi.

For an aggregate, such as daily sales, we may need to recompute or adjust the affected day's result.

#### How Flink & Hudi Fix It (3 Simple Steps)

1. Flink Holds the Door Open (Allowed Lateness)

When the late order arrives on Tuesday, Flink wakes up Monday's calculation.

2. Flink Routes Very Late Data to a "Side Door" (Side Output)

3. Hudi Rewrites Only the Old File (Smart Upserts)

## Watermarks

Flink uses watermarks to tell the system roughly:

"I believe events before this point have mostly arrived."

Events:
10:01
10:02
10:05
10:03  ← arrived late

A watermark helps Flink decide when it is safe to finalize time-based calculations.

-----------------------------------------------------------------------------------

Kafka
  ↓
Flink
  ├── Checkpointing
  ├── Deduplication
  ├── Ordering
  ├── Event-time processing
  └── Late-event handling
          ↓
        Hudi
        
------------------------------------------------------------------------














