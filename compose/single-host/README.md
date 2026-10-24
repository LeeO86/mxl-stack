# Single host

One host with an NMOS registry, a shared MediaMTX and the core chain:

```
test-player ──► multiviewer ─┐
            ├─► scopes ──────┤
            ├─► audio-mixer ─┼─► webrtc-monitor ──► MediaMTX ──► browser
            ├─► replay ──────┤
browser-source ──────────────┘
```

Nothing is routed at start. Every arrow is an IS-05 connection you make with
an NMOS controller or the functions' own UIs.

## Before you start

Prepare the host as in [docs/requirements.md](../../docs/requirements.md):
the tmpfs at `/Volumes/mxl` (writable for uid 1000), TAI time and the inotify
limit.

```bash
cp example.env .env
$EDITOR .env                       # HOST_IP at least
docker compose config -q           # check
docker compose up -d
```

Optional:

```bash
docker compose --profile extras up -d                      # + colour corrector, SRT gateway
docker compose -f compose.yaml -f compose.gpu.yaml up -d   # NVIDIA GPU (NVENC, CUDA)
```

The registry advertises itself over mDNS on the host network. The functions
do not need that (`NMOS_DNS_SD=false`); if mDNS is unwanted on your network,
run the registry on a separate host or block 5353/udp.

## Open

| UI | URL |
| --- | --- |
| Test player | `http://HOST_IP:8130/` |
| Multiviewer | `http://HOST_IP:8110/` |
| Scopes | `http://HOST_IP:8170/` |
| Audio mixer | `http://HOST_IP:8190/` |
| Replay | `http://HOST_IP:8150/` |
| Browser source | `http://HOST_IP:8160/` |
| WebRTC monitor | `http://HOST_IP:8100/` |
| Colour corrector (extras) | `http://HOST_IP:8140/` |
| SRT gateway (extras) | `http://HOST_IP:8120/` |
| Registry Query API | `http://HOST_IP:3211/x-nmos/query/v1.3/senders` |

## Route a flow

Find the sender's flow (`flow_id`) and its domain in the Query API, then
connect a receiver with IS-05:

```bash
curl -X PATCH -H 'content-type: application/json' \
  -d '{"master_enable":true,"activation":{"mode":"activate_immediate"},
       "transport_params":[{"mxl_domain_id":"<domain id>","mxl_flow_id":"<flow id>"}]}' \
  "http://HOST_IP:<receiver NMOS port>/x-nmos/connection/v1.1/single/receivers/<receiver id>/staged"
```

## What the file sets

All values come from the functions' own Compose and Kubernetes examples:

- `network_mode: host` everywhere; `NMOS_HOST_ADDRESS=HOST_IP`.
- `user: "1000:1000"` and the host MXL root bind-mounted at `/Volumes/mxl`
  (read-only for the monitor).
- One registry, `NMOS_DNS_SD=false`, a unique `NMOS_SEED` per function.
- Shared preview: `PREVIEW_PUBLISH_URL=rtsp://127.0.0.1:8554`, a
  `PREVIEW_PATH_PREFIX` per function, `PREVIEW_WHEP_URL=http://HOST_IP:8889`.
- Real time: the audio mixer gets `SYS_NICE` and `MIXER_RT_PRIORITY=49`; the
  browser source `rtprio: 10`, `shm_size: 1gb`, a tmpfs `/tmp`.
- `cpuset` lines are commented out. Pin real-time functions to whole cores of
  your machine (see `lscpu -e`).
- The replay buffer is a named volume. In production put
  `REPLAY_STORAGE_DIR` on a dedicated local NVMe.

## Not included

These need hardware; use the Compose file in their repository and the same
MXL root:

- [mxl-decklink](https://github.com/LeeO86/mxl-decklink/blob/main/docker/docker-compose.yaml)
- [mxl-st2110-gateway](https://github.com/LeeO86/mxl-st2110-gateway#readme)
  (set `NMOS_PORT=8090` when DeckLink runs on the same host)
