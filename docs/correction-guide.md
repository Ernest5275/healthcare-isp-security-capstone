# Configuration Correction Guide

Commands used to correct the gaps found in my configuration review. Parts 1–4 were applied and tested (see [test results](../tests/test-results.md)); Part 5 is the planned hardening step and has not been applied.

> Device names: in the Packet Tracer file, the NIH routers are labeled `NH1-R1` and `NHI-R2`. Several R1 routers still use the default hostname `Router`.


## Part 1 – Extend OSPF To Cedars-Sinai And Mayo Clinic

**Why:** Only NIH runs OSPF. Cedars-Sinai and Mayo Clinic use static routes on the ISP cores, so a LAN failure at CEDARS-R1 or MAYO-R1 leaves the core sending traffic to a dead path. The static routes must be removed, because a static route (AD 1 or 10) always beats OSPF (AD 110).


**CEDARS-R1 (display name CEDARS-R1)**
```
enable
configure terminal
router ospf 1
 router-id 3.3.3.1
 passive-interface GigabitEthernet0/0.10
 passive-interface GigabitEthernet0/0.30
 network 172.16.0.4 0.0.0.3 area 0
 network 10.30.10.0 0.0.0.255 area 0
 network 10.30.30.0 0.0.0.255 area 0
 exit
end
write memory
```

**CEDARS-R2**
```
enable
configure terminal
router ospf 1
 router-id 3.3.3.2
 passive-interface GigabitEthernet0/0.10
 passive-interface GigabitEthernet0/0.30
 network 172.16.0.20 0.0.0.3 area 0
 network 10.30.10.0 0.0.0.255 area 0
 network 10.30.30.0 0.0.0.255 area 0
 exit
end
write memory
```

**MAYO-R1**
```
enable
configure terminal
router ospf 1
 router-id 4.4.4.1
 passive-interface GigabitEthernet0/0.10
 passive-interface GigabitEthernet0/0.30
 network 172.16.0.8 0.0.0.3 area 0
 network 10.40.10.0 0.0.0.255 area 0
 network 10.40.30.0 0.0.0.255 area 0
 exit
end
write memory
```

**MAYO-R2**
```
enable
configure terminal
router ospf 1
 router-id 4.4.4.2
 passive-interface GigabitEthernet0/0.10
 passive-interface GigabitEthernet0/0.30
 network 172.16.0.24 0.0.0.3 area 0
 network 10.40.10.0 0.0.0.255 area 0
 network 10.40.30.0 0.0.0.255 area 0
 exit
end
write memory
```

**ISP-CORE-R1**
```
enable
configure terminal
router ospf 1
 network 172.16.0.4 0.0.0.3 area 0
 network 172.16.0.8 0.0.0.3 area 0
 exit
no ip route 10.30.10.0 255.255.255.0 172.16.0.6
no ip route 10.30.30.0 255.255.255.0 172.16.0.6
no ip route 10.40.10.0 255.255.255.0 172.16.0.10
no ip route 10.40.30.0 255.255.255.0 172.16.0.10
no ip route 10.30.10.0 255.255.255.0 172.16.0.14 10
no ip route 10.30.30.0 255.255.255.0 172.16.0.14 10
no ip route 10.40.10.0 255.255.255.0 172.16.0.14 10
no ip route 10.40.30.0 255.255.255.0 172.16.0.14 10
no ip route 172.16.0.16 255.255.255.252 172.16.0.14
no ip route 172.16.0.20 255.255.255.252 172.16.0.14
no ip route 172.16.0.24 255.255.255.252 172.16.0.14
end
write memory
```

**ISP-CORE-R2**
```
enable
configure terminal
router ospf 1
 network 172.16.0.20 0.0.0.3 area 0
 network 172.16.0.24 0.0.0.3 area 0
 exit
no ip route 10.30.10.0 255.255.255.0 172.16.0.22
no ip route 10.30.30.0 255.255.255.0 172.16.0.22
no ip route 10.40.10.0 255.255.255.0 172.16.0.26
no ip route 10.40.30.0 255.255.255.0 172.16.0.26
end
write memory
```

