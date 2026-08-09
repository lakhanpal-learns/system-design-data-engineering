# Building Data Pipelines At Facebook - How To Manage An Exabyte Data Warehouse

<!-- data engineering at meta high level overview of the internal tech stack (meta blog) -->

#  problem 

# problem 1 
Sure. The problem is basically **data is spread across different physical locations**.

Imagine Facebook's warehouse is not one building. It is like:

**India data center** → some tables
**USA data center** → some tables
**Europe data center** → some tables

Now suppose a query needs data from different places.

If the data is scattered randomly, the system has to **move huge amounts of data between locations**. That is slow and expensive.

### Facebook's solution: Namespaces

They group related tables together in the **same location**.

For example:

```text
Namespace: User Data
   ├── users
   ├── user_profiles
   └── user_activity

Namespace: Ads Data
   ├── ads
   ├── ad_clicks
   └── ad_impressions
```

So if a query needs **User Data**, the related tables are already close together.

### Simple idea

❌ **Without namespaces**

`Data A (USA) ←→ Data B (Europe) ←→ Data C (India)`

Lots of data movement → **slow**

✅ **With namespaces**

`User tables → USA`

`Ads tables → Europe`

Related data stays together → **less data movement → faster queries**

### The key problem

> **How do we organize a huge distributed warehouse so that queries don't have to constantly move massive amounts of data between different locations?**

That's the important system-design idea here: **physical + logical partitioning to reduce data movement.**


# additonal 01 
![Distributed Database Architecture Diagram](image.png)

## problems 
### Problem 1: Primary region failure 💥

**Problem:** If the primary region goes down, the data/service could become unavailable.

**Solution:** Keep a replicated copy of the shard in a **secondary region**. If the primary fails, the secondary can take over.

---

### Problem 2: Users are far away from the primary region 🌍

**Problem:** Users in Europe or Asia would have to communicate with a database in the US, causing higher latency.

**Solution:** Keep replicas in different geographic regions so users can access data **closer to them**.

---

### Problem 3: Too many read requests 🚦

**Problem:** Millions of users may request the same data. Sending every request directly to MySQL would overload it.

**Solution:** Use **caches and followers** to distribute read traffic and reduce the load on MySQL.

---

### Problem 4: Database reads are slow 🐌

**Problem:** Going to MySQL for every request takes more time.

**Solution:** Store frequently requested data in **cache**, allowing fast reads.

---

### Problem 5: Cache contains old data ❌

**Problem:** When data changes in MySQL, the old value may still exist in the cache.

**Solution:** **Invalidate/update the cache** when a write happens so users don't receive stale data.

---

### Problem 6: One region cannot handle everything 📈

**Problem:** Facebook operates globally with enormous traffic. One region shouldn't have to handle all requests.

**Solution:** Distribute data and traffic across **multiple regions, replicas, caches, and followers**.

---

### 🎯 The main idea

**Problem:** Facebook needs to serve enormous global traffic while keeping data available and fast.

**Solution:**
**Replication + multiple regions + caching + followers**

That's the core architecture.

# additonal 02 
![alt text](image-1.png)

## problem 01 
This image is showing a very common **data architecture problem**:

> **In theory, you want one central data platform. In reality, data gets scattered across different systems.**

### 1. In an ideal world

Imagine the company has one central **Apache Iceberg** data platform.

Inside it:

```text
                 ICEBERG
        ┌─────────────────────┐
        │ Sales │ HR │ Marketing│
        │ Ops   │ Core │ Data   │
        └─────────────────────┘
```

All departments put their data into the **same platform**.

Then different tools can access that same data:

* Google BigQuery
* Trino
* Snowflake
* Databricks

So you have:

> **One data storage layer → many tools can use it.**

That's the dream.

---

### 2. In reality 😅

The data doesn't stay in one place.

Instead:

```text
Iceberg
 ├── Sales
 └── Marketing

Snowflake
 └── HR

BigQuery
 └── Operations

PostgreSQL
 └── European Sales
```

Now the company has **data silos**.

For example, suppose you want:

> **Total sales + HR employee information + European operations**

You may need to query:

```text
Iceberg
   +
Snowflake
   +
BigQuery
   +
PostgreSQL
```

That's much more complicated.

---

### Why does this happen?

Different teams have different needs.

For example:

**HR team** might already use Snowflake.

**Operations team** might have data in BigQuery.

**European team** might have an old PostgreSQL database.

**Data team** might use Iceberg/Databricks.

Nobody wants to immediately migrate everything.

So over time:

```text
New team → new tool
Old system → stays
Acquisition → brings another system
Business requirement → another database
```

And eventually you get **data fragmentation**.

---

### 🎯 The actual system-design problem

**Problem:**

> Data is spread across multiple storage systems and platforms, making it difficult to access, combine, govern, and analyze data consistently.

**Ideal solution:**

> Have a common data layer, such as Iceberg, where data is organized centrally while different compute/query engines can access it.

### The important concept for you

This image is really teaching:

**Centralized data layer vs. data silos**

## system desgin question 

And this becomes an important **system-design trade-off**:

> **How do we allow different teams and tools to work independently without creating a mess of disconnected data?**

Bro, the answer is basically **“separate storage from compute.”** This is one of the most important ideas in modern data architecture.

### Problem

Different teams want to use different tools:

* Analytics → Snowflake
* Data engineering → Databricks
* SQL users → Trino
* Other workloads → BigQuery

If every tool has its **own copy of the data**, you get:

> duplicate data + inconsistent data + expensive movement + difficult governance.

### Solution

Create **one common data/storage layer** and let different tools access it.

For example:

```text
                DATA STORAGE
             Apache Iceberg
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
     Trino      Databricks   Snowflake
        ↓           ↓           ↓
      Team A      Team B      Team C
```

The key is:

> **Teams can choose their own compute/query tool, but they don't need to create their own separate copy of the data.**

### But how?

Apache Iceberg acts as the **common table/storage layer**.

It manages things like:

* Where the data files are
* Table schema
* Partitions
* Metadata
* Table versions
* Transactions

Then different engines can understand and work with those tables.

So:

```text
             One Data Layer
                  ↓
      ┌───────────┼───────────┐
      ↓           ↓           ↓
    Trino     Databricks   Snowflake
```

### The important architecture principle

**Don't force every team to use the same compute engine.**

Instead:

> **Standardize the data layer, but allow flexibility in the compute layer.**

That's how you get both:

**Team independence + centralized data management.**

This is exactly why technologies like **Apache Iceberg** are important in modern data platforms.


# 
