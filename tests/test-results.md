# Test Results

All tests were run in Cisco Packet Tracer on a corrected copy of the team's master file (November 2026). Screenshots are in [`/images`](../images).

## 1. OSPF at all hospitals
| Test | Expected | Actual | Evidence |
|---|---|---|---|
| ISP-CORE-R1 OSPF neighbors (before) | 2 | 2 (ISP-CORE-R2, NIH-R1) | [01](../images/01-core-r1-before-ospf.png) |
| ISP-CORE-R1 OSPF neighbors (after) | 4 FULL | 4 FULL (+ CEDARS-R1, MAYO-R1) | [02](../images/02-core-r1-after-ospf.png) |
| ISP-CORE-R2 OSPF neighbors (after) | 4 FULL | 4 FULL (+ CEDARS-R2, MAYO-R2) | [03](../images/03-core-r2-after-ospf.png) |
| Hospital networks on the cores | Learned as OSPF (O) routes | 10.20.x, 10.30.x, 10.40.x all OSPF, metric 65 | [02](../images/02-core-r1-after-ospf.png) |

## 1a. Cedars-Sinai failover (CEDARS-R1 Gi0/0 shut down)
| Test | Expected | Actual | Evidence |
|---|---|---|---|
| HSRP takeover | CEDARS-R2 Active for groups 10 and 30 | Active for both groups | [05](../images/05-cedars-r2-hsrp-active.png) |
| Core reroute | 10.30.10.0/24 via 172.16.0.14, metric 66 | `[110/66] via 172.16.0.14` | [04](../images/04-cedars-failover-route.png) |
| Clinical PC → EHR (10.10.10.10) | Replies | 3/4 replies (first lost during convergence) | [06](../images/06-cedars-failover-ping.png) |
| Restore | CEDARS-R1 back to Active (preempt) | Restored; route back to metric 65 via 172.16.0.6 | — |

## 2. ASA0 backup path (ISP-CORE-R1 Gi0/0 to ASA1 shut down)
| Test | Expected | Actual | Evidence |
|---|---|---|---|
| Floating default route | `S* 0.0.0.0/0 [20/0] via 172.16.0.14` | As expected | [07](../images/07-asa0-floating-route.png) |
| NIH Clinical PC → server.com (ping) | Replies through ASA0 | 3/4 replies | [09](../images/09-asa0-failover-ping.png) |
| ASA0 NAT translation | Entry for the NIH PC | `ICMP PAT from inside:10.20.10.21 to outside:11.0.0.3` | [08](../images/08-asa0-xlate.png) |
| Web access to http://server.com | Page loads | Page loaded | [10](../images/10-asa0-failover-web.png) |

## 3. Guest isolation (all three hospitals)
| Test (from Guest PC) | Expected | NIH | Cedars-Sinai | Mayo Clinic |
|---|---|---|---|---|
| DNS lookup of server.com | Resolves | ✅ | ✅ | ✅ |
| Ping EHR server 10.10.10.10 | Blocked | ✅ | ✅ | ✅ |
| Ping another hospital's Guest gateway | Blocked | — | ✅ | ✅ |
| Ping WAN link 172.16.0.1 | Blocked | ✅ | — | — |
| Ping server.com (Internet) | Replies | ✅ | ✅ | ✅ |

The ACL match counters on CEDARS-R1 show hits on the new deny entries ([11](../images/11-cedars-guest-acl-counters.png)). Mayo Clinic test: [12](../images/12-mayo-guest-isolation-test.png).

## 4. Backup DHCP (NIH-R1 Gi0/0 shut down)
| Test | Expected | Actual | Evidence |
|---|---|---|---|
| NIH-CLINICAL-PC renews DHCP | Address .151 or higher from NIH-R2 | 10.20.10.151, gateway 10.20.10.1 | [13](../images/13-nih-backup-dhcp.png) |
| NIH-R2 configuration | Backup pools and 13-entry Guest ACL | As expected | [14](../images/14-nih-r2-dhcp-acl.png) |

## Lessons from testing
- **Confirm the device before every change.** Commands were once pasted into the wrong router; running `show ip interface brief` and checking a unique IP address became a standard check.
- **Test the outcome, not just the configuration.** HSRP takeover worked even before the fix; the routing table and end-to-end ping proved the OSPF correction.
- **Expect one lost packet during failover.** The first ping often timed out while the path converged.

## Not tested in simulation
Physical circuit diversity, 5G backup, Azure EHR services, and device hardening.
