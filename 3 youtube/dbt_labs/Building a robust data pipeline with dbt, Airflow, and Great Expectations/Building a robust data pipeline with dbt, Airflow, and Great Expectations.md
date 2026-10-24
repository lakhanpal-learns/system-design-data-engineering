
# Building a Robust Data Pipeline with dbt, Airflow, and Great Expectations

## 1. Video Information

- **Video:** [Building a Robust Data Pipeline with dbt, Airflow, and Great Expectations](https://youtu.be/9iN6iw7Lamo)
- **Speaker:** Sam Bail
- **Topic:** Data Engineering, Data Quality, Pipeline Orchestration
- **Main tools:** dbt, Apache Airflow, Great Expectations
- **Level:** Beginner to Intermediate

### Learning Objectives

After studying this presentation, I should understand:

1. Why data quality is important in data engineering.
2. How dbt transforms data using SQL.
3. How Apache Airflow orchestrates data pipelines.
4. How Great Expectations validates data.
5. How these tools work together.
6. Where validation should occur in a pipeline.
7. How to handle pipeline failures caused by bad data.
8. How to design a reliable data pipeline instead of merely moving data.

> Note: These notes are based on the presentation slides and associated materials. They are structured study notes, not a verbatim transcript of every spoken sentence.

---

## 2. The Main Problem: Data Pipelines Can Produce Incorrect Results

A data pipeline can execute successfully while producing incorrect data.

Consider a company that receives transaction records from an application.

The pipeline performs these steps:

1. Extract transactions from the source.
2. Load them into a database.
3. Transform the records using SQL.
4. Generate analytical tables.
5. Display the results in a dashboard.

Suppose the source contains these records:

| transaction_id | amount | customer_id |
|---|---:|---|
| TX001 | 500 | C101 |
| TX002 | 800 | C102 |
| TX002 | 800 | C102 |
| TX003 | -500 | C103 |
| TX004 | NULL | C104 |

Potential problems:

- TX002 appears twice.
- TX003 has a negative amount that may violate the business rules.
- TX004 has a missing amount.

The pipeline might complete without any software exception. However, the output could still be incorrect.

For example, duplicate transactions can inflate total revenue.

### Key lesson

**A successful pipeline execution does not automatically mean that the resulting data is correct.**

Data engineers need to validate data at different stages of the pipeline.

Important quality dimensions include:

- Completeness: Are required values present?
- Uniqueness: Are duplicate records present?
- Validity: Do values follow expected rules?
- Consistency: Do related datasets agree?
- Freshness: Has the data arrived on time?
- Accuracy: Does the data correctly represent the underlying reality?

A technical validation can detect many problems, but it cannot prove every value is factually accurate without a trusted reference or business rule.

---

## 3. Understanding the dAG Stack

The presentation combines three tools:

- **dbt:** Data transformation and transformation testing.
- **Apache Airflow:** Workflow orchestration.
- **Great Expectations:** Data validation and quality reporting.

The name "dAG stack" refers to dbt, Airflow, and Great Expectations.

Each tool has a different responsibility.

### 3.1 dbt — Data Transformation

dbt stands for data build tool.

It allows engineers to organize SQL transformations into reusable models.

For example, suppose a raw transactions table contains:

- transaction_id
- customer_id
- amount
- transaction_timestamp

We want to create a clean model for analytics.

Example SQL model:

```sql
SELECT
    transaction_id,
    customer_id,
    amount,
    transaction_timestamp
FROM raw_transactions
WHERE transaction_id IS NOT NULL;
```

This query filters records without transaction IDs.

In a dbt project, the query can be stored in a model file such as:

`models/staging/stg_transactions.sql`

dbt can then build the model in the target data platform.

### What dbt provides

- SQL-based transformation models.
- Dependencies between models.
- Reusable SQL logic.
- Built-in data tests.
- Custom SQL tests.
- Documentation and model lineage.

### Example dbt tests

A model might require:

- transaction_id must be unique.
- transaction_id must not be null.
- customer_id must not be null.
- customer_id must reference an existing customer.

Example YAML configuration:

```yaml
version: 2

models:
  - name: stg_transactions
    columns:
      - name: transaction_id
        data_tests:
          - not_null
          - unique

      - name: customer_id
        data_tests:
          - not_null
```

This example uses the current-style `data_tests` configuration. Older dbt projects may use `tests`.

The exact supported tests depend on the dbt version and project configuration.

### Important distinction

Filtering invalid records is not the same as reporting that they are invalid.

If the pipeline silently removes bad records, engineers may lose visibility into source-data problems.

A robust system should decide whether to reject, quarantine, repair, or accept each record according to documented business rules.

---

## 4. Apache Airflow — Workflow Orchestration

Apache Airflow schedules and coordinates pipeline tasks.

It represents workflows as DAGs.

DAG stands for Directed Acyclic Graph.

A DAG defines:

- Tasks.
- Dependencies between tasks.
- Execution order.
- Scheduling.
- Retry behavior.
- Failure states.

Example workflow:

```text
Load Source Data
       |
       v
Validate Source
       |
       v
Run dbt Models
       |
       v
Run dbt Tests
       |
       v
Publish Analytics Data
```

A downstream task should not run until its required upstream tasks have completed successfully.

For example, analytics data should not be published if a critical validation step fails.

### Why orchestration matters

Imagine that a pipeline contains five separate Python scripts.

Without orchestration, engineers might have to:

- Execute the scripts manually.
- Remember the correct order.
- Inspect logs individually.
- Rerun failed scripts themselves.
- Determine whether a previous task completed.

Airflow manages these dependencies and execution states.

### Airflow DAG example

The following illustrates task dependencies. It is a simplified example rather than a complete production DAG.

```python
from airflow.sdk import dag, task
from datetime import datetime


@dag(
    dag_id="transaction_quality_pipeline",
    start_date=datetime(2025, 1, 1),
    schedule=None,
    catchup=False,
)
def transaction_quality_pipeline():

    @task
    def load_data():
        print("Loading transaction data")

    @task
    def validate_source():
        print("Validating source data")

    @task
    def run_transformations():
        print("Running transformations")

    @task
    def run_tests():
        print("Testing transformed data")

    @task
    def publish_results():
        print("Publishing analytics data")

    load = load_data()
    validate = validate_source()
    transform = run_transformations()
    test = run_tests()
    publish = publish_results()

    load >> validate >> transform >> test >> publish


transaction_quality_pipeline()
```

This example requires a compatible Airflow installation and does not implement actual database operations.

### Important concept

Airflow coordinates the work. It does not automatically make the underlying transformations or validation rules correct.

The actual logic still needs to be implemented and tested.

---

## 5. Great Expectations — Data Validation

Great Expectations is an open-source framework for expressing and checking expectations about data.

An expectation is a rule that describes what the data should satisfy.

Examples:

- A column must not contain null values.
- Values must be within an acceptable range.
- A field must contain unique values.
- Data must follow an expected schema.
- A dataset must satisfy a business constraint.

The presentation uses an example in which passenger counts must be between 1 and 6.

Conceptually, the rule is:

```python
expect_column_values_to_be_between(
    column="passenger_count",
    min_value=1,
    max_value=6,
)
```

This illustrates the older Great Expectations API used in the presentation. Current APIs and configuration differ by version, so this snippet should be treated as a conceptual example unless the corresponding version is installed.

### Example business rule

Suppose a taxi trip dataset contains a passenger_count column.

An acceptable rule could be:

- Passenger count must be at least 1.
- Passenger count must not exceed the chosen upper limit for the dataset.

A record containing a passenger count of 0 would fail the rule.

However, the acceptable upper limit should come from the dataset's schema or documented business requirements, not from an arbitrary assumption.

### How expectations are created

A typical workflow is:

1. Inspect historical data.
2. Profile the data to understand its values and distributions.
3. Consult business requirements.
4. Define expectations.
5. Validate new data against those expectations.
6. Inspect validation results.
7. Investigate unexpected failures.
8. Update expectations only when the underlying requirements justify the change.

### Why historical profiling is useful

Suppose a dataset normally contains between 10,000 and 15,000 records per daily load.

A new load contains only 200 records.

A record-count check could identify an unexpected drop.

However, historical behavior alone does not establish the correct business rule. A genuine holiday or source-system change might explain the difference.

An alert should trigger investigation rather than automatically assume that the data is wrong.

---

## 6. How dbt and Great Expectations Differ

Both tools support data quality, but their strengths differ.

### dbt

Best suited to testing transformed data models inside the analytical data platform.

Examples:

- Unique transaction identifiers.
- Required customer identifiers.
- Referential integrity.
- Custom SQL assertions.
- Transformation correctness.

### Great Expectations

Can validate different data assets, including database tables, files, and in-memory data, depending on the configured execution environment and supported integrations.

Examples:

- Source file schema validation.
- Expected column presence.
- Numeric range validation.
- Record-count anomalies.
- Distribution checks.
- Validation reporting.

### Comparison

| Capability | dbt | Great Expectations |
|---|---|---|
| SQL transformations | Core purpose | Not its main purpose |
| Tests on warehouse models | Yes | Can validate data assets |
| Custom SQL tests | Yes | Different validation approach |
| File validation | Not its primary purpose | Supported use case |
| Data quality reporting | Test results and documentation | Validation results and data documentation |
| Workflow scheduling | External orchestrator commonly used | External orchestrator commonly used |

### Key lesson

These tools can complement each other.

Using both is not automatically necessary for every project. Select them based on the kinds of data assets and validation requirements involved.

---

## 7. Where Should Data Quality Checks Run?

One of the most important ideas in the presentation is that validation can occur at multiple points in a pipeline.

### Stage 1 — Validate Source Data

Before transforming data, verify that the source has the expected structure.

Possible checks:

- Required columns exist.
- Data types are correct.
- Row counts are within expected bounds.
- Required values are present.
- Source data is not unexpectedly empty.

Example:

```text
Expected source columns:
transaction_id
customer_id
amount
transaction_timestamp

Actual source columns:
transaction_id
customer_id
amount
```

The timestamp column is missing.

The pipeline should identify this schema change before downstream models fail or produce misleading output.

### Stage 2 — Validate Data Loading

After ingestion, check whether the load completed correctly.

Possible checks:

- Compare source and destination row counts.
- Verify that expected records were loaded.
- Identify rejected records.
- Check whether duplicate ingestion occurred.
- Confirm that the load is associated with the correct batch.

For example:

```text
Source records:       10,000
Loaded records:        9,800
Rejected records:       200
```

This may be correct if the 200 rejected records are documented and quarantined.

But if 200 records disappeared without explanation, the load needs investigation.

### Stage 3 — Test Transformations

During development, test that transformations implement the intended logic.

Examples:

- Revenue calculations are correct.
- Joins do not unexpectedly multiply records.
- Aggregations produce the expected totals.
- Required columns are populated.
- Business rules are applied consistently.

A join can produce more rows than expected when the join key is not unique on one side.

This is commonly called a fan-out join.

### Stage 4 — Validate Final Analytical Tables

After transformations, check whether the final output is suitable for analytics.

Examples:

- Primary keys are unique.
- Foreign keys reference valid dimension records.
- Metrics reconcile against trusted source totals.
- Required business fields are present.
- Data freshness is within the defined service expectation.

### Key lesson

Testing only the final table can make it difficult to identify where a problem entered the pipeline.

Checks at critical boundaries help narrow the investigation.

---

## 8. Integrating dbt with Airflow

The presentation describes different ways to represent dbt work inside an Airflow DAG.

### Approach A — One Airflow Task Runs the dbt Pipeline

```text
Airflow
   |
   v
Run Entire dbt Project
   |
   v
All dbt Models and Tests
```

Advantages:

- Simpler orchestration.
- Fewer Airflow tasks.
- Easier initial setup.

Disadvantages:

- The dbt execution can appear as one large task.
- Individual model-level retries and reruns may be harder to manage through Airflow.

### Approach B — One Airflow Task per dbt Model

```text
Airflow DAG
    |
    v
stg_customers
    |
    v
stg_transactions
    |
    v
fct_transactions
    |
    v
Run Tests
```

Advantages:

- More granular execution visibility.
- Individual task failure states.
- More control over reruns and dependencies.

Disadvantages:

- More orchestration complexity.
- More tasks to manage.
- Additional metadata parsing and execution overhead may be involved.

### Which approach should I choose?

For a small project, start with one Airflow task that invokes dbt.

Move to model-level orchestration only if granular scheduling, observability, or retry requirements justify the additional complexity.

The goal is to solve a real engineering problem, not to maximize the number of tasks.

---

## 9. What Does a Robust Pipeline Look Like?

A simplified architecture:

```text
            SOURCE SYSTEM
                  |
                  v
          INGESTION / LOAD
                  |
                  v
       SOURCE DATA VALIDATION
          Great Expectations
                  |
                  v
          DATA TRANSFORMATION
                  dbt
                  |
                  v
       TRANSFORMATION TESTS
                  dbt
                  |
                  v
       FINAL QUALITY VALIDATION
                  |
                  v
          ANALYTICS TABLES
                  |
                  v
            BI DASHBOARD
```

Apache Airflow coordinates the execution of the steps and their dependencies.

A production-oriented implementation should also persist execution metadata and validation results.

For example:

- Pipeline run ID.
- Source batch ID.
- Start and end timestamps.
- Rows received.
- Rows accepted.
- Rows rejected.
- Checks executed.
- Failed check names.
- Overall execution status.
- Error details.

This information makes it possible to investigate incidents instead of relying on terminal output alone.

---

## 10. How Should the Pipeline Respond to Failed Validation?

Not every failure should be handled identically.

### Critical failures

Examples:

- A required source column disappears.
- The transaction identifier is missing from a large portion of the dataset.
- A critical reconciliation check fails.
- The source data cannot be read correctly.

Possible response:

1. Stop the affected downstream tasks.
2. Record the validation result.
3. Preserve the source data for investigation when appropriate.
4. Alert the responsible engineer.
5. Fix the underlying issue.
6. Rerun the affected tasks safely.

### Non-critical failures

Examples:

- A small number of optional descriptions are missing.
- An informational metric changes slightly.
- A non-critical field contains an unusual but permitted value.

Possible response:

1. Record the issue.
2. Continue only if the documented policy permits it.
3. Report the issue.
4. Monitor the trend.

### Important distinction

A failed data-quality check is not the same as a software exception.

The validation system can execute correctly and report that the data violates a rule.

The orchestration layer must then decide whether the failed check should block downstream work.

---

## 11. Important Data Engineering Concepts

### Data profiling

Examining data to understand its structure, value distributions, null rates, and anomalies.

### Data validation

Checking whether data satisfies defined rules.

### Data testing

Verifying that data models and transformations behave as expected.

### Data documentation

Recording schemas, models, expectations, and validation results in a form that engineers can inspect.

### Workflow orchestration

Managing dependencies, scheduling, retries, and task execution.

### Data lineage

Understanding how source data flows through transformations into downstream models.

### Idempotency

Ensuring that repeating an operation does not unintentionally duplicate its effects.

For example, rerunning a transaction batch should not insert the same transactions twice.

### Reconciliation

Comparing source and destination records or totals to establish whether data has been transferred correctly.

### Schema drift

A change in the structure of incoming data, such as a renamed column, missing field, or changed data type.

### Fan-out join

A join that unexpectedly multiplies rows because one or both join keys contain multiple matching records.

---

## 12. Applying These Lessons to My Project 3

### Project Name

Data Quality & Observability Platform

### Objective

Build a platform that detects data-quality problems, records validation results, prevents unreliable data from reaching analytics, and helps engineers investigate pipeline failures.

### Proposed technology stack

- Python: ingestion and custom validation logic.
- PostgreSQL: raw data, curated data, and validation history.
- dbt: SQL transformations and warehouse tests.
- Great Expectations: source-data and dataset validation.
- Apache Airflow: scheduling and orchestration.
- Streamlit: quality monitoring dashboard.
- pytest: unit and integration tests.
- GitHub Actions: automated test execution.

These tools should be added incrementally. A working Python/PostgreSQL validation engine is the first milestone.

### Example pipeline

```text
Public Dataset
      |
      v
Python Ingestion
      |
      v
PostgreSQL Raw Layer
      |
      v
Source Validation
      |
      +---- Invalid records
      |          |
      |          v
      |      Quarantine
      |
      v
dbt Transformations
      |
      v
dbt Tests
      |
      v
Curated Tables
      |
      v
Quality Dashboard
```

Airflow can be added to coordinate these tasks after the underlying logic works.

### Suggested validation rules

1. Transaction IDs must not be null.
2. Transaction IDs must be unique in the relevant business scope.
3. Required customer identifiers must be present.
4. Amounts must satisfy documented business rules.
5. Timestamps must be parseable and within the permitted range.
6. Referential integrity must be maintained where applicable.
7. Source and destination counts must reconcile.
8. Freshness must satisfy the defined expectation.
9. Unexpected schema changes must be detected.

The exact rules must reflect the selected dataset. For example, a taxi trip dataset has different valid values from a banking transaction dataset.

### Dashboard metrics

- Total records processed.
- Valid records.
- Rejected records.
- Quality pass rate.
- Failed rules.
- Latest successful run.
- Data freshness.
- Pipeline execution duration.
- Historical failure trends.

Define each metric explicitly so that the dashboard cannot silently confuse rejected rows, failed checks, and failed pipeline runs.

---

## 13. Suggested Repository Structure

```text
data-quality-observability/
|
|-- config/
|   |-- quality_rules.yaml
|
|-- data/
|   |-- sample/
|
|-- ingestion/
|   |-- load_data.py
|
|-- quality/
|   |-- schema_checks.py
|   |-- validation_engine.py
|   |-- quarantine.py
|
|-- dbt/
|   |-- models/
|   |-- tests/
|
|-- dags/
|   |-- quality_pipeline.py
|
|-- dashboard/
|   |-- app.py
|
|-- tests/
|   |-- test_validation_engine.py
|   |-- test_quarantine.py
|
|-- docs/
|   |-- architecture.md
|   |-- quality_rules.md
|   |-- failure_recovery.md
|
|-- docker-compose.yml
|-- requirements.txt
|-- README.md
```

This is a proposed structure for our implementation, not the exact directory structure from the video.

---

# 14. Interview Questions and Answers


## Q1. Why is data quality important?

Data quality means ensuring that data is correct, complete, and usable before it reaches reports or business applications.

### Example

Suppose a bank receives these transactions:

| Transaction ID | Amount |
|---|---:|
| T101 | 500 |
| T102 | 800 |
| T102 | 800 |
| T103 | 200 |

The actual total should be ₹1,500, but because T102 is duplicated, the pipeline calculates ₹2,300.

The pipeline may run successfully, but the result is wrong.

We can detect duplicates using SQL:

```sql
SELECT transaction_id, COUNT(*)
FROM transactions
GROUP BY transaction_id
HAVING COUNT(*) > 1;
```

**Interview answer:** Data quality prevents incorrect, missing, or duplicated data from affecting business decisions. We use validation rules and automated tests to detect problems before they reach dashboards or reports.

---

## Q2. What is the difference between dbt and Airflow?

**dbt transforms data; Airflow manages the workflow.**

### Example

Suppose we need to process daily sales data.

1. Load sales data into PostgreSQL.
2. Calculate total sales for each customer.
3. Check whether the results are valid.
4. Publish the final table.

**dbt** performs step 2 by running SQL transformations and can test the resulting table.

**Airflow** coordinates the steps and makes sure the transformation runs after the data has been loaded.

```text
Load Data
    |
    v
Run dbt Transformation
    |
    v
Run Tests
    |
    v
Publish Results
```

Airflow controls the sequence. dbt performs the transformation and its associated tests.

**Interview answer:** dbt is mainly used for SQL transformations and data testing, while Airflow orchestrates tasks, schedules workflows, and manages dependencies and retries.

---

## Q3. What is Great Expectations used for?

Great Expectations is a data quality framework used to define rules about what data should look like and check whether the data follows those rules.

### Example

Suppose a taxi dataset has a `passenger_count` column.

Our business rule says the passenger count must be between 1 and 6.

| Trip ID | Passenger Count | Result |
|---|---:|---|
| TR001 | 2 | Pass |
| TR002 | 4 | Pass |
| TR003 | 0 | Fail |
| TR004 | 9 | Fail |

Great Expectations can validate the passenger-count rule and report the records that violate it, depending on the validation configuration.

**Interview answer:** Great Expectations helps detect data quality problems by validating data against defined expectations, such as required columns, valid ranges, and missing values.

---

## Q4. Why use dbt and Great Expectations together?

Both tools can check data quality, but they are useful for different needs.

- **dbt:** Particularly useful for testing transformed warehouse models.
- **Great Expectations:** Useful for validating datasets, including source data and files, as well as applying more detailed validation rules.

### Example

A company receives a CSV file containing sales transactions.

**Great Expectations** checks whether the file has the expected columns and whether sales amounts follow the required rules.

After loading the data, **dbt** transforms it into a customer sales summary and tests whether customer IDs are unique and required fields are populated.

```text
CSV File
   |
   v
Great Expectations
Validate Source Data
   |
   v
PostgreSQL
   |
   v
dbt
Transform and Test
   |
   v
Sales Dashboard
```

**Interview answer:** We can use Great Expectations for source-data validation and dbt for transformation and warehouse tests. Using both is useful when the project needs both capabilities, but it is not mandatory for every pipeline.

---

## Q5. What is a DAG in Airflow?

DAG stands for **Directed Acyclic Graph**. It represents tasks and the dependencies between them.

In simple terms, it tells Airflow which task must run before another task.

### Example

Suppose a daily sales pipeline has three tasks:

```text
Extract Data
     |
     v
Transform Data
     |
     v
Validate Data
```

The transformation should not begin until extraction finishes. Validation should not begin until transformation finishes.

Airflow uses the DAG to coordinate this execution order.

A DAG has no circular dependency. For example, task A cannot depend on task B if task B ultimately depends on task A.

**Interview answer:** A DAG defines tasks and their execution dependencies in Airflow. It helps coordinate pipeline execution and ensures dependent tasks run in the correct order.

---

## Q6. What happens when a critical data validation fails?

A critical validation failure should prevent unsafe data from reaching downstream systems.

The pipeline should record the failure and follow a defined recovery process.

### Example

Suppose a sales pipeline expects a column called `sale_amount`, but the incoming file contains `amount`.

The validation fails because the expected schema has changed.

The correct response is:

1. Record the schema validation failure.
2. Stop the affected downstream tasks.
3. Alert the responsible engineer.
4. Investigate whether the source schema changed intentionally.
5. Update the pipeline only after confirming the correct schema.
6. Rerun the pipeline safely.

We should not simply ignore the error and publish incomplete results.

**Interview answer:** When a critical validation fails, we record the error, stop affected downstream processing, and alert the responsible engineer. After fixing the underlying issue, we rerun the pipeline and verify the results.

---

## Q7. What is the difference between data testing and data validation?

The terms overlap, but we can distinguish them by their purpose.

- **Data validation:** Checks whether data satisfies defined rules.
- **Data testing:** Checks whether data models, transformations, and expected behavior are correct.

### Example

Suppose a sales dataset contains an `amount` column.

**Validation:** Check that the amount is not null and follows the permitted range.

**Testing:** Check that the transformation calculates total sales correctly.

For example:

Input sales:

| Sale ID | Amount |
|---|---:|
| S1 | 100 |
| S2 | 200 |

Expected total: ₹300.

If the transformation returns ₹600, the test should fail.

**Interview answer:** Validation checks whether data meets defined quality rules, while testing verifies that transformations and data models behave as expected. In practice, the two concepts overlap.

---

## Q8. Why not use only dbt?

dbt may be sufficient when our quality requirements mainly concern SQL transformations and warehouse models.

A separate validation framework can help when we need to validate source files, inspect incoming data before loading, or implement validation workflows that are not conveniently handled by dbt alone.

### Example

Suppose a pipeline receives a CSV file every morning.

Before loading it, we want to check:

- Are all required columns present?
- Is the file empty?
- Are timestamps valid?
- Are amounts within the permitted range?

After loading, we want to transform the data and test our warehouse models.

We might use Great Expectations for the source file and dbt for the warehouse models.

**Interview answer:** We do not always need both tools. I would start with dbt tests if they meet the requirements, and add a separate validation framework when source-data checks or additional validation capabilities justify it.

---

## Q9. What is data lineage?

Data lineage shows where data comes from, how it is transformed, and which downstream tables or reports depend on it.

It helps engineers investigate problems and understand the impact of changes.

### Example

A revenue dashboard displays an incorrect total.

The data lineage might show:

```text
Sales API
    |
    v
Raw Sales Table
    |
    v
Clean Sales Model
    |
    v
Daily Revenue Table
    |
    v
Revenue Dashboard
```

An engineer can trace the incorrect result back through the transformations and investigate where the problem started.

For example, duplicate records may have entered the raw table or a join may have multiplied rows.

**Interview answer:** Data lineage tracks the flow of data from its source through transformations to its final destination. It helps with debugging, impact analysis, and understanding dependencies.

---

## Q10. How can you prove your data quality platform works?

We should demonstrate that the platform handles both valid data and deliberately introduced problems correctly.

### Example

Suppose our platform processes 1,000 sales records.

Initially, all 1,000 records pass validation.

Next, we deliberately introduce:

- Five duplicate transaction IDs.
- Three missing customer IDs.
- Two invalid amounts.

We rerun the pipeline and verify that the appropriate checks fail.

The platform should record the failed rules and handle invalid records according to the defined policy.

For example:

```text
Records received:       1,000
Valid records:             990
Rejected records:           10
Failed validation rules:     3
Pipeline status:          FAILED
```

These figures are illustrative. The actual number of failed checks can differ from the number of rejected records because one record may violate multiple rules.

After correcting the problems, we rerun the pipeline and verify that:

- Valid records are loaded correctly.
- Invalid records are handled as expected.
- Reruns do not create duplicates.
- The validation results show the successful recovery.

**Interview answer:** I would test the platform using valid data and deliberately corrupted data. I would verify that the expected rules detect the problems, invalid records are handled correctly, failures are logged, and the pipeline can recover safely without duplicating records.


# 15. Practical Exercises

### Exercise 1 — Basic validation

Create a transactions table with 100 records.

Introduce:

- Five duplicate transaction IDs.
- Three missing customer IDs.
- Two invalid timestamps.

Write checks that identify the deliberately introduced problems.

### Exercise 2 — dbt tests

Create a staging model and add:

- A uniqueness test.
- A not-null test.
- A relationship test.

Run the tests and inspect the results.

### Exercise 3 — Airflow orchestration

Create a DAG that runs:

1. Data loading.
2. Source validation.
3. dbt transformation.
4. dbt tests.

Configure dependencies so that a failed critical validation blocks downstream tasks.

### Exercise 4 — Failure recovery

Cause a validation failure deliberately.

Then:

1. Inspect the run metadata.
2. Identify the failed rule.
3. Correct the data or rule, depending on the cause.
4. Rerun safely.
5. Confirm that accepted records are not duplicated.

### Exercise 5 — Monitoring

Store validation results in PostgreSQL.

Build a Streamlit dashboard showing:

- Run history.
- Quality pass rate.
- Failed checks.
- Rejected records.
- Freshness.
- Execution duration.

---

## 16. Final Revision Notes

Remember these core ideas:

- **dbt transforms data and tests analytical models.**
- **Airflow orchestrates tasks and manages workflow dependencies.**
- **Great Expectations validates data against defined expectations.**
- **Validation should happen at critical stages of the pipeline.**
- **A successful execution does not prove data correctness.**
- **Critical validation failures should block unsafe downstream work.**
- **Data-quality failures need recorded results and actionable diagnostics.**
- **Historical profiling helps establish expectations, but business context is necessary to interpret anomalies.**
- **Safe retries and idempotency prevent reruns from creating duplicate effects.**
- **A monitoring dashboard should display persisted, well-defined metrics.**

### Official and Supporting Resources

1. [Original YouTube video](https://youtu.be/9iN6iw7Lamo)
2. [Airflow Summit presentation and slides](https://airflowsummit.org/sessions/2021/building-a-robust-data-pipeline-with-the-dag-stack/)
3. [Great Expectations presentation materials](https://greatexpectations.io/blog/dbt-coalesce-2020-building-a-robust-data-pipeline-with-dbt-airflow-and-great/)
4. [O'Reilly course overview](https://www.oreilly.com/live-events/build-a-robust-data-pipeline-with-airflow-dbt-and-great-expectationswith-interactivity/0636920058631/)

**Main takeaway:** Data engineering is not only about moving and transforming data. Reliable engineering also requires proving that the data is fit for its intended use, detecting failures, and recovering without corrupting downstream results.
