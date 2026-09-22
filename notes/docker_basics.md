# Docker basics

Notes from using Docker 24.0 on Ubuntu 22.04 and Docker Desktop 4.28 on macOS 14.
If you're on Windows, mostly the same, except volume paths are a different kind
of pain.

## Images vs containers (the mental model that finally stuck)

- An **image** is a read-only template. Layers, content-addressed.
- A **container** is a running (or stopped) instance of an image plus a thin
  writable layer on top.
- Deleting a container does NOT delete the image. Deleting the image does NOT
  free the container's writable layer until the container is gone.

That last point is why `docker rmi` sometimes says "image is being used by
stopped container".

## Everyday commands

```bash
docker ps -a                 # all containers, including stopped
docker images                # local images
docker logs -f --tail 100 app   # follow last 100 lines
docker exec -it app bash        # get a shell inside a running container
docker inspect app | jq '.[0].State'   # is it actually running?
```

`docker exec -it` fails with `exec: "bash": executable file not found` on slim
images (alpine, distroless). Use `sh`:

```bash
docker exec -it app sh
```

## Cleanup: the honest version

`docker system df` is the reality check I run first. On my laptop it once showed
38GB of build cache I didn't know existed.

```bash
docker system df
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          41        9         21.4GB    15.1GB (70%)
# Containers      17        2         88MB      61MB (69%)
# Local Volumes   12        3         1.2GB     900MB (75%)
# Build Cache     203       0         38.1GB    38.1GB
```

Then prune, in increasing order of how much you'll regret it:

```bash
docker container prune          # removes stopped containers
docker image prune              # dangling images only
docker image prune -a           # ALL unused images (be careful)
docker builder prune            # build cache
docker system prune -a --volumes  # nukes basically everything unused
```

> **gotcha**: `docker system prune --volumes` deletes unused named volumes.
> Unused means "not attached to any container". If you stopped a Postgres
> container to save RAM and forgot, your data volume can be "unused". I lost a
> local test DB this way. Named volumes are not in your git repo. Slow down.

## Volumes

```bash
docker volume ls
docker volume create pgdata
docker run -d --name pg -v pgdata:/var/lib/postgresql/data postgres:16
```

Bind mount vs volume:
- Bind mount (`-v /host/path:/container/path`) — you control the path. Good for
  source code in dev. Bad permissions stories on macOS (slow, and file ownership
  is weird).
- Named volume (`-v pgdata:/data`) — Docker manages it. Good for databases.

> **gotcha**: on macOS, bind mounts over `virtiofs` are fast, but `gRPC-FUSE`
> (older default) is slow for node_modules. If your build is 10x slower inside
> Docker than outside, it's probably this.

## Networking, briefly

```bash
docker network ls
docker network inspect bridge
# containers on the same user-defined network resolve each other by name
docker run -d --name db --network mynet postgres:16
docker run --rm --network mynet curlimages/curl -s http://db:5432
```

Containers on the default `bridge` network do NOT get DNS by name. You need a
user-defined network for that. This trips everyone once.

```bash
docker network create mynet
```

## Building

```bash
docker build -t myapp:dev .
docker build --no-cache -t myapp:dev .    # when the cache lies to you
docker build --progress=plain -t myapp .  # see the real build output
```

Multi-stage build, because shipping the compiler is embarrassing:

```dockerfile
FROM python:3.11-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.11-slim
COPY --from=build /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY . /app
WORKDIR /app
CMD ["python", "-m", "myapp"]
```

## Notes

- `docker compose` (v2, no hyphen) is the plugin. `docker-compose` (v1) is the
  old standalone Python script. If `docker compose` says "is not a docker
  command", your plugin isn't installed.
- `--rm` on `docker run` auto-removes the container on exit. Great for one-off
  commands, terrible if you wanted the logs.
- Container exit code 137 = SIGKILL, usually OOM. Check `docker inspect <c>
  --format '{{.State.OOMKilled}}'`.
