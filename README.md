# pgslice

Postgres partitioning as easy as pie. Works great for both new and existing tables, with zero downtime and minimal app changes. No need to install anything on your database server. Archive older data on a rolling basis to keep your database size under control.

:tangerine: Battle-tested at [Instacart](https://www.instacart.com/opensource)

[![Build Status](https://travis-ci.org/ankane/pgslice.svg?branch=master)](https://travis-ci.org/ankane/pgslice)

## Install

pgslice is a command line tool. To install, run:

```sh
gem install pgslice
```

This will give you the `pgslice` command. If installation fails, you may need to install [dependencies](#dependencies).

## Steps

1. Ensure the table you want to partition has been created. We’ll refer to this as `<table>`.

2. Specify your database credentials

  ```sh
  export PGSLICE_URL=postgres://localhost/myapp_development
  ```

3. Create an intermediate table

  ```sh
  pgslice prep <table> <column> <period>
  ```

  Period can be `day`, `month`, or `year`.

  This creates a table named `<table>_intermediate` with the appropriate trigger for partitioning.

4. Add partitions

  ```sh
  pgslice add_partitions <table> --intermediate --past 3 --future 3
  ```

  This creates child tables that inherit from the intermediate table.

  Use the `--past` and `--future` options to control the number of partitions.

5. *Optional, for tables with data* - Fill the partitions in batches with data from the original table

  ```sh
  pgslice fill <table>
  ```

  Use the `--batch-size` and `--sleep` options to control the speed.

  To sync data across different databases, check out [pgsync](https://github.com/ankane/pgsync).

6. Analyze tables

  ```sh
  pgslice analyze <table>
  ```

7. Swap the intermediate table with the original table

  ```sh
  pgslice swap <table>
  ```

  The original table is renamed `<table>_retired` and the intermediate table is renamed `<table>`.

8. Fill the rest (rows inserted between the first fill and the swap)

  ```sh
  pgslice fill <table> --swapped
  ```

9. Back up the retired table with a tool like [pg_dump](https://www.postgresql.org/docs/current/static/app-pgdump.html) and drop it

  ```sql
  pg_dump -c -Fc -t <table>_retired $PGSLICE_URL > <table>_retired.dump
  psql -c "DROP <table>_retired" $PGSLICE_URL
  ```

## Sample Output

pgslice prints the SQL commands that were executed on the server. To print without executing, use the `--dry-run` option.

```sh
pgslice prep visits created_at month
```

```sql
BEGIN;

CREATE TABLE "public"."visits_intermediate" (LIKE "public"."visits" INCLUDING DEFAULTS INCLUDING CONSTRAINTS INCLUDING STORAGE INCLUDING COMMENTS) PARTITION BY RANGE ("created_at");

CREATE INDEX ON "public"."visits_intermediate" USING btree ("created_at");

COMMENT ON TABLE "public"."visits_intermediate" is 'column:createdAt,period:day,cast:date,version:3';

COMMIT;
```

```sh
pgslice add_partitions visits --intermediate --past 1 --future 1
```

```sql
BEGIN;

CREATE TABLE "public"."visits_201808" PARTITION OF "public"."visits_intermediate" FOR VALUES FROM ('2018-08-01') TO ('2018-09-01');

ALTER TABLE "public"."visits_201808" ADD PRIMARY KEY ("id");

CREATE TABLE "public"."visits_201809" PARTITION OF "public"."visits_intermediate" FOR VALUES FROM ('2018-09-01') TO ('2018-10-01');

ALTER TABLE "public"."visits_201809" ADD PRIMARY KEY ("id");

CREATE TABLE "public"."visits_201810" PARTITION OF "public"."visits_intermediate" FOR VALUES FROM ('2018-10-01') TO ('2018-11-01');

ALTER TABLE "public"."visits_201808" ADD PRIMARY KEY ("id");

COMMIT;
```

```sh
pgslice fill visits
```

```sql
/* 1 of 3 */
INSERT INTO "public"."visits_intermediate" ("id", "user_id", "ip", "created_at")
    SELECT "id", "user_id", "ip", "created_at" FROM "public"."visits"
    WHERE "id" > 0 AND "id" <= 10000 AND "created_at" >= '2018-08-01'::date AND "created_at" < '2018-11-01'::date

/* 2 of 3 */
INSERT INTO "public"."visits_intermediate" ("id", "user_id", "ip", "created_at")
    SELECT "id", "user_id", "ip", "created_at" FROM "public"."visits"
    WHERE "id" > 10000 AND "id" <= 20000 AND "created_at" >= '2018-08-01'::date AND "created_at" < '2018-11-01'::date

/* 3 of 3 */
INSERT INTO "public"."visits_intermediate" ("id", "user_id", "ip", "created_at")
    SELECT "id", "user_id", "ip", "created_at" FROM "public"."visits"
    WHERE "id" > 20000 AND "id" <= 30000 AND "created_at" >= '2018-08-01'::date AND "created_at" < '2018-11-01'::date
```

```sh
pgslice analyze visits
```

```sql
ANALYZE VERBOSE "public"."visits_201808";

ANALYZE VERBOSE "public"."visits_201809";

ANALYZE VERBOSE "public"."visits_201810";

ANALYZE VERBOSE "public"."visits_intermediate";
```

```sh
pgslice swap visits
```

```sql
BEGIN;

SET LOCAL lock_timeout = '5s';

ALTER TABLE "public"."visits" RENAME TO "visits_retired";

ALTER TABLE "public"."visits_intermediate" RENAME TO "visits";

ALTER SEQUENCE "visits_id_seq" OWNED BY "public"."visits"."id";

COMMIT;
```

## Adding Partitions

To add partitions, use:

```sh
pgslice add_partitions <table> --future 3
```

Add this as a cron job to create a new partition each day, month, or year.

```sh
# day
0 0 * * * pgslice add_partitions <table> --future 3 --url ...

# month
0 0 1 * * pgslice add_partitions <table> --future 3 --url ...

# year
0 0 1 1 * pgslice add_partitions <table> --future 3 --url ...
```

Add a monitor to ensure partitions are being created.

```sql
SELECT 1 FROM
    pg_catalog.pg_class c
INNER JOIN
    pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE
    c.relkind = 'r' AND
    n.nspname = 'public' AND
    c.relname = '<table>_' || to_char(NOW() + INTERVAL '3 days', 'YYYYMMDD')
    -- for months, use to_char(NOW() + INTERVAL '3 months', 'YYYYMM')
    -- for years, use to_char(NOW() + INTERVAL '3 years', 'YYYY')
```

## Archiving Partitions

Back up and drop older partitions each day, month, or year.

```sh
pg_dump -c -Fc -t <table>_201809 $PGSLICE_URL > <table>_201809.dump
psql -c "DROP <table>_201809" $PGSLICE_URL
```

If you use [Amazon S3](https://aws.amazon.com/s3/) for backups, [s3cmd](https://github.com/s3tools/s3cmd) is a nice tool.

```sh
s3cmd put <table>_201809.dump s3://<s3-bucket>/<table>_201809.dump
```

## Additional Commands

To undo prep (which will delete partitions), use:

```sh
pgslice unprep <table>
```

To undo swap, use:

```sh
pgslice unswap <table>
```

## App Considerations

This set up allows you to read and write with the original table name with no knowledge it’s partitioned. However, there are a few things to be aware of.

### Reads

When possible, queries should include the column you partition on to limit the number of partitions the database needs to check. For instance, if you partition on `created_at`, try to include it in queries:

```sql
SELECT * FROM
    visits
WHERE
    user_id = 123 AND
    -- for performance
    created_at >= '2018-09-01' AND created_at < '2018-09-02'
```

For this to be effective, ensure `constraint_exclusion` is set to `partition` (default value) or `on`.

```sql
SHOW constraint_exclusion;
```

### Writes

Before Postgres 10, if you use `INSERT` statements with a `RETURNING` clause (as frameworks like Rails do), you’ll no longer receive the id of the newly inserted record(s) back. If you need this, you can either:

1. Insert directly into the partition
2. Get value before the insert with `SELECT nextval('sequence_name')` (for multiple rows, append `FROM generate_series(1, n)`)

## One Off Tasks

You can also use pgslice to reduce the size of a table without partitioning by creating a new table, filling it with a subset of records, and swapping it in.

```sh
pgslice prep <table> --no-partition
pgslice fill <table> --where "id > 1000" # use any conditions
pgslice swap <table>
```

## Schema Updates

Once a table is partitioned, here’s how to change the schema:

To add, remove, or modify a column, make the update on the master table only.

To add or remove an index or foreign key:

- For Postgres 11+, make the update on the master table only.
- For Postgres 10, make the update on partitions only.
- For Postgres < 10, make the update on the master table and all partitions.

## Declarative Partitioning

Postgres 10 introduces [declarative partitioning](https://www.postgresql.org/docs/10/static/ddl-partitioning.html#ddl-partitioning-declarative). A major benefit is `INSERT` statements with a `RETURNING` clause work as expected. If you prefer to use trigger-based partitioning instead (not recommended), pass the `--trigger-based` option to the `prep` command.

## Data Protection

Always make sure your [connection is secure](https://ankane.org/postgres-sslmode-explained) when connecting to your database over a network you don’t fully trust. Your best option is to connect over SSH or a VPN. Another option is to use `sslmode=verify-full`. If you don’t do this, your database credentials can be compromised.

## Dependencies

If installation fails, your system may be missing Ruby or libpq.

On Mac, run:

```sh
brew install postgresql
```

On Ubuntu, run:

```sh
sudo apt-get install ruby-dev libpq-dev build-essential
```

## Upgrading

Run:

```sh
gem install pgslice
```

To use master, run:

```sh
gem install specific_install
gem specific_install https://github.com/ankane/pgslice.git
```

## Docker

```sh
docker build -t pgslice .
alias pgslice="docker run --rm -e PGSLICE_URL pgslice"
```

This will give you the `pgslice` command.

## Reference

- [PostgreSQL Manual](https://www.postgresql.org/docs/current/static/ddl-partitioning.html)
- [PostgreSQL Wiki](https://wiki.postgresql.org/wiki/Table_partitioning)

## Related Projects

Also check out:

- [PgHero](https://github.com/ankane/pghero) - A performance dashboard for Postgres
- [pgsync](https://github.com/ankane/pgsync) - Sync Postgres data to your local machine

## Contributing

Everyone is encouraged to help improve this project. Here are a few ways you can help:

- [Report bugs](https://github.com/ankane/pgslice/issues)
- Fix bugs and [submit pull requests](https://github.com/ankane/pgslice/pulls)
- Write, clarify, or fix documentation
- Suggest or add new features

To run tests, do:

```sh
git clone https://github.com/ankane/pgslice.git
cd pgslice
bundle install
createdb pgslice_test
bundle exec rake
```

To test against different versions of Postgres with Docker, use:

```sh
docker run -p=8000:5432 postgres:10
TZ=Etc/UTC PGSLICE_URL=postgres://postgres@localhost:8000/postgres bundle exec rake
```

On Mac, you must use [Docker for Mac](https://www.docker.com/docker-mac) for the port mapping to localhost to work.
