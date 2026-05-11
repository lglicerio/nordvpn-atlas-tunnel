# nordvpn-atlas-tunnel

Docker stack that runs a NordVPN (OpenVPN) tunnel through a dedicated IP and
exposes a SOCKS5 proxy on `127.0.0.1:1080` so MongoDB Atlas clusters protected
by an IP allow-list become reachable from both the host and other containers.

## How it works

```
  ┌──────────────────────────┐         ┌──────────────────────────────┐
  │ host                     │         │ docker                       │
  │                          │         │                              │
  │  mongosh / Compass ──────┼─SOCKS5──┼─► nordvpn-socks5 ─┐          │
  │                          │  :1080  │                   │ shares   │
  │  app container ──────────┼─────────┼─► (network_mode:  │ netns    │
  │                          │         │       service:vpn)│          │
  │                          │         │                   ▼          │
  │                          │         │              ┌────────────┐  │
  │                          │         │              │ nordvpn    │  │
  │                          │         │              │ (Gluetun,  │──┼──► NordVPN
  │                          │         │              │  OpenVPN)  │  │    dedicated IP
  │                          │         │              └────────────┘  │      │
  └──────────────────────────┘         └──────────────────────────────┘      ▼
                                                                       MongoDB Atlas
                                                                       (IP allow-list)
```

* `vpn` (Gluetun) holds the OpenVPN tunnel to a specific NordVPN dedicated-IP server.
* `socks5` shares the VPN container's network namespace and serves SOCKS5 on port 1080.
* Mongo drivers connect via `?proxyHost=127.0.0.1&proxyPort=1080` so DNS and TCP
  travel through the VPN, while TLS stays end-to-end to the real Atlas shard hosts.
* Any other container can drop into the same tunnel with `network_mode: "service:vpn"`
  and connect to Atlas with the unmodified `mongodb+srv://…` URI.

## Why OpenVPN and not NordLynx (WireGuard)

NordVPN's dedicated-IP plan currently only exposes OpenVPN for these accounts —
the dashboard's Manual Setup page does not offer a WireGuard config, and the
NordVPN API does not return WireGuard endpoints for the dedicated-IP group.
We use OpenVPN because that's what works; the rest of the stack is identical.

## Prerequisites

* Docker + Docker Compose (Docker Desktop is fine on macOS).
* An active NordVPN subscription with the **Dedicated IP** add-on.
* The NordVPN **service credentials** (auto-generated, different from your
  account login) from <https://my.nordaccount.com/> → NordVPN → Set up
  NordVPN manually → Service credentials.
* The `.ovpn` file NordVPN gives you for your dedicated IP — needed once to
  read the server hostname out of the `remote` line.

## Setup

```bash
cp .env.example .env
```

Edit `.env`:

| Variable           | Where it comes from                                                                                                     |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `NORDVPN_USER`     | Service credentials → username (long random string).                                                                    |
| `NORDVPN_PASSWORD` | Service credentials → password.                                                                                         |
| `NORDVPN_SERVER`   | Hostname from the `remote <hostname> <port>` line in your dedicated-IP `.ovpn` file, e.g. `us9999.nordvpn.com`.         |
| `TZ`               | Optional. IANA TZ string, e.g. `Europe/London`.                                                                         |

`.env` is gitignored — credentials never leave your machine.

## Run

```bash
docker compose pull
docker compose up -d
docker compose logs -f vpn
```

Successful startup looks like:

```
INFO [openvpn] OpenVPN 2.6 …
INFO [openvpn] Initialization Sequence Completed
INFO [healthcheck] healthy!
INFO [vpn] You are running with the dedicated IP …
```

## Verify

The egress IP from inside the VPN container should match your assigned
NordVPN dedicated IP — this is the value you'll add to Atlas Network Access:

```bash
docker compose exec vpn sh -c 'wget -qO- https://ifconfig.me; echo'
```

