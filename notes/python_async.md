# Python asyncio notes

Python 3.11 (also checked against 3.12). I write async code for API clients and
scrapers, not for CPU work. If you're doing CPU work, async is the wrong tool.

## The one rule

`async def` does not make anything faster by itself. It only helps when you're
waiting on I/O. `await time.sleep(1)` inside an async function still blocks the
whole event loop.

```python
import asyncio, time

async def bad():
    time.sleep(1)      # BLOCKS the event loop. everything else stalls.

async def good():
    await asyncio.sleep(1)   # yields control back
```

> **gotcha**: a single blocking call (`requests.get`, `time.sleep`, a heavy
> pandas operation) inside an async function stalls every other task. If a
> library is sync-only, run it in a thread:
> `await asyncio.to_thread(requests.get, url)` (3.9+).

## gather vs TaskGroup

`asyncio.gather` is the old standby. `asyncio.TaskGroup` (3.11+) is what I reach
for now because of how it handles failures.

```python
# gather: by default, one exception does NOT cancel the others,
# and you lose the ones that succeed unless you read results carefully.
results = await asyncio.gather(fetch(u) for u in urls)

# return_exceptions=True: you get exceptions as values, no raise.
results = await asyncio.gather(*tasks, return_exceptions=True)
for r in results:
    if isinstance(r, Exception):
        print("failed:", r)
```

```python
# TaskGroup: if one task raises, the rest are cancelled, and you get
# an ExceptionGroup. This is usually what you actually want.
async with asyncio.TaskGroup() as tg:
    t1 = tg.create_task(fetch(url1))
    t2 = tg.create_task(fetch(url2))
# after the block, t1.result() and t2.result() are ready
```

> **gotcha**: with plain `gather`, if the first URL raises, the other coroutines
> keep running in the background. If one of them also raises, you can get
> "exception was never retrieved" warnings at shutdown. I spent an hour
> confused by those. `TaskGroup` avoids this by cancelling siblings.

## Timeouts

```python
# 3.11+ idiomatic
async with asyncio.timeout(5):
    await fetch(url)

# older
await asyncio.wait_for(fetch(url), timeout=5)
```

`asyncio.timeout` raises `TimeoutError` and cancels the inner task. Note it's
the *3.11* builtin `TimeoutError`, not `asyncio.TimeoutError` (they merged).

## Semaphores: don't hammer the server

My scraper got rate-limited after ~50 concurrent requests. Fix:

```python
import asyncio

sem = asyncio.Semaphore(10)   # max 10 in flight

async def limited_fetch(session, url):
    async with sem:
        async with session.get(url) as resp:
            return await resp.text()

async def main(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [limited_fetch(session, u) for u in urls]
        return await asyncio.gather(*tasks)
```

## Cancellation

Cancellation is delivered as `asyncio.CancelledError`, which is a
`BaseException` since 3.8. Don't catch bare `Exception` and assume you're safe
from it.

```python
async def worker():
    try:
        await long_running()
    except asyncio.CancelledError:
        # cleanup, then re-raise. Swallowing it breaks cancellation.
        await cleanup()
        raise
```

> **gotcha**: `except Exception` will NOT catch `CancelledError` (good), but
> `except BaseException` will, and if you don't re-raise, your program hangs on
> shutdown. I've done this.

## Running things

```python
asyncio.run(main())        # entry point. don't call inside a running loop.
```

If you're already in a loop (Jupyter, some frameworks), `asyncio.run` raises
`RuntimeError: asyncio.run() cannot be called from a running event loop`. In
notebooks, just `await main()` directly.

## Debug mode

```bash
PYTHONASYNCIODEBUG=1 python app.py
```

It logs slow callbacks and "coroutine was never awaited". Also:
`asyncio.run(main(), debug=True)`. Turn this on when something mysteriously
hangs. It usually tells you which callback blocked the loop.

## Notes

- `asyncio.Queue` is not thread-safe. For cross-thread use `queue.Queue` +
  `asyncio.to_thread`, or `janus`.
- A `Task` must be kept alive (store a reference) or it can be garbage
  collected mid-flight. `tg.create_task` handles this for you; a bare
  `asyncio.create_task` without storing the result does not.
- `async for` over aiohttp streaming responses is much friendlier on memory
  than `await resp.text()` for big payloads.
