# Apache Hudi™ at Uber: Engineering for Trillion-Record-Scale Data Lake Operations
# Article 1 - Section 1

The Foundation of Uber's Data Platform

## 1 Business Problem

Before discussing technology, Uber explains why they need a data platform.

Think about Uber's businesses:

🚗 Ride booking
🍔 Uber Eats
📦 Delivery
📍 Maps
📢 Ads
💳 Payments
🛡 Safety
🤖 Machine Learning

Every one of these continuously produces data.

## 2. Where does the data come from?

The article says:

Hundreds of microservices

Ride Service
Payment Service
Maps Service
Pricing Service
Notification Service
Fraud Service
Ads Service
Driver Service
Customer Service
...

**Each service generates data independently.**

**This is a classic microservice architecture.**

## 3 Scale

Uber mentions:

Thousands of cities
Millions of riders
Continuous stream of data

Imagine one ride.

Ride Requested
↓

Driver Assigned
↓

Driver Accepted
↓

GPS every few seconds
↓

Fare Updated

↓

Payment

↓

Rating

↓

Receipt

↓

Support Events

<!-- One ride may generate hundreds of events.

Now multiply that by millions of rides every day.

That is why they say an unending stream of data. -->

## 4 What is the Data Lake?

Uber says

Multi-hundred-petabyte repository

----------------------------------------

All company data

↓

Stored in one huge storage system

↓

Accessible for analytics

↓

Machine Learning

↓

Dashboards

↓

Experiments

↓

Reports

----------------------------------

<!-- Notice something important.

A Data Lake is not a database.

It is the central storage where raw and processed data live. -->

## 5. What uses this data?

The article lists:

Operational decisions
Machine Learning
Experimentation
Business Intelligence

Meaning one dataset is consumed by many teams.

### Example
Trips Dataset

↓

Pricing Team

↓

ML Team

↓

Finance

↓

Fraud Detection

↓

Dashboard Team

↓

Marketing

<!-- One dataset serves many consumers. -->

## 6 The real engineering challenge

Most beginners think:
<!-- 
"Just store lots of data." -->

Uber says that's not enough.

They mention **four major problems.**

### 6.1 Constant Mutation
Data changes after it's written.

Example

Ride Status

Requested

↓

Accepted

↓

Started

↓

Completed

The same record keeps changing.


#### problem (old datalake problem )
When a customer changes their phone number or address, you aren't allowed to use an eraser. Because of how the notebook was bound, every time you want to update one single line on a page, you have to throw away the whole page and rewrite all 1,000 names on it from scratch.

**Why Data Lakes Suffer From This**
In data engineering, a Data Lake is like that giant notebook. Data is stored in large files called Parquet files, which are built to be read very fast, but cannot be edited line by line.

When database updates keep pouring in constantly (orders placed, user profiles updated, items canceled):

**Option A (Rewrite everything):** You spend a fortune on computing power rewriting giant data files just to change a few rows.

**Option B (Write tiny sticky notes):** Every time something changes, you create a tiny new file. Soon, you have millions of tiny files, and searching through them takes forever.

#### solution 
Engineers use modern table tools (like Apache Hudi, Apache Iceberg, or Delta Lake) that act like sticky-note management systems:

**The "Sticky Note" Trick (Merge-on-Read):** When a record changes, the system quickly writes a small "sticky note" saying "Hey, Row 45 was updated to X." When someone runs a report, the system reads the main notebook page and applies the sticky notes on the fly.

**Nightly Cleanup (Compaction):** At night, when traffic is low, an automated process takes all the sticky notes and neatly rewrites the notebook pages once, keeping everything tidy.




### 6.2 High Cardinality

Millions or billions of unique values.

Examples

Trip ID

Customer ID

Driver ID

Restaurant ID

GPS Coordinates

**Every record is different.**

Indexes become huge.

Queries become harder.


### Problem 3 — Fast-changing Schem

**Today**

Trip

Fare

**Tomorrow**

Trip

Fare

Discount

Weather

Surge Multiplier

**Next month**

Carbon Emission

Electric Vehicle

Delivery Mode

The structure evolves continuously.


### 6.3 Freshness

This is probably the biggest requirement.

Uber says

Minutes, not hours.

**Why?**

Imagine surge pricing.

If data arrives 2 hours late,

Pricing is wrong.

Drivers are sent incorrectly.

Customers wait longer.

Revenue decreases.

Freshness directly affects the business.

## 7 exiting datalake fail
This is the key sentence.

Uber says

Existing technologies could not handle

huge scale
constant updates
schema evolution
fast freshness

This created a gap.

## 8 Their Solution

pache Hudi

Notice something important.

They didn't create Hudi because they wanted another tool.

They created it because their business requirements couldn't be met by existing technology.

**This is exactly how system design starts.**

Business Problem

↓

Requirements

↓

Limitations

↓

Design

↓

Technology

## 9 Hudi introduces Database features to Data Lakes
This sentence is huge.

They mention:

ACID Transactions
Indexing
Incremental Processing

We'll spend separate lessons on each because they're core system design concepts.