And the same IP should appear when going through the SOCKS proxy from the host:

```bash
curl --socks5-hostname 127.0.0.1:1080 https://ifconfig.me
```

If both match the dedicated IP from your NordVPN dashboard, the tunnel is ready.

## Whitelist the IP in MongoDB Atlas

For **each** cluster you need to reach:

1. Atlas → Project → **Network Access** → **+ ADD IP ADDRESS**.
2. Paste the dedicated IP as a `/32`.
3. Add a comment like `nordvpn-dedicated-ip` so it's obvious later.

The whitelist applies to all shards in the cluster automatically.

## Connecting clients

The same SOCKS proxy serves both clusters; the cluster is chosen by the
hostname in the connection string.

### `mongosh` (host)

```bash
mongosh "mongodb+srv://USER:PASS@<cluster-01>.<atlas-id>.mongodb.net/?proxyHost=127.0.0.1&proxyPort=1080"
mongosh "mongodb+srv://USER:PASS@<cluster-02>.<atlas-id>.mongodb.net/?proxyHost=127.0.0.1&proxyPort=1080"
```

### MongoDB Compass

New Connection → **Advanced Connection Options** → **Proxy/SSH** tab →
**SOCKS5** → host `127.0.0.1`, port `1080`, no username/password.

### Node.js driver

```js
new MongoClient("mongodb+srv://USER:PASS@<cluster-01>.<atlas-id>.mongodb.net/", {
  proxyHost: "127.0.0.1",
  proxyPort: 1080,
});
```

### Python (PyMongo ≥ 4.6)

```python
MongoClient(
    "mongodb+srv://USER:PASS@<cluster-01>.<atlas-id>.mongodb.net/",
    proxyHost="127.0.0.1",
    proxyPort=1080,
)
```

### Containerised app (same Compose project)

Uncomment the `app:` block in `docker-compose.yml` and adapt the image —
no proxy options needed, the unmodified `mongodb+srv://…` URI just works
because the app shares the VPN's network namespace.

## Day-to-day

```bash
docker compose up -d            # start
docker compose down             # stop
docker compose restart vpn      # rotate the tunnel (bounces socks5 too — expected)
docker compose logs -f vpn      # follow VPN logs
docker compose pull && docker compose up -d   # update Gluetun
```

## Troubleshooting

**`server hostname xxx not found in NordVPN servers list`** — Gluetun's
bundled server list is stale relative to NordVPN's API. Confirm the hostname
spelling against the `remote` line in your `.ovpn`. If it's correct, fall
back to mounting the raw `.ovpn` file (see Gluetun docs on custom OpenVPN
configs) — ping back here and we can wire it up.

**`AUTH_FAILED` in VPN logs** — `NORDVPN_USER` / `NORDVPN_PASSWORD` are
likely your account login. Use the **service credentials** from Manual Setup;
they're different strings.

**Atlas connection still times out after whitelisting** — confirm the
`?proxyHost=127.0.0.1&proxyPort=1080` is in the URI exactly (some tools
strip query params on paste). Re-run the `curl --socks5-hostname` check —
if that succeeds, the proxy path is fine and Atlas allow-list is the
remaining suspect (check both clusters individually).

**`nordvpn-socks5 exited with code 1 … REQUIRE_AUTH is true`** — make sure
the `REQUIRE_AUTH: "false"` line is present in `docker-compose.yml` (or
provide `PROXY_USER`/`PROXY_PASSWORD`).

**Host can't reach `127.0.0.1:1080`** — `docker compose ps` should show
`nordvpn` with `127.0.0.1:1080->1080/tcp`. If `socks5` is in a crash loop
it won't be serving; check `docker compose logs socks5`.

## Files

* `docker-compose.yml` — VPN + SOCKS5 services (and a commented app template).
* `.env.example` — environment variable template; copy to `.env` and fill in.
* `.gitignore` — keeps `.env` out of version control.
