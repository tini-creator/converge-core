Converge Core
A cloud-native 5G Standalone (SA) core deployment exploring RAN-core convergence, built on **free5GC** and **UERANSIM**. This project simulates IoT device fleet connectivity, a custom network-exposure API layer, multi-operator/roaming concepts, and device-level isolation: patterns that are relevant to API-first, cloud-native mobile core providers.

> **Status:** Work in progress (Day 1 of 10 complete). This README will be updated as the project develops.

## Overview

| | |
|---|---|
| Core network | free5GC v3.4.0 |
| RAN simulator | UERANSIM |
| Deployment | Docker Compose on a cloud VM (DigitalOcean, Ubuntu 22.04) |
| Focus areas | RAN-core signaling, IoT device fleet simulation, network exposure APIs, roaming concepts, device isolation |

## Why a cloud VM instead of local

Initial attempts to run this locally (WSL2, then VirtualBox) hit two blocking issues: WSL2's kernel can't load the `gtp5g` out-of-tree kernel module the UPF depends on without a full custom kernel rebuild, and VirtualBox conflicted with WSL2's underlying Hyper-V virtualization layer on this machine. Moving to a standard Linux cloud VM sidestepped both issues entirely — and arguably better reflects how a real cloud-native core would be deployed. See `NOTES.md` for the full debugging trail.

## Architecture (current state)

Full free5GC control plane (AMF, AUSF, CHF, NRF, NSSF, PCF, SMF, UDM, UDR) + UPF + WebUI, deployed via Docker Compose, with a UERANSIM gNB for RAN simulation. WebConsole is accessed via SSH tunnel only — deliberately not exposed on the public interface.

[UERANSIM gNB/UE] -- N1/N2/N3 --> [free5GC Core: AMF/SMF/UPF/...] -- N6 --> [Internet/DN]

## Setup

Prerequisites: Docker, Docker Compose, a Linux host with a standard kernel (see note above on why not WSL2).

```bash
git clone --recursive https://github.com/free5gc/free5gc-compose.git
cd free5gc-compose
git checkout v3.4.0
```

**Important:** the UPF requires the `gtp5g` kernel module, version `>=0.8.1, <0.9.0` specifically (not the latest release — see `NOTES.md`):

```bash
git clone https://github.com/free5gc/gtp5g.git
cd gtp5g
git checkout v0.8.10
make clean && make
sudo make install
sudo depmod -a
sudo modprobe gtp5g
```

Then bring up the stack:
```bash
cd ../free5gc-compose
docker compose up -d
docker compose ps -a   # confirm all services, including upf, show Up
```

## Roadmap

- [x] Day 1 — Core deployment, UPF stabilized
- [ ] Day 2 — RAN simulation, UE registration + PDU session
- [ ] Day 3 — Signaling capture and sequence diagram
- [ ] Day 4 — Multi-device fleet scenario
- [ ] Day 5 — Network exposure API
- [ ] Day 6 — Multi-operator/roaming concept
- [ ] Day 7 — Device isolation
- [ ] Day 8 — Resilience/failure testing
- [ ] Day 9 — Documentation polish
- [ ] Day 10 — Final review

## Notes

See `NOTES.md` for day-by-day technical findings and debugging details.
EOF