## 10 HLD 
From just this introduction, we can already infer a simplified HLD:

                  Users
                     │
                     ▼
           Uber Applications
                     │
                     ▼
         Hundreds of Microservices
                     │
          Continuous Event Streams
                     │
                     ▼
              Data Ingestion
                     │
                     ▼
      Hudi-powered Data Lake
                     │
      ┌────────┬────────┬────────┐
      ▼        ▼        ▼        ▼
 Machine     BI      Analytics   ML
 Learning  Dashboards  Teams    Models

 Key takeaways
**Business drives architecture**. Uber's massive, real-time operations created requirements that existing tools couldn't satisfy.
**A data lake is shared infrastructure.** Many teams depend on the same underlying data.
**Scale alone isn't the hardest problem.** Continuous updates, evolving schemas, and low-latency freshness make the system much more challenging.
**Hudi was a response to specific engineering problems,** not an end goal.




# section 2 
## System Design Thinking
They ask:

"What problem are we trying to solve?"


## What changed in 2015?
Uber says:

Our data systems were expanding faster than off-the-shelf technology.

Business Growth
      ↓
More users
More cities
More products
More microservices
      ↓
More data
      ↓
Existing tools cannot keep up

## problem 
### Problem 1 — New Events
Ride Started
Ride Completed
Ride Cancelled
Driver Offline
Driver Online
Payment Failed
Coupon Applied

**Every new feature creates new events.**

### problem 2 — Backfills
What is a backfill?

Imagine yesterday's ETL job failed.

Monday ❌

Tuesday ✅

Wednesday ✅

Now you need to process Monday's missing data.

That is called a backfill.

Without good system design, backfills become slow and expensive.

### problem 3 late arive data 

Suppose a driver's phone loses internet.

10:00 Ride Completed

↓

No Network

↓

10:10 Internet Returns

↓

Event reaches server

<!-- The event arrives 10 minutes late.

A good data platform must still process it correctly. -->

### problem 4 — Schema Evolution
Yesterday

Trip
Fare

Today

Trip
Fare
Discount

Next month

Trip
Fare
Discount
Weather

The system must keep working without rewriting everything.

## Why Traditional Data Lakes Failed

Uber says data lakes were:

File-based
Append-only
Batch-oriented

Let's understand each

### File-based
Data was stored as files.
trip_001.parquet

trip_002.parquet

trip_003.parquet

**But updating one row means rewriting files.**

### apend only 
Append-only

Imagine a ride.

Ride ID = 101

Status = Requested

Later:

Status = Accepted

Append-only systems don't overwrite.

They keep adding:

Ride 101 Requested

Ride 101 Accepted

Ride 101 Started

Ride 101 Completed

Now one ride exists four times.

**Finding the latest version becomes harder.**

### Batch-oriented

Every few hours

Read everything

↓

Process everything

↓

Write everything

Even if only 100 rows changed.

Very inefficient.

## uber requirements 

### Mutable Data

Data changes after it's written.

Requested

↓

Accepted

↓

Started

↓

Completed

**The storage system must support updates efficiently**.

### High Update Rate

Millions of updates every minute.

Think of GPS updates, order status changes, payments, etc

### Incremental Processing

This is one of the biggest ideas in Data Engineering.

Instead of:

Read 100 TB

Read only:

500 MB changed today

This saves:

Time
CPU
Memory
Money

### Big Requirement
Uber writes:

We needed database performance without losing the flexibility and scalability of a data lake.

This sentence explains why Hudi exists.

| Database          | Data Lake             |
| ----------------- | --------------------- |
| Fast updates      | Massive storage       |
| ACID transactions | Cheap and scalable    |
| Indexes           | Flexible file formats |
| Great for OLTP    | Great for analytics   |

## Hudi's Three Superpowers

Instead of thinking of Hudi as software, think of it as adding capabilities to a data lake

### ACID Transactions

If two jobs update the same data simultaneously, the table remains consistent.

**What happens WITHOUT ACID (The Old Data Lake Problem)**

In a traditional file-based data lake, both jobs try to update the driver's data file at the exact same moment.

Job A opens the file (reads balance: $100), adds $15, and writes a new file with $115.

Job B opens the same file at the exact same time (reads balance: $100), subtracts $5, and writes a new file with $95.

Whichever job finishes writing last overwrites the other. If Job B finishes last, the passenger's $15 tip vanishes into thin air! The driver now has $95 instead of $110.

This is called a Race Condition or Data Corruption. In traditional data lakes, to avoid this, engineers had to lock entire folders or prevent multiple systems from touching the same data at once, slowing everything down.

### Indexing + Fast Upserts

Without an index:

Need to find Trip 123?

↓

Search every file

With an index:

Trip 123

↓

Jump directly to the correct file

Much faster.

### Incremental Processing

Instead of:

Yesterday

↓

Read entire lake

Use:

Read only changes since last successful run

This is a huge optimization for ETL pipelines.

### The Last Paragraph  🔴
Uber says they later built:

Metadata Table
Record Index
Multi-data-center sync
Optimized table services

**These are advanced scaling features.** We don't need to understand them yet. The important point is that Hudi itself had to evolve as Uber continued to grow.