**Verify:**
```
  ISP-CORE-R1#  show ip ospf neighbor      -> 4 FULL neighbors (NIH-R1, CEDARS-R1, MAYO-R1, ISP-CORE-R2)
  ISP-CORE-R2#  show ip ospf neighbor      -> 4 FULL neighbors (NIH-R2, CEDARS-R2, MAYO-R2, ISP-CORE-R1)
  ISP-CORE-R1#  show ip route ospf         -> 10.30.10.0/24 [110/65] via 172.16.0.6 and 10.40.10.0/24 [110/65] via 172.16.0.10
  Failover test: CEDARS-R1 -> interface Gi0/0 -> shutdown
     CEDARS-R2#   show standby brief       -> Active for groups 10 and 30
     ISP-CORE-R1# show ip route ospf       -> 10.30.10.0/24 [110/66] via 172.16.0.14
     CEDARS-CLINICAL-PC> ping 10.10.10.10  -> replies
     CEDARS-R1 -> interface Gi0/0 -> no shutdown  (repeat the same test at Mayo Clinic)
```

## Part 2 – Add A Backup Path To The Secondary Firewall (ASA0)

**Why:** ISP-CORE-R1 sends all Internet traffic to ASA1 and has no backup default route, so traffic reaches ASA0 only through ISP-CORE-R2. A floating default route (AD 20) on each core sends traffic to the other core if its own firewall link fails. ASA0 is also missing "inspect http," which ASA1 has; adding it is the first thing to try for the open ASA0 return-traffic issue.


**ISP-CORE-R1**
```
enable
configure terminal
ip route 0.0.0.0 0.0.0.0 172.16.0.14 20
end
write memory
```

**ISP-CORE-R2**
```
enable
configure terminal
ip route 0.0.0.0 0.0.0.0 172.16.0.13 20
end
write memory
```

**ASA0**
```
enable
configure terminal
policy-map global_policy
 class inspection_default
  inspect http
  exit
 exit
end
write memory
```

**ASA1 (fix default hostname)**
```
enable
configure terminal
hostname ASA1
end
write memory
```

**Verify:**
```
  Test: ISP-CORE-R1 -> interface Gi0/0 -> shutdown   (simulates losing ASA1)
     ISP-CORE-R1#  show ip route              -> S* 0.0.0.0/0 [20/0] via 172.16.0.14
     NIH-CLINICAL-PC> ping server.com         -> replies
     NIH-CLINICAL-PC Web Browser: http://server.com -> page loads
     ASA0#  show xlate                        -> translations for the hospital PC
     ISP-CORE-R1 -> interface Gi0/0 -> no shutdown
  If the web page still fails through ASA0, record it as an open issue in the journal.
```

## Part 3 – Allow Secondary DNS And Tighten Guest Isolation

**Why:** The Guest ACLs allow DNS only to 10.10.10.20, so Guests cannot use secondary DNS 10.10.10.21. The final "permit ip any any" also lets Guests reach other hospitals’ Guest networks, WAN links, and firewall links. These scripts delete and re-create each ACL in one step (it stays applied to the interfaces). Paste the same ACL on BOTH routers at each hospital.


**NH1-R1 and NHI-R2 (paste on both)**
```
enable
configure terminal
no ip access-list extended NIH-GUEST-ISOLATION
ip access-list extended NIH-GUEST-ISOLATION
 permit udp 10.20.30.0 0.0.0.255 host 10.10.10.20 eq domain
 permit tcp 10.20.30.0 0.0.0.255 host 10.10.10.20 eq domain
 permit udp 10.20.30.0 0.0.0.255 host 10.10.10.21 eq domain
 permit tcp 10.20.30.0 0.0.0.255 host 10.10.10.21 eq domain
 deny ip 10.20.30.0 0.0.0.255 10.20.10.0 0.0.0.255
 deny ip 10.20.30.0 0.0.0.255 10.30.10.0 0.0.0.255
 deny ip 10.20.30.0 0.0.0.255 10.40.10.0 0.0.0.255
 deny ip 10.20.30.0 0.0.0.255 10.10.10.0 0.0.0.255
 deny ip 10.20.30.0 0.0.0.255 10.30.30.0 0.0.0.255
 deny ip 10.20.30.0 0.0.0.255 10.40.30.0 0.0.0.255
 deny ip 10.20.30.0 0.0.0.255 172.16.0.0 0.0.255.255
 deny ip 10.20.30.0 0.0.0.255 10.10.254.0 0.0.0.255
 permit ip any any
 exit
end
write memory
```

