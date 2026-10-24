# Kubernetes notes

Hints, not manifests. Each function repository has its own example; start
from those:

| Function | Example |
| --- | --- |
| mxl-st2110-gateway | [deploy/k8s](https://github.com/LeeO86/mxl-st2110-gateway/tree/main/deploy/k8s) (kustomize, SR-IOV, two-host fabrics variant) |
| mxl-srt-gateway | [deploy/mxl-srt-gateway.yaml](https://github.com/LeeO86/mxl-srt-gateway/blob/main/deploy/mxl-srt-gateway.yaml), [-gpu](https://github.com/LeeO86/mxl-srt-gateway/blob/main/deploy/mxl-srt-gateway-gpu.yaml) |
| mxl-decklink | [deploy/mxl-decklink.yaml](https://github.com/LeeO86/mxl-decklink/blob/main/deploy/mxl-decklink.yaml), [generic-device-plugin.yaml](https://github.com/LeeO86/mxl-decklink/blob/main/deploy/generic-device-plugin.yaml) |
| mxl-test-player | [deploy/k8s/deployment.yaml](https://github.com/LeeO86/mxl-test-player/blob/main/deploy/k8s/deployment.yaml) |
| mxl-multiviewer | [deploy/mxl-multiviewer.yaml](https://github.com/LeeO86/mxl-multiviewer/blob/main/deploy/mxl-multiviewer.yaml), [-gpu](https://github.com/LeeO86/mxl-multiviewer/blob/main/deploy/mxl-multiviewer-gpu.yaml) |
| mxl-webrtc-monitor | [deploy/mxl-webrtc-monitor.yaml](https://github.com/LeeO86/mxl-webrtc-monitor/blob/main/deploy/mxl-webrtc-monitor.yaml) |
| mxl-color-corrector | [deploy/k8s/deployment.yaml](https://github.com/LeeO86/mxl-color-corrector/blob/main/deploy/k8s/deployment.yaml) |
| mxl-scopes | [deploy/k8s/deployment.yaml](https://github.com/LeeO86/mxl-scopes/blob/main/deploy/k8s/deployment.yaml) |
| mxl-replay | [deploy/k8s/deployment.yaml](https://github.com/LeeO86/mxl-replay/blob/main/deploy/k8s/deployment.yaml) |
| mxl-browser-source | [deploy/mxl-browser-source.yaml](https://github.com/LeeO86/mxl-browser-source/blob/main/deploy/mxl-browser-source.yaml), [-gpu](https://github.com/LeeO86/mxl-browser-source/blob/main/deploy/mxl-browser-source-gpu.yaml) |
| mxl-audio-mixer | [deploy/k8s/deployment.yaml](https://github.com/LeeO86/mxl-audio-mixer/blob/main/deploy/k8s/deployment.yaml) |
| mxl-fabrics-agent | [deploy/mxl-fabrics-agent.yaml](https://github.com/LeeO86/mxl-fabrics-agent/blob/main/deploy/mxl-fabrics-agent.yaml) (DaemonSet) |

## One pod per function

- One Deployment with `replicas: 1` and `strategy: Recreate` per function
  instance. Two copies would write the same MXL domain and NMOS ids.
- `terminationGracePeriodSeconds` greater than `SHUTDOWN_TIMEOUT_S` so a
  function can remove its domain and deregister.
- A writable `/config` (PVC or emptyDir) for the saved settings and IS-05
  routes. Probes on `/livez` and `/readyz` of the web port.
- `NMOS_HOST_ADDRESS` from `status.podIP` (downward API) on the pod network;
  the node IP for host-network pods.
- `NMOS_SEED` unique and stable per instance (e.g. `<production>-<function>`).

## The MXL domain

MXL is shared memory through files, so all pods that exchange flows must be on
the **same node** and see the **same tmpfs**.

- Recommended: a host tmpfs at `/Volumes/mxl` on every media node (see
  [requirements](requirements.md)) and a `hostPath` volume
  (`type: Directory`) in each pod. Pods of one node then share every domain;
  the fabrics agent mirrors flows from other nodes into it.
- `emptyDir: {medium: Memory}` is a tmpfs too, but it belongs to one pod.
  Use it only for containers in the same pod (e.g. a writer sidecar), or for a
  function's own output domain when nothing on the node reads it.
- Schedule functions that work together onto one node (node selector or pod
  affinity), unless the fabrics agent bridges the nodes.
- Count the tmpfs (Memory) usage: it is node memory outside the pods' limits
  when it is a hostPath.

## Privileges

No function needs `privileged: true`. Typical security context:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["SYS_NICE"]        # only where a real-time priority is set
```

- No `hostIPC` (MXL does not use SysV IPC).
- `IPC_LOCK`: fabrics agent (RDMA), DeckLink, ST 2110 gateway.
- Fabrics agent: also `hostNetwork`, `hostPID`, `SYS_PTRACE` and
  `appArmorProfile: {type: Unconfined}` for its mirror reader check.
- ST 2110 gateway with DPDK: `runAsUser: 0` (VFIO device nodes), hugepages
  resources, host network.
- Browser source: an `rtprio` ulimit cannot be set in Kubernetes; see its
  README for the real-time options.

## Real time: CPU manager

- Kubelet `cpuManagerPolicy: static` (optionally
  `cpuManagerPolicyOptions: full-pcpus-only: "true"`) and
  `reservedSystemCPUs` for the system daemons.
- Guaranteed QoS (requests equal limits, whole CPUs) gives a pod exclusive
  cores. The audio mixer wants an even count (whole physical cores, e.g. `"4"`).
- Topology manager `single-numa-node` (or `restricted`) keeps CPUs, NIC
  (SR-IOV/RDMA) and GPU of a pod on one NUMA node.
- Keep RT priorities below 50 (threaded IRQs run at 50). Move NIC IRQs off the
  exclusive cores.

## Devices

- **GPU:** [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/)
  or the device plugin; `runtimeClassName: nvidia`, `nvidia.com/gpu: 1`.
  Time-slicing lets several functions share a GPU; NVENC session limits on
  consumer cards still apply per GPU.
- **RDMA:** an RDMA shared device plugin, or the generic device plugin pattern
  exposing `/dev/infiniband` (see the DeckLink and fabrics agent examples).
- **DeckLink:** generic device plugin for `/dev/blackmagic`, the host's
  `libDeckLinkAPI.so` as a read-only hostPath.
- **ST 2110 NICs:** SR-IOV Network Device Plugin with one resource per media
  port, and [Multus](https://github.com/k8snetworkplumbingwg/multus-cni) when
  a pod needs a second network (RDMA or media) next to the pod network.

## NMOS registry

- Run one nmos-cpp registry (Deployment + Service). Registration port `P`,
  Query API `P+1`.
- Set `NMOS_DNS_SD=false` and `NMOS_REGISTRY_ADDRESS` (Service name or IP) in
  every function. mDNS does not cross the pod network; no Avahi needed.
- The IS-05 controller must reach every node's `NMOS_HOST_ADDRESS`.

## MediaMTX

- One shared MediaMTX (Deployment + Service) for all previews: functions get
  `PREVIEW_PUBLISH_URL=rtsp://mediamtx:8554`, a unique
  `PREVIEW_PATH_PREFIX`, `MEDIAMTX_API_URL=http://mediamtx:9997`.
- Browsers need WHEP over an ingress (`PREVIEW_WHEP_URL`) and the ICE UDP
  port reachable (host port or a LoadBalancer with UDP), plus
  `webrtcAdditionalHosts` set to the public address.

## Time

- Every media node runs PTP (`linuxptp`: `ptp4l` + `phc2sys`) or chrony with
  the TAI offset, outside the pods (systemd or a privileged DaemonSet such as
  the PTP operator). Pods read `CLOCK_TAI` from the node kernel.

## Multi-node

- Fabrics agent as a DaemonSet on host network, one per media node, with a
  per-node peer map (ConfigMap selected by `HOST_ID` = node name).
- Set `LOCAL_NODE_CIDRS` to the node's pod CIDR so NMOS nodes on the pod
  network count as local.
