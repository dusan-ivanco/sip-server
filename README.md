# SIP Server

The server side of a SIP video intercom for Home Assistant: [Asterisk](https://www.asterisk.org) for SIP/WebRTC, [coturn](https://github.com/coturn/coturn) for STUN/TURN, and a small script that rings every open panel and sends push notifications through a Home Assistant webhook.

It is the companion to [sip-doorbell](https://github.com/dusan-ivanco/sip-doorbell), a Lovelace card that registers as a WebRTC SIP client and shows the intercom video in a dashboard.

## How it works

```
 ┌──────────────┐  SIP/TLS (5061)  ┌───────────────────────────┐  WSS + WebRTC  ┌──────────────────┐
 │  Intercom    │ ───────────────► │  Asterisk                 │ ◄────────────► │  HA dashboard    │
 │  (ext. 999)  │   dials 001      │  ConfBridge doorbell_room │                │  sip-doorbell    │
 └──────────────┘                  │  + scripts/doorbell       │                │  card (ext. 001) │
                                   └─────────────┬─────────────┘                └────────┬─────────┘
                                                 │ webhook                               │ STUN/TURN
                                                 ▼ sip_notify?state=wakeup|cancel        ▼
                                   ┌───────────────────────────┐                ┌──────────────────┐
                                   │  Home Assistant           │                │  coturn          │
                                   │  automation "SIP Notify"  │                │  (3478 / 5349)   │
                                   └───────────────────────────┘                └──────────────────┘
```

1. The intercom (endpoint `999`, TLS) dials a three-digit extension, here `001`.
2. The dialplan (`[default]`, `_XXX`) answers and puts the intercom into the ConfBridge room `doorbell_room`. The room uses SFU video, and the intercom hears music on hold while it is alone. The dialplan then starts the `doorbell` script in the background with the caller number and the dialled extension as arguments.
3. The script:
   - calls the HA webhook with `state=wakeup`;
   - reads every registered contact of the dialled extension from the Asterisk registrar (up to 25 per AOR, one per open card) and originates a call to each one into the context `[doorbell]`, which joins the same room.
4. The first panel that answers joins the room. The script hangs up the panels that are still ringing and calls the webhook with `state=cancel`.
5. The script ends and clears the room when:
   - the intercom leaves;
   - nobody answers within **60 s**;
   - the answering panel leaves;
   - the call has lasted **1 hour**.

## Repository layout

The repository mirrors the target file system, so files are copied to the same paths on the server.

| Path | Purpose |
|------|---------|
| `etc/docker/compose/asterisk/docker-compose.yml` | Asterisk 22 (`andrius/asterisk:22`), host network, runs as UID/GID 1000 |
| `…/asterisk/data/etc/asterisk/pjsip.conf` | Transports (UDP/TCP 5060, TLS 5061, WSS), NAT settings and endpoints `001` and `999` |
| `…/asterisk/data/etc/asterisk/http.conf` | HTTP server for the WebSocket: `8088` (plain) and `8089` (TLS) |
| `…/asterisk/data/etc/asterisk/extensions.conf` | Dialplan (contexts `default` and `doorbell`) |
| `…/asterisk/data/etc/asterisk/confbridge.conf` | Profiles `doorbell_bridge` (SFU video, no join/leave sounds), `doorbell_device` (music on hold when alone), `doorbell_agents` |
| `…/asterisk/data/etc/asterisk/rtp.conf` | RTP range `10000–11000` and ICE host candidate mapping (private → public IP) |
| `…/asterisk/data/etc/asterisk/modules.conf` | Autoload with unused modules disabled (ARI, voicemail, queues, CDR/CEL backends, …) |
| `…/asterisk/data/var/lib/asterisk/scripts/doorbell` | Ring-all, cleanup and webhook script |
| `…/asterisk/data/var/lib/asterisk/scripts/bin/jq` | Static `jq` binary used by the script (**Linux ARM64**). Replace it on other architectures. |
| `…/asterisk/data/var/lib/asterisk/moh/simplicity.wav` | Music on hold played to the intercom while it waits |
| `etc/docker/compose/coturn/docker-compose.yml` | coturn (`coturn/coturn:latest`), host network |
| `etc/docker/compose/coturn/data/server.conf` | coturn configuration (relay ports `12000–13000`) and bootstrap commands |
| `etc/certs/base/` | Empty placeholders for the TLS certificate (`full.pem`, full chain) and key (`cert.key`). Asterisk and coturn share them. |
| `etc/nftables.conf` | Firewall rules |
| `etc/systemd/system/asterisk.service`, `coturn.service` | systemd units that run `docker compose up -d` / `down` |
| `var/Home Assistant/*.sample` | HA `input_boolean` and webhook automation |

## Requirements

- A Linux host with a public IP (or a port-forwarded NAT), Docker with the Compose plugin, systemd and nftables.
- A domain name (e.g. `sip.domain.tld`) and a valid TLS certificate for it. Browsers require valid TLS for WSS.
- A SIP intercom that supports TLS and the codecs **G.711 A-law** and **H.264**. Other codecs are disabled.
- Home Assistant with the [sip-doorbell](https://github.com/dusan-ivanco/sip-doorbell) card and the Companion app for push notifications.

## Installation

### 1. Copy the files

```sh
sudo cp -r etc/docker/compose/asterisk etc/docker/compose/coturn /etc/docker/compose/
sudo cp etc/systemd/system/asterisk.service etc/systemd/system/coturn.service /etc/systemd/system/
sudo cp etc/nftables.conf /etc/nftables.conf
```

Put your certificate chain and key in `/etc/certs/base/full.pem` and `/etc/certs/base/cert.key`. Both containers must be able to read them.

### 2. Replace the placeholders

| File | What to set |
|------|-------------|
| `pjsip.conf` | `external_media_address` and `external_signaling_address` (public IP, `1.2.3.4`); `local_net` (your private ranges); `from_domain`; the `password` of `001` and `999` |
| `rtp.conf` | `[ice_host_candidates]`: map the private IP of the host (`10.20.30.40`) to the public IP |
| `scripts/doorbell` | Home Assistant webhook URL (`https://hass.domain.tld/api/webhook/sip_notify`) |
| `coturn/data/server.conf` | `listening-ip`, `realm`, `cli-password` |

The endpoint `001` is the WebRTC account the cards use (transport `wss-ipv4`, `webrtc = yes`). The endpoint `999` is the intercom (transport `tls-ipv4`). To add another group of panels or another intercom, copy the `aor`/`auth`/`endpoint` sections. Any three-digit extension works with the dialplan.

### 3. Initialise coturn

Start coturn once, then create the CLI password hash and the users with `turnadmin` inside the container. The commands are at the end of `server.conf`:

```sh
turnadmin -P -p "cli-password"
turnadmin -A -b "/var/lib/coturn/turndb" -r "sip.domain.tld" -u "admin" -p "secret"
turnadmin -a -b "/var/lib/coturn/turndb" -r "sip.domain.tld" -u "turn"  -p "secret"
```

Put the hash from the first command into `cli-password`. The `turn` user is the one the card uses in `server.ice`.

### 4. Firewall and services

```sh
sudo systemctl enable --now nftables
sudo systemctl daemon-reload
sudo systemctl enable --now asterisk coturn
```

Ports opened by `nftables.conf`:

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 5060 | UDP/TCP | SIP |
| 5061 | TCP | SIP over TLS |
| 3478, 5349 | UDP/TCP | STUN/TURN (plain and TLS) |
| 10000–11000 | UDP | Asterisk RTP |
| 12000–13000 | UDP | coturn relay |

> The WebSocket port (`8089`) is **not** opened by `nftables.conf`. Either add it, or put a reverse proxy in front of Asterisk and point the card at `wss://sip.domain.tld/ws`.

### 5. Home Assistant

- Add `var/Home Assistant/configuration.yaml.sample` (the `input_boolean.sip_notify` helper) to your configuration.
- Add `var/Home Assistant/automations.yaml.sample` (the **SIP Notify** automation). Replace `notify.user` and `camera.doorbell` with your own entities.

The automation listens on the webhook `sip_notify`:

- `state=wakeup`: turns on `input_boolean.sip_notify` and sends a time-sensitive push notification. The notification shows a live picture from the camera and opens `/lovelace/doorbell`.
- `state=cancel`: turns the helper off and clears the notification.

The webhook is `local_only`, so HA accepts it only from the local network. If the Asterisk host calls HA through a public address, set `local_only: false`.

### 6. The card

Install [sip-doorbell](https://github.com/dusan-ivanco/sip-doorbell) and configure it with the `001` account and the TURN user:

```yaml
type: custom:sip-doorbell
server:
  sip:
    url: wss://sip.domain.tld/ws
    username: 001@sip.domain.tld
    credential: secret
  ice:
    - urls: stun:sip.domain.tld:3478
    - urls: turn:sip.domain.tld:3478
      username: turn
      credential: secret
camera: camera.doorbell
```

## Troubleshooting

Open the Asterisk console with `docker exec -it ASTERISK asterisk -rvvv`. Useful commands:

| Command | Shows |
|---------|-------|
| `pjsip show contacts` | registered cards and intercom |
| `database show registrar` | registrar contacts the script rings |
| `confbridge list doorbell_room` | who is in the room |
| `core show channels` | active and ringing channels |
| `pjsip set logger on` | SIP signalling trace |

Common problems:

- **Card does not register**: the WSS port is not reachable, the certificate is invalid for the domain, or the password is wrong.
- **Call connects but no audio or video**: NAT. Check `external_*_address`, the `rtp.conf` ICE mapping, the TURN credentials in the card and the UDP ranges `10000–11000` (Asterisk) and `12000–13000` (coturn).
- **No push notification**: run the script's `curl` from the host by hand and check the HA log for the webhook and the `local_only` setting.
- **Script fails with `jq: cannot execute`**: the bundled `jq` is for ARM64. Replace it with a binary for your architecture.
