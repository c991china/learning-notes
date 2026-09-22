# learning-notes

My own study notes. Not a tutorial series, not a course. Just the stuff I keep
forgetting and the stuff that bit me once and I never want to debug twice.

I started this because I kept re-googling the same six things (how to undo a
`git commit`, why `docker system df` shows 40GB of garbage, why `asyncio.gather`
swallows an exception) and reading the same three blog posts. Now I write it down
once, badly, and fix it later.

## Index

| Note | What's in it | Last touched |
|------|--------------|--------------|
| [docker_basics](notes/docker_basics.md) | images vs containers, cleanup, volumes, the `system df` reality check | 2026-08 |
| [git_workflow](notes/git_workflow.md) | branch/switch/rebase, undo recipes, reflog recovery | 2026-09 |
| [python_async](notes/python_async.md) | asyncio.gather vs TaskGroup, cancellation, the blocking-call trap | 2026-07 |
| [linux_networking](notes/linux_networking.md) | ss/ip/tcpdump, DNS resolution order, ports in TIME_WAIT | 2026-06 |
| [postgres_tips](notes/postgres_tips.md) | EXPLAIN, index gotchas, locks, `pg_stat_activity` | 2026-08 |
| [kubernetes_101](notes/kubernetes_101.md) | pods/deployments/services, probes, crashloop debugging | 2026-05 |
| [redis_notes](notes/redis_notes.md) | data types, TTL pitfalls, keyspace, persistence basics | 2026-04 |
| [troubleshooting](notes/troubleshooting.md) | grab-bag of "this one weird error" fixes | 2026-09 |

## How I use these

- Each note is standalone. No required reading order.
- Commands are copy-pasteable but I note the OS when it matters (mostly Linux;
  some macOS; Windows only where I actually tested it).
- `gotcha` blocks are things that cost me real time. Read those first if you're
  skimming.
- Versions in the notes are whatever I was running at the time. I try to keep
  them honest rather than "latest".

## Caveats

- This is a personal notebook. It is not reviewed, not exhaustive, and probably
  wrong somewhere. If a command deletes your data, that's on you to check first.
- I don't cover Windows PowerShell equivalents for most of the Linux stuff
  because I don't use it daily and I'd rather leave a gap than write something
  untested.

## Conventions

```
$ command          # a command I run
# output           # what it printed (trimmed)
```

Comments marked `# NOTE:` are reminders to future me. `# FIXME:` means the note
itself is incomplete.
