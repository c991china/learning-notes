# Linux networking notes

Debian/Ubuntu mostly, some notes for macOS. I'm not a network engineer; these
are the commands I actually use when "the service can't reach the database".

## First things I run

```bash
ip addr                    # my interfaces and IPs (replaces ifconfig)
ip route                   # routing table, default gateway
ss -tulpn                  # listening TCP/UDP sockets + owning process
```

`ss` replaced `netstat`. If `ss` isn't there, install `iproute2`. `netstat` is
deprecated but still ships on some boxes.

```bash
ss -tulpn
# Netid State  Local Address:Port  Process
# tcp   LISTEN 0.0.0.0:5432         users:(("postgres",pid=812,fd=7))
# tcp   LISTEN 127.0.0.1:6379        users:(("redis-server",pid=903))
```

Note the second line: Redis is bound to `127.0.0.1` only. That's why a remote
host gets connection refused. This is the #1 thing I check.

## "Can't connect" decision tree

1. Is it listening at all? `ss -tulpn | grep 5432`
2. Is it listening on the right interface? (`127.0.0.1` vs `0.0.0.0`)
3. Is the firewall dropping it? `sudo iptables -L -n` or `sudo ufw status`
4. Can I reach the host? `ping host` (ICMP may be blocked; not conclusive)
5. Can I reach the port? `nc -vz host 5432` or `timeout 2 bash -c '</dev/tcp/host/5432'`

> **gotcha**: connection refused vs timeout means different things.
> - **Refused** = something answered and said no. Nothing is listening, or a
>   firewall sent a reject.
> - **Timeout** = packets went into a black hole. Usually a firewall DROP rule,
>   or a security group, or wrong IP entirely.
> Confusing these wastes a lot of time. A timeout is almost never "the app is
> down"; it's the network in the way.

## DNS

```bash
dig example.com            # full answer
dig +short example.com     # just the A record
dig @1.1.1.1 example.com   # query a specific resolver
getent hosts example.com   # what the SYSTEM resolves (respects /etc/hosts)
```

> **gotcha**: `dig` queries DNS directly and ignores `/etc/hosts`. The app uses
> the system resolver, which DOES read `/etc/hosts` and `/etc/nsswitch.conf`.
> Use `getent hosts` to see what the app sees. I've debugged "DNS is broken"
> that was actually a stale `/etc/hosts` entry.

Resolution order on Linux (`/etc/nsswitch.conf`):

```
hosts: files dns
```

`files` first means `/etc/hosts` wins over DNS.

## tcpdump: see the actual packets

```bash
sudo tcpdump -i any -n port 5432
sudo tcpdump -i eth0 -n -c 20 host 10.0.0.5 and port 443
sudo tcpdump -i any -n -A port 80      # -A prints ASCII payload
sudo tcpdump -i any -n -w cap.pcap     # write to file, open in Wireshark
```

> **gotcha**: in a container, tcpdump on the host interface may not show
> container-to-container traffic. Use `nsenter` into the container's netns, or
> tcpdump inside the container. And on the host, `-i any` sees a lot; filter.

## Ports in TIME_WAIT

```bash
ss -tan state time-wait | wc -l
ss -tan state time-wait | head
```

TIME_WAIT is normal: the side that closes first waits ~60s (2*MSL) before
reusing the port. It only becomes a problem if you're opening thousands of
short connections and exhausting ephemeral ports.

```bash
sysctl net.ipv4.ip_local_port_range
# 32768	60999   -> ~28k ephemeral ports
```

Fixes: use keep-alive / connection pooling (real fix), or
`net.ipv4.tcp_tw_reuse=1` (band-aid).

## Useful one-liners

```bash
# what process is on port 8080
sudo lsof -i :8080
sudo ss -tulpn 'sport = :8080'

# is the port open from here?
nc -vz db.internal 5432

# watch a route
ip route get 8.8.8.8

# capture DNS queries
sudo tcpdump -i any -n port 53

# curl with timing breakdown
curl -o /dev/null -s -w 'dns:%{time_namelookup} conn:%{time_connect} tls:%{time_appconnect} total:%{time_total}\n' https://example.com
```

## Notes

- On macOS, `ip`/`ss` don't exist. Use `ifconfig`, `netstat -an`, `lsof -i`.
- `ping` uses ICMP; many cloud security groups block it. A failed ping does not
  mean the host is down.
- `localhost` resolving to `::1` (IPv6) instead of `127.0.0.1` breaks services
  that only bind IPv4. Symptom: `curl localhost:5432` refused but
  `curl 127.0.0.1:5432` works. Check `/etc/hosts` for a `::1 localhost` line.
- `curl -v` is usually faster than tcpdump for "is TLS even happening".
