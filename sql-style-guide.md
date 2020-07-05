# SQL Style Guide

## Introduction

_Note: This style guide is primarily written with AWS-related forms of SQL (e.g. PostgreSQL/Redshift, Presto/Athena) in mind, but its principles can be broadly applied to other flavors of SQL._

Having readable and consistent code is a core tenet of observing best practices on any data team. Code is almost always read more often than it is written, so care towards optimizing for readability should be observed. This style guide aims to strike a balance between readability and practicality, favoring readability so long as ease of writing of code is relatively unimpaired. Therefore, adhering to this style guide will generally stress creating new lines and indenting as you write, eschewing the effort needed to maintain something like Joe Celko's [river-style formatting](https://www.red-gate.com/simple-talk/sql/t-sql-programming/formatting-sql-code-part-second/), which is difficult to preserve without frequently having to go back and edit.

In particular, this style guide is intended to integrate smoothly with modern data science tools and workflows in Python. The original motivation for creating this guide arose from a perceived lack of preexisting SQL style guidelines that coexist nicely with Python/Spark, especially as libraries such as `SQLAlchemy`, `psycopg2`, `sqlite3`, and `PyAthena` have become commonplace. Thus, this guide primarily looks towards Python style conventions and defaults while maintaining general principles of good SQL style.

For cases not explicitly covered, Python's [PEP 8](https://www.python.org/dev/peps/pep-0008/) guidelines can be extended to this guide.

## Overarching Principles

- Always capitalize SQL [reserved keywords](https://www.sqlstyle.guide/#reserved-keyword-reference) (e.g. ` SELECT`, `WHERE`, `AS`, `COUNT`, `DISTINCT`, `NOT`, etc.)
    - Never use uppercase in field or variable names
- `AS` should always be used when aliasing table or field names
- Use snake case (underscore_separated_names) as the naming convention for common table expressions (CTEs), aliases, etc.
- Indent size should be 4, using spaces instead of tab (most SQL editors allow tab size to be set with spaces).
    - _Note: The choice to use 4 here mirrors common use cases integrating Python + SQL, wherein writing Python forces indents to be 4 spaces_
- Always left align main columns (i.e. `SELECT`, `FROM`, `LEFT JOIN`, `WHERE`, `GROUP BY`, `ORDER BY`, `LIMIT`)
- Aim to choose descriptive names, especially when shortening multiple tables with similar names
    - It is common practice to use the first letter of each word for tables (e.g. `FROM event_log AS el`)
    - Try to rename fields as necessary to avoid ambiguous similar field names after joining
- Suggested max line length is 80 characters; long lines run the risk of being more difficult to read
- Comment to briefly explain what the query is doing whenever it is not obvious, using `--` for single-line or inline comments, and `/* */` for multi-line comments

## Indenting clauses

### SELECT

When selecting multiple columns, give `SELECT` (keep `DISTINCT` together) its own line and give each column its own row. Commas should go at the end of field names, never at the beginning of the line.

Good:
```
SELECT 
    user_id,
    start_date,
    payment_method
FROM subscriptions
```

Bad:
```
SELECT user_id
       ,start_date
       ,payment_method
FROM subscriptions
```

### WHERE

Similarly, when there are multiple `WHERE` clauses, indent and give each condition its own line. Avoid writing logical expressions for boolean columns to avoid confusion over redundancy.

Good:
```
SELECT COUNT(DISTINCT userid)
FROM subscriptions
WHERE 
    start_date = '2019-11-29' -- Black Friday
    AND is_current_subscriber
    AND payment_method = 'PayPal'
```

Bad:
```
SELECT COUNT(DISTINCT userid)
FROM subscriptions
WHERE start_date = '2019-11-29'
    AND is_current_subscriber = True
    AND payment_method = 'PayPal'
```

### GROUP/ORDER BY

When writing GROUP BY or ORDER BY, using implicit numbers (i.e. 1, 2, etc.) instead of field names is preferred. When there are multiple arguments, indent on a new line after the `BY`, but keep the numbers together on the subsequent line. Put spaces after commas for multiple arguments.

Good:
```
SELECT
    start_date,
    payment_method,
    COUNT(*)
FROM subscriptions
GROUP BY
    1, 2
ORDER BY 3
```

Bad:
```
SELECT
    start_date,
    payment_method,
    COUNT(*)
FROM subscriptions
GROUP BY 1,2
ORDER BY
3
```

### Exceptions

The only exceptions for new line requirements are when only 1 column/argument or `*` in the SELECT clause or 1 condition in the `WHERE` clause. If grouping or ordering by only 1 column, feel free to keep everything on a single line. `FROM` clauses may also be indented on a new line, if desired, though maintaining a single line is preferred.

Good:
```
SELECT user_id
FROM users
WHERE signup_date = '2020-01-01'
```

Also good:
```
SELECT 
    user_id
FROM
    users
WHERE
    signup_date = '2020-01-01'
```

## CASE... WHEN

When writing `CASE` `WHEN` statements, indent new lines after `CASE`, and left align subsequent `WHEN` and `ELSE` clauses. Align the closing `END` keyword on the same column as `CASE`. 

Generally, it is fine to keep `THEN` on the same line as `WHEN`, but indent `THEN` onto a new line if the line length gets out of hand.

Good:
```
SELECT
    user_id,
    CASE 
        WHEN lifetime_purchases > 100 THEN 'whale'
        WHEN lifetime_purchases = 0 THEN 'free_only'
        ELSE 'regular'
    END AS user_type
FROM users
```

Bad:
```
SELECT
    user_id,
    CASE
    WHEN lifetime_purchases > 100 THEN 'whale'
    WHEN lifetime_purchases = 0 THEN 'free_only' 
    ELSE 'regular' END AS user_type
FROM users
```

## Common Table Expressions (CTEs) & Parentheses

CTEs written as `WITH` statements at the top of a query are strongly preferred to subqueries embedded within the main body of the SQL query. When using parentheses, aim to open the parentheses on the same line as the relevant keyword, and close the parentheses on a new dedicated line, aligned column-wise with the keyword of the starting line. All lines within a set of parentheses should start 1 indent from the parent line's column.

Good:
```
WITH black_friday_start_users AS (
    SELECT user_id
    FROM subscriptions
    WHERE start_date = '2019-11-29'
),
old_users AS (
    SELECT 
        user_id,
	country
    FROM users
    WHERE signup_date < '2019-01-01'
)
...
```

Bad:
```
WITH black_friday_start_users
    AS
        (SELECT
            user_id
            FROM subscriptions
            WHERE start_date = '2019-11-29')                  
, old_users
    AS (
    SELECT 
        user_id,
	country_code
    FROM users
    WHERE signup_date < '2019-01-01')
...         
```

For more complicated parentheses usage, try to prioritize readability and consistency. This style guide cannot cover all possible cases.

Example:
```
-- Select data for all users acquired from non-paid efforts before 2020
SELECT *
FROM users
WHERE
    (
    	(acquisition_type = 'Organic' AND NOT is_partnership_user)
    	OR is_legacy
    )
    AND signup_date < '2020-01-01'
```

Here, the parentheses are admittedly a bit awkward in how much space they consume, but they present these sets of conditions in a clear and readable format, separating the first pair of conditions from the rest of the relevant `WHERE` clauses, while also indicating a logical relatedness between them.

## Window Functions

Long Window functions should be split across multiple lines: one for the `PARTITION`, `ORDER BY` and frame clauses, aligned to the `PARTITION` keyword. Partition keys should be one-per-line, left-aligned to the first key. `ORDER BY` (`ASC`, `DESC`) should always be explicit. All window functions should be aliased.

Good:
```
SELECT
    *
    ROW_NUMBER() OVER (PARTITION BY user_id,
                                    dt
                       ORDER BY dt DESC) AS transaction_num
FROM transactions
```

Bad:
```
SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY user_id, dt
                                    ORDER BY dt desc)
FROM transactions
```
Source: [Kickstarter](https://gist.github.com/fredbenenson/7bb92718e19138c20591)

## Other Resources

Feel free to consult these example guides from various organizations to get a sense of how other teams approach and converge on certain conventions, as well as to solicit additional opinions on how to resolve cases not covered here. This style guide took much inspiration from all 3 of these:
- [Kickstarter](https://gist.github.com/fredbenenson/7bb92718e19138c20591)
- [GitLab](https://about.gitlab.com/handbook/business-ops/data-team/sql-style-guide/)
- [Mozilla](https://docs.telemetry.mozilla.org/concepts/sql_style.html)
