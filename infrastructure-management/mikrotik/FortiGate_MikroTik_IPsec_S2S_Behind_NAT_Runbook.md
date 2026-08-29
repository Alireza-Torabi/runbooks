# FortiGate ↔ MikroTik Site-to-Site IPsec VPN Runbook

## MikroTik Behind NAT / No Public IP

**Document type:** Production Runbook
**VPN type:** Site-to-Site IPsec
**IKE version:** IKEv2
**Authentication:** Pre-Shared Key (PSK)
**FortiGate role:** Publicly reachable responder / hub
**MikroTik role:** NATed initiator / branch
**Traffic model:** Bidirectional LAN-to-LAN communication after the tunnel is established
**RouterOS baseline:** RouterOS v7
**FortiOS baseline:** FortiOS 7.x

> All addresses in this document are examples only. They are taken from private or documentation address ranges and must be replaced before deployment.

---

## 1. Purpose

This runbook describes how to build a site-to-site IPsec VPN between:

* a **FortiGate with a public/static Internet address**, and
* a **MikroTik that does not have a public IP address and is located behind an upstream NAT device**.

The important design point is that the MikroTik cannot be treated as a normal fixed-IP IPsec peer.

The FortiGate is therefore configured as a **dynamic/dial-up IPsec responder**, while the MikroTik always initiates the IKEv2 session toward the FortiGate.

The MikroTik identifies itself using a fixed IKE Peer ID rather than relying on its translated public IP address.

---

## 2. Example Topology

```text
                       Internet
                           |
                           |
                 Public IP: 203.0.113.10
                    +-------------+
                    |  FortiGate  |
                    |   wan1      |
                    +-------------+
                           |
                    10.10.10.1/24
                           |
                FortiGate Internal LAN
                    10.10.10.0/24
                           |
             Example servers / applications
                    10.10.10.20
                    10.10.10.30
                    10.10.10.53 DNS
                           |
===================== IPsec / IKEv2 =====================
                 NAT-T over UDP/4500
==========================================================
                           |
                    Upstream NAT / ISP
                 Public IP may be dynamic
                  or even ISP/CGNAT based
                           |
                  192.0.2.1/24
                           |
                    192.0.2.10/24
                    +-------------+
                    |  MikroTik   |
                    |    WAN      |
                    +-------------+
                           |
                    10.20.20.1/24
                           |
                  MikroTik Branch LAN
                    10.20.20.0/24
                           |
                       Clients
```

---

## 3. Example Addressing

| Component                  | Example Value          |
| -------------------------- | ---------------------- |
| FortiGate WAN interface    | `wan1`                 |
| FortiGate public IP        | `203.0.113.10`         |
| FortiGate LAN interface    | `lan`                  |
| FortiGate LAN subnet       | `10.10.10.0/24`        |
| FortiGate LAN gateway      | `10.10.10.1`           |
| Example central DNS server | `10.10.10.53`          |
| MikroTik WAN private IP    | `192.0.2.10/24`        |
| MikroTik upstream gateway  | `192.0.2.1`            |
| MikroTik LAN subnet        | `10.20.20.0/24`        |
| MikroTik LAN gateway       | `10.20.20.1`           |
| MikroTik IKE Peer ID       | `branch-mikrotik`      |
| IKE version                | `IKEv2`                |
| Phase 1 encryption         | `AES-256`              |
| Phase 1 integrity          | `SHA-256`              |
| DH group                   | `14 / MODP-2048`       |
| Phase 2 encryption         | `AES-256-CBC`          |
| Phase 2 integrity          | `SHA-256`              |
| PFS                        | `Group 14 / MODP-2048` |
| Phase 1 lifetime           | `86400 seconds`        |
| Phase 2 lifetime           | `3600 seconds`         |

---

# 4. Critical Design Notes

## 4.1 MikroTik is behind NAT

The MikroTik WAN interface only has a private address.

The upstream router or ISP translates its traffic before it reaches the Internet.

Therefore:

```text
FortiGate cannot reliably identify the MikroTik by source public IP.
```

The FortiGate must use:

```text
Dynamic / Dial-up IPsec peer
```

and identify the MikroTik using:

```text
IKE Peer ID = branch-mikrotik
```

---

## 4.2 MikroTik must initiate the tunnel

The initial tunnel establishment is:

