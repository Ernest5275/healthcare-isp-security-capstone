# Security Assessment – Healthcare ISP Network

**Scope:** Running configurations of all 17 network devices in the team's Cisco Packet Tracer model: 6 hospital routers, 2 ISP core routers, 1 Internet router, 6 switches, and 2 Cisco ASA firewalls.
**Method:** Configuration review against the design requirements, followed by remediation and validation testing in a copy of the file.
**Assessor:** Ernest Foubi (Network Architect / Engineer), November 2026.
**Context:** Academic capstone based on a simulated RFP. Findings apply to the simulation model, not to any real organization.

## Risk rating
| Rating | Meaning |
|---|---|
| **High** | Could expose patient data or allow unauthorized device control |
| **Medium** | Weakens isolation, availability, or detection |
| **Low** | Hygiene issue; makes operations or incident response harder |

## Findings summary
| ID | Finding | Rating | HIPAA Security Rule | NIST CSF 2.0 | Status |
|---|---|---|---|---|---|
| F-01 | Guest ACLs end in `permit ip any any`, so Guest networks can reach other hospitals' Guest networks, WAN links, and firewall links | High | §164.312(a)(1) Access control | PR.IR-01 | ✅ Remediated |
| F-02 | No enable secret, no console/VTY credentials, no SSH on any device | High | §164.312(a)(1), §164.312(d) Authentication | PR.AA-01, PR.PS-01 | ⏳ Open |
| F-03 | No logging, syslog, or NTP configured, so events cannot be collected or correlated | Medium | §164.312(b) Audit controls | DE.CM-01 | ⏳ Open |
| F-04 | Unused switch ports enabled; no port security on access ports | Medium | §164.310 / §164.312(a)(1) | PR.PS-01, PR.IR-01 | ⏳ Open |
| F-05 | No backup route to the secondary firewall (ASA0); ASA0 lacked HTTP inspection | Medium | §164.308(a)(7) Contingency plan | PR.IR-03 | ✅ Remediated |
| F-06 | Cedars-Sinai and Mayo Clinic used static routes, so a router LAN failure would black-hole traffic | Medium | §164.308(a)(7) | PR.IR-03 | ✅ Remediated |
| F-07 | DHCP only on the primary router at each hospital (single point of failure) | Medium | §164.308(a)(7) | PR.IR-03 | ✅ Remediated |
| F-08 | Guest ACLs blocked the secondary DNS server, so DNS had no fallback | Low | §164.308(a)(7) | PR.IR-03 | ✅ Remediated |
| F-09 | Default hostnames (`Router`, `ciscoasa`) and mislabeled devices (`NH1-R1`, `NHI-R2`) | Low | — | ID.AM-01 | ⏳ Open |

**Result:** 5 of 9 findings remediated and tested; 4 open findings are device-hardening and monitoring items with documented fixes.

## Remediated findings

### F-01 – Guest network isolation (High)
- **Risk:** An untrusted guest device could scan other hospitals' guest networks and network infrastructure, supporting reconnaissance and lateral movement.
- **Fix:** Rebuilt all six Guest ACLs (13 entries each). Guests may use DNS on 10.10.10.20 and 10.10.10.21 and reach the Internet. Guests are denied all Clinical networks, the server network, other Guest networks, 172.16.0.0/16 WAN links, and 10.10.254.0/24 firewall links. The ACL is applied inbound on VLAN 30 at **both** routers per site, so isolation holds during failover.
- **Validation:** Guest PCs at all three hospitals blocked from the EHR server and other Guest networks; ACL counters show hits on the new deny entries; Internet access still works.

### F-05 – Firewall path redundancy (Medium)
- **Fix:** Floating default routes (administrative distance 20) on both core routers; `inspect http` added to ASA0 to match ASA1.
- **Validation:** With the ASA1 link down, the floating route became active, ASA0 translated hospital traffic (`show xlate`), and ping to `server.com` succeeded through ASA0.

### F-06 / F-07 / F-08 – Availability
- **Fix:** OSPF at all hospital routers with 15 conflicting static routes removed; backup DHCP pools on R2 routers using split ranges; secondary DNS permitted in Guest ACLs.
- **Validation:** Cores went from 2 to 4 OSPF neighbors each; during a Cedars-Sinai LAN failure, traffic rerouted automatically and a Clinical PC still reached the EHR; during an NIH failure, a client received an address from the backup router.

Evidence: [test results](../tests/test-results.md) · Commands: [correction guide](correction-guide.md)

## Open findings – recommended remediation
| ID | Recommendation |
|---|---|
| F-02 | `enable secret`, local admin user with `secret`, `login local` on console and VTY, `transport input ssh`, SSH version 2 with RSA keys; in production, centralized AAA (TACACS+/RADIUS) with MFA |
| F-03 | Send logs to a central syslog/SIEM server, enable timestamps, and synchronize clocks with NTP so events can be correlated across devices |
| F-04 | Shut down unused ports, enable port security (maximum 1 MAC, sticky, violation restrict) on PC ports, and move trunks to an unused native VLAN |
| F-09 | Set hostnames to match the asset inventory and correct the device labels |

Hardening (F-02, F-04) was deliberately not applied in the class submission so the team and instructor can still open and verify every device.

## Limitations
- The assessment covers the Packet Tracer model only; it does not test real circuits, 5G service, or the proposed Azure EHR environment.
- Findings are mapped to HIPAA and NIST CSF for context. A simulation does not establish HIPAA compliance.
