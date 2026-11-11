# Changelog

## 0.3.0 - unreleased

- README: a summary of what the collection contains, a "What you can build"
  table, the functions grouped by role with hardware needs and licenses, and
  release badges instead of version numbers that went stale.
- Added mxl-avsync (now released) and more complementary projects (MXL
  docs and awesome-mxl, EASY MXL, mxl-k8s, BCP-007-03).
- States that this is an independent community project.
- The Compose examples are unchanged (same pinned images as 0.2.0).

## 0.2.0 - 2026-10-24

- FlowXer vision mixer (19.26.68) as a media function: README table,
  requirements and ports, single-host (`--profile flowxer` with the GPU
  overlay) and host-b of the multi-host example.
- NMOS crosspoint controller (dmf-app-nmos-crosspoint 4.36) next to the
  registry in both Compose examples, with routing instructions.
- Single host: `monitor-config-owner` is now `volume-owner` (also FlowXer's
  volumes).

## 0.1.0 - 2026-10-24

- First version: overview of the MXL media functions, host requirements,
  single-host and two-host (RDMA) Docker Compose examples, Kubernetes notes.
- Pinned images: mxl-st2110-gateway 1.0.20, mxl-srt-gateway 1.4.4,
  mxl-decklink 1.0.7, mxl-test-player 1.2.4, mxl-multiviewer 1.4.6,
  mxl-webrtc-monitor 1.4.2, mxl-color-corrector 1.2.3, mxl-scopes 1.1.0,
  mxl-replay 1.8.1, mxl-browser-source 1.3.3, mxl-audio-mixer 1.4.3,
  mxl-fabrics-agent 1.7.3.
