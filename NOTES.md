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

# Day 2 — RAN Simulation & End-to-End Data Plane

## Issue: registration/PDU session succeeded, but no data-plane traffic
- UE registration and PDU session establishment both succeeded (NAS Registration
  Accept, PDU Session Establishment Accept), TUN interface `uesimtun0` came up
  with a valid IP.
- However, pinging through the tunnel showed 100% packet loss.

## Investigation
- Ruled out UPF-to-internet reachability separately (confirmed NAT/masquerade
  rule existed, ip_forward enabled).
- tcpdump on the host revealed ICMP requests reached 8.8.8.8 and replies came
  back to the UPF's data-network interface — but were never re-encapsulated
  into GTP-U and sent back toward the UE. Confirmed by climbing TX error
  counters on the UPF's `upfgtp` interface.
- Investigated NIC checksum/segmentation offload as a possible cause (a
  documented pattern for this exact symptom on cloud VMs) — disabled
  toggleable offload features on the host interface; did not resolve it.

## Root cause
- The deployment was pinned to an older combination: free5gc-compose v3.4.0's
  docker-compose.yaml referenced UPF image v3.3.0, paired with gtp5g v0.8.10
  (chosen to satisfy that UPF's version-range check). This older UPF/gtp5g
  pairing appears to have a genuine return-path GTP-U encapsulation bug,
  separate from and easily mistaken for a checksum-offload issue.

## Fix
- Upgraded to free5gc-compose v3.4.5, which pulls a newer, rewritten
  "Go-UPF" (free5gc/upf:v3.4.5) expecting gtp5g v0.9.5 specifically
  (per the project's current documented requirement).
- Rebuilt gtp5g at v0.9.5, reloaded the kernel module, redeployed the stack.
- Also hit and resolved a corrupted MongoDB volume (WiredTiger error) from
  repeated version switches — resolved by removing the named `dbdata` volume
  and letting it reinitialize.

## Result
- Full end-to-end connectivity confirmed: UE registers, establishes a PDU
  session, and passes real ICMP traffic through the tunnel to the public
  internet with 0% packet loss (~1.5-5ms RTT from the droplet).
