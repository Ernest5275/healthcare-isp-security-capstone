# Addressing Plan

## Hospital networks
| Site | Clinical VLAN 10 | Guest VLAN 30 | HSRP gateway (.1) | R1 / R2 addresses |
|---|---|---|---|---|
| NIH | 10.20.10.0/24 | 10.20.30.0/24 | 10.20.10.1 / 10.20.30.1 | .2 (NIH-R1, active) / .3 (NIH-R2) |
| Cedars-Sinai | 10.30.10.0/24 | 10.30.30.0/24 | 10.30.10.1 / 10.30.30.1 | .2 (CEDARS-R1, active) / .3 (CEDARS-R2) |
| Mayo Clinic | 10.40.10.0/24 | 10.40.30.0/24 | 10.40.10.1 / 10.40.30.1 | .2 (MAYO-R1, active) / .3 (MAYO-R2) |

HSRP: R1 priority 105 with preempt, tracking its primary serial WAN link; R2 default priority 100 with preempt.

### DHCP (after corrections)
| Router | Hands out | Gateway | DNS |
|---|---|---|---|
| R1 at each site | .21 – .150 | HSRP virtual address (.1) | 10.10.10.20 |
| R2 at each site (backup) | .151 – .254 | HSRP virtual address (.1) | 10.10.10.20 |

## WAN links (serial, /30)
| Link | Network | Addresses |
|---|---|---|
| ISP-CORE-R1 ↔ NIH-R1 | 172.16.0.0/30 | .1 / .2 |
| ISP-CORE-R1 ↔ CEDARS-R1 | 172.16.0.4/30 | .5 / .6 |
| ISP-CORE-R1 ↔ MAYO-R1 | 172.16.0.8/30 | .9 / .10 |
| ISP-CORE-R1 ↔ ISP-CORE-R2 (Ethernet transit) | 172.16.0.12/30 | .13 / .14 |
| ISP-CORE-R2 ↔ NIH-R2 | 172.16.0.16/30 | .17 / .18 |
| ISP-CORE-R2 ↔ CEDARS-R2 | 172.16.0.20/30 | .21 / .22 |
| ISP-CORE-R2 ↔ MAYO-R2 | 172.16.0.24/30 | .25 / .26 |

## ISP core and perimeter
| Link / network | Addresses |
|---|---|
| ISP-CORE-R1 ↔ ASA1 (inside) | 10.10.254.0/30 (.1 core, .2 ASA1) |
| ISP-CORE-R2 ↔ ASA0 (inside) | 10.10.254.4/30 (.5 core, .6 ASA0) |
| ASA outside / Internet edge | 11.0.0.0/29 (ASA1 .1, Internet router .2, ASA0 .3) |
| External web server (server.com) | 11.1.1.10 |

## Server network (10.10.10.0/24)
| Server | Address |
|---|---|
| Gateway (HSRP on core routers) | 10.10.10.1 |
| EHR server / standby EHR | 10.10.10.10 / 10.10.10.11 |
| Primary DNS / secondary DNS | 10.10.10.20 / 10.10.10.21 |
| Backup server | 10.10.10.30 |

## Routing
- **OSPF area 0** on all hospital and core routers (hospital LAN subinterfaces are passive)
- Hospital routers: static default route to their core router
- Core routers: default route to their own firewall, plus a **floating default route (AD 20)** through the other core
- ASAs: dynamic PAT (NAT) for 10.0.0.0/8 to their outside interface
