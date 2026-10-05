# Computer Networks: Team 3
## Private Network Service Platform

A local, no-cloud networking project on three macOS laptops: private DNS, an nginx HTTPS edge with load balancing, and two HTTP backends.

**URL:** `https://app.team3.test:8443/`

---

## Team

| Member | Enrollment No. | Machine | Role | IP |
|---|---|---|---|---|
| Shashwat | `2401010438` | Mac 1 | Private DNS (dnsmasq) | `10.7.21.46` |
| Adamya Tiwari | `2401010025` | Mac 2 | nginx edge / HTTPS / load balancer | `10.7.12.54` |
| Aditi Singh | `2401020006` | Mac 3 | Backend A + Backend B | `10.7.18.8` |

---

## Architecture

```
Client
  |  1. DNS query (UDP/53)
  v
Mac 1: dnsmasq (10.7.21.46)
  |  app.team3.test -> 10.7.12.54
  v
Mac 2: nginx (10.7.12.54:8443, TLS terminated)
  |  round-robin
  +--> Backend A  10.7.18.8:3001  (X-Backend: A)
  +--> Backend B  10.7.18.8:3002  (X-Backend: B)
```

| Component | Technology |
|---|---|
| DNS | dnsmasq (`.test` domain) |
| Edge / load balancer | nginx upstream, round-robin |
| TLS | mkcert certificate, TLS 1.3 |
| Backends | Python HTTP services |
| Analysis | Wireshark, curl, dig |

---

## How to Run

### 1. Backends (Mac 3, 10.7.18.8)
Each backend must listen on the LAN IP, not just `127.0.0.1`.

```bash
python3 <backend_a_file>.py   # listens on 10.7.18.8:3001
python3 <backend_b_file>.py   # listens on 10.7.18.8:3002
```

Verify from Mac 2:
```bash
curl http://10.7.18.8:3001/    # {"backend": "A", "status": "running"}
curl http://10.7.18.8:3002/    # {"backend": "B", "status": "running"}
```

**Endpoints:** `GET /` and `GET /api/status` (returns JSON and an `X-Backend` header; `/api/status` also sends `Cache-Control: max-age=60` and `ETag`).

### 2. DNS (Mac 1)
dnsmasq maps `app.team3.test` and `api.team3.test` to `10.7.12.54`.

```bash
dig @10.7.21.46 app.team3.test    # A 10.7.12.54
dig @8.8.8.8 app.team3.test       # NXDOMAIN (private only)
```

### 3. nginx edge (Mac 2)
Upstream group (`max_fails=1 fail_timeout=5s`) with both backends, HTTPS on port 8443, and `proxy_next_upstream` for failover.

```bash
nginx -t          # check config
nginx             # start
nginx -s reload   # apply changes
```

---

## Verification

```bash
# HTTPS with certificate validation (never use -k)
curl -v https://app.team3.test:8443/

# Load balancing: A and B should both appear
for i in {1..6}; do
  curl -s -D - https://app.team3.test:8443/api/status -o /dev/null | grep X-Backend
done

# Caching headers
curl -sI https://app.team3.test:8443/api/status
```

**Failover:** stop Backend A and rerun the loop. All responses should come from B. Restart A and both return.

**Wireshark filters:** `dns`, `tcp.flags.syn==1`, `tls`

---

## Troubleshooting Order
LAN (`ping`) -> DNS (`dig`) -> Backends (`curl` on 3001/3002) -> nginx (`nginx -t`) -> HTTPS (`curl -v`) -> load balancing.