```text
MikroTik
   |
   | IKEv2 UDP/500
   v
Upstream NAT
   |
   | NAT detection
   v
FortiGate
   |
   | NAT-T
   v
UDP/4500 IPsec session
```

No inbound port-forward is normally required on the NAT router in front of the MikroTik.

The upstream NAT device must permit outbound UDP traffic and the corresponding return traffic.

Required protocols/ports are normally:

```text
UDP 500   - IKE
UDP 4500  - IPsec NAT Traversal
```

When NAT is detected, encrypted ESP traffic is encapsulated inside UDP/4500.

---

## 4.3 FortiGate cannot bring up a completely down tunnel from scratch

Because the MikroTik is behind an unknown NAT mapping, the FortiGate normally cannot initiate a brand-new IKE session toward the MikroTik.

The MikroTik must create the IKE session first.

However, once the tunnel and IPsec Security Associations are established, traffic may be initiated in either direction:

```text
MikroTik LAN  -> FortiGate LAN
FortiGate LAN -> MikroTik LAN
```

Therefore hosts behind the FortiGate can connect to hosts behind the MikroTik **as long as the VPN is established and the required policies are present**.

---

# 5. Pre-Deployment Checklist

Before configuring IPsec, verify the following.

## FortiGate

* FortiGate has a working Internet connection.
* The FortiGate WAN IP is reachable from the Internet.
* UDP/500 is not blocked.
* UDP/4500 is not blocked.
* No restrictive Local-In Policy blocks IKE/NAT-T.
* The FortiGate LAN subnet does not overlap the MikroTik LAN subnet.

## MikroTik

* MikroTik has working Internet access through the upstream NAT router.
* MikroTik can reach the FortiGate public IP.
* The upstream NAT/firewall permits outbound UDP/500 and UDP/4500.
* MikroTik LAN does not overlap the FortiGate LAN.
* Existing masquerade rules are known.
* Existing FastTrack rules are known.

Basic MikroTik reachability test:

```routeros
/ping 203.0.113.10
```

---

# 6. Generate a Strong Pre-Shared Key

Do not use the sample strings in this document.

Use a long randomly generated PSK, preferably at least 32 random characters.

Example placeholder:

```text
<STRONG-RANDOM-IPSEC-PSK>
```

The exact same PSK must be configured on both devices.

---

# 7. FortiGate Configuration

## 7.1 Create Address Objects

```fortios
config firewall address
    edit "FG-LAN-10.10.10.0_24"
        set subnet 10.10.10.0 255.255.255.0
    next

    edit "MT-LAN-10.20.20.0_24"
        set subnet 10.20.20.0 255.255.255.0
    next
end
```

Optional DNS host object:

```fortios
config firewall address
    edit "CENTRAL-DNS-10.10.10.53"
        set subnet 10.10.10.53 255.255.255.255
    next
end
```

---

# 8. FortiGate Phase 1 — Dynamic IKEv2 Responder

The FortiGate must **not** be configured with the MikroTik's public IP as a fixed remote gateway.

Configure it as a dynamic peer.

```fortios
config vpn ipsec phase1-interface
    edit "S2S-MT-DIALUP"
        set type dynamic
        set interface "wan1"

        set ike-version 2

        set peertype one
        set peerid "branch-mikrotik"

        set proposal aes256-sha256
        set dhgrp 14
        set keylife 86400

        set nattraversal enable

        set dpd on-idle
        set dpd-retryinterval 10

        set add-route enable

        set psksecret "<STRONG-RANDOM-IPSEC-PSK>"
    next
end
```

### Important parameters

```text
type dynamic
```

Allows peers whose public source address is not known in advance.

```text
peerid "branch-mikrotik"
```

Identifies the MikroTik independently of the translated source IP.

```text
nattraversal enable
```

Allows IPsec to operate when NAT exists between the peers.

```text
add-route enable
```

Allows FortiGate to install the remote subnet route when the dynamic tunnel is established.

---

# 9. FortiGate Phase 2

```fortios
config vpn ipsec phase2-interface
    edit "S2S-MT-DIALUP-P2"
        set phase1name "S2S-MT-DIALUP"

        set proposal aes256-sha256

        set pfs enable
        set dhgrp 14

        set keylifeseconds 3600

        set src-subnet 10.10.10.0 255.255.255.0
        set dst-subnet 10.20.20.0 255.255.255.0
    next
end
```

