# Day 1 — Core Deployment

## Environment
- DigitalOcean droplet, Ubuntu 24.04 (LTS) x64, 4GB/2vCPU, Frankfurt region
- Switched from local WSL2/VirtualBox after hitting the gtp5g kernel module
  incompatibility (WSL2's custom kernel can't load out-of-tree modules
  without a full kernel rebuild) and a Hyper-V/VirtualBox coexistence
  conflict on the local machine.

## Deployment
- Cloned free5gc-compose, checked out tag v3.4.0
- `docker compose up -d` — all NFs (AMF, AUSF, CHF, NRF, NSSF, PCF, SMF,
  UDM, UDR, N3IWF, N3IWUE, UERANSIM gNB, WebUI) came up cleanly on first try

## Issue: UPF crash-looping — gtp5g version mismatch
- UPF (free5gc/upf:v3.3.0) requires gtp5g kernel module in range
  0.8.1 <= version < 0.9.0
- Default `git clone` of free5gc/gtp5g pulls latest (0.10.x at time of
  writing) — incompatible, UPF fails with `open Gtp5g: version mismatch`
- Fix: built gtp5g from tag v0.8.10 specifically. Note: rebuilding alone
  wasn't enough — the old module stayed resident in the kernel until
  explicitly `rmmod`'d and `depmod -a` was run to refresh module
  dependencies before `modprobe` would load the correct version.

## Result
- Full free5GC core stack healthy, UPF confirmed running (PFCP server +
  GTP forwarder started, no crash on restart)
- WebConsole reachable via SSH tunnel (port 5000 kept off the public
  interface deliberately — ufw was inactive on the droplet, so tunneling
  avoids exposing the console to the internet)