## High-Level Design (HLD) from this section
                Microservices
                      │
         Millions of Events & Updates
                      │
                Data Ingestion
                      │
                      ▼
             Apache Hudi Data Lake
      ┌────────────┼──────────────┐
      │            │              │
  ACID Txns     Indexing     Incremental Reads
      │            │              │
      └────────────┼──────────────┘
                   ▼
          Analytics / ML / BI


# section 3 Data at Uber Scale

Uber is saying:

"Hudi isn't special because it exists. It's special because it works reliably at one of the largest scales in the world."

## Different Businesses, Different Data
Uber has many businesses:
Uber Rides
Uber Eats
Uber Freight
Maps
Ads
Pricing
Safety
ML

Each generates data differently.

| Business | Data Type     | Speed        |
| -------- | ------------- | ------------ |
| Rides    | GPS, Trips    | Seconds      |
| Eats     | Orders        | Minutes      |
| Ads      | Clicks        | Milliseconds |
| Maps     | Locations     | Continuous   |
| ML       | Training Data | Daily        |

Lesson: One platform must support many different workloads.

## Hudi is the Center of the Platform

Microservices
      │
      ▼
 Ingestion
      │
      ▼
   Apache Hudi
      │
 ┌────┼─────┐
 ▼    ▼     ▼
 BI   ML   Analytics

 Hudi becomes the single source of truth for many teams.

 ## Understanding the Numbers

 19,500 datasets

Means:

Thousands of tables
Different teams
Different schemas

**6 trillion rows/day**

Means:

Massive daily data growth
Storage and processing must scale automatically

**350 PB storage**

Millions of files

↓

Thousands of folders

↓

Hundreds of petabytes

Finding data efficiently becomes difficult.

**10 PB ingested/day**

Meaning:

Every day, Uber receives 10 petabytes of new data.

The ingestion pipeline can never stop.

350,000 commits/day

A commit means data is successfully written.

**350,000/day means:**

Every few seconds

↓

Thousands of successful writes

The system must remain consistent even with constant writes.

**70,000 table services/day**

These background tasks keep the data lake healthy.

Examples:

Cleaning old files
Compaction
Clustering

Think of them as maintenance jobs.

**4 million queries/week**

Many users and applications read the data simultaneously.

The storage layer must support both:

Heavy writes
Heavy reads

**400+ billion-row tables**

Some tables never stop growing.


## Important Sentence

Every workload has its own latency requirement.

Fraud Detection

↓

Needs seconds

----------------

ML Training

↓

Can wait hours

----------------

Monthly Report

↓

Can wait days

## Biggest Engineering Lesson

Uber says:

**We invented new operational patterns.**

Meaning:

Existing systems couldn't solve every problem.

So Uber created:

Better indexes
Better metadata
Better maintenance jobs

This is how large companies innovate.



# intreview question 

# section 1 
## Why wasn't a traditional data lake sufficient for Uber?

A strong answer would mention:

Frequent updates (mutations)
Large-scale datasets (trillions of records)
Evolving schemas
Need for data freshness within minutes
Requirement for database-like guarantees (such as ACID transactions) on top of scalable data lake storage

# section 2 
1. Why are append-only data lakes inefficient for frequently updated data?

Answer:
Because every update creates a new record instead of modifying the existing one. This increases storage usage and makes queries slower because the system must determine the latest version.

2. What is a backfill, and why should a data platform support it?

Answer:
A backfill is reprocessing historical or missed data after a failure or delay. The platform should support it to recover missing data and maintain data accuracy

3. What is late-arriving data? How would you handle it?

Answer:
Late-arriving data is data that reaches the system after its expected time (for example, due to network issues). Design the pipeline to allow updates to existing records instead of ignoring late data.

4. Why is incremental processing better than full-table recomputation?

Answer:
Incremental processing reads only changed data, reducing processing time, compute resources, and cost.

5. Why would a company want database-like guarantees while using a data lake?

Answer:
To get ACID transactions, reliable updates, and data consistency while still benefiting from the scalability and low cost of a data lake.

6. Uber has a 100 TB table, but only 500 MB changed today. What would you do?

Answer:
Read only the 500 MB of changed data because it's much faster, cheaper, and avoids unnecessary processing.

7. Why couldn't traditional data lakes meet Uber's needs?

Answer:
They were file-based, append-only, and batch-oriented, making frequent updates and real-time processing difficult.

8. What business challenges led Uber to build Hudi?

Answer:
Rapid growth, increasing data volume, frequent updates, evolving schemas, and the need for near real-time analytics.

9. What is an upsert?

Answer:
An upsert updates an existing record if it exists; otherwise, it inserts a new record.

10. Why is schema evolution important?

Answer:
It allows new columns or changes to the data structure without breaking existing pipeline

11. Why are indexes important in large datasets?

Answer:
Indexes help locate records quickly, avoiding full-table scans and improving query and update performance.

12. What is the biggest lesson from this section?

Answer:
Business requirements drive technology decisions. Uber built Hudi because existing solutions couldn't satisfy its scale, freshness, and update requirements.




# section 3 


