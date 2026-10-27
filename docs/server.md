# Personal WhatsApp bridges: server setup (admin)

This sets up **bridge-gateway** next to Synapse so tjena users can run their
own WhatsApp bridge on a phone at home (tjena Bridge app). Synapse needs
**one restart at setup** (and one per new batch of slots), never one per user.

```
home phone                                   server (Docker network docker_local_net)
┌──────────────────────────────┐   wss/https  ┌────────────────────────┐  internal  ┌─────────┐
│ mautrix-whatsapp (unmodified)│──► relay ───►│ bridge-gateway :29331  │───────────►│ Synapse │
│ bridge-relay (127.0.0.1)     │◄────────────│ (public: wap.tjena.eu) │◄───────────│  :8008  │
└──────────────────────────────┘              │ bridge-gateway :29332  │ transactions└─────────┘
                                              │ (internal, Synapse only)│
                                              └────────────────────────┘
```

- **Slot pool.** The gateway creates N appservice registrations
  (`tjena-wa01` … `tjena-waNN`), each with its own namespace (`@wa07bot`,
  `@wa07_*`) and real tokens. Synapse loads them once.
- **Provisioner.** A user signs in from the bridge app; the gateway checks the
  Matrix token with Synapse and assigns a free slot. The app gets **its own
  bridge tokens**; the gateway swaps them for the slot's real tokens on every
  request. Revoking = deleting the bridge tokens: instant, no restart. Real
  tokens never leave the server.
- **Relay.** Synapse pushes transactions to the internal listener; the gateway
  forwards them verbatim over the bridge's websocket and acknowledges Synapse
  only after the bridge did. The bridge's normal API calls come back through
  the public listener.
- Burned (revoked) slots are never reused. Add slots in batches.

Source: this repo. Design and test evidence:
tjena_bridge `docs/m0-spikes.md`.

## 1. Build the image

```bash
cd tjena_proxy
docker build -t tjena-bridge-gateway:latest .
```

## 2. Directories and config

```bash
BASE=/mnt/SSD/PhD_local_only/docker_hosting/tjena_matrix/bridge-gateway
mkdir -p $BASE/data $BASE/registrations
cp example-config.yaml $BASE/data/config.yaml
# the container runs as uid 1337
sudo chown -R 1337:1337 $BASE/data $BASE/registrations
sudo chmod 700 $BASE/data
```

Edit `$BASE/data/config.yaml`. The defaults match the current setup
(`tjena_matrix` on `docker_local_net`, server name `tjena.eu`). Keep
`registration_dir: /registrations` and
`registration_dir_in_synapse: /data/personal-bridges`.

## 3. docker-compose (Portainer stack)

Add to the stack that runs Synapse (or a new stack on the same network):

```yaml
services:
  tjena-bridge-gateway:
    image: tjena-bridge-gateway:latest
    container_name: tjena-bridge-gateway
    restart: unless-stopped
    volumes:
      - /mnt/SSD/PhD_local_only/docker_hosting/tjena_matrix/bridge-gateway/data:/data
      - /mnt/SSD/PhD_local_only/docker_hosting/tjena_matrix/bridge-gateway/registrations:/registrations
    networks: [docker_local_net]
    # No published ports. Cloudflare reaches :29331 over the Docker network;
    # only Synapse uses :29332.

networks:
  docker_local_net:
    external: true
```

And mount the registrations into Synapse (read-only):

```yaml
  tjena_matrix:
    volumes:
      # ...existing mounts...
      - /mnt/SSD/PhD_local_only/docker_hosting/tjena_matrix/bridge-gateway/registrations:/data/personal-bridges:ro
```

## 4. Create the slot pool

Before the first start (works offline on the state file):

```bash
docker run --rm \
  -v $BASE/data:/data -v $BASE/registrations:/registrations \
  tjena-bridge-gateway:latest slots-add 100
```

It prints the `app_service_config_files` lines to add.

**Make the pool big once** (e.g. 100) so Synapse rarely needs a restart.
Idle slots cost Synapse almost nothing. The gateway logs
`few free bridge slots` (and `/status/v1` says `"low": true`) when fewer
than `low_slots_warning` (default 5) are free. To grow the pool, with the
gateway running, then restart Synapse once:

```bash
docker exec tjena-bridge-gateway bridge-gateway -config /data/config.yaml slots-add 20
```

