# Host requirements

Distilled from the function repositories. Details and the reasons behind each
rule are in each repository's README and SPECIFICATION.

## Summary

| Need | For | Required? |
| --- | --- | --- |
| Linux x86-64, Docker with Compose v2 | all | yes |
| tmpfs MXL root (`/Volumes/mxl`) | all | yes |
| `CLOCK_TAI` disciplined, TAI offset 37 s | all | yes |
| NMOS IS-04 registry | routing and discovery | yes, one per facility or stack |
| `fs.inotify.max_user_instances` raised | many flows per host | yes on busy hosts |
| MediaMTX | WebRTC previews | optional (several functions bring their own) |
| NVIDIA GPU + Container Toolkit | NVENC previews, CUDA, GPU rendering | optional |
| RDMA NIC + rdma-core | multi-host (fabrics agent) | only multi-host |
| Intel E810 + DPDK/VFIO + hugepages | ST 2110 gateway | only ST 2110 |
| Blackmagic Desktop Video driver | DeckLink | only SDI |

Ubuntu 24.04 and Debian 13 hosts are known to work. The container userland is
independent of the host distribution.

## MXL root on tmpfs

All MXL domains of a host live under one tmpfs, by convention `/Volumes/mxl`
(from [mxl-hands-on](https://github.com/cbcrc/mxl-hands-on)). Every MXL
container of the host mounts it at the same path. Functions create their own
domain below it (`MXL_OUTPUT_DOMAIN_DIR`) and scan it for other domains
(`MXL_DOMAIN_SCAN_PATH`), including the `mirror-*` domains of the fabrics agent.

```bash
sudo mkdir -p /Volumes/mxl
echo 'tmpfs /Volumes/mxl tmpfs defaults,noatime,size=8G,uid=1000,gid=1000,mode=0775 0 0' \
  | sudo tee -a /etc/fstab
sudo mount /Volumes/mxl
```

- Containers run as uid/gid 1000. The tmpfs must be writable for that user
  (`uid=1000,gid=1000`, or `mode=1777`).
- Mount the dedicated tmpfs, not the whole `/dev/shm`.
- No `--ipc=host` / `hostIPC` is needed: MXL uses files on the tmpfs, not SysV
  shared memory.
- The browser source refuses to start when its domain is not on a tmpfs.

**Size.** A flow's ring holds `history_duration` (default 200 ms) of media:

| Flow | Payload | 200 ms ring |
| --- | --- | --- |
| 1080p50 v210 | ~5.3 MB/frame | ~53 MB |
| 2160p50 v210 | ~21 MB/frame | ~210 MB |
| 1080p50 v210a (fill + key) | | ~1.6x v210 |
| 16 ch 48 kHz float32 | 64 B/sample | ~31 MB |
| `video/smpte291` (ANC) | 4 KB/grain | ~40 KB |

```
tmpfs size ~ local flows + mirrored flows + 512 MB reserve (fabrics agent)
```

8 GB is a comfortable start for a 1080p host. The memory is only used when
flows exist.

## Time: PTP and TAI

Grain indices are counted from TAI. Every host that writes, reads or mirrors a
flow needs `CLOCK_TAI` disciplined to the facility clock **and** the kernel TAI
offset set (currently 37 s). Without the offset, `CLOCK_TAI` equals UTC and every
index is 37 s off; readers on another host then see nothing or far-late grains.
IS-05 scheduled activations are also compared against TAI.

- With PTP (ST 2059-2): `linuxptp`, `ptp4l` on a kernel-owned NIC port and
  `phc2sys -a -r`. `phc2sys` sets the TAI offset from the PTP UTC offset.
- Without PTP (test hosts): chrony with `leapsectz right/UTC`, which sets the
  kernel TAI offset.
- Check: `adjtimex --print | grep tai` or compare `date` with a TAI clock tool.
- The ST 2110 gateway runs PTP inside MTL on its DPDK ports; the host clock
  still needs TAI for the other functions.

## NMOS registry

Each function is an NMOS node (IS-04 Node API, IS-05 Connection API with the
MXL transport `urn:x-nmos:transport:mxl`, BCP-007-03). Use one static registry:

- [nmos-cpp](https://github.com/sony/nmos-cpp) registry, e.g. the container in
  [compose/single-host](../compose/single-host/compose.yaml).
- Registration port `P` (3210 in the examples), Query API `P+1`.
- Set `NMOS_DNS_SD=false` and `NMOS_REGISTRY_ADDRESS` / `NMOS_REGISTRY_PORT` on
  every function. No Avahi daemon is needed then.
- `NMOS_HOST_ADDRESS` must be an IPv4 literal the controller and the other
  nodes can reach (not a hostname, not loopback).
- `NMOS_SEED` makes every NMOS id (and the output domain id) a stable UUIDv5.
  Give each instance a unique, stable seed.
- Routing needs an IS-05 controller (for example nmos-js, or `curl` against
  `/x-nmos/connection/v1.1/single/receivers/<id>/staged`). Several functions
  also have an IS-05 form in their own UI.

## inotify limit

MXL opens one inotify instance per MXL instance (roughly one per domain a
process reads). `fs.inotify.max_user_instances` (default 128) counts per uid on
the whole host, and all functions run as uid 1000.

```bash
echo 'fs.inotify.max_user_instances=1024' | sudo tee /etc/sysctl.d/90-mxl.conf
sudo sysctl --system
```

## CPU and real time

- **SIMD.** Images are plain x86-64 and pick code at start. AVX2 is used where
  present (colour corrector, scopes, WebRTC monitor). AVX-512 is optional; on
  Skylake-SP / Cascade Lake it can slow the sibling hyperthread, so the colour
  corrector's `auto` picks AVX2 there. See `CC_SIMD` and `SCOPES_SIMD`.
- **SCHED_FIFO.** Some functions run timing threads real time when allowed:
  - audio mixer: `MIXER_RT_PRIORITY` (1-49) and `cap_add: [SYS_NICE]`
  - DeckLink: `RT_SCHED=true`, `cap_add: [SYS_NICE, IPC_LOCK]`
  - fabrics agent: `RT_PRIORITY`, `cap_add: [SYS_NICE]`
  - browser source: `ulimits: {rtprio: 10}` (`SYS_NICE` alone does not help a
    non-root process)
  - ST 2110 gateway: `cap_add: [IPC_LOCK, SYS_NICE]`

  Keep priorities below 50: threaded IRQ handlers run at SCHED_FIFO 50.
- **Pinning.** Give RT functions whole physical cores (`cpuset` in Compose,
  static CPU manager in Kubernetes). The audio mixer wants an even number of
  CPUs so its DSP thread gets a core whose SMT sibling is not shared.
- **Isolation (ST 2110, optional elsewhere).** `isolcpus`, `nohz_full`,
  `rcu_nocbs` for the MTL lcores; keep those cores out of `irqbalance`; keep
  NIC IRQs and the media threads on the NIC's NUMA node.
- **IRQ affinity.** Pin RDMA and ST 2110 NIC IRQs away from the pinned media
  cores (`/proc/irq/<n>/smp_affinity_list`, or `irqbalance --banirq`).

## GPU (optional)

- NVIDIA driver and the
  [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/).
  Compose: a device reservation (see `compose.gpu.yaml`).
- `NVIDIA_DRIVER_CAPABILITIES` must include `video` for NVENC/NVDEC
  (`compute,video,utility`), and `graphics` for the browser source's GPU
  rendering.
- Users: multiviewer (CUDA compositor, NVENC preview), replay (NVENC mosaic,
  optical flow), SRT gateway (NVDEC/NVENC), WebRTC monitor and scopes (NVENC
  previews), browser source (GPU rendering). All of them fall back to the CPU.
- **NVENC session limit.** Consumer GeForce cards allow only a few concurrent
  NVENC sessions per system. Every preview channel and every SRT egress counts.
  Data-centre and professional cards (e.g. L4, A4000/A16) do not have that
  limit. Functions fall back to x264 per channel when a session fails.

## MediaMTX (WebRTC previews)

Previews are H.264 published over RTSP to
[MediaMTX](https://github.com/bluenviron/mediamtx) and played in the browser
over WHEP. Two modes per function:

- **Built-in:** the image starts its own MediaMTX (each function has distinct
  default ports).
- **Shared:** set `PREVIEW_PUBLISH_URL=rtsp://<mediamtx>:8554`,
  `PREVIEW_PATH_PREFIX` (unique per function), `PREVIEW_WHEP_URL` /
  `PREVIEW_HLS_URL` (the base the browser uses) and `MEDIAMTX_API_URL`.

Open WHEP (8889/tcp), HLS (8888/tcp) and ICE (8189/udp+tcp) to the viewers. Set
the host address MediaMTX announces in ICE (`webrtcAdditionalHosts`).

## Multi-host: RDMA

The fabrics agent replicates flows with the MXL Fabrics API (libfabric):

- Provider `verbs` (RoCEv2) on NVIDIA/Mellanox ConnectX-4 or later (`mlx5`) or
  Intel E810 (`irdma` in RoCEv2 mode). `tcp` works as a fallback.
- Host: the kernel RDMA driver, `rdma-core` (`ibv_devinfo` must list the
  device), matching MTU per link.
- Container: `/dev/infiniband`, `cap_add: [IPC_LOCK]`, `ulimits: memlock: -1`,
  host network. For mirror cleanup also `pid: host`, `SYS_PTRACE` and AppArmor
  unconfined.
- RoCE needs a (nearly) lossless network: PFC on the RoCE priority end to end,
  ECN/DCQCN, consistent DSCP trust. Each lost packet costs one retransmit
  timeout.
- No hugepages are needed for the fabrics agent.

See [compose/multi-host-rdma](../compose/multi-host-rdma/).

## ST 2110 (MTL / DPDK)

Only for mxl-st2110-gateway. Its README has the full host preparation:

- Intel E810 NIC, BIOS with VT-d (and SR-IOV for VFs).
- Kernel command line: `intel_iommu=on iommu=pt default_hugepagesz=1G
  hugepagesz=1G hugepages=<n>` (4 GiB covers several 1080p50 groups).
- Media ports bound to `vfio-pci` (e.g. `driverctl set-override <pci> vfio-pci`).
- Container: host network, `user: "0:1000"` (root for VFIO), `/dev/vfio`,
  `/dev/hugepages`, `IPC_LOCK`, `SYS_NICE`, `memlock: -1`.
- CPU isolation for the MTL lcores on the NIC's NUMA node.

## DeckLink

Only for mxl-decklink:

- Blackmagic Desktop Video 16.0 or later on the host, kernel modules
  `blackmagic` / `blackmagic-io` loaded.
- Container: `/dev/blackmagic`, `group_add: [video]`, the host's
  `libDeckLinkAPI.so` mounted read-only (`DECKLINK_LIB_MODE=hostmount`).
- One container per physical card.

## Ports

All examples use host networking, so every port below shares one space. Two
instances of the same function need different values. The NMOS port's next
port (`+1`) is the IS-04 WebSocket where a function opens one.

| Function | Web / API / metrics | NMOS | Other |
| --- | --- | --- | --- |
| nmos-cpp registry (example) | | 3210 registration, 3211 query, 3212 query WS | |
| MediaMTX (shared) | 9997 API | | 8554 RTSP, 8889 WHEP, 8888 HLS, 8189 ICE |
| mxl-decklink | 8080 | 3212, 3213 | |
| mxl-st2110-gateway | on `NMOS_PORT` | 8080 (ST 2110 node +1); use 8090 next to DeckLink | |
| mxl-fabrics-agent | 8095 | 3232, 3233 | 23500-23599 fabric targets |
| mxl-webrtc-monitor | 8100 | 3242, 3243 | built-in MediaMTX 8554, 8889, 8888, 8189, 9997, 9998; TSL 8912/8913 |
| mxl-multiviewer | 8110 | 3262, 3263 | TSL 8910/8911; built-in MediaMTX 8754, 8789, 8788, 8389 |
| mxl-srt-gateway | 8120 | 3272, 3273 | SRT 9000-9099/udp |
| mxl-test-player | 8130 | 3282 | |
| mxl-color-corrector | 8140 | 3292 | |
| mxl-replay | 8150 | 3302, 3303 | TSL 8914/8915; built-in MediaMTX 8854, 8689, 8688, 8489 |
| mxl-browser-source | 8160 | 3312 | |
| mxl-scopes | 8170 | 3322 | built-in MediaMTX 8574, 8579, 8578, 8279 |
| mxl-audio-mixer | 8190 | 3342 | TSL 8910/8911 (move it next to a multiviewer) |

Known clashes on one host: the example registry's query WebSocket (3212) and
mxl-decklink's NMOS port; mxl-decklink and mxl-st2110-gateway on 8080; the
multiviewer's and audio mixer's TSL ports. Move one side.

## Security

The web and NMOS APIs have no authentication. Keep them on a protected network
or put an authenticating proxy in front. No function stores secrets in its
configuration export.
