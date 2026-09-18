# SCD Type 1 vs Type 2

When a customer's information changes, do you want to remember the old value or not?

SCD means Slowly Changing Dimension.

It is mainly used for dimension or master data where attributes can change over time.

## Example

Suppose a customer's city changes.

### Type 1: Forget the old value

Before:

```text
Customer 101 -> Delhi
```

After:

```text
Customer 101 -> Mumbai
```

The original Delhi value is overwritten.

### Type 2: Remember the history

```text
Customer 101 -> Delhi    -> valid until Jan 2026
Customer 101 -> Mumbai   -> valid from Feb 2026
```

Both records are retained.

## One example

Suppose your customer dimension initially contains:

| customer_id | name  | city   |
| --- | --- | --- |
| 101 | Rahul | Delhi |
| 102 | Priya | Mumbai |

Later, Rahul moves from Delhi to Bangalore.

### SCD Type 1

The table becomes:

| customer_id | name  | city |
| --- | --- | --- |
| 101 | Rahul | Bangalore |
| 102 | Priya | Mumbai |

The old Delhi value is gone.

Conceptually:

```text
Delhi
       |
       v
UPDATE
       |
       v
Bangalore
```

No history is kept.

### When to use Type 1

Use it when historical values are not important.

Examples:

- Customer email correction
- Typo in customer name
- Current phone number
- Current address where historical address is not required

### SCD Type 2

Now suppose the business says:

> We need to know where the customer lived when each historical transaction occurred.

Then simply overwriting Delhi would be a problem.

Instead, keep history:

| customer_id | name  | city | start_date | end_date | is_current |
| --- | --- | --- | --- | --- | --- |
| 101 | Rahul | Delhi | 2025-01-01 | 2026-01-31 | false |
| 101 | Rahul | Bangalore | 2026-02-01 | NULL | true |
| 102 | Priya | Mumbai | 2025-01-01 | NULL | true |

Now we know:

```text
Rahul
|-- Delhi
|   Jan 2025 -> Jan 2026
|
`-- Bangalore
              Feb 2026 -> current
```

## The key idea

When Rahul's city changes:

```text
Existing record
                     |
                     v
Close old record
                     |
                     v
Insert new record
```

So Type 2 generally involves UPDATE plus INSERT, rather than simply overwriting the row.

## The easiest mental model

### Type 1

Think:

> What is the customer's value now?

```text
101 -> Bangalore
```

Only the current state matters.

### Type 2

Think:

> What was the customer's value at each point in time?

```text
101 -> Delhi     -> historical
101 -> Bangalore -> current
```

History matters.

## Databricks example

Imagine your source table:

| customer_id | name  | city |
| --- | --- | --- |
| 101 | Rahul | Bangalore |
| 102 | Priya | Mumbai |

And your existing dimension:

| customer_id | name  | city | start_date | end_date | is_current |
| --- | --- | --- | --- | --- | --- |
| 101 | Rahul | Delhi | 2025-01-01 | 2026-01-31 | false |
| 101 | Rahul | Bangalore | 2026-02-01 | NULL | true |

With Type 2, when another change occurs:

```text
Bangalore -> Pune
```

You do not simply run:

```sql
UPDATE customer_dim
SET city = 'Pune'
WHERE customer_id = 101;
```

That would destroy the historical Bangalore information.

Instead, conceptually:

1. Close the Bangalore record.
        - end_date = today
        - is_current = false
2. Insert the Pune record.
        - start_date = today
        - end_date = NULL
        - is_current = true

Result:

| customer_id | city | start_date | end_date | is_current |
| --- | --- | --- | --- | --- |
| 101 | Delhi | 2025-01-01 | 2026-01-31 | false |
| 101 | Bangalore | 2026-02-01 | 2026-09-18 | false |
| 101 | Pune | 2026-09-18 | NULL | true |

## Databricks interview or exam perspective

A common scenario question is:

> A company wants to preserve the previous address of a customer whenever the address changes. Which SCD approach should be used?

Answer: SCD Type 2.

The requirement contains the keyword preserve history.

Another question:

> A company only needs the customer's latest email address and does not care about previous addresses.

Answer: SCD Type 1.

Because the old value can be overwritten.

## Side by side

| Feature | SCD Type 1 | SCD Type 2 |
| --- | --- | --- |
| Old value retained? | No | Yes |
| History? | No | Yes |
| Existing row | Overwritten | Closed |
| New row | Usually no | Inserted |
| Storage | Lower | Higher |
| Complexity | Simple | More complex |
| Typical columns | Business attributes | start_date, end_date, is_current |
| Example | Correct email typo | Customer moves city |
| Main question | What is current? | What was true when? |

## One sentence to remember

Type 1 overwrites the past. Type 2 preserves the past.

In Databricks, Delta Lake MERGE makes these patterns easier to implement, especially Type 1. Type 2 requires maintaining the old version and inserting the new version with appropriate validity information.