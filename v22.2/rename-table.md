---
title: ALTER TABLE ... RENAME TO
summary: The ALTER TABLE ... RENAME TO statement changes the name of a table.
toc: true
docs_area: reference.sql
---

{% assign rd = site.data.releases | where_exp: "rd", "rd.major_version == page.version.version" | first %}

{% if rd %}
{% assign remote_include_version = page.version.version | replace: "v", "" %}
{% else %}
{% assign remote_include_version = site.versions["stable"] | replace: "v", "" %}
{% endif %}

The `RENAME TO` [statement](sql-statements.html) is part of [`ALTER TABLE`](alter-table.html), and changes the name of a table.

{{site.data.alerts.callout_info}}
`ALTER TABLE ... RENAME TO` cannot be used to move a table from one schema to another. To change a table's schema, use [`SET SCHEMA`](set-schema.html).

`ALTER TABLE ... RENAME TO` cannot be used to move a table from one database to another. To change a table's database, use [`BACKUP`](backup.html#backup-a-table-or-view) and [`RESTORE`](restore.html#restore-a-table).
{{site.data.alerts.end}}

{{site.data.alerts.callout_info}}
It is not possible to rename a table referenced by a view. For more details, see <a href="views.html#view-dependencies">View Dependencies</a>.
{{site.data.alerts.end}}

{% include {{ page.version.version }}/misc/schema-change-stmt-note.md %}

## Required privileges

The user must have the `DROP` [privilege](security-reference/authorization.html#managing-privileges) on the table and the `CREATE` on the parent database. When moving a table from one database to another, the user must have the `CREATE` privilege on both the source and target databases.

## Synopsis

<div>
{% remote_include https://raw.githubusercontent.com/cockroachdb/generated-diagrams/release-{{ remote_include_version }}/grammar_svg/rename_table.html %}
</div>

## Parameters

 Parameter | Description
-----------|-------------
 `IF EXISTS` | Rename the table only if a table with the current name exists; if one does not exist, do not return an error.
 `current_name` | The current name of the table.
 `new_name` | The new name of the table, which must be unique within its database and follow these [identifier rules](keywords-and-identifiers.html#identifiers). When the parent database is not set as the default, the name must be formatted as `database.name`.<br><br>The [`UPSERT`](upsert.html) and [`INSERT ON CONFLICT`](insert.html) statements use a temporary table called `excluded` to handle uniqueness conflicts during execution. It's therefore not recommended to use the name `excluded` for any of your tables.

## Viewing schema changes

{% include {{ page.version.version }}/misc/schema-change-view-job.md %}

## Examples

{% include {{page.version.version}}/sql/movr-statements.md %}

### Rename a table

{% include_cached copy-clipboard.html %}
~~~ sql
> SHOW TABLES;
~~~

~~~
  schema_name |         table_name         | type  | estimated_row_count
--------------+----------------------------+-------+----------------------
  public      | promo_codes                | table |                1000
  public      | rides                      | table |                 500
  public      | user_promo_codes           | table |                   0
  public      | users                      | table |                  50
  public      | vehicle_location_histories | table |                1000
  public      | vehicles                   | table |                  15
(6 rows)
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER TABLE users RENAME TO riders;
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
> SHOW TABLES;
~~~

~~~
  schema_name |         table_name         | type  | estimated_row_count
--------------+----------------------------+-------+----------------------
  public      | promo_codes                | table |                1000
  public      | rides                      | table |                 500
  public      | user_promo_codes           | table |                   0
  public      | riders                     | table |                  50
  public      | vehicle_location_histories | table |                1000
  public      | vehicles                   | table |                  15
(6 rows)
~~~

To avoid an error in case the table does not exist, you can include `IF EXISTS`:

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER TABLE IF EXISTS customers RENAME TO clients;
~~~

## See also

- [`CREATE TABLE`](create-table.html)
- [`ALTER TABLE`](alter-table.html)
- [`SHOW TABLES`](show-tables.html)
- [`DROP TABLE`](drop-table.html)
- [`SHOW JOBS`](show-jobs.html)
- [SQL Statements](sql-statements.html)
- [Online Schema Changes](online-schema-changes.html)
