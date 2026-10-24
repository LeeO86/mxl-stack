# Single host

One host with an NMOS registry, an NMOS crosspoint controller, a shared
MediaMTX and the core chain:

```
test-player ──► multiviewer ─┐
            ├─► scopes ──────┤
            ├─► audio-mixer ─┼─► webrtc-monitor ──► MediaMTX ──► browser
            ├─► replay ──────┤
browser-source ──────────────┘
FlowXer (optional) ◄── sources;  FlowXer PGM ──► multiviewer
```

Nothing is routed at start. Every arrow is an IS-05 connection you make in
the crosspoint (below).

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
docker compose -f compose.yaml -f compose.gpu.yaml --profile flowxer up -d   # + FlowXer
```

FlowXer needs the GPU overlay: its media engine runs only on an NVIDIA GPU.
Set `FLOWXER_API_TOKEN` in `.env` on any shared network.

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
| NMOS crosspoint (login admin / admin, change it) | `http://HOST_IP:8099/` |
| FlowXer (profile flowxer) | `http://HOST_IP:9620/` |
| Registry Query API | `http://HOST_IP:3211/x-nmos/query/v1.3/senders` |

## Route a flow

With the crosspoint: open `http://HOST_IP:8099/`, sign in (default
`admin` / `admin`; change it under Setup -> Change Login & Password). The
matrix lists every MXL sender (rows) and receiver (columns) from the registry.
Click a sender and a receiver to connect them (autotake, or stage and TAKE).
Only senders and receivers of the same transport (MXL) can be connected; route
video and audio separately. Examples:

- Test player output 1 video -> multiviewer input 1 video; same for audio.
- Test player -> scopes "Scopes In Video", audio mixer inputs, replay cameras,
  WebRTC monitor channel 1.
- Test player / browser source -> FlowXer live inputs; FlowXer ME 1 PGM video
  -> a multiviewer input.

Its registry is set in `crosspoint-settings.json` (the Query API,
`127.0.0.1:3211`, discovery off). The file is copied into the `crosspoint-config`
volume on the first start only; later changes are made on its Setup page.
It still sends mDNS queries (browse only, no announcement); the static
registry wins.

Without the crosspoint, find the sender's flow (`flow_id`) and its domain in
the Query API, then connect a receiver with IS-05:

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
