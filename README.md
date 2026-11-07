# RecNet 2018 Backend

A from-scratch Python reimplementation of the REST services a 2018-era Rec Room client talks to: login, accounts, avatars, settings, friends, messages, rooms, matchmaking, images, config and CDN, plus a small notification websocket stub.

- **One file** (`recnet2018.py`), **standard library only**, no `pip install`
- Runs on Python 3.8+ (Ubuntu 22.04's Python 3.10 is fine)
- **SQLite** storage, created automatically on first run
- Logs every request it doesn't recognise to `data/unhandled.jsonl`, so you can see exactly what your client asks for

> **Status: experimental.** This is a clean-room reconstruction. It contains no original Rec Room code or assets, and the endpoint paths and JSON shapes are best-effort. They have **not** been verified against every 2018 client build. Expect to fill in gaps; see [Debugging](#debugging-your-client).
>
> This project is not affiliated with or endorsed by Rec Room Inc. You need your own copy of the client.

## What's implemented

| Area | Endpoints |
| --- | --- |
| Auth | `POST /connect/token` (cached login, auto-creates accounts, refresh tokens, HS256 JWTs), `GET /cachedlogin/forplatformid/{platform}/{id}` |
| Accounts | `/account/me`, `/account/{id}`, `/account/bulk`, `/account/search`, `/account/create`, `/account/me/displayname`, `/api/players/v1/...` (get, list, search, progression) |
| Config | `/api/versioncheck/v*`, `/api/config/v*`, `/api/config/v1/amplitude`, `/api/gameconfigs/v*/all`, `/api/sanitize/v*/isPure` |
| Avatar & settings | `/api/avatar/v*` (get, set, saved, items), `/api/settings/v*` (get, set) |
| Social | `/api/relationships/v*` (get, send/accept/remove friend request), `/api/messages/v*` (get, send, delete), `/api/playerReputation/v*` |
| Matchmaking | `/player/login`, `/player/logout`, `/player/heartbeat`, `/player`, `/goto/room/{name}`, `/goto/player/{id}`, `/goto/none`, `/roominstance/{id}` |
| Rooms | `/api/rooms/v*/...` and `/rooms/...`: by id, by name, bulk, hot, search, owned/created by me, create |
| Images | `/api/images/v*/save` (multipart upload), `/api/images/v*/player/{id}`, `/room/{id}`, `/me`, served from `/img/{name}` |
| Notifications | SignalR negotiate (`/hub/negotiate`, `/signalr/negotiate`) and a websocket that completes the handshake and keeps the connection alive. **No push yet.** |
| Static / CDN | Any file under `static/` is served by path, e.g. `static/cdn/foo.bundle` at `/cdn/foo.bundle` |
| Stubs | Objectives, moderation block details, game rewards, equipment, consumables, challenges, community board, storefronts, telemetry |

## Quick start (local)

```bash
python3 recnet2018.py
curl http://127.0.0.1:8080/health
```

Data is written next to the script in `data/` (database, signing key, uploaded images, `unhandled.jsonl`). Static files go in `static/`.

## Install on an Ubuntu VPS

Copy `recnet2018.py` and `install.sh` to the server, then:

```bash
sudo bash install.sh                 # listens on port 8080
sudo PORT=80 bash install.sh         # or listen on port 80 directly
```

The script installs Python if needed, creates a locked-down `recnet` service user, installs the app to `/opt/recnet2018`, registers a systemd service that restarts on failure, and opens the port in `ufw` if it is active. Re-running it upgrades the code and keeps your data.

Useful commands:

```bash
systemctl status recnet2018
journalctl -u recnet2018 -f
cat /opt/recnet2018/data/unhandled.jsonl
```

### Behind a reverse proxy (Caddy example)

If you already run Caddy (or nginx) on ports 80/443, keep the backend on 8080 and proxy to it. Websockets work without extra config in Caddy.

```
api.example.com {
    reverse_proxy 127.0.0.1:8080
}
```

If Caddy runs in Docker, point it at the host instead, e.g. `host.docker.internal:8080` (add `extra_hosts: ["host.docker.internal:host-gateway"]` to the container).

Then tell the backend its public address so config responses hand out the right CDN URL:

```bash
sudo PUBLIC_URL=https://api.example.com bash install.sh
```

## Pointing a client at the server

The client has to be sent to your server instead of the original RecNet hosts. How you do that depends on your build:

- If the build reads its service URLs from a config file or can be patched, change them to your server.
- Otherwise, map the hostnames the client uses to your server's IP (for example with the hosts file on the machine running the client).
- Check whether your build requires HTTPS. If it does, put a reverse proxy with a certificate the client trusts in front of this server.

The `Host` header of every unhandled request is recorded in `data/unhandled.jsonl`, which is the quickest way to see which hostnames your client is really calling.

## Configuration

All settings are environment variables.

| Variable | Default | Meaning |
| --- | --- | --- |
| `RECNET_HOST` | `0.0.0.0` | Bind address |
| `RECNET_PORT` | `8080` | Bind port |
| `RECNET_PUBLIC_URL` | `http://<ip>[:port]` | Public base URL, used to build the CDN URL in `/api/config` |
| `RECNET_DATA` | `./data` | Database, signing key, images, logs |
| `RECNET_STATIC` | `./static` | Static/CDN files |
| `RECNET_SECRET` | auto | JWT signing secret (generated and saved to `data/secret.key` if unset) |
| `RECNET_OPEN_REGISTRATION` | `1` | Create an account automatically for an unknown platform ID |
| `RECNET_LENIENT_AUTH` | `0` | **Debug only.** Accept requests with no/invalid token as the first account |
| `RECNET_UNKNOWN_GET` | `404` | `404`, or `empty` to answer unknown GETs with `{}` |
| `RECNET_MOTD` | welcome text | Message of the day |
| `RECNET_LOG_LEVEL` | `INFO` | Python logging level |

## Debugging your client

1. Start the server and launch the client.
2. Watch `journalctl -u recnet2018 -f` (or the terminal). Lines marked `UNHANDLED` are requests the server has no route for.
3. Open `data/unhandled.jsonl`. Each line has the method, path, query, key headers and the first 4 KB of the body.
4. Add a handler with the `@route` decorator in `recnet2018.py`, or drop a stub into the `_stub(...)` list:

```python
@route("GET", r"/api/something/v1/new", auth=True)
def something_new(c):
    a = need_account(c)          # the logged-in account row
    return {"AccountId": a["id"]}
```

If the client never gets past login, try `RECNET_LENIENT_AUTH=1` temporarily to separate token problems from missing endpoints.

## Customising

- **Rooms:** seeded placeholder rooms live in `SEED_ROOMS` at the top of the database section. Edit them, or use the `rooms` table / `POST /rooms`. The client's `/goto/room/<name>` lookup is case-insensitive.
- **Level curve:** `LevelProgressionMaps` in `config_v2`.
- **Assets:** put asset bundles and other downloadable files under `static/`; they are served by path.
- **Bans:** `UPDATE accounts SET banned=1 WHERE id=...;` in `data/recnet.db`.

## Known limitations

- **No gameplay server.** In-room multiplayer was handled by a separate real-time service (Photon), which this project does not include. This backend gives the client menus, accounts, rooms and matchmaking responses; the matchmaking response carries a `photonRoomId` you can wire to a real-time server.
- **Notifications are a stub.** The websocket completes the handshake and stays alive, but friend requests, messages and presence changes are not pushed, so they only show up when the client next requests them.
- Economy, inventory, objectives and storefront endpoints return empty data.
- No rate limiting, moderation tooling or anti-cheat. Don't expose it to the public internet without something in front of it.
- Endpoint shapes are reconstructed, not captured from real traffic. See [Debugging](#debugging-your-client).

## Development

```bash
python3 -m py_compile recnet2018.py
RECNET_PORT=18080 RECNET_DATA=/tmp/recnet-dev python3 recnet2018.py
```