**CEDARS-R1 and CEDARS-R2 (paste on both)**
```
enable
configure terminal
no ip access-list extended CEDARS-GUEST-ISOLATION
ip access-list extended CEDARS-GUEST-ISOLATION
 permit udp 10.30.30.0 0.0.0.255 host 10.10.10.20 eq domain
 permit tcp 10.30.30.0 0.0.0.255 host 10.10.10.20 eq domain
 permit udp 10.30.30.0 0.0.0.255 host 10.10.10.21 eq domain
 permit tcp 10.30.30.0 0.0.0.255 host 10.10.10.21 eq domain
 deny ip 10.30.30.0 0.0.0.255 10.30.10.0 0.0.0.255
 deny ip 10.30.30.0 0.0.0.255 10.20.10.0 0.0.0.255
 deny ip 10.30.30.0 0.0.0.255 10.40.10.0 0.0.0.255
 deny ip 10.30.30.0 0.0.0.255 10.10.10.0 0.0.0.255
 deny ip 10.30.30.0 0.0.0.255 10.20.30.0 0.0.0.255
 deny ip 10.30.30.0 0.0.0.255 10.40.30.0 0.0.0.255
 deny ip 10.30.30.0 0.0.0.255 172.16.0.0 0.0.255.255
 deny ip 10.30.30.0 0.0.0.255 10.10.254.0 0.0.0.255
 permit ip any any
 exit
end
write memory
```

**MAYO-R1 and MAYO-R2 (paste on both)**
```
enable
configure terminal
no ip access-list extended MAYO-GUEST-ISOLATION
ip access-list extended MAYO-GUEST-ISOLATION
 permit udp 10.40.30.0 0.0.0.255 host 10.10.10.20 eq domain
 permit tcp 10.40.30.0 0.0.0.255 host 10.10.10.20 eq domain
 permit udp 10.40.30.0 0.0.0.255 host 10.10.10.21 eq domain
 permit tcp 10.40.30.0 0.0.0.255 host 10.10.10.21 eq domain
 deny ip 10.40.30.0 0.0.0.255 10.40.10.0 0.0.0.255
 deny ip 10.40.30.0 0.0.0.255 10.20.10.0 0.0.0.255
 deny ip 10.40.30.0 0.0.0.255 10.30.10.0 0.0.0.255
 deny ip 10.40.30.0 0.0.0.255 10.10.10.0 0.0.0.255
 deny ip 10.40.30.0 0.0.0.255 10.20.30.0 0.0.0.255
 deny ip 10.40.30.0 0.0.0.255 10.30.30.0 0.0.0.255
 deny ip 10.40.30.0 0.0.0.255 172.16.0.0 0.0.255.255
 deny ip 10.40.30.0 0.0.0.255 10.10.254.0 0.0.0.255
 permit ip any any
 exit
end
write memory
```

**Verify:**
```
  <site>-R1#  show ip access-lists              -> 13 entries, ending with permit ip any any
  NIH-GUEST-PC>  nslookup server.com          -> resolves
  NIH-GUEST-PC>  ping 10.20.10.1 (Clinical)   -> fails (blocked)
  NIH-GUEST-PC>  ping 10.30.30.1 (Cedars Guest) -> fails (blocked)
  NIH-GUEST-PC>  ping 172.16.0.1 (WAN link)   -> fails (blocked)
  NIH-GUEST-PC Web Browser: http://server.com -> loads;  http://10.10.10.10 -> times out
  NIH-CLINICAL-PC Web Browser: http://10.10.10.10 -> EHR page loads
```

## Part 4 – Add Backup DHCP On The R2 Routers

**Why:** DHCP pools exist only on the R1 router at each hospital. The fix splits each subnet: R1 keeps .21–.150 and R2 serves .151–.254, so both can run at once without handing out duplicate addresses. Both use the HSRP virtual gateway (.1), so clients work through either router.


