# Two hosts over RDMA

host-a produces a flow, host-b shows it. The
[fabrics agent](https://github.com/LeeO86/mxl-fabrics-agent) on each host
mirrors flows between them over RDMA (RoCEv2, libfabric `verbs`), with TCP as
fallback.

```
host-a                                       host-b
 registry (3210/3211)                         multiviewer
 test-player ─► /Volumes/mxl/player           webrtc-monitor
                       │                          ▲
 fabrics-agent ◄── RDMA 198.51.100.1/.2 ──► fabrics-agent ─► /Volumes/mxl/mirror-<domain id>
```

How it works: a controller connects a receiver on host-b to the test player's
sender with ordinary IS-05 (`mxl_domain_id`, `mxl_flow_id`). The agent on
host-b sees the connection, finds the domain on host-a, and pulls the flow into
a local mirror domain with the same ids. With `MIRROR_MODE=eager` the mirror
exists before the receiver opens it. The media functions know nothing about
hosts or RDMA.

## Files

| File | Use |
| --- | --- |
| `fabrics-agent.yaml` | The agent, included by both hosts |
| `host-a/compose.yaml` | Registry, test player, agent |
| `host-b/compose.yaml` | Multiviewer, WebRTC monitor (built-in MediaMTX), agent |
| `host-*/example.env` | Copy to `.env` on that host |

```bash
# on host-a
cd host-a && cp example.env .env && $EDITOR .env && docker compose up -d
# on host-b
cd host-b && cp example.env .env && $EDITOR .env && docker compose up -d
```

## Network

- **Management network** (`HOST_IP`, here 192.0.2.11 and 192.0.2.12): NMOS,
  the agents' control API (8095), UIs, WebRTC.
- **Fabric link** (here a direct link 198.51.100.1 <-> 198.51.100.2): RDMA.
  `PEERS` lists, per peer, the local and remote fabric address. On a switched
  fabric leave both empty and set `FABRIC_INTERFACE` to the local fabric
  address instead. Same MTU on both ends.
- Fabric target ports 23500-23599 (TCP or RDMA CM) must be open between the
  hosts.
- RoCE needs a nearly lossless network: PFC on the RoCE priority end to end,
  ECN/DCQCN, same DSCP trust on NIC and switch. On Intel E810 the retransmit
  timer is the `irdma` device setting `roce_rtomin`; see the agent's README
  ("Audio timing").

## Host checklist

- `ibv_devinfo` lists the RDMA device (`mlx5_*` on ConnectX, `irdma*` on
  E810 in RoCEv2 mode). Install `rdma-core`.
- `/dev/infiniband` exists. The agent gets it with `devices:`.
- `memlock` unlimited and `IPC_LOCK` for memory registration (in
  `fabrics-agent.yaml`). No hugepages needed.
- `pid: host`, `SYS_PTRACE`, `apparmor=unconfined`: the agent removes a mirror
  only when no other process maps it, and reads every `/proc/<pid>/maps` to
  know. Without them it keeps all mirrors and reports `readers_known: false`.
- Both hosts: `CLOCK_TAI` disciplined (PTP or chrony) with the 37 s TAI
  offset. Grain indices are TAI; a wrong offset on one host shows as 37 s lag
  or no grains.
- tmpfs big enough for local plus mirrored flows plus `TMPFS_RESERVE_MB`
  (512 MB). A mirror that does not fit is refused (`insufficient_space`).
- `fs.inotify.max_user_instances` raised (one instance per mirrored flow
  lane).

## Check

```bash
curl http://192.0.2.12:8095/api/v1/peers          # host-a: up, link_up
curl http://192.0.2.12:8095/api/v1/replications   # grains, lag_ms after routing
curl http://192.0.2.12:8095/metrics | grep mxl_fabrics_agent_mirror_stale
```

No RDMA hardware? Set `"provider":"tcp"` in `PEERS` and `DEFAULT_PROVIDER=tcp`
to try the same setup over TCP.