Encryption domain:

```text
FortiGate LAN:
10.10.10.0/24

MikroTik LAN:
10.20.20.0/24
```

Both sides must use matching traffic selectors.

---

# 10. FortiGate Firewall Policy — MikroTik to FortiGate

Allow the branch network to access the FortiGate-side network.

```fortios
config firewall policy
    edit 0
        set name "VPN-MT-to-FG-LAN"

        set srcintf "S2S-MT-DIALUP"
        set dstintf "lan"

        set srcaddr "MT-LAN-10.20.20.0_24"
        set dstaddr "FG-LAN-10.10.10.0_24"

        set action accept
        set schedule "always"
        set service "ALL"

        set nat disable
        set logtraffic all
    next
end
```

---

# 11. FortiGate Firewall Policy — FortiGate to MikroTik

This policy is required if systems behind the FortiGate must initiate connections toward the MikroTik LAN.

```fortios
config firewall policy
    edit 0
        set name "FG-LAN-to-VPN-MT"

        set srcintf "lan"
        set dstintf "S2S-MT-DIALUP"

        set srcaddr "FG-LAN-10.10.10.0_24"
        set dstaddr "MT-LAN-10.20.20.0_24"

        set action accept
        set schedule "always"
        set service "ALL"

        set nat disable
        set logtraffic all
    next
end
```

For production environments, replace `ALL` with only the services actually required.

For example:

```text
DNS
HTTPS
RDP
SSH
Application-specific TCP ports
```

---

# 12. FortiGate Routing

With a dynamic/dial-up tunnel and:

```text
add-route enable
```

FortiGate can dynamically install a route for the MikroTik LAN when Phase 2 is established.

Verify:

```fortios
get router info routing-table all
```

Or inspect the remote prefix:

```fortios
get router info routing-table details 10.20.20.0
```

Expected behavior while the tunnel is up:

```text
10.20.20.0/24 -> S2S-MT-DIALUP
```

The dynamic route may disappear when the dial-up tunnel is down.

---

# 13. Optional FortiGate Blackhole Route

A high-distance blackhole route prevents traffic for the remote private subnet from accidentally following the normal default route when the VPN is down.

```fortios
config router static
    edit 0
        set dst 10.20.20.0 255.255.255.0
        set blackhole enable
        set distance 254
    next
end
```

The dynamically installed VPN route should have a lower administrative distance and therefore take precedence while the VPN is active.

---

# 14. MikroTik IPsec Phase 1 Profile

```routeros
/ip ipsec profile
add name=fg-s2s \
    hash-algorithm=sha256 \
    enc-algorithm=aes-256 \
    dh-group=modp2048 \
    lifetime=1d \
    dpd-interval=10s
```

RouterOS IKEv2 performs NAT detection automatically.

The MikroTik is the initiator in this design.

---

# 15. MikroTik Phase 2 Proposal

```routeros
/ip ipsec proposal
add name=fg-s2s \
    auth-algorithms=sha256 \
    enc-algorithms=aes-256-cbc \
    pfs-group=modp2048 \
    lifetime=1h
```

---

# 16. MikroTik IKEv2 Peer

```routeros
/ip ipsec peer
add name=fg-s2s \
    address=203.0.113.10/32 \
    exchange-mode=ike2 \
    profile=fg-s2s \
    passive=no \
    send-initial-contact=yes
```

Important:

```text
passive=no
```

The MikroTik actively initiates the connection.

The FortiGate does not initiate toward the MikroTik's NATed WAN address.

---

# 17. MikroTik Identity and Peer ID

```routeros
/ip ipsec identity
add peer=fg-s2s \
    auth-method=pre-shared-key \
    secret="<STRONG-RANDOM-IPSEC-PSK>" \
    my-id=key-id:branch-mikrotik
```

The value:

```text
branch-mikrotik
```

must match the FortiGate Phase 1:

```fortios
set peerid "branch-mikrotik"
```

This is what allows the FortiGate to identify the branch even though the MikroTik has no fixed public IP.

---

# 18. MikroTik IPsec Policy

```routeros
/ip ipsec policy
add peer=fg-s2s \
    src-address=10.20.20.0/24 \
    dst-address=10.10.10.0/24 \
    tunnel=yes \
    action=encrypt \
    level=require \
    proposal=fg-s2s
```

