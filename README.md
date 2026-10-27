# tjena proxy (bridge gateway)

Server side of the [tjena Bridge](https://github.com/tjena-eu/tjena_bridge): the
gateway at `wap.tjena.eu` between people's home bridges and the tjena Synapse
server. It is the `bridge_proxy` entry on the tjena status page.

- **Slot pool:** pre-registered appservices (`tjena-waNN`), so a new bridge needs
  no Synapse restart.
- **Token swap:** the home bridge only ever holds a gateway token; the real
  appservice token never leaves the server. Revoking a bridge takes effect at once.
- **Provisioning:** the tjena app claims and revokes a slot with the user's Matrix
  login (OpenID).
- **Forwarding:** appservice transactions (including to-device / E2EE data) go to
  the bridge over a websocket unchanged; the bridge's client-API calls go to Synapse.
- **Call signalling** for WhatsApp calls (`/_tjena/call/v1/`), and a public status
  endpoint (`/status/v1`).

## Build, test, run

```bash
go test -race ./...
docker build -t tjena-bridge-gateway:latest .
```

Deployment, Synapse settings, slots and updates: [docs/server.md](docs/server.md).
The full-stack test (gateway + relay + unmodified bridge + throwaway Synapse) lives in
tjena_bridge: `itest/run.sh` (expects this repo next to it, or `GATEWAY_DIR`).

## License

AGPL-3.0-or-later, see [LICENSE](LICENSE).
