# GSoC 2026 Proposal: Add `generate_series()` support to `contrib.postgres`

## 1. Introduction and Motivation

PostgreSQL's set-returning functions (SRFs), such as `generate_series()`, are powerful tools for building complex reports, filling gaps in time-series data, or generating numeric sequences. 

This project idea was inspired by the official **Django Project GSoC 2026 ideas list**. After researching the proposal, I realized that while many developers have used raw SQL or third-party packages to achieve this, there is a clear need for a native, first-class implementation within the ORM. Relying on "phantom models" or raw SQL fragments often leads to brittle code that breaks standard QuerySet features like `.count()` or manual aggregation.

My goal this summer is to implement `generate_series()` in `contrib.postgres` in a way that feels native to Django, ensuring it is robust, well-tested, and consistent with the ORM’s architecture.

## 2. Technical Research and Implementation Path

I have researched the current state of set-returning functions in Django by examining recent discussions among core developers and studying the source code.

### 2.1. Foundation: The `set_returning` Flag
I have studied the work in **[PR #18412](https://github.com/django/django/pull/18412)**, authored by [**Devin Cox**](https://github.com/devin13cox) and [**Simon Charette**](https://github.com/charettes), which added support for set-returning database functions. This PR introduced the `set_returning = False` attribute to the `BaseExpression` class. By following the logic in `django/db/models/sql/query.py`, I learned how this flag ensures that the SQL compiler handles rows correctly when a function might multiply the result set.

My implementation will build directly on this foundation. I plan to use this flag to help the compiler distinguish between scalar expressions and expressions that should be treated as table sources in the `FROM` clause.

### 2.2. Goal 1: A Generic `Relation` API
Core contributors like [**Simon Charette**](https://github.com/charettes) and [**Adam Johnson**](https://github.com/adamchainz) have discussed the potential for a generic "Table-Valued Function" API on the [Django Developers forum](https://forum.djangoproject.com/t/proposal-add-generate-series-support-to-contrib-postgres/21947). 

However, as **Lily Foote** noted in [new-features Issue #25](https://github.com/django/new-features/issues/25), the main challenge is managing aliases and composite values (functions returning multiple columns). 

**My approach:**
*   I will first aim to implement a generic `Relation` abstraction that allows an expression to be treated as a table source.
*   This would ideally allow SRFs to be joined or aliased using a syntax similar to `FilteredRelation`.
*   I will focus on implementing this through `LATERAL` joins where appropriate, ensuring the solution is compatible with existing QuerySet behaviors.

### 2.3. Goal 2: `GenerateSeries` in `contrib.postgres` (The Core Deliverable)
While a generic API is the long-term goal, the primary success of my project will be a working `GenerateSeries` implementation in `contrib.postgres`. 

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

#### The Advanced Reporting Example
A common real-world use case is ensuring every day in a reporting period is shown, even if no sales occurred:

```python
from django.contrib.postgres.expressions import GenerateSeries
from django.db.models import F, Sum, Q, DateTimeField
from django.db.models.functions import Coalesce
from datetime import datetime, timedelta

# Sales report for the last week
report = Sales.objects.alias(
    day=Relation(
        GenerateSeries(
            start=datetime.now() - timedelta(days=6),
            stop=datetime.now(),
            step=timedelta(days=1),
            output_field=DateTimeField(),
        ),
        on=Q(day=F('created_at__date'))
    )
).values("day").annotate(
    total_sales=Coalesce(Sum("amount"), 0)
).order_by("day")
```

## 3. Testing Strategy and Edge Cases

A major part of this project will be ensuring the implementation is robust across all PostgreSQL-supported types and edge cases. Based on my research of existing solutions like `django-generate-series` and PostgreSQL documentation, I have identified the following testing priorities:

### 3.1. Edge Case Validation
*   **Boundary Inclusion**: Verifying that `stop` is inclusive when it aligns perfectly with the `step`, and exclusive otherwise.
*   **Negative Steps**: Ensuring decreasing sequences (e.g., `start=10, stop=1, step=-1`) work correctly.
*   **Zero/Invalid Steps**: Handling cases where `step` is zero or invalid to prevent infinite loops or database crashes.
*   **Empty Ranges**: Ensuring the QuerySet is empty if `start > stop` (with positive step) or `start < stop` (with negative step).
*   **Timezone & DST**: Carefully testing `DateTimeField` sequences across daylight saving time boundaries to ensure consistency with PostgreSQL's timezone logic.

### 3.2. Integration and Performance
*   **Gap Filling**: Verifying that `LEFT JOIN` operations correctly preserve all generated rows when joined against sparse data.
*   **Cartesian Products**: Testing implicit cross-joins when multiple `generate_series` calls are present in a single query.
*   **Large Sequences**: Stress testing the memory and performance impact when generating millions of rows, ensuring the ORM's "lazy" evaluation remains efficient.

## 4. Implementation Plan (350 Hours)

*   **Bonding Period**: Finalize technical design for the `Relation` class with mentors; build a comprehensive "Edge Case Matrix" for testing.
*   **Weeks 1-4**: ORM Compiler changes. Allow expressions with `set_returning=True` to be injected into the `FROM` clause.
*   **Weeks 5-8**: Implementing the `Relation` API and testing against common reporting patterns.
*   **Weeks 9-11**: Finalizing the `GenerateSeries` implementation in `contrib.postgres`, including deep support for intervals and field type safety.
*   **Final Weeks**: Documentation, final polish, and responding to mentor feedback.

## 5. About Me

My name is [Pravin](https://www.linkedin.com/in/pravin-206069235/), I am a software Engineer with **2.8 years of professional experience** using Python and Django. Django is the framework I have grown with, and it is the primary tool I use for my work. 

For the last year, I have been working **full-time on open source**, focusing on contributing back to the tools I rely on. I have been active in the Django community, learning from experienced contributors. I understand the importance of writing tests and following Django's coding standards.

**My merged contributions to Django core include:**
*   [Fixed #36929 -- Dropped support for GEOS 3.9. (PR #20720)](https://github.com/django/django/pull/20720)
*   [Fixed #36769 -- Avoided visiting deeply nested nodes in XML deserializer. (PR #20377)](https://github.com/django/django/pull/20377)
*   [Fixed #32568 -- Replaced obvious mark_safe usages with SafeString for performance. (PR #20287)](https://github.com/django/django/pull/20287)
*   [Bump checkout action to newer versions (PR #20430)](https://github.com/django/django/pull/20430)
*   [fixes #36789 -- added missing .pdf version contribution_process.svg (PR #20387)](https://github.com/django/django/pull/20387)

Currently, I am already spending **9 to 10 hours every day** on open source work.

## 6. Conclusion

This is really great opportunity for me to add value in Django.