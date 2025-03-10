---
title: SHOW CONSTRAINTS
summary: The SHOW CONSTRAINTS statement lists the constraints on a table.
toc: true
docs_area: reference.sql
---

{% assign rd = site.data.releases | where_exp: "rd", "rd.major_version == page.version.version" | first %}

{% if rd %}
{% assign remote_include_version = page.version.version | replace: "v", "" %}
{% else %}
{% assign remote_include_version = site.versions["stable"] | replace: "v", "" %}
{% endif %}

The `SHOW CONSTRAINTS` [statement](sql-statements.html) lists all named [constraints](constraints.html) as well as any unnamed [`CHECK`](check.html) constraints on a table.

## Required privileges

The user must have any [privilege](security-reference/authorization.html#managing-privileges) on the target table.

## Aliases

`SHOW CONSTRAINT` is an alias for `SHOW CONSTRAINTS`.

## Synopsis

<div>
{% remote_include https://raw.githubusercontent.com/cockroachdb/generated-diagrams/release-{{ remote_include_version }}/grammar_svg/show_constraints.html %}
</div>

## Parameters

Parameter | Description
----------|------------
`table_name` | The name of the table for which to show constraints.

## Response

The following fields are returned for each constraint.

Field | Description
------|------------
`table_name` | The name of the table.
`constraint_name` | The name of the constraint.
`constraint_type` | The type of constraint.
`details` | The definition of the constraint, including the column(s) to which it applies.
`validated` | Whether values in the column(s) match the constraint.

## Example

{% include_cached copy-clipboard.html %}
~~~ sql
> CREATE TABLE orders (
    id INT PRIMARY KEY,
    date TIMESTAMP NOT NULL,
    priority INT DEFAULT 1,
    customer_id INT UNIQUE,
    status STRING DEFAULT 'open',
    CHECK (priority BETWEEN 1 AND 5),
    CHECK (status in ('open', 'in progress', 'done', 'cancelled')),
    FAMILY (id, date, priority, customer_id, status)
);
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
> SHOW CONSTRAINTS FROM orders;
~~~

~~~
+------------+------------------------+-----------------+--------------------------------------------------------------------------+-----------+
| table_name |    constraint_name     | constraint_type |                                 details                                  | validated |
+------------+------------------------+-----------------+--------------------------------------------------------------------------+-----------+
| orders     | check_priority         | CHECK           | CHECK (priority BETWEEN 1 AND 5)                                         |   true    |
| orders     | check_status           | CHECK           | CHECK (status IN ('open':::STRING, 'in progress':::STRING,               |   true    |
|            |                        |                 | 'done':::STRING, 'cancelled':::STRING))                                  |           |
| orders     | orders_customer_id_key | UNIQUE          | UNIQUE (customer_id ASC)                                                 |   true    |
| orders     | primary                | PRIMARY KEY     | PRIMARY KEY (id ASC)                                                     |   true    |
+------------+------------------------+-----------------+--------------------------------------------------------------------------+-----------+
(4 rows)
~~~

## See also

- [Constraints](constraints.html)
- [`ADD CONSTRAINT`](add-constraint.html)
- [`RENAME CONSTRAINT`](rename-constraint.html)
- [`DROP CONSTRAINT`](drop-constraint.html)
- [`VALIDATE CONSTRAINT`](validate-constraint.html)
- [`CREATE TABLE`](create-table.html)
- [Information Schema](information-schema.html)
- [SQL Statements](sql-statements.html)