**NH1-R1**
```
enable
configure terminal
ip dhcp excluded-address 10.20.10.151 10.20.10.254
ip dhcp excluded-address 10.20.30.151 10.20.30.254
end
write memory
```

**NHI-R2**
```
enable
configure terminal
ip dhcp excluded-address 10.20.10.1 10.20.10.150
ip dhcp excluded-address 10.20.30.1 10.20.30.150
ip dhcp pool NIH-CLINICAL-BACKUP
 network 10.20.10.0 255.255.255.0
 default-router 10.20.10.1
 dns-server 10.10.10.20
 exit
ip dhcp pool NIH-GUEST-BACKUP
 network 10.20.30.0 255.255.255.0
 default-router 10.20.30.1
 dns-server 10.10.10.20
 exit
end
write memory
```

**CEDARS-R1**
```
enable
configure terminal
ip dhcp excluded-address 10.30.10.151 10.30.10.254
ip dhcp excluded-address 10.30.30.151 10.30.30.254
end
write memory
```

**CEDARS-R2**
```
enable
configure terminal
ip dhcp excluded-address 10.30.10.1 10.30.10.150
ip dhcp excluded-address 10.30.30.1 10.30.30.150
ip dhcp pool CEDARS-CLINICAL-BACKUP
 network 10.30.10.0 255.255.255.0
 default-router 10.30.10.1
 dns-server 10.10.10.20
 exit
ip dhcp pool CEDARS-GUEST-BACKUP
 network 10.30.30.0 255.255.255.0
 default-router 10.30.30.1
 dns-server 10.10.10.20
 exit
end
write memory
```

**MAYO-R1**
```
enable
configure terminal
ip dhcp excluded-address 10.40.10.151 10.40.10.254
ip dhcp excluded-address 10.40.30.151 10.40.30.254
end
write memory
```

**MAYO-R2**
```
enable
configure terminal
ip dhcp excluded-address 10.40.10.1 10.40.10.150
ip dhcp excluded-address 10.40.30.1 10.40.30.150
ip dhcp pool MAYO-CLINICAL-BACKUP
 network 10.40.10.0 255.255.255.0
 default-router 10.40.10.1
 dns-server 10.10.10.20
 exit
ip dhcp pool MAYO-GUEST-BACKUP
 network 10.40.30.0 255.255.255.0
 default-router 10.40.30.1
 dns-server 10.10.10.20
 exit
end
write memory
```

**Verify:**
```
  Test: NH1-R1 -> interface Gi0/0 -> shutdown
     NIH-CLINICAL-PC -> Desktop -> IP Configuration -> Static, then DHCP  -> address 10.20.10.151 or higher, gateway 10.20.10.1
     NH1-R1 -> interface Gi0/0 -> no shutdown;  repeat at Cedars-Sinai and Mayo Clinic
  Note: Packet Tracer PCs accept only one DNS server, so 10.10.10.21 cannot be handed out by DHCP in the lab.
```

## Part 5 – Basic Device Hardening (Planned – Not Applied)

**Why:** No device has an enable secret, login passwords, or SSH, and unused switch ports are active. IMPORTANT: once these passwords are set, nobody (including teammates and the instructor) can configure the devices without them. Agree on the passwords with the team and list them in your submission notes. The example passwords below are placeholders – change them.


**NH1-R1**
```
enable
configure terminal
hostname NIH-R1
enable secret <ENABLE-SECRET>
service password-encryption
no ip domain-lookup
ip domain-name teamecho.local
username echoadmin privilege 15 secret <ADMIN-PASSWORD>
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
banner motd #Authorized access only. Activity is monitored.#
line con 0
 login local
 exit
line vty 0 4
 login local
 transport input ssh
 exit
end
write memory
```

**NHI-R2**
```
enable
configure terminal
hostname NIH-R2
enable secret <ENABLE-SECRET>
service password-encryption
no ip domain-lookup
ip domain-name teamecho.local
username echoadmin privilege 15 secret <ADMIN-PASSWORD>
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
banner motd #Authorized access only. Activity is monitored.#
line con 0
 login local
 exit
line vty 0 4
 login local
 transport input ssh
 exit
end
write memory
```