This defines the protected networks:

```text
10.20.20.0/24 <-> 10.10.10.0/24
```

A separate static route to the FortiGate LAN is normally not required for this traditional policy-based RouterOS IPsec configuration as long as the router already has a valid default route to the Internet.

---

# 19. MikroTik NAT Bypass — Mandatory

This is one of the most important steps.

Most MikroTik Internet configurations contain a masquerade rule similar to:

```routeros
/ip firewall nat
add chain=srcnat action=masquerade out-interface=<WAN>
```

If branch traffic is masqueraded before matching the IPsec policy, the IPsec selector may no longer match.

Create a NAT exemption:

```routeros
/ip firewall nat
add chain=srcnat \
    src-address=10.20.20.0/24 \
    dst-address=10.10.10.0/24 \
    action=accept \
    comment="NO-NAT: Branch LAN to FortiGate LAN"
```

### Rule order is critical

This rule must be **above the Internet masquerade rule**.

Check:

```routeros
/ip firewall nat print
```

Conceptual order:

```text
1. ACCEPT 10.20.20.0/24 -> 10.10.10.0/24
2. Other special NAT rules
3. MASQUERADE -> Internet
```

---

# 20. MikroTik Firewall — Branch to FortiGate

Allow branch users to traverse the tunnel:

```routeros
/ip firewall filter
add chain=forward \
    src-address=10.20.20.0/24 \
    dst-address=10.10.10.0/24 \
    action=accept \
    comment="Allow Branch LAN to FortiGate LAN"
```

Place this rule before generic forward-chain drop rules.

---

# 21. MikroTik Firewall — FortiGate to Branch

To allow traffic initiated from the FortiGate-side network:

```routeros
/ip firewall filter
add chain=forward \
    src-address=10.10.10.0/24 \
    dst-address=10.20.20.0/24 \
    action=accept \
    comment="Allow FortiGate LAN to Branch LAN"
```

Place the rule before generic forward-chain drop rules.

---

# 22. MikroTik FastTrack Consideration

FastTrack can bypass processing required for IPsec traffic.

If FastTrack is enabled, make sure IPsec traffic is accepted before the FastTrack rule.

Useful inspection command:

```routeros
/ip firewall filter print
```

Recommended conceptual order:

```text
1. Accept established/related IPsec traffic
2. Accept Branch -> FortiGate protected traffic
3. Accept FortiGate -> Branch protected traffic
4. FastTrack ordinary Internet traffic
5. Other firewall rules
```

MikroTik explicitly documents that FastTrack can interfere with policy-based IPsec if protected connections are FastTracked.

---

# 23. Upstream NAT Router Requirements

The NAT router in front of the MikroTik does **not normally require port forwarding**.

It should permit:

```text
Outbound UDP/500
Outbound UDP/4500
Established/related return traffic
```

Normal NAT behavior is:

```text
MikroTik private WAN
192.0.2.10:500
       |
       | NAT
       v
Translated Internet address
198.51.100.x:<translated-port>
       |
       v
FortiGate
203.0.113.10:500
```

After NAT detection:

```text
MikroTik
   |
UDP/4500
   |
Upstream NAT
   |
UDP/4500
   |
FortiGate
```

The upstream NAT public address may change without requiring the FortiGate IPsec definition to be rewritten because the FortiGate is configured as a dynamic responder.

---

# 24. Initial Tunnel Verification — MikroTik

Check active peers:

```routeros
/ip ipsec active-peers print detail
```

Expected indicators include:

```text
state=established
side=initiator
natt-peer=yes
```

Check installed Security Associations:

```routeros
/ip ipsec installed-sa print detail
```

Check policies:

```routeros
/ip ipsec policy print detail
```

The policy should become active when the Phase 2 SA exists.

---

# 25. Initial Tunnel Verification — FortiGate

Check tunnel summary:

```fortios
get vpn ipsec tunnel summary
```

Inspect IKE gateways:

```fortios
diagnose vpn ike gateway list
```

Inspect IPsec tunnel details:

```fortios
diagnose vpn tunnel list
```

Confirm that the remote peer is recognized using the configured peer ID.

---

# 26. Basic Connectivity Test — MikroTik to FortiGate LAN

From the MikroTik, explicitly use the branch LAN IP as source:

```routeros
/ping 10.10.10.1 src-address=10.20.20.1
```

Test a server:

```routeros
/ping 10.10.10.20 src-address=10.20.20.1
```

From a branch Windows client:

```powershell
ping 10.10.10.20
```

Test an application port:

```powershell
Test-NetConnection 10.10.10.20 -Port 443
```

---

# 27. Reverse Connectivity Test — FortiGate LAN to MikroTik LAN

After the VPN is established, test from a host behind the FortiGate.

Example:

```powershell
ping 10.20.20.10
```

Test a TCP port:

```powershell
Test-NetConnection 10.20.20.10 -Port 3389
```

If this direction fails while MikroTik-to-FortiGate works, check:

1. FortiGate `LAN -> VPN` firewall policy.
2. MikroTik `FortiGate LAN -> Branch LAN` forward rule.
3. Host firewall on the branch device.
4. FortiGate dynamic route to `10.20.20.0/24`.
5. IPsec Phase 2 selectors.
6. Whether the tunnel is currently established.

---

# 28. Optional DNS Architecture

A common design is:

```text
Branch Clients
     |
DNS = 10.20.20.1
     |
     v
MikroTik DNS Cache / Forwarder
     |
     | IPsec
     v
Central DNS
10.10.10.53
```

Branch clients use:

```text
10.20.20.1
```

as their DNS server.

The MikroTik forwards DNS requests to:

```text
10.10.10.53
```

---

# 29. MikroTik DNS Forwarder

Configure the central DNS server:

```routeros
/ip dns
set servers=10.10.10.53 allow-remote-requests=yes
```

Verify:

```routeros
/ip dns print
```

Expected:

```text
servers: 10.10.10.53
allow-remote-requests: yes
```

---

# 30. Restrict DNS Access to the Branch LAN

Do not expose the MikroTik DNS resolver to arbitrary WAN clients.

Allow DNS from the branch LAN:

```routeros
/ip firewall filter
add chain=input \
    src-address=10.20.20.0/24 \
    protocol=udp \
    dst-port=53 \
    action=accept \
    comment="Allow LAN DNS UDP to MikroTik"

add chain=input \
    src-address=10.20.20.0/24 \
    protocol=tcp \
    dst-port=53 \
    action=accept \
    comment="Allow LAN DNS TCP to MikroTik"
```

These rules must be placed before generic input-chain drop rules.

---

# 31. Important DNS Source-Address Check

The IPsec policy protects traffic whose source is:

```text
10.20.20.0/24
```

When testing DNS generated by the MikroTik itself, confirm that the packet toward the central DNS server uses a source address inside that protected subnet.

Test explicit source reachability first:

```routeros
/ping 10.10.10.53 src-address=10.20.20.1
```

If this succeeds but MikroTik DNS queries do not, inspect traffic and source-address selection.

For installations where RouterOS selects the private WAN IP for locally generated traffic, a more-specific route with a preferred source can be used after validating the WAN gateway:

```routeros
/ip route
add dst-address=10.10.10.53/32 \
    gateway=192.0.2.1 \
    pref-src=10.20.20.1 \
    comment="Preferred source for central DNS through IPsec"
```

This should only be added after confirming the actual RouterOS routing behavior in the target environment.

Verify again:

```routeros
/ping 10.10.10.53 src-address=10.20.20.1
```

Then test DNS:

```routeros
:put [:resolve example.internal]
```

---

# 32. DHCP Configuration for Branch Clients

If the MikroTik provides DHCP, configure the MikroTik itself as the DNS server delivered to clients.

Example:

```routeros
/ip dhcp-server network
set [find address="10.20.20.0/24"] \
    gateway=10.20.20.1 \
    dns-server=10.20.20.1
```

Client configuration should become:

```text
IP Address:  10.20.20.x
Gateway:     10.20.20.1
DNS Server:  10.20.20.1
```

The clients do not need direct awareness of the central DNS server.

---

# 33. DNS Test from a Windows Branch Client

```cmd
ipconfig /all
```

Confirm:

```text
DNS Servers . . . . . . . . . . : 10.20.20.1
```

Test:

```cmd
nslookup example.internal 10.20.20.1
```

The logical path should be:

```text
Windows Client
10.20.20.x
     |
     v
MikroTik DNS
10.20.20.1
     |
     | IPsec
     v
Central DNS
10.10.10.53
```