## 5. Synapse changes (one restart)

In `synapse-data/homeserver.yaml`:

```yaml
app_service_config_files:
  - /data/mautrix-signal-registration.yaml
  - /data/mautrix-whatsapp-registration.yaml
  # personal bridge slots (printed by slots-add):
  - /data/personal-bridges/tjena-wa01.yaml
  - /data/personal-bridges/tjena-wa02.yaml
  # ...

experimental_features:
  # End-to-bridge encryption in appservice mode. Synapse refuses /sync for
  # appservice users, so the bridges get to-device messages, device lists and
  # one-time-key counts inside transactions instead.
  msc2409_to_device_messages_enabled: true
  msc3202_transaction_extensions: true

# Optional but recommended: lets the tjena app show a sign-in QR code for the
# bridge app, so users never type their password on the bridge phone.
login_via_existing_session:
  enabled: true
  require_ui_auth: true     # tjena asks for the password once
  token_timeout: 5m
```

Then restart Synapse and check the logs for appservice errors.

> These experimental flags also add to-device data to the transactions of the
> existing shared bridges (`mautrix-whatsapp`, `mautrix-signal`). Bridges that
> don't use it ignore it. Check their logs after the restart.

Each slot registration contains:

| Field | Value |
|---|---|
| `id` | `tjena-waNN` |
| `url` | `http://tjena-bridge-gateway:29332` (internal listener) |
| `sender_localpart` | `waNNbot` (not a server admin) |
| `namespaces.users` | `^@waNNbot:tjena\.eu$`, `^@waNN_.*:tjena\.eu$` (both exclusive) |
| `namespaces.rooms/aliases` | none |
| `org.matrix.msc3202`, `io.element.msc4190`, `receive_ephemeral` | true |

`wa` + digits never overlaps `@whatsapp_*`, `@signal_*`, or other slots
(`wa10_` doesn't match `wa100_`). Before adding slots, make sure no real user
matches `@wa[0-9]+`: `SELECT name FROM users WHERE name ~ '^@wa[0-9]';`.

## 6. Public hostname wap.tjena.eu

**Cloudflare tunnel (current setup).** In Zero Trust → Networks → Tunnels →
(your tunnel) → Public hostnames → Add:

- Subdomain `wap`, domain `tjena.eu`
- Service: `HTTP` → `tjena-bridge-gateway:29331`
- Leave Path empty. The public listener only serves the bridge websocket, the
  bridge's API calls, `/provision/` and the call signalling websockets under
  `/_tjena/call/`; everything else returns 404.
- Make sure WebSockets are enabled for the zone (Network → WebSockets).

Do **not** add a hostname for port 29332.

Cloudflare's free plan limits request bodies to 100 MB, which caps media the
bridge can upload. Synapse's `max_upload_size` usually caps it lower anyway.

**Later on a VPS (nginx).**

```nginx
server {
    listen 443 ssl http2;
    server_name wap.tjena.eu;
    # ssl_certificate ...;

    client_max_body_size 100m;

    location = /_matrix/client/unstable/fi.mau.as_sync {
        proxy_pass http://127.0.0.1:29331;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 1h;
        proxy_send_timeout 1h;
    }
    # WhatsApp call signalling (websockets; app <-> bridge phone).
    location ^~ /_tjena/call/ {
        proxy_pass http://127.0.0.1:29331;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 1h;
    }
    location ~ ^/(_matrix/client|_matrix/media|provision)/ {
        proxy_pass http://127.0.0.1:29331;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 10m;     # /sync long-polls, media
        proxy_request_buffering off;
    }
    location / { return 404; }
}
```

## 7. Start and check

```bash
docker compose up -d tjena-bridge-gateway
docker exec tjena-bridge-gateway bridge-gateway -config /data/config.yaml slots-list
curl https://wap.tjena.eu/provision/v1/info
```

`slots-list` shows `ACTIVE true` once Synapse has loaded a slot (checked
every 60 s). `info` should say `"slots_available": true`.

### Public status (for the tjena website)

`GET https://wap.tjena.eu/status/v1` needs no login and allows any origin
(CORS). It is cached for 30 s, so visitors can't load Synapse or the bridges.

```json
{
  "updated": "2026-10-24T12:00:00Z",
  "services": [
    {"name": "synapse", "online": true, "latency_ms": 4},
    {"name": "whatsapp_bridge", "online": true, "latency_ms": 2},
    {"name": "bridge_proxy", "online": true}
  ],
  "personal_bridge_slots": {"free": 63, "total": 100, "low": false}
}
```

- `synapse` is always checked (`synapse_url` + `/health`).
- Other services come from `status_checks` in the config. A mautrix bridge
  answers on `/_matrix/mau/live` at its appservice address (the `url:` in its
  registration file). The gateway must be on the same Docker network:
  ```yaml
  status_checks:
    - name: whatsapp_bridge
      url: http://mautrix-whatsapp:29318/_matrix/mau/live
  ```
- `bridge_proxy` is the gateway itself. If the request fails altogether,
  the website should show the proxy (and everything behind it) as unknown.
- `total` counts slots Synapse has loaded and that aren't revoked; `free`
  is how many new users can still set up their own bridge.
- It shows no user names, slot names or per-bridge state.

Update: rebuild and recreate the gateway as below, add `status_checks`, and
check with `curl https://wap.tjena.eu/status/v1`.

### Update for WhatsApp calls

Calls need the new gateway (call signalling under `/_tjena/call/v1/`).
No config, Synapse or slot changes:

```bash
cd tjena_proxy
docker build -t tjena-bridge-gateway:latest .
docker compose up -d --force-recreate tjena-bridge-gateway
# Should answer 401 (needs a token), not 404:
curl -s -o /dev/null -w '%{http_code}\n' https://wap.tjena.eu/_tjena/call/v1/app
```

Restarting the gateway drops the bridge websockets for a few seconds; the
bridges reconnect by themselves. Calls connect through Synapse's TURN server
(`turn_uris` in homeserver.yaml) when the two phones can't reach each other
directly, so that must be set up, as it already is for tjena calls.

## Admin CLI

Run inside the container: `docker exec tjena-bridge-gateway bridge-gateway -config /data/config.yaml <cmd>`

| Command | What it does |
|---|---|
| `slots-list` | All slots: state (free/assigned/burned), active, owner, online, last seen |
| `slots-add N` | Create N more slots and print the Synapse lines (needs one restart) |
| `snippet` | Print `app_service_config_files` lines for all slots |
| `claim @user:tjena.eu` | Give a user a bridge (prints the bundle for manual setup) |
| `assign wa05 @user:tjena.eu` | Same, for a specific slot |
| `revoke wa05` | Remove a bridge right away (slot is burned, never reused) |
| `check-activation` | Re-check which slots Synapse has loaded |

Files in `data/`: `state.json` (slots and tokens, mode 0600), `audit.jsonl`
(create/claim/reclaim/revoke with user and time, no tokens or content),
`admin_token` (protects the internal admin API).

## Security notes

- Real `as_token`s exist only in `state.json` and the registration files.
  Bridge apps get separate tokens. Revoking or rotating them cuts the bridge
  off immediately.
- The gateway does not decide which users a bridge may act as. Synapse
  enforces each slot's namespace, so a gateway bug can't reach another slot's
  users.
- A revoked slot's real token stays valid in Synapse until the next restart,
  but nobody outside the server has it. The slot is never reassigned, so its
  old ghosts can't be reused in other people's rooms.
- Bridge bots are not server admins. Slots have no room or alias namespaces.
- Encryption is mandatory in every personal bridge (`encryption.require`). To
  check that the server only stores ciphertext for portal rooms:
  ```sql
  SELECT e.type, count(*) FROM events e
  JOIN room_memberships m ON m.room_id = e.room_id AND m.user_id = '@wa01bot:tjena.eu'
  WHERE e.state_key IS NULL GROUP BY e.type;
  -- expect m.room.encrypted (plus reactions/redactions), no m.room.message
  ```

## Troubleshooting

| Symptom | Check |
|---|---|
| App: "no free bridge slots" | `slots-list`; `slots-add` + restart |
| Slot stays `ACTIVE false` | Registration file mounted in Synapse? listed in `app_service_config_files`? restarted? |
| Bridge app: connection to tjena fails, HTTP 401 | The bridge was revoked or another phone took over: use "Make this phone the active bridge" in the app |
| Messages arrive late | Gateway log: `synapse ping` after reconnect should be 200 (flushes Synapse's retry queue) |
| Bot doesn't answer in an encrypted DM | Synapse experimental flags missing; bridge log "failed to decrypt" |