**CEDARS-R1**
```
enable
configure terminal
hostname CEDARS-R1
enable secret <ENABLE-SECRET>
service password-encryption
no ip domain-lookup
ip domain-name teamecho.local
username echoadmin privilege 15 secret <ADMIN-PASSWORD>
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
banner motd #Authorized access only. Activity is monitored.#
line con 0
 login local
 exit
line vty 0 4
 login local
 transport input ssh
 exit
end
write memory
```

**CEDARS-R2**
```
enable
configure terminal
hostname CEDARS-R2
enable secret <ENABLE-SECRET>
service password-encryption
no ip domain-lookup
ip domain-name teamecho.local
username echoadmin privilege 15 secret <ADMIN-PASSWORD>
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
banner motd #Authorized access only. Activity is monitored.#
line con 0
 login local
 exit
line vty 0 4
 login local
 transport input ssh
 exit
end
write memory
```

**MAYO-R1**
```
enable
configure terminal
hostname MAYO-R1
enable secret <ENABLE-SECRET>
service password-encryption
no ip domain-lookup
ip domain-name teamecho.local
username echoadmin privilege 15 secret <ADMIN-PASSWORD>
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
banner motd #Authorized access only. Activity is monitored.#
line con 0
 login local
 exit
line vty 0 4
 login local
 transport input ssh
 exit
end
write memory
```

**MAYO-R2**
```
enable
configure terminal
hostname MAYO-R2
enable secret <ENABLE-SECRET>
service password-encryption
no ip domain-lookup
ip domain-name teamecho.local
username echoadmin privilege 15 secret <ADMIN-PASSWORD>
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
banner motd #Authorized access only. Activity is monitored.#
line con 0
 login local
 exit
line vty 0 4
 login local
 transport input ssh
 exit
end
write memory
```

**ISP-CORE-R1**
```
enable
configure terminal
hostname ISP-CORE-R1
enable secret <ENABLE-SECRET>
service password-encryption
no ip domain-lookup
ip domain-name teamecho.local
username echoadmin privilege 15 secret <ADMIN-PASSWORD>
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
banner motd #Authorized access only. Activity is monitored.#
line con 0
 login local
 exit
line vty 0 4
 login local
 transport input ssh
 exit
end
write memory
```

**ISP-CORE-R2**
```
enable
configure terminal
hostname ISP-CORE-R2
enable secret <ENABLE-SECRET>
service password-encryption
no ip domain-lookup
ip domain-name teamecho.local
username echoadmin privilege 15 secret <ADMIN-PASSWORD>
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
banner motd #Authorized access only. Activity is monitored.#
line con 0
 login local
 exit
line vty 0 4
 login local
 transport input ssh
 exit
end
write memory
```

**NIH-Sw1, CEDARS-SW1, MAYO-SW1 (paste on each)**
```
enable
configure terminal
interface FastEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 exit
interface FastEthernet0/2
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 exit
interface range FastEthernet0/3 - 24
 shutdown
 exit
end
write memory
```

**ISP-SW1**
```
enable
configure terminal
interface FastEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 exit
interface FastEthernet0/2
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 exit
interface FastEthernet0/4
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 exit
interface range FastEthernet0/3
 shutdown
 exit
interface range FastEthernet0/5 - 23
 shutdown
 exit
interface range GigabitEthernet0/2
 shutdown
 exit
end
write memory
```

**ISP-SW2**
```
enable
configure terminal
interface FastEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 exit
interface FastEthernet0/2
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 exit
interface range FastEthernet0/3 - 23
 shutdown
 exit
interface range GigabitEthernet0/2
 shutdown
 exit
end
write memory
```

**ISP-EDGE-SW1**
```
enable
configure terminal
interface range FastEthernet0/4 - 24
 shutdown
 exit
interface range GigabitEthernet0/1 - 2
 shutdown
 exit
end
write memory
```

**Verify:**
```
  <router>#  show ip ssh                    -> SSH Enabled - version 2.0
  <switch>#  show port-security             -> Fa0/1 and Fa0/2 secure, violation Restrict
  <switch>#  show interfaces status         -> unused ports "disabled"
  NIH-CLINICAL-PC>  ssh -l echoadmin 10.20.10.2  -> password prompt (SSH works)
  Re-run the Part 1–4 tests to confirm nothing broke.
```
