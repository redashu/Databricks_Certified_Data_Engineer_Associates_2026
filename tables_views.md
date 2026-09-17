# Tables, Views, and Materialized Views in Databricks

A clean way to remember the main types is:

| Type | Stores data physically? | Recomputes query when queried? | Typical use |
| --- | --- | --- | --- |
| View | No | Yes | Reusable SQL logic |
| Temporary View | No | Yes | Within the current Spark session |
| Global Temporary View | No | Yes | Shared within a Spark application |
| Materialized View | Yes, maintains result | Usually no | BI, aggregations, performance |
| Delta Table | Yes | N/A | Persistent data storage |
| Streaming Table | Yes | Incrementally maintained | Streaming and incremental pipelines |

## 1. Regular View

A normal SQL view is essentially a saved query.

```sql
CREATE VIEW customer_orders AS
SELECT
    customer_id,
    COUNT(*) AS order_count
FROM orders
GROUP BY customer_id;
```

Think of it like this:

```text
VIEW
  │
  └── saved SQL definition
           │
           ▼
      underlying tables
```

It generally does not store a separate copy of the result.

When someone runs:

```sql
SELECT * FROM customer_orders;
```

the underlying query is evaluated against the source data.

Use case: reusable business logic.

## 2. Temporary View

Created with:

```python
df.createOrReplaceTempView("orders_temp")
```

or:

```sql
CREATE TEMP VIEW orders_temp AS
SELECT * FROM orders;
```

### Important characteristic

It is session-scoped. Once the Spark session ends, the temporary view disappears.

This is a common pattern in notebooks:

```text
Notebook
   ↓
Create intermediate dataframe
   ↓
Temporary View
   ↓
Run SQL against it
```

This is very common when mixing PySpark and SQL.

## 3. Global Temporary View

You may also encounter:

```python
df.createOrReplaceGlobalTempView("orders")
```

It can be referenced using:

```sql
SELECT *
FROM global_temp.orders;
```

The important distinction is that it is available across sessions within the Spark application, but it is still temporary and not a persistent Unity Catalog object.

For modern Databricks work, you will generally see temporary views and persistent Unity Catalog views more often than global temporary views.

## 4. Materialized View

This is the interesting one.

```sql
CREATE MATERIALIZED VIEW daily_sales AS
SELECT
    store_id,
    DATE(order_date) AS order_date,
    SUM(amount) AS sales
FROM orders
GROUP BY store_id, DATE(order_date);
```

### Conceptually

A regular view is:

```text
Query
  ↓
Underlying data
  ↓
Calculate result
  ↓
Return result
```

A materialized view is:

```text
Underlying data
       ↓
  Compute result
       ↓
  Store/maintain result
       ↓
   Query MV
       ↓
 Faster access to result
```

This is particularly useful for expensive aggregations and BI workloads.

## 5. View vs Materialized View vs Table

This is probably the most important distinction for Databricks students:

```text
                    DATA OBJECTS
                         │
          ┌──────────────┼──────────────┐
          │              │              │
        TABLE           VIEW       MATERIALIZED VIEW
          │              │              │
      Stores data    Stores SQL     Maintains
                     definition     query result
          │              │              │
          ▼              ▼              ▼
       Physical       No separate    Precomputed/
        data           result data     maintained data
```

### Example

Suppose you have `orders` with 10 billion rows.

And you frequently need:

```sql
SELECT
    store_id,
    SUM(amount)
FROM orders
GROUP BY store_id;
```

A view means:

> Save this query so I do not have to write it again.

A materialized view means:

> Maintain the result of this query so users do not have to recalculate this expensive aggregation.

A table means:

> Store this dataset as a persistent data asset.

## Databricks terminology trap ⚠️

Do not confuse:

- SQL View
- Streaming Table
- Materialized View

They are different concepts.

For example, in a Lakeflow Declarative Pipeline you might have:

```text
          Raw files / Kafka
                  │
                  ▼
        ┌───────────────────┐
        │ Streaming Table   │
        │     BRONZE        │
        └───────────────────┘
                  │
                  ▼
        ┌───────────────────┐
        │ Streaming Table   │
        │     SILVER        │
        └───────────────────┘
                  │
                  ▼
        ┌───────────────────┐
        │ Materialized View │
        │      GOLD         │
        └───────────────────┘
```

But you could also have:

```text
Bronze Delta Table
       ↓
Silver Delta Table
       ↓
Gold Delta Table
```

or:

```text
Bronze Streaming Table
       ↓
Silver Streaming Table
       ↓
Gold Materialized View
```

The architecture does not force one particular object type for each medallion layer. The choice depends on the processing pattern and requirements.