# Local Headscale test harness

A minimal Headscale instance used to **develop and test the
`login_server` path** in this component against a real Tailscale-
compatible coordinator without touching Tailscale SaaS.

**Not a production setup** — no TLS, default secrets, single container.

## What this proves / what it doesn't

- ✅ The YAML `login_server` value reaches microlink at runtime.
- ✅ DNS resolves, TCP connects on the configured port, and the
  HTTP/1.1 Upgrade request makes it into Headscale's
  `NoiseUpgradeHandler`.
- ✅ Microlink fetches the server's Noise public key from Headscale's
  Tailscale-compatible `/key?v=88` endpoint and the Noise IK
  handshake completes.
- ✅ The node registers via `/machine/register` and the initial
  `/machine/map` response returns a tailnet IP. The node shows up in
  `headscale nodes list` with its IP (typically `100.64.0.1` on a
  fresh install).
- ⚠ The MapResponse long-poll is not yet stable: the device re-auths
  and re-registers on roughly a 60 s cycle, so Headscale reports the
  node as offline between cycles. Fine for iterating on the
  registration path; not suitable as an always-on Headscale endpoint.
  This is a microlink-side higher-protocol issue and is tracked for a
  follow-up release.

The Noise server pubkey is fetched dynamically, so every Headscale
install works without baking a per-install key into the firmware.
This harness binds Headscale to host port **80** to match the
compose file's `80:80` mapping; `login_server` also accepts explicit
ports (`host:port` or `http://host:port`) if you remap it.

## Prerequisites

- Docker (Docker Desktop on Windows/Mac works fine)
- Your workstation must be reachable from the ESP32 over WiFi — i.e., the
  ESP32 and the Docker host must share a LAN segment.
- Nothing else listening on host port 80 (IIS, Apache, local web
  servers). On Windows, check with `netstat -ano | findstr :80`.
- Inbound TCP 80 open in the host firewall. On Windows run (elevated):
  ```
  netsh advfirewall firewall add rule name="Headscale 80" dir=in action=allow protocol=TCP localport=80
  ```

## 1. Set the server URL

Figure out your host's LAN IP (the address the ESP32 will reach). On
Windows `ipconfig` and look at the adapter that is on the same network
as your ESP32. Edit `config/config.yaml` and replace the
`HOST_LAN_IP` placeholder with that address, for example:

```yaml
server_url: http://192.168.1.42
listen_addr: 0.0.0.0:80
```

Headscale bakes `server_url` into preauth keys and the registration
flow, so clients must reach it exactly as written — `localhost` will not
work for the ESP32.

## 2. Start Headscale

From this directory:

```bash
docker compose up -d
docker compose logs -f   # optional: watch it boot
```

First boot creates the SQLite database and keys under the
`headscale-data` volume. You should see:

```
INF listening and serving HTTP on: 0.0.0.0:80
```

## 3. Create a user and a preauth key

```bash
docker compose exec headscale headscale users create esp32
docker compose exec headscale headscale preauthkeys create \
    --user esp32 \
    --reusable \
    --expiration 24h
```

The second command prints a raw hex preauth key (not `tskey-auth-…` —
Headscale uses its own format). Copy it.

## 4. Point the ESPHome device at Headscale

In your device YAML:

```yaml
tailscale:
  auth_key: "<hex preauth key from step 3>"
  hostname: "esp32-test"
  login_server: "http://192.168.1.42:80"
  # Also accepted: "192.168.1.42", "192.168.1.42:80",
  # "http://192.168.1.42". "https://..." is rejected.
```

Flash, then `esphome logs example-dev.yaml`. On a successful run you
will see the device fetch the Headscale Noise pubkey, complete the
handshake, and register:

```
[I][tailscale]: Calling microlink_init with auth_key=abc123def456... ctrl_host=http://192.168.1.42:80
[I][ml_coord]: Control plane from config: 192.168.1.42 port=80
[I][ml_coord]: Fetching server pubkey from http://192.168.1.42:80/key?v=88
[I][ml_coord]: Server pubkey OK (first4=... last2=...)
[I][ml_coord]: Noise handshake complete
[I][ml_coord]: Register OK
[I][microlink]: State: CONNECTED
```

And on the Headscale side (`docker compose logs`):

```
INF Successfully authenticated via AuthKey node=esp32-test
```

After that, `docker compose exec headscale headscale nodes list`
shows the node with its tailnet IP (typically `100.64.0.1` on a
fresh install).

**Known caveat:** the MapResponse long-poll is not yet stable, so the
device re-authenticates on roughly a 60 s cycle and Headscale reports
the node as offline between cycles. See the parent README's *Custom
control plane (Headscale)* section for the full picture.

## 5. Tear down

```bash
docker compose down          # keep the volume (keys + db persist)
docker compose down -v       # full wipe
```

## Notes

- HTTP only. Real Tailscale clients use the ts2021 Noise protocol over
  plain TCP anyway (the Noise layer provides confidentiality), so TLS
  isn't required end-to-end — but this harness also doesn't attempt
  TLS at all. Do not expose this Headscale to the public internet.
- DERP is disabled in this config. The harness verifies the
  control-plane handshake and registration flow; exercising
  DERP-relayed peer traffic needs a different setup and is not part
  of this harness.
- This harness exists as a development and reproduction environment,
  not as a supported end-user deployment target. Tailscale SaaS
  remains the recommended production control plane for this
  component until the MapResponse long-poll issue is resolved.