---

# 34. Troubleshooting Matrix

| Symptom                                                 | Likely Cause                                                   |
| ------------------------------------------------------- | -------------------------------------------------------------- |
| No IKE session                                          | UDP/500 blocked, wrong FortiGate public IP, wrong PSK          |
| IKE establishes but no Phase 2                          | Proposal mismatch or traffic selector mismatch                 |
| `natt-peer=no` unexpectedly                             | NAT not detected or test topology differs                      |
| Branch cannot reach FortiGate LAN                       | NAT bypass missing, firewall policy missing, selector mismatch |
| FortiGate LAN cannot reach branch                       | Reverse FortiGate policy or MikroTik forward rule missing      |
| Internet stops after adding IPsec                       | NAT rule order incorrect                                       |
| VPN works until FastTrack is enabled                    | FastTrack bypassing IPsec processing                           |
| FortiGate has no route to branch                        | Dynamic route not installed / Phase 2 not up                   |
| DNS clients can query MikroTik but names do not resolve | MikroTik cannot reach central DNS through protected selector   |
| Tunnel drops after idle periods                         | Upstream NAT timeout, DPD/keepalive behavior, ISP filtering    |
| VPN fails only behind some ISPs                         | UDP/4500 filtering or problematic carrier NAT                  |

---

# 35. MikroTik IPsec Logging

Temporarily enable IPsec logging:

```routeros
/system logging
add topics=ipsec
```

For deeper troubleshooting:

```routeros
/system logging
add topics=ipsec,debug
```

View logs:

```routeros
/log print where topics~"ipsec"
```

Remove excessive debug logging after troubleshooting.

---

# 36. FortiGate IKE Debugging

Use debugging only during troubleshooting.

```fortios
diagnose debug reset
diagnose vpn ike log-filter clear
diagnose debug application ike -1
diagnose debug enable
```

Reproduce the issue.

Then stop debugging:

```fortios
diagnose debug disable
diagnose debug reset
```

---

# 37. Packet Capture — FortiGate

Capture IKE and NAT-T:

```fortios
diagnose sniffer packet any 'udp port 500 or udp port 4500' 4 0 l
```

Capture traffic between protected networks:

```fortios
diagnose sniffer packet any 'net 10.10.10.0/24 and net 10.20.20.0/24' 4 0 l
```

Stop with:

```text
Ctrl+C
```

---

# 38. Packet Inspection — MikroTik

Check IPsec state:

```routeros
/ip ipsec active-peers print detail
/ip ipsec installed-sa print detail
/ip ipsec policy print detail
```

Check counters while generating traffic.

If the policy's packet counters increase but the remote application does not respond, investigate the remote firewall or return path.

---

# 39. Security Hardening

After basic connectivity works:

1. Replace broad `ALL` FortiGate services with specific required services.
2. Restrict branch-to-datacenter access by source host/subnet where possible.
3. Restrict datacenter-to-branch access similarly.
4. Use a long randomly generated PSK.
5. Do not reuse the PSK for unrelated tunnels.
6. Store the PSK in a secure password/secrets manager.
7. Enable logging on both FortiGate VPN firewall policies.
8. Monitor IKE and Phase 2 status.
9. Keep FortiOS and RouterOS on supported patched releases.
10. Avoid overlapping address space between sites.
11. Do not expose MikroTik DNS recursion to the WAN.
12. Keep the upstream NAT/router firmware patched.

---

# 40. Production Validation Checklist

## Phase 1

* [ ] MikroTik can reach FortiGate public IP.
* [ ] IKEv2 Phase 1 is established.
* [ ] MikroTik reports itself as initiator.
* [ ] NAT-T is active when NAT is present.
* [ ] FortiGate matches Peer ID `branch-mikrotik`.

## Phase 2

* [ ] Protected subnet selectors match.
* [ ] Phase 2 SA is installed.
* [ ] Encryption/decryption packet counters increase.

## NAT

* [ ] VPN NAT-bypass rule exists on MikroTik.
* [ ] VPN NAT-bypass rule is above masquerade.
* [ ] NAT is disabled in FortiGate VPN policies.

## Firewall

* [ ] MikroTik -> FortiGate policy exists.
* [ ] FortiGate -> MikroTik policy exists if reverse initiation is required.
* [ ] MikroTik forward rules permit both required directions.
* [ ] Endpoint operating-system firewalls allow required services.

