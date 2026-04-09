# GSoC 2026 Proposal: Add `generate_series()` support to `contrib.postgres`

## 1. Introduction and Motivation

PostgreSQL's set-returning functions (SRFs), such as `generate_series()`, are powerful tools for building complex reports, filling gaps in time-series data, or generating numeric sequences. 

 While many developers have used raw SQL or third-party packages to achieve this, there is a clear need for a native, first-class implementation within the ORM. Relying on "phantom models" or raw SQL fragments often leads to brittle code that breaks standard QuerySet features like `.count()` or manual aggregation.

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

First, I plan to create a `Relation` object. This object acts as the **source** of data in the `FROM` clause. When you pass a function like `generate_series()` to a `Relation`, the ORM knows to treat it like a table. 

My plan is to use `CROSS JOIN LATERAL` for this. This is very flexible because it allows the series to depend on values from other tables already in the query.

### 3.2. How the ORM "Finds" the Data
Once the source is in the `FROM` clause, we need a way to **reference** its data in the `SELECT`, `WHERE`, or `ORDER BY` clauses. I plan to use a new expression called `RelationCol`. 

Think of it this way:
*   `Relation` is the **Table** (the source).
*   `RelationCol` is the **Pointer** (the column reference).

By using `RelationCol`, we allow the ORM to resolve field lookups (like `__gt` or `__year`) against the series alias just like it does for a normal model field. This ensures that features like `.filter()` and `.order_by()` work perfectly without extra complex logic.

## 4. Simple Usage Examples

I want the API to be as "magical" as possible for most users, handling the `Relation` and `RelationCol` behind the scenes.

### Transparent Usage (Standard)
For most cases, a user just needs to call `.annotate()`. The ORM will see the `set_returning=True` flag on `GenerateSeries`, automatically create the `Relation` for the `FROM` clause, and give the user a `RelationCol` to work with.

```python
from django.contrib.postgres.expressions import GenerateSeries

# The ORM handles the Relation vs RelationCol distinction for you
qs = MyModel.objects.annotate(v=GenerateSeries(1, 10)).filter(v__gt=5)
```

### Advanced Usage (Explicit `Relation`)
For more complex queries, a developer can explicitly define a data source using a new `.relation()` QuerySet method. 

While `.annotate()` is great for simply adding a series to existing model data, we need the explicit `Relation` version in a few important cases:

1. **Gap Filling (Left Joins):** If you want to group sales by day over a month, but no sales happened on certain days, `.annotate()` on the `Sales` model will just skip those days. By putting the `GenerateSeries` Relation first in the `FROM` clause and using a `LEFT JOIN` to connect the `Sales` model to it, we can make sure every generated date shows up in the final result (with `0` or null for days with no sales).
2. **Dealing with Multiple Columns:** If a Postgres function returns multiple columns at once, explicit relations let you name the whole result table. Then, you can use `RelationCol` to pick specific columns from that table safely.
3. **Preventing Duplicated Rows:** When using more than one set-returning function in the same query, `.annotate()` might get tricky and accidentally multiply your rows. Explicit relations force you to clearly name your sources, giving you full control over how the database joins them.

```python
from django.contrib.postgres.expressions import GenerateSeries
from django.db.models import Sum

# Create the series we want to use as a source
series = GenerateSeries(start=date(2026, 1, 1), stop=date(2026, 1, 31), step=timedelta(days=1))

# By using .relation() instead of .annotate(), we explicitly place the series in the 
# FROM clause under the clear alias "calendar". This gives us a safe, named 
# virtual table to join our Sales model against without duplicating rows.
daily_sales = Sales.objects.relation(
    calendar=series
).values("calendar").annotate(total_sales=Sum("amount"))
```

### Direct Function Querying (Standalone)
Because `Relation` works so well as a table source, it gives us the ability to query a set-returning function directly, without needing a Django `Model` to start with.

As a reviewer suggested, letting developers use `Relation` directly as a QuerySet source means they can evaluate a Postgres function completely on its own. 

