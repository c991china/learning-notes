# Postgres tips

Postgres 15/16. Notes from running a small OLTP database and debugging slow
queries at 2am. Nothing here is exotic; it's the 20% I use constantly.

## Getting oriented

```sql
SELECT version();
SELECT current_database(), current_user;
\d+ orders            -- table structure, indexes, size (psql)
\di                   -- list indexes
\df                   -- list functions
```

```sql
-- table sizes, biggest first
SELECT relname,
       pg_size_pretty(pg_total_relation_size(relid)) AS total
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;
```

## EXPLAIN: always use ANALYZE when it's safe

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

`EXPLAIN` alone shows the plan; `EXPLAIN ANALYZE` actually runs it and shows
real timings. The gap between estimated rows and actual rows is the whole game.

```
Seq Scan on orders  (cost=0.00..18334.00 rows=1 width=...) (actual time=0.015..95.221 rows=8420 loops=1)
```

> **gotcha**: `rows=1` estimated but `rows=8420` actual means the planner's stats
> are stale or the predicate is unindexable. Run `ANALYZE orders;` and re-check.
> A bad estimate is why Postgres picks a nested loop over a hash join and the
> query takes 30 seconds.

Look for:
- `Seq Scan` on a big table with a selective filter -> probably missing index.
- `rows=` estimate wildly off from actual -> run `ANALYZE`.
- `Nested Loop` with high `loops=` -> consider an index on the join key.

## Index gotchas

Indexes aren't free; each one slows writes and eats disk.

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders (customer_id);
```

> **gotcha**: use `CONCURRENTLY` in production. Plain `CREATE INDEX` takes an
> ACCESS EXCLUSIVE lock and blocks writes for the duration. `CONCURRENTLY`
> can't run inside a transaction block, and if it fails it leaves an INVALID
> index you must drop and retry.

The classic "why isn't my index used" traps:

- **Function on the column**: `WHERE lower(email) = 'a@b.com'` won't use a plain
  index on `email`. Create `CREATE INDEX ... ON users (lower(email));`.
- **Type mismatch**: `WHERE id = '42'` where `id` is int can force a cast. Keep
  types aligned.
- **Leading wildcard**: `LIKE '%foo'` can't use a btree. Use a trigram index
  (`pg_trgm`) if you need it.
- **Low selectivity**: an index on a boolean column is usually ignored; the
  planner would rather scan.

Partial index for the common case:

```sql
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';
```

## Locks and blocked queries

```sql
-- who is blocked, and by whom
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';

-- find the blocking pid
SELECT blocked.pid AS blocked_pid,
       blocking.pid AS blocking_pid,
       blocked.query AS blocked_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks kl ON kl.locktype = bl.locktype
  AND kl.relation = bl.relation AND kl.pid <> bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid
WHERE NOT bl.granted;
```

Then, if you must:

```sql
SELECT pg_terminate_backend(12345);   -- kill the blocking session
```

> **gotcha**: `pg_terminate_backend` on the wrong pid kills your own session or a
> critical job. Double-check the pid and read the query text first. It's the
> Postgres equivalent of `kill -9`.

Long-running queries:

```sql
SELECT pid, now() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE state <> 'idle' AND now() - query_start > interval '1 minute'
ORDER BY duration DESC;
```

## Vacuum and bloat

Autovacuum usually keeps up. When it doesn't, tables bloat.

```sql
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 10;
```

```sql
VACUUM (VERBOSE, ANALYZE) orders;   -- non-blocking, safe
VACUUM FULL orders;                 -- rewrites the table, takes an exclusive lock
```

> **gotcha**: `VACUUM FULL` locks the table for writes for the whole rewrite. On
> a big table that's an outage. Prefer `pg_repack` if you can install it.

## Handy

```sql
-- current connections vs limit
SELECT count(*), (SELECT setting::int FROM pg_settings WHERE name='max_connections')
FROM pg_stat_activity;

-- kill all idle connections older than 10 min
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE state = 'idle' AND state_change < now() - interval '10 minutes';
```

## Notes

- `count(*)` on a large table is a full scan in Postgres (MVCC). If you need an
  approximate count, use an estimate from `pg_class.reltuples`.
- `timestamptz` stores UTC; `timestamp` doesn't. Always use `timestamptz` unless
  you have a very specific reason. I learned this the hard way with DST.
- Set `statement_timeout` per session so a runaway query can't pin a connection
  forever: `SET statement_timeout = '5s';`.