## Routing

* [ ] FortiGate installs route to the MikroTik LAN when tunnel is up.
* [ ] MikroTik has working Internet/default route toward FortiGate public IP.
* [ ] No overlapping networks exist.

## DNS

* [ ] Branch DHCP advertises MikroTik LAN IP as DNS.
* [ ] MikroTik DNS server is configured with the central DNS server.
* [ ] MikroTik can reach the central DNS server through the VPN.
* [ ] DNS TCP/53 and UDP/53 are permitted as required.
* [ ] DNS recursion is not exposed to the Internet.

---

# 41. Expected Final Traffic Flow

## Branch client to FortiGate-side application

```text
10.20.20.10
    |
    v
MikroTik
    |
    | IPsec policy match
    | No NAT
    | IKEv2 / NAT-T
    v
Upstream NAT
    |
    v
Internet
    |
    v
FortiGate 203.0.113.10
    |
    | Decrypt
    v
10.10.10.20
```

---

## FortiGate-side server to branch client

The tunnel must already be established by the MikroTik.

```text
10.10.10.20
    |
    v
FortiGate
    |
    | Dynamic VPN route
    | Encrypt
    v
Existing IPsec NAT-T session
    |
    v
Upstream NAT
    |
    v
MikroTik
    |
    | Decrypt
    v
10.20.20.10
```

---

# 42. Key Operational Behavior

This architecture should be understood as:

```text
Tunnel establishment:
MikroTik -> FortiGate

Traffic after establishment:
MikroTik LAN <-> FortiGate LAN
```

The fact that the MikroTik is behind NAT does **not** make the protected LAN traffic inherently one-way.

It only changes **which endpoint can reliably initiate the IKE session**.

Once the IKE and IPsec Security Associations exist, application sessions can be initiated from either protected LAN, assuming routing and firewall policies permit them.

---

# 43. Configuration Summary

## FortiGate

```text
VPN type:          Dynamic / Dial-up
IKE:               IKEv2
Remote peer IP:    Not fixed
Peer identity:     branch-mikrotik
NAT-T:             Enabled
Local subnet:      10.10.10.0/24
Remote subnet:     10.20.20.0/24
VPN NAT:           Disabled
Routing:           Dynamic route while tunnel is established
```

## MikroTik

```text
Role:              Initiator
FortiGate address: 203.0.113.10
IKE:               IKEv2
My ID:             branch-mikrotik
Local subnet:      10.20.20.0/24
Remote subnet:     10.10.10.0/24
NAT bypass:        Required
FastTrack bypass:  Required where applicable
Public IP:         Not required
Inbound DNAT:      Normally not required
```

---

# 44. Rollback

## MikroTik

Disable the IPsec peer first:

```routeros
/ip ipsec peer disable [find name="fg-s2s"]
```

Then remove or disable:

```text
IPsec policy
IPsec identity
IPsec peer
IPsec proposal
IPsec profile
VPN NAT-bypass rule
VPN-specific firewall rules
DNS-specific configuration if added only for this VPN
```

## FortiGate

Disable or remove:

```text
VPN firewall policies
Phase 2
Phase 1
VPN address objects if unused elsewhere
Optional blackhole route
```

Do not delete shared objects or policies that are used by other production services.

---

# 45. References

Fortinet Documentation:

* FortiGate IPsec VPN administration documentation
  https://docs.fortinet.com/

* Fortinet dynamic/dial-up IPsec documentation
  https://docs.fortinet.com/document/fortigate/

MikroTik Documentation:

* RouterOS IPsec
  https://help.mikrotik.com/docs/spaces/ROS/pages/11993097/IPsec

* MikroTik IPsec with Fortinet / Encryption Domain
  https://help.mikrotik.com/docs/spaces/RKB/pages/245792775/IPSEC%20with%20Fortinet%20and%20Encryption%20Domain

---

## Final Note

The most important difference between this design and a conventional fixed-IP site-to-site VPN is:

```text
The MikroTik does not have a public IP address.
```

Therefore the production design is:

```text
FortiGate = Dynamic IPsec responder
MikroTik  = IKEv2 initiator behind NAT
Identity  = Fixed IKE Peer ID
Transport = NAT-T / UDP 4500
Traffic   = Bidirectional after tunnel establishment
```