```python
from django.contrib.postgres.expressions import GenerateSeries
from django.db.models.query import QuerySet

# No "MyModel" required! Query the function directly as the primary table
qs = Relation(GenerateSeries(1, 10)).values()
# Compiles simply to: SELECT * FROM generate_series(1, 10)
```

**Technical Implementation Details:** Right now, Django's `QuerySet` strictly requires a base `self.model` so it knows which database table to query (`self.model._meta.db_table`). To support `Relation(...).values()`, I will need to explore creating a "dummy" model in memory, or changing the SQL compiler so it doesn't crash when the base table is a `Relation` object instead of a real database table.

### Smart Type Inference
The ORM will also be smart about types. By looking at the `start` value, it can decide if the result should be an `IntegerField`, a `DateField`, or a `DateTimeField`.

```python
# The ORM sees the date objects and automatically creates a DateField series
GenerateSeries(
    start=date(2026, 1, 1),
    stop=date(2026, 12, 1),
    step=timedelta(weeks=4)
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

1. Refs #35440 - Optimized `parse_header_parameters` for common cases. ([PR #20532](https://github.com/django/django/pull/20532))
1. Fixed #36929 - Dropped support for GEOS 3.9. ([PR #20720](https://github.com/django/django/pull/20720))
1. Fixed #36769 - Avoided visiting deeply nested nodes in XML deserializer. ([PR #20377](https://github.com/django/django/pull/20377))
1. Fixed #32568 - Replaced obvious `mark_safe` usages with `SafeString` for performance. ([PR #20287](https://github.com/django/django/pull/20287))
1. Bump checkout action to newer versions. ([PR #20430](https://github.com/django/django/pull/20430))
1. Fixed #36789 - Added missing PDF version of `contribution_process.svg`. ([PR #20387](https://github.com/django/django/pull/20387))

Other than Django core I have contributed to some Django community software too.

*   **django-rusty-templates**: [PRs](https://github.com/LilyFirefly/django-rusty-templates/pulls?q=is%3Amerged+is%3Apr+author%3Ap-r-a-v-i-n+)
*   **wagtail**: [PRs](https://github.com/wagtail/wagtail/commits?author=p-r-a-v-i-n)
*   **django-rest-framework**: [PRs](https://github.com/encode/django-rest-framework/pulls?q=is%3Apr+author%3Ap-r-a-v-i-n+is%3Amerged)
*   **django-tasks**: [PRs](https://github.com/RealOrangeOne/django-tasks/pulls?q=is%3Amerged+is%3Apr+author%3Ap-r-a-v-i-n+)
*   **django-debug-toolbar**: [PRs](https://github.com/django-commons/django-debug-toolbar/pulls?q=is%3Amerged+is%3Apr+author%3Ap-r-a-v-i-n)
*   **djangoproject.com**: [PRs](https://github.com/django/djangoproject.com/pulls?q=is%3Apr+is%3Amerged+author%3Ap-r-a-v-i-n+)
*   **django-click**: [PRs](https://github.com/django-commons/django-click/pulls?q=is%3Amerged+is%3Apr+author%3Ap-r-a-v-i-n+)
*   **drf-excel**: [PRs](https://github.com/django-commons/drf-excel/pulls?q=is%3Amerged+is%3Apr+author%3Ap-r-a-v-i-n+)
*   **django-tinymce**: [PRs](https://github.com/jazzband/django-tinymce/pulls?q=is%3Apr+is%3Amerged+author%3Ap-r-a-v-i-n+)
*   **django-extensions**: [PRs](https://github.com/django-extensions/django-extensions/pulls?q=is%3Amerged+is%3Apr+author%3Ap-r-a-v-i-n+)
*   **django-packages**: [PRs](https://github.com/djangopackages/djangopackages/pulls?q=is%3Apr+is%3Amerged+author%3Ap-r-a-v-i-n+)

I am very excited about this project because it solves a real problem I have faced in my own work. I have already done the technical research to make sure this plan is realistic and follows Django's internal architecture.
