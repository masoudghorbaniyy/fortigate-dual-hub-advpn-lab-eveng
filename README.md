# FortiGate Dual-Hub ADVPN + SD-WAN Lab

This repository documents a dual-hub ADVPN lab built in EVE-NG using FortiGate, FortiManager, BGP, SD-WAN, and ADVPN shortcut tunnels.

The goal of this lab was to validate a dual-hub, dual-transport design where branch-to-branch traffic can initially pass through the hubs and then move to direct on-demand ADVPN shortcuts.

## Lab Overview

The lab includes:

- Dual hubs: Hub1 and Hub2
- Dual transports: INET and MPLS
- Two branch FortiGates
- EVE-NG topology file
- FortiManager-based configuration using CLI scripts, policy package, and metadata variables
- BGP route reflection
- BGP additional-path and iBGP multipath
- SD-WAN overlay zone
- On-demand branch-to-branch ADVPN shortcuts

## Topology Summary

| Device | Role | LAN |
|---|---|---|
| FGT_61 | Hub1 | 10.10.61.0/24 |
| FGT_62 | Hub2 | 10.10.62.0/24 |
| FGT_63 | Branch1 | 10.10.63.0/24 |
| FGT_64 | Branch2 | 10.10.64.0/24 |

## Overlay Summary

| Overlay | Transport | Hub |
|---|---|---|
| Overlay1_INET | INET | Hub1 |
| Overlay2_MPLS | MPLS | Hub1 |
| Overlay3_INET | INET | Hub2 |
| Overlay4_MPLS | MPLS | Hub2 |

## Repository Structure

```text
.
├── Diagram/
│   └── Topology diagrams for the EVE-NG lab
│
├── EVE-NG/
│   └── EVE-NG lab/topology file used to build the environment
│
├── Metadata Var - FMG/
│   └── FortiManager metadata variable references used for dynamic templates
│
├── Policy Package - FMG/
│   └── FortiManager policy package configuration and related notes
│
├── Screenshots/
│   └── Validation screenshots, BGP outputs, SD-WAN outputs, and ADVPN shortcut proof
│
├── Scripts - FMG/
│   └── FortiManager CLI scripts for IPsec, BGP, SD-WAN, and policy configuration
│
└── Underlay Config/
    └── Underlay router/interface configuration for INET and MPLS transport simulation
```

## Design Notes

This lab uses the traditional BGP-per-overlay ADVPN design.

Each branch has separate iBGP sessions over each overlay tunnel:

- Branch to Hub1 over INET
- Branch to Hub1 over MPLS
- Branch to Hub2 over INET
- Branch to Hub2 over MPLS

The hubs act as BGP route reflectors and advertise branch routes back to the spokes. BGP additional-path and iBGP multipath are used so the branches can install multiple usable paths for the remote branch LANs.

SD-WAN is then configured on the branch overlay interfaces so traffic can be steered across the available ADVPN overlays.

## Important Troubleshooting Lessons

### 1. SD-WAN member reference issue

Tunnel interfaces cannot be added as SD-WAN members while they are still referenced directly in firewall policies.

In this lab, the old per-overlay policies had to be removed first. After that, the tunnel interfaces could be added to the SD-WAN zone.

Example issue:

```text
entry not found in datasource
value parse error before 'Overlay1_INET'
```

The fix was to remove or replace policies that referenced the individual overlays, then recreate the policies using the SD-WAN zone.

### 2. Hub inter-overlay forwarding policies were required

During testing, traffic could enter a hub on one overlay but leave through another overlay depending on the hub's BGP route selection.

For example:

```text
Traffic entered Hub1 on Overlay2_MPLS
Hub1 selected Overlay1_INET to reach Branch2
Traffic was dropped because no policy allowed Overlay2_MPLS -> Overlay1_INET
```

The fix was to add broader hub spoke-to-spoke policies:

```text
Hub1:
Overlay1_INET / Overlay2_MPLS -> Overlay1_INET / Overlay2_MPLS

Hub2:
Overlay3_INET / Overlay4_MPLS -> Overlay3_INET / Overlay4_MPLS
```

### 3. SD-WAN controls the local decision only

Forcing Branch1 to use a specific SD-WAN member only controls how Branch1 sends traffic out.

Once the traffic reaches a hub, the hub still performs its own route lookup. This means the hub may forward traffic out a different overlay unless the ADVPN shortcut has already formed.

### 4. ADVPN shortcuts are on-demand

The hub tunnels stay up for control-plane and normal forwarding.

The branch-to-branch shortcuts are created dynamically when traffic needs them.

A hub hairpin path may look like:

```text
BR1 -> Hub2 -> BR2

1  10.10.63.1
2  172.16.4.1
3  172.16.3.64
4  10.10.64.2
```

A direct ADVPN shortcut path may look like:

```text
BR1 -> BR2 directly

1  10.10.63.1
2  172.16.4.64
3  10.10.64.2
```

## Final Validation

The final validation confirmed that branch-to-branch traffic moved from hub hairpin forwarding to direct ADVPN shortcut forwarding.

Examples of successful shortcut validation:

```text
Overlay4_MPLS_0
198.18.63.2:500 -> 198.18.64.2:500
peer-id: BR2
```

Session output showed the direct gateway to Branch2:

```text
gwy=198.18.64.2
sdwan_mbr_seq=4
reply=84/1/1
```

Traceroute also confirmed that the hub was no longer in the path:

```text
1   10.10.63.1
2   172.16.4.64
3   10.10.64.2
```

## Suggested Reading Order

1. Open/import the lab file from `EVE-NG/`
2. Review the topology in `Diagram/`
3. Review the underlay configuration in `Underlay Config/`
4. Review FortiManager metadata variables in `Metadata Var - FMG/`
5. Review IPsec, BGP, and SD-WAN scripts in `Scripts - FMG/`
6. Review policy package notes in `Policy Package - FMG/`
7. Review validation screenshots in `Screenshots/`

## Useful Verification Commands

```bash
get router info bgp summary
get router info bgp network 10.10.64.0/24
get router info routing-table details 10.10.64.0
get router info routing-table bgp | grep port1

diagnose sys sdwan member
diagnose sys sdwan health-check
diagnose sys sdwan service4
diagnose firewall proute list

diagnose vpn ike gateway list
diagnose sys session filter clear
diagnose sys session filter proto 1
diagnose sys session filter dst 10.10.64.2
diagnose sys session list
```

## Notes

This is a lab environment and not a production-ready design. The purpose is to understand the interaction between ADVPN, BGP, SD-WAN, route recursion, firewall policies, and dynamic shortcut formation.

For newer FortiOS designs, BGP over loopback and dynamic BGP can also be considered, but this lab focuses on the traditional BGP-per-overlay approach because it is easy to visualize and works well for learning and FortiManager template practice.

## Disclaimer

All IP addresses, device names, and topology details in this repository are for lab and educational purposes only.
