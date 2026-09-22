# Troubleshooting grab-bag

A running list of "this exact error, this exact fix". If you found this via a
search engine, you're probably dealing with one of these right now.

## `Address already in use`

```
OSError: [Errno 98] Address already in use
```

Something is already bound to the port. Find it:

```bash
sudo lsof -i :8000
sudo ss -tulpn 'sport = :8000'
```

Kill it, or pick another port. If `lsof` shows nothing but the error persists,
you may have a process in `TIME_WAIT`:

```bash
ss -tan state time-wait | grep 8000
```

Set `SO_REUSEADDR` on your socket, or wait ~60s. Most frameworks already set it;
if you're writing raw sockets, you have to.

> **gotcha**: on macOS, AirPlay Receiver (ControlCenter) squats on port 5000 and
> 7000. "Address already in use" on 5000 with no obvious process is usually this.
> Disable it in System Settings > General > AirDrop & Handoff.

## `Permission denied (publickey)` over SSH

```bash
ssh -v user@host        # -v, -vv, -vvv for more
```

Usual causes, in order:
1. Wrong key. `ssh-add -l` and check `-i` path.
2. Key permissions too open. `chmod 600 ~/.ssh/id_ed25519` and `chmod 700 ~/.ssh`.
3. The server doesn't have your public key in `~/.ssh/authorized_keys`.
4. `~/.ssh` on the server has wrong ownership (not yours).

> **gotcha**: SSH refuses to use a private key that's group/world readable. It
> often says nothing useful about why. If `-vvv` says "ignoring key", it's
> permissions.

## `ModuleNotFoundError` even though it's installed

You installed into a different Python than the one running. Check:

```bash
which python && python -c 'import sys; print(sys.executable)'
pip -V            # which pip, and for which python
```

Fix by using `python -m pip install ...` so pip and python can't disagree. If
you're in a venv, confirm it's activated (`echo $VIRTUAL_ENV`).

> **gotcha**: `pip` and `python` pointing at different interpreters is the most
> common cause of "I installed it, I swear". `python -m pip` removes the doubt.

## `No space left on device` but `df` says there's space

```bash
df -h
df -i          # inodes! often the real problem
```

If inodes are exhausted (millions of tiny files, e.g. a runaway cache or mail
queue), you're out even with disk free. Find the directory with a huge file
count:

```bash
sudo find /var -xdev -type f | wc -l      # careful, slow on big trees
```

Another cause: deleted files still held open by a process.

```bash
sudo lsof +L1 | head        # files with link count 0 (deleted but open)
```

Restart the holding process to free the space. Classic with log files deleted
while a daemon still has them open.

## `Connection refused` from a container to another container

1. Both on the same user-defined network? (default `bridge` has no DNS by name)
2. Using the service name, not `localhost`?
3. The target is actually listening on `0.0.0.0`, not `127.0.0.1`?

```bash
docker exec -it app sh -c 'nc -vz db 5432'
```

`localhost` inside a container means the container itself, not the host and not
another container. This trips up almost everyone once.

## YAML: `found character '\t' that cannot start any token`

A literal tab character in YAML. YAML only allows spaces for indentation.

```bash
grep -Pn '\t' file.yaml
```

Most editors have "insert spaces for tabs" somewhere. Turn it on. I also run
`yamllint` in pre-commit so this never reaches CI.

## Python: `TypeError: Object of type datetime is not JSON serializable`

`json.dumps` doesn't know how to encode `datetime`.

```python
import json
from datetime import datetime

def default(o):
    if isinstance(o, datetime):
        return o.isoformat()
    raise TypeError(f"not serializable: {type(o)}")

json.dumps({"t": datetime.now()}, default=default)
```

I keep this `default=` helper around. It's small and saves the same fix twice a
month.

## `kubectl`: `error: You must be logged in to the server (Unauthorized)`

Your kubeconfig context is stale (token expired or wrong cluster). Check:

```bash
kubectl config current-context
kubectl config view --minify
```

Re-authenticate (e.g. `aws eks update-kubeconfig --name <cluster>`) or switch
context. Usually it's "I'm pointed at the wrong cluster".

## `npm`: `EACCES: permission denied` on a global install

You're trying to write to a root-owned global dir without sudo, or you used
sudo before and now own nothing.

```bash
npm config get prefix
```

Fix properly: use a Node version manager (`nvm`) or set the prefix to a
user-owned dir. `sudo npm install -g` is how people end up with this problem in
the first place. Avoid it.

## Notes

- When stuck, the fastest move is usually: reproduce with the smallest possible
  command, add `-v`/`--verbose`, read the FIRST error not the last. Stack traces
  cascade; the first line is the cause.
- `strace -f -e trace=network,openat <cmd>` answers "what is this binary
  actually doing". Noisy but powerful.
- Write down the fix when you find it. That's literally why this repo exists.
