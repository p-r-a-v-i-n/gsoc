# GSoC 2026 Proposal: Add `generate_series()` support to `contrib.postgres`

## 1. Introduction and Motivation

PostgreSQL's set-returning functions (SRFs), such as `generate_series()`, are powerful tools for building complex reports, filling gaps in time-series data, or generating numeric sequences. 

 while many developers have used raw SQL or third-party packages to achieve this, there is a clear need for a native, first-class implementation within the ORM. Relying on "phantom models" or raw SQL fragments often leads to brittle code that breaks standard QuerySet features like `.count()` or manual aggregation.

My goal this summer is to implement `generate_series()` in `contrib.postgres` in a way that feels native to Django, ensuring it is robust, well-tested, and consistent with the ORM’s architecture.

## 2. Technical Research (Background)

I have researched the current state of set-returning functions in Django by examining recent discussions among core developers and studying the source code.

### 2.1. Foundation: The `set_returning` Flag
I have studied the work in **[PR #18412](https://github.com/django/django/pull/18412)**, authored by [**Devin Cox**](https://github.com/devin13cox) and [**Simon Charette**](https://github.com/charettes), which added support for set-returning database functions. This PR introduced the `set_returning = False` attribute to the `BaseExpression` class. By following the logic in `django/db/models/sql/query.py`, I learned how this flag ensures that the SQL compiler handles rows correctly when a function might multiply the result set.

My implementation will build directly on this foundation. I plan to use this flag to help the compiler distinguish between scalar expressions and expressions that should be treated as table sources in the `FROM` clause.

### 2.2. A Generic `Relation` API
Core contributors like [**Simon Charette**](https://github.com/charettes) and [**Adam Johnson**](https://github.com/adamchainz) have discussed the potential for a generic "Table-Valued Function" API on the [Django Developers forum](https://forum.djangoproject.com/t/proposal-add-generate-series-support-to-contrib-postgres/21947). 

However, as **Lily Foote** noted in [new-features Issue #25](https://github.com/django/new-features/issues/25), the main challenge is managing aliases and composite values (functions returning multiple columns). 

## 3. My approach (Picture in my mind)

When I look at how Django works, I see that it keeps a very clear line between "tables" (which go in the `FROM` clause) and "functions" (which usually go in `SELECT`). To solve this, I need to build a "bridge" that allows a function to act like a table.

### 3.1. Turning a Function into a Table Source
From **[PR #18412](https://github.com/django/django/pull/18412)**, I found that we can use the `set_returning` flag on an expression. If this is `True`, the ORM should know that this is not just a value, but a source of data.

My plan is to create a `Relation` object. Inside the ORM, this object will wrap the function. When the SQL is being built, the ORM will see this `Relation` and place it in the `FROM` clause. I've chosen to use `CROSS JOIN LATERAL` for this because it is very flexible, it allows the series to use values from the main table, like `generate_series(1, user.visit_count)`.

### 3.2. How the ORM "Finds" the Data
The biggest challenge is making sure the rest of the query can talk to this new "table". For example, if I name my series `v`, and I want to do `.filter(v__gt=5)`, Django needs to know where `v` is.

I plan to solve this with a new expression called `RelationCol` (this name bcs there is already `Col` class this will give more info). This is my "pointer". 
*   In the `SELECT` clause, it says: "Give me the value from the table named `v`".
*   In the `WHERE` clause, it allows the ORM to resolve lookups (`__gt`, `__lt`) against that specific alias instead of looking for a field on the Model.

By doing this, we don't have to change how filtering works in Django. We just give the ORM a new type of "Column" that knows it belongs to a function-relation.

## 4. `GenerateSeries` in `contrib.postgres`

Using this foundation, I will build the `GenerateSeries` expression. It should be very simple to use but powerful enough to handle different types.

### Automatic Type Inference
PostgreSQL is strict about types. If you start a series with an integer, it's an integer series. If you start with a timestamp, it's a timestamp series. 

I want the Django API to be smart about this. By looking at the first argument to `GenerateSeries`, the ORM can automatically decide the `output_field`. This means the user doesn't have to manually tell Django what the type is most of the time.

```python
# The ORM will see the datetime and know this is a DateTimeField series
GenerateSeries(start=datetime(2026, 1, 1), stop=datetime(2026, 1, 2), step=timedelta(hours=1))
```

#### Simple Usage Example
The API is designed to be as simple as possible for basic needs. For example, generating a simple sequence of numbers:

```python
from django.contrib.postgres.expressions import GenerateSeries

# Generate numbers from 1 to 10
series = GenerateSeries(start=1, stop=10)

# In a more complex QuerySet context
results = MyModel.objects.alias(val=Relation(series)).values("val")
```

Or generating a simple date range:

```python
from datetime import date, timedelta

# Monthly range for 2026
series = GenerateSeries(
    start=date(2026, 1, 1),
    stop=date(2026, 12, 1),
    step=timedelta(weeks=4) # or a custom interval string
)
```

## 5. Testing Strategy and Edge Cases
Because this touches the core SQL generation, testing is very important. I have identified several areas that need careful checks:

*   **Inclusive vs Exclusive**: Making sure the `stop` value appears correctly.
*   **Empty Results**: If the start is greater than the stop, the query should return zero rows without crashing.
*   **Timezones**: Testing how series behave when they cross into Daylight Saving Time.
*   **Integrity**: Ensuring that calling `.count()` or `.aggregate()` on a QuerySet with a series returns the correct mathematical results.

### 5.2. Integration and Performance
*   **Gap Filling**: Verifying that `LEFT JOIN` operations correctly preserve all generated rows when joined against sparse data.
*   **Cartesian Products**: Testing implicit cross-joins when multiple `generate_series` calls are present in a single query.
*   **Large Sequences**: Stress testing the memory and performance impact when generating millions of rows, ensuring the ORM's "lazy" evaluation remains efficient.

## Other
Currently, I am already spending **9 to 10 hours every day** on open source work. So I can give sufficient time to implement this.

## 7. About Me

My name is [Pravin](https://www.linkedin.com/in/pravin-206069235/), I am a software Engineer with **2.8 years of professional experience** using Python and Django. Django is the framework I have grown with, and it is the primary tool I use for my work. 

For the last year, I have been working **full-time on open source**, focusing on contributing back to the tools I rely on. I have been active in the Django community, learning from experienced contributors. I understand the importance of writing tests and following Django's coding standards.

**My merged contributions to Django core include:**
*   [Fixed #36929 -- Dropped support for GEOS 3.9. (PR #20720)](https://github.com/django/django/pull/20720)
*   [Fixed #36769 -- Avoided visiting deeply nested nodes in XML deserializer. (PR #20377)](https://github.com/django/django/pull/20377)
*   [Fixed #32568 -- Replaced obvious mark_safe usages with SafeString for performance. (PR #20287)](https://github.com/django/django/pull/20287)
*   [Bump checkout action to newer versions (PR #20430)](https://github.com/django/django/pull/20430)
*   [fixes #36789 -- added missing .pdf version contribution_process.svg (PR #20387)](https://github.com/django/django/pull/20387)



I am very excited about this project because it solves a real problem I have faced in my own work. I have already done the technical research to make sure this plan is realistic and follows Django's internal architecture.