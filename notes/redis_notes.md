# Redis notes

Redis 7.2. I use it as a cache and for a few small counters/queues. These notes
are from that use, not from running a big Redis cluster.

## Connect and poke

```bash
redis-cli -h 127.0.0.1 -p 6379
redis-cli -u redis://:password@host:6379/0
```

```redis
PING                 -- PONG
INFO server          -- version, uptime
DBSIZE               -- number of keys in current db
SELECT 1             -- switch database (0-15 by default)
```

> **gotcha**: `DBSIZE` only counts the current database. If you're "missing"
> keys, check you're in the right `SELECT` index. I've lost time to keys sitting
> in db 1 while I looked at db 0.

## Data types and what I use them for

```redis
-- strings: cache values, counters
SET session:abc "data" EX 3600
GET session:abc
INCR pageviews:home              -- atomic counter
SETNX lock:job1 "1"              -- simple lock, returns 0 if it existed

-- hashes: objects without serializing
HSET user:42 name "ann" email "a@b.com"
HGETALL user:42
HINCRBY user:42 logins 1

-- lists: queues (LPUSH + BRPOP), recent items
LPUSH queue:emails "job1"
BRPOP queue:emails 5             -- block up to 5s waiting for an item

-- sets: unique membership, tags
SADD post:7:tags go redis
SMEMBERS post:7:tags

-- sorted sets: leaderboards, rate limiting windows
ZADD leaderboard 100 "alice"
ZREVRANGE leaderboard 0 9 WITHSCORES

-- streams: proper event log (better than lists for queues)
XADD events '*' type signup user 42
XREAD COUNT 10 BLOCK 5000 STREAMS events '$'
```

## TTL: the part that bites

```redis
EXPIRE session:abc 3600
TTL session:abc            -- seconds remaining, -1 = no expiry, -2 = missing
PERSIST session:abc        -- remove expiry
```

> **gotcha**: `SET key value` on an existing key wipes its TTL. `SET k v EX 60`
> resets it every call, which is usually what you want for a sliding cache but
> NOT for a hard session timeout. To keep the original TTL, use `SET k v KEEPTTL`
> (6.0+). I've had sessions silently live forever because of a bare `SET`.

> **gotcha**: Redis expires keys lazily (on access) and via a background sweep.
> A key can exist in memory past its TTL and only vanish when touched. Don't use
> Redis as a precise timer; use it as an approximate expiry.

## Keyspace: never use KEYS in production

```redis
KEYS user:*                 -- BLOCKS the whole server. do not.
SCAN 0 MATCH user:* COUNT 100    -- cursor-based, non-blocking
```

> **gotcha**: `KEYS` is O(N) and single-threaded, so it stalls every other
> client. On a production instance with millions of keys it can hang for
> seconds. Always `SCAN`. `SCAN` may return duplicates and doesn't guarantee a
> full snapshot, so dedupe client-side if it matters.

## Atomic patterns

Increment with a cap:

```redis
-- Lua runs atomically on the server
EVAL "local v = redis.call('INCR', KEYS[1]); if v > tonumber(ARGV[1]) then redis.call('DECR', KEYS[1]); return 0 end; return v" 1 rate:user:42 100
```

Compare-and-delete (safe lock release):

```redis
-- only delete if we still own the lock
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:job1 <token>
```

A plain `DEL lock:job1` can delete someone else's lock if yours already expired.
The Lua version checks ownership first.

## Memory

```redis
INFO memory
# used_memory_human: 42.11M
# maxmemory_human: 256.00M
MEMORY USAGE user:42
```

Eviction policy matters a lot:

```redis
CONFIG GET maxmemory-policy
-- allkeys-lru / volatile-lru / noeviction ...
```

> **gotcha**: the default is `noeviction`. When Redis hits `maxmemory` with
> `noeviction`, writes fail with `OOM command not allowed when used memory >
> 'maxmemory'`. If you're using Redis as a cache, you want `allkeys-lru` (or
> `allkeys-lfu` on 4.0+). If you use it as a store, `noeviction` is correct and
> the error is a real signal. Know which one you are.

## Persistence, in one paragraph

RDB = periodic snapshots, fast restart, can lose recent writes. AOF = append
every write, more durable, bigger files, slower. Default config usually has RDB
on and AOF off. For a pure cache, persistence off is fine. For anything you'd
cry about losing, turn AOF on (`appendonly yes`).

## Notes

- Keys are binary-safe but keep them short and namespaced (`app:entity:id`).
- `MGET`/`MSET` batch round-trips. A loop of `GET` over a network is 100x slower.
- Pipelines batch commands without waiting for each reply:
  `redis-cli --pipe` or a client-side pipeline. Big latency win.
- Redis is single-threaded for command execution. One slow Lua script or `KEYS`
  blocks everyone. Keep scripts tiny.
