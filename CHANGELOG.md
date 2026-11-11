# Changelog

## 0.3.0 - unreleased

- README: a summary of what the collection contains, a "What you can build"
  table, the functions grouped by role with hardware needs and licenses, and
  release badges instead of version numbers that went stale.
- Added mxl-avsync (now released) and more complementary projects (MXL
  docs and awesome-mxl, EASY MXL, mxl-k8s, BCP-007-03).
- States that this is an independent community project.
- Compose examples: current releases. mxl-test-player 1.3.1, mxl-scopes 1.2.1,
  mxl-audio-mixer 1.6.0, mxl-replay 1.8.7, mxl-browser-source 1.4.0,
  mxl-srt-gateway 1.5.1, mxl-fabrics-agent 1.7.4 (the others are unchanged).
  The single-host set was run together on one host with an NVIDIA GPU, every
  function routed over IS-05. The two-host example passes `docker compose
  config` only.
- Single host: the replay could not open its database, because its image runs
  as uid 1000 and the named volumes for `/config` and `/data/replay` were
  owned by root. `volume-owner` now hands them to uid 1000 as well.

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
