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











