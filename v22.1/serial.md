---
title: SERIAL
summary: The SERIAL pseudo-type produces integer values automatically.
toc: true
docs_area: reference.sql
---

The `SERIAL` pseudo [data type](data-types.html) is a keyword that can be used *in lieu* of a real data type when defining table columns. It is approximately equivalent to using an [integer type](int.html) with a [`DEFAULT` expression](default-value.html) that generates different values every time it is evaluated. This default expression in turn ensures that inserts that do not specify this column will receive an automatically generated value instead of `NULL`.

{{site.data.alerts.callout_info}}
`SERIAL` is provided only for compatibility with PostgreSQL. New applications should use real data types and a suitable `DEFAULT` expression.

In most cases, we recommend using the [`UUID`](uuid.html) data type with the `gen_random_uuid()` function as the default value, which generates 128-bit values (larger than `SERIAL`'s maximum of 64 bits) and more uniformly scatters them across all of a table's underlying key-value ranges. UUIDs ensure more effectively that multiple nodes share the insert load when a UUID column is used in an index or primary key.

See [this FAQ entry](sql-faqs.html#how-do-i-auto-generate-unique-row-ids-in-cockroachdb) for more details.
{{site.data.alerts.end}}

## Modes of operation

The keyword `SERIAL` is recognized in `CREATE TABLE` and is automatically translated to a real data type and a [`DEFAULT` expression](default-value.html) during table creation. The result of this translation is then used internally by CockroachDB, and can be observed using [`SHOW CREATE`](show-create.html).

The chosen `DEFAULT` expression ensures that different values are automatically generated for the column during row insertion.  These are not guaranteed to increase monotonically, see [this section below](#auto-incrementing-is-not-always-sequential) for details.

There are three possible translation modes for `SERIAL`:

 Mode                | Description
---------------------|------------------
 `rowid` (default)   | `SERIAL` implies `DEFAULT unique_rowid()`. The real data type is always `INT8`, regardless of the `default_int_size` [session variable](set-vars.html) or `sql.defaults.default_int_size` [cluster setting](cluster-settings.html).
 `virtual_sequence`  | `SERIAL` creates a virtual sequence and implies `DEFAULT nextval(<seqname>)`.  The real data type is always `INT8`, regardless of the `default_int_size` session variable or `sql.defaults.default_int_size` cluster setting.
 `sql_sequence`      | `SERIAL` creates a regular SQL sequence and implies `DEFAULT nextval(<seqname>)`. The real data type depends on `SERIAL` variant.
 `sql_sequence_cached`| `SERIAL` creates a regular SQL sequence and implies `DEFAULT nextval(<seqname>)`, caching the results for reuse in the current session. The real data type depends on `SERIAL` variant. When the cache is empty, the underlying sequence will only be incremented once to populate the cache.<br>When this mode is set, the `sql.defaults.serial_sequences_cache_size` [cluster setting](cluster-settings.html) controls the number of values to cache in a user's session, with a default of 256.

These modes can be configured with the [session variable](set-vars.html) `serial_normalization`.

{{site.data.alerts.callout_info}}
The particular choice of `DEFAULT` expression when clients use the `SERIAL` keyword is subject to change in future versions of CockroachDB. Applications that wish to use `unique_rowid()` specifically must use the full explicit syntax `INT DEFAULT unique_rowid()` and avoid `SERIAL` altogether.
{{site.data.alerts.end}}

### Generated values for modes `rowid` and `virtual_sequence`

In both modes `rowid` and `virtual_sequence`, a value is automatically generated using the `unique_rowid()` function. The difference between `rowid` and `virtual_sequence` is that the latter setting also creates a virtual (pseudo) sequence in the database. However, in both cases the `unique_rowid()` function is ultimately used to generate new values. This function produces a 64-bit integer (i.e., [`INT8`](int.html)) from the current timestamp and ID of the node executing the [`INSERT`](insert.html) or [`UPSERT`](upsert.html) operation. This behavior is statistically likely to be globally unique except in extreme cases (see [this FAQ entry](sql-faqs.html#how-do-i-auto-generate-unique-row-ids-in-cockroachdb) for more details).

The `unique_rowid()` function creates a unique integer composed of the current time at a 10-microsecond granularity and the `instance-id`. The `instance-id` is stored in the lower 15 bits of the returned value and the timestamp is stored in the upper 48 bits. The top-most bit is left empty so that negative values are not returned. The 48-bit timestamp field provides for 89 years of timestamps. CockroachDB uses a custom epoch (Jan 1, 2015) in order to utilize the entire timestamp range.

Also, because value generation using `unique_rowid()` does not require inter-node coordination, it is much faster than the other mode [`sql_sequence`](#generated-values-for-mode-sql_sequence-and-sql_sequence_cached) discussed below when multiple SQL clients are writing to the table from different nodes.

{{site.data.alerts.callout_info}}
Values generated by the `unique_rowid()` function do not respect the `default_int_size` [session variable](set-vars.html) or `sql.defaults.default_int_size` [cluster setting](cluster-settings.html).
{{site.data.alerts.end}}

### Generated values for mode `sql_sequence` and `sql_sequence_cached`

In both modes, a regular [SQL sequence](create-sequence.html) is automatically created alongside the table where `SERIAL` is specified.

The actual data type is determined as follows:

| `SERIAL` variant | Real data type |
|---|---|
| `SERIAL2`, `SMALLSERIAL` | `INT2` |
| `SERIAL4` | `INT4` |
| `SERIAL` | `INT` |
| `SERIAL8`, `BIGSERIAL` | `INT8` |

Every insert or upsert into the table will then use `nextval()` to increment the sequence and produce increasing values.

Because SQL sequences persist the current sequence value in the database, inter-node coordination is required when multiple clients use the sequence concurrently via different nodes. This can cause [contention](sql-faqs.html#what-is-transaction-contention) and impact performance negatively.

Therefore, applications should consider using `unique_rowid()` or `gen_random_uuid()` as discussed in [this FAQ entry](sql-faqs.html#how-do-i-auto-generate-unique-row-ids-in-cockroachdb) instead of sequences when possible.

{{site.data.alerts.callout_info}}
Note that `sql_sequence_cached` will perform fewer distributed calls to increment sequences, resulting in better performance than `sql_sequence`. However, cached sequences may result in large gaps between serial sequence numbers if a session terminates before using all the values in its cache.
{{site.data.alerts.end}}

### Generated values for mode `unordered_rowid`

In this mode, a value is generated using the `unordered_unique_rowid()` [function](functions-and-operators.html#id-generation-functions). This function is similar to the [`rowid` mode](#generated-values-for-modes-rowid-and-virtual_sequence) but does not guarantee ordering.

## Auto-incrementing is not always sequential

It's a common misconception that the auto-incrementing types in PostgreSQL and MySQL generate strictly sequential values. However, there can be gaps and the order is not completely guaranteed:

- Each insert increases the sequence by one, even when the insert is not committed. This means that auto-incrementing types may leave gaps in a sequence.
- Two concurrent transactions can commit in a different order than their use of sequences, and thus "observe" the values to decrease relative to each other. This effect is amplified by automatic transaction retries.

These are fundamental properties of a transactional system with non-transactional sequences. PostgreSQL, MySQL, and CockroachDB do not increase sequences transactionally with other SQL statements, so these effects can happen in any case.

To experience this for yourself, run through the following example in PostgreSQL:

1. Create a table with a `SERIAL` column:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE increment (a SERIAL PRIMARY KEY);
    ~~~

2. Run four transactions for inserting rows:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    > INSERT INTO increment DEFAULT VALUES;
    > ROLLBACK;
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    > INSERT INTO increment DEFAULT VALUES;
    > COMMIT;
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    > INSERT INTO increment DEFAULT VALUES;
    > ROLLBACK;
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > BEGIN;
    > INSERT INTO increment DEFAULT VALUES;
    > COMMIT;
    ~~~

3. View the rows created:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > SELECT * from increment;
    ~~~

    ~~~
              a
    ----------------------
      645994218934534145
      645994218978082817
    (2 rows)
    ~~~

    Since each insert increased the sequence in column `a` by one, the first committed insert got the value `645994218934534145`, and the second committed insert got the value `645994218978082817`. As you can see, the values aren't strictly sequential, and the last value doesn't give an accurate count of rows in the table.

In summary, the `SERIAL` type in PostgreSQL and CockroachDB, and the `AUTO_INCREMENT` type in MySQL, all behave the same in that they do not create strict sequences. CockroachDB will likely create more gaps than these other databases, but will generate these values much faster. An alternative is to use [`SEQUENCE`](create-sequence.html).

### Additional scenarios

If two transactions occur concurrently, CockroachDB cannot guarantee monotonically increasing IDs (i.e., first commit is smaller than second commit). Here are three more scenarios that demonstrate this:

#### Scenario 1

- At time 1, transaction `T1` `BEGIN`s.
- At time 2, transaction `T2` `BEGIN`s on the same node (from a different client).
- At time 3, transaction `T1` creates a `SERIAL` value, `x`.
- At time 3 + 2 microseconds, transaction `T2` creates a `SERIAL` value, `y`.
- At time 4, transaction `T1` `COMMIT`s.
- At time 5, transaction `T2` `COMMIT`s.

If this happens, CockroachDB cannot guarantee whether `x < y` or `x > y`, despite the fact `T1` and `T2` began and were committed in different times. In this particular example, it's even likely that `x = y` because there is less than a 10-microsecond difference and the `SERIAL` values are constructed from the number of microseconds in the current time.

#### Scenario 2

- At time 1, transaction `T1` `BEGIN`s.
- At time 1, transaction `T2` `BEGIN`s somewhere else, on a different node.
- At time 2, transaction `T1` creates a `SERIAL` value, `x`.
- At time 3, transaction `T2` creates a `SERIAL` value, `y`.
- At time 4, transaction `T1` `COMMIT`s.
- At time 4, transaction `T2` `COMMIT`s.

If this happens, CockroachDB cannot guarantee whether `x < y` or `x > y`. Both can happen, even though the transactions began and committed at the same time. However it's sure that `x != y` because the values were generated on different nodes.

#### Scenario 3

- At time 1, transaction `T1` `BEGIN`s.
- At time 2, transaction `T1` creates a `SERIAL` value, `x`.
- At time 3, transaction `T1` `COMMIT`s.
- At time 4, transaction `T2` `BEGIN`s somewhere else, on a different node.
- At time 5, transaction `T2` creates a `SERIAL` value, `y`.
- At time 6, transaction `T2` `COMMIT`s.

There is less than a 250-microsecond difference between the system clocks of the two nodes.

If this happens, CockroachDB cannot guarantee whether `x < y` or `x > y`. Even though the transactions "clearly" occurred one "after" the other, perhaps there was a clock skew between the two nodes and the system time of the second node is set earlier than the first node.

## Example

### Use `SERIAL` to auto-generate primary keys

In this example, we create a table with the `SERIAL` column as the primary key so we can auto-generate unique IDs on insert.

{% include_cached copy-clipboard.html %}
~~~ sql
> CREATE TABLE serial (a SERIAL PRIMARY KEY, b STRING, c BOOL);
~~~

The [`SHOW COLUMNS`](show-columns.html) statement shows that the `SERIAL` type is just an alias for `INT` with `unique_rowid()` as the default.

{% include_cached copy-clipboard.html %}
~~~ sql
> SHOW COLUMNS FROM serial;
~~~

~~~
  column_name | data_type | is_nullable | column_default | generation_expression |  indices  | is_hidden
--------------+-----------+-------------+----------------+-----------------------+-----------+------------
  a           | INT8      |    false    | unique_rowid() |                       | {primary} |   false
  b           | STRING    |    true     | NULL           |                       | {}        |   false
  c           | BOOL      |    true     | NULL           |                       | {}        |   false
(3 rows)
~~~

When we insert rows without values in column `a` and display the new rows, we see that each row has defaulted to a unique value in column `a`.

{% include_cached copy-clipboard.html %}
~~~ sql
> INSERT INTO serial (b,c) VALUES ('red', true), ('yellow', false), ('pink', true);
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
> INSERT INTO serial (a,b,c) VALUES (123, 'white', false);
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
> SELECT * FROM serial;
~~~

~~~
          a          |   b    |   c
---------------------+--------+--------
                 123 | white  | false
  645993277462937601 | red    | true
  645993277462970369 | yellow | false
  645993277463003137 | pink   | true
(4 rows)
~~~

## See also

- [FAQ: How do I auto-generate unique row IDs in CockroachDB?](sql-faqs.html#how-do-i-auto-generate-unique-row-ids-in-cockroachdb)
- [Data Types](data-types.html)
