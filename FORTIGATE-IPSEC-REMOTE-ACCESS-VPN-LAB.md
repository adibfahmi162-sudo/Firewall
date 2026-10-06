# FortiGate IPsec Remote Access VPN Lab

A practical lab guide for building, verifying, troubleshooting, and hardening a **dial-up IPsec remote access VPN** on a FortiGate, using **FortiClient** as the remote client.

| | |
|---|---|
| **Tested on** | FortiGate 80D, FortiOS 6.2.13 |
| **Client** | FortiClient 7.4.3 (Windows) |
| **VPN type** | Dial-up IPsec, IKEv1 Aggressive Mode, Pre-Shared Key + XAuth |
| **Environment** | Isolated lab. All addresses and names are fictional |

> **Golden rule:** Find the exact point where the packet stops, then fix only that layer.

---

## Overview

This lab places a FortiGate **behind an upstream NAT router** (the common home-lab and small-office layout) and lets a remote PC connect with FortiClient. The upstream router must forward the IPsec ports to the FortiGate.

The guide follows the order in which problems were met and solved in the lab:

1. Build the WAN and LAN links.
2. Forward UDP 500 and 4500 and **prove** the packets arrive.
3. Bring up Phase 1 and fix a DH group mismatch.
4. Bring up XAuth and Mode Config.
5. Fix a Phase 2 failure caused by a **PFS mismatch**.
6. Add a split-tunnel route **and** a firewall policy to reach a second network.
7. Isolate host-level and local-in problems with packet captures.

Key lessons:

- A **connected** VPN does not mean the network is reachable. You still need **routes + firewall policy + return path**.
- Phase 1, XAuth, Phase 2, routing, and firewall policy are **separate layers**. Troubleshoot one at a time.
- A split-tunnel route on the client does **not** open the firewall. A FortiGate policy is always required.

## Lab Objectives

- Build a working IKEv1 dial-up VPN with PSK + XAuth.
- Understand why UDP 500, UDP 4500, and NAT-T are required behind a NAT device.
- Use packet captures and IKE debug to locate the failing stage.
- Apply a hardening checklist before any real use.

## Lab Topology

```text
Internet
   |
Upstream router (NAT mode, NOT bridge)
LAN 10.10.10.1/24
   |
FortiGate  port3 (WAN) = 10.10.10.2/24
   |
FortiGate  port4 (LAN) = 10.10.20.1/24
   |
Office LAN 10.10.20.0/24 (lab VMs)


Remote user (FortiClient)
   |
Internet --> upstream public IP --> UDP 500/4500 forwarded --> FortiGate port3
   |
IPsec tunnel
   |
VPN client IP 10.10.30.10 (from pool)
```

## Address Plan

| Function | Value |
|---|---|
| WAN / upstream network | `10.10.10.0/24` |
| Upstream router | `10.10.10.1` |
| FortiGate WAN (`port3`) | `10.10.10.2/24`, gateway `10.10.10.1` |
| FortiGate LAN (`port4`) | `10.10.20.1/24` |
| Office LAN | `10.10.20.0/24` |
| VPN client pool | `10.10.30.10 - 10.10.30.100` |
| Example host reachable through the tunnel | `10.10.10.21` |
| Example host used in the troubleshooting scenario | `10.10.10.209` |
| VPN tunnel name | `ipsec-lab` |
| VPN user group | `VPN-Lab-Users` |
| VPN test account | `vpn-test-user` |

Placeholders used in this guide:

| Placeholder | Meaning |
|---|---|
| `<STRONG_LAB_PSK>` | Pre-shared key. Supplied by you |
| `<STRONG_LAB_PASSWORD>` | Password for the test account. Supplied by you |
| `<UPSTREAM_PUBLIC_IP>` | Public IPv4 of the upstream router |
| `<REMOTE_PUBLIC_IP>` | Public IPv4 of the remote test PC |
| `<DDNS_HOSTNAME>` | Optional DDNS name for the upstream router |

**Why separate subnets?** Using `.10 / .20 / .30` keeps the WAN side, office LAN, and VPN clients clearly apart. In a packet capture you can tell from the IP alone where a packet came from.

## Prerequisites

- A FortiGate with console or GUI access and two usable ports (WAN and LAN).
- An upstream router with port-forwarding support, in **router/NAT mode**.
- A public IPv4 on the upstream router that is **not** in `100.64.0.0/10` (that range means CGNAT, and inbound forwarding will not work).
- FortiClient installed on a test PC.
- The test PC on a **different network** (for example a mobile hotspot). Testing from inside the lab proves nothing about the Internet path.

## Security Notice

> **Never commit secrets to Git.**
>
> - Every PSK and password in this guide is a placeholder. You must generate and supply your own.
> - Never commit PSKs, passwords, certificates, private keys, tokens, config backups, or logs from a real environment.
> - A FortiGate config backup or `show` output may contain the PSK in encrypted form. Treat that string as sensitive too.
> - Never put a PSK in a screenshot, chat message, or document. If one leaks, treat it as compromised and replace it.
> - Use a **different** PSK and password for any real deployment.
> - Do not expose the FortiGate GUI or SSH on the WAN interface.

Suggested `.gitignore` entries for a repo that holds lab files:

```text
*.conf
*.cfg
*.pcap
*.pcapng
*.log
*.key
*.pem
```

Use this guide only on equipment you own or are authorized to test.

---

## 1. Prepare the Lab

**Do one step, verify it, then move on. Never change two things at once.**

Each step below uses the same layout: what and why, the config, how to verify, what to expect, and what to do if it fails.

### 1.1 Reserve the FortiGate WAN address on the upstream router

**Why:** If the upstream DHCP pool can hand out `10.10.10.2`, a duplicate IP will randomly break the VPN.

Set the upstream DHCP range so it skips the FortiGate address:

```text
DHCP start: 10.10.10.6
DHCP end:   10.10.10.254
```

This leaves `.1` for the router, `.2` for the FortiGate, and `.3-.5` spare.

**Verify:** The new range is applied and no device holds `.2` through `.5`.

**If it fails:** If something already holds `.2`, disconnect or re-address that device first.

> **Safety:** Do not factory reset anything or change unrelated port mappings on the upstream router unless a step says so.

---

## 2. Configure FortiGate Interfaces

**Why:** The FortiGate needs a path to the Internet (through the upstream router) and an internal network to protect.

WAN interface and default route:

```bash
config system interface
    edit "port3"
        set mode static
        set ip 10.10.10.2 255.255.255.0
        set allowaccess ping
        set role wan
    next
end

config router static
    edit 1
        set gateway 10.10.10.1
        set device "port3"
    next
end
```

After the WAN test passes, configure the LAN interface:

```bash
config system interface
    edit "port4"
        set mode static
        set ip 10.10.20.1 255.255.255.0
        set role lan
    next
end
```

- `allowaccess ping` on the WAN lets you test the link but does **not** expose HTTPS or SSH. Keep management off the WAN.
- The static route sends all unknown traffic to the upstream router.

**Verify:**

```bash
execute ping 10.10.10.1
execute ping 8.8.8.8
get router info routing-table all
```

**Expect:** 0% packet loss to both, and a default route like `S* 0.0.0.0/0 via 10.10.10.1 port3`.

**If it fails:** Check the cable, the `port3` IP and mask, and that the gateway is exactly `10.10.10.1`. Do not continue until this passes.

---

## 3. Configure Upstream NAT / Port Forwarding

**Why:** The FortiGate is behind NAT. IKE uses **UDP 500**. Once NAT is detected, IPsec moves to **UDP 4500** (NAT-T). Both must reach the FortiGate.

Create two port-forward rules on the upstream router:

| Rule name | Protocol | External port | Internal host | Internal port |
|---|---|---|---|---|
| IPsec-UDP-500 | UDP | 500 | `10.10.10.2` | 500 |
| IPsec-UDP-4500 | UDP | 4500 | `10.10.10.2` | 4500 |

Do **not** add ESP (protocol 50) forwarding. With NAT-T, ESP is wrapped inside UDP 4500.

**Verify (prove it, do not assume).** On the FortiGate, while the remote PC tries to connect:

```bash
diagnose sniffer packet port3 'udp port 500 or udp port 4500' 4 0 l
```

To focus on one client:

```bash
diagnose sniffer packet port3 'host <REMOTE_PUBLIC_IP> and (udp port 500 or udp port 4500)' 6 0 l
```

Press `Ctrl+C` to stop.

**Expect:**

```text
<client>      -> 10.10.10.2:500     (request in)
10.10.10.2    -> <client>:500       (reply out)
<client>      -> 10.10.10.2:4500
10.10.10.2    -> <client>:4500
```

**If it fails:**

| Symptom | Likely cause | Fix |
|---|---|---|
| No packets at all | Wrong port forward, wrong public IP, ISP CGNAT, or the client network blocks UDP | Re-check the forwards and the public IP. Confirm the WAN IP is not in `100.64.0.0/10` |
| Packets in, no replies out | FortiGate not listening on that interface | Confirm Phase 1 `interface` is `port3` |

> This is the cheapest way to split the problem in half: if UDP does not arrive, nothing else matters.

> The upstream public IP is often assigned dynamically and can change. Check the current value before configuring FortiClient, or use a DDNS name (`<DDNS_HOSTNAME>`).

---

## 4. Create VPN Users and Groups

**Why:** The PSK only proves the device knows a shared secret. **XAuth** identifies the person.

```bash
config user local
    edit "vpn-test-user"
        set type password
        set passwd <STRONG_LAB_PASSWORD>
    next
end

config user group
    edit "VPN-Lab-Users"
        set member "vpn-test-user"
    next
end
```

Supply the password yourself. Do not reuse a password from any other system.

**Verify:**

```bash
show user group VPN-Lab-Users
```

**Expect:** `vpn-test-user` listed as a member.

**Good practice:** Use one account per person. Create extra groups (for example `VPN-Admins`, `VPN-Contractors`) only when people need different access.

---

## 5. Create Firewall Address Objects

**Why:** Firewall policies and the split tunnel both use these objects.

```bash
config firewall address
    edit "ipsec-lab_range"
        set type iprange
        set start-ip 10.10.30.10
        set end-ip 10.10.30.100
    next
    edit "lab-office-lan"
        set subnet 10.10.20.0 255.255.255.0
    next
    edit "lab-upstream-lan"
        set subnet 10.10.10.0 255.255.255.0
    next
end

config firewall addrgrp
    edit "ipsec-lab_split"
        set member "lab-office-lan" "lab-upstream-lan"
    next
end
```

| Object | Meaning |
|---|---|
| `ipsec-lab_range` | The VPN clients themselves |
| `lab-office-lan` | Office LAN `10.10.20.0/24` |
| `lab-upstream-lan` | Upstream network `10.10.10.0/24` |
| `ipsec-lab_split` | Networks FortiClient routes into the tunnel (split tunnel) |

> The FortiGate VPN wizard creates a client range, an office LAN object, and a split group automatically, named after the tunnel. The upstream network object was added manually after the lab proved it was needed.

**Verify:**

```bash
show firewall addrgrp ipsec-lab_split
```

**Expect:** Both member objects listed.

---

## 6. Configure IPsec Phase 1

**Why:** Phase 1 builds the secure channel between FortiClient and the FortiGate and authenticates the peer. This is where IKEv1, Aggressive Mode, PSK, XAuth, the DH group, and Mode Config are set.

Working configuration from the lab:

```bash
config vpn ipsec phase1-interface
    edit "ipsec-lab"
        set type dynamic
        set interface "port3"
        set mode aggressive
        set peertype any
        set net-device disable
        set mode-cfg enable
        set proposal aes128-sha256 aes256-sha256 aes128-sha1 aes256-sha1
        set dhgrp 14
        set wizard-type dialup-forticlient
        set xauthtype auto
        set authusrgrp "VPN-Lab-Users"
        set ipv4-start-ip 10.10.30.10
        set ipv4-end-ip 10.10.30.100
        set dns-mode auto
        set ipv4-split-include "ipsec-lab_split"
        set save-password enable
        set nattraversal enable
        set psk <STRONG_LAB_PSK>
    next
end
```

| Setting | Why |
|---|---|
| `type dynamic` | Clients have changing IPs (mobile, home), so the FortiGate accepts any peer |
| `interface "port3"` | Listen on the WAN port |
| `mode aggressive` | Required for dial-up PSK with FortiClient on IKEv1 |
| `mode-cfg enable` | Lets the FortiGate hand the client an IP, DNS, and routes |
| `proposal ...` | Encryption and hash combinations the FortiGate accepts |
| `dhgrp 14` | **Only one DH group.** Offering several in Aggressive Mode caused failures |
| `xauthtype auto` + `authusrgrp` | Ask for username and password, and check them against the group |
| `ipv4-start-ip` / `ipv4-end-ip` | The VPN address pool |
| `ipv4-split-include` | Only these networks go through the tunnel (split tunnel) |
| `nattraversal enable` | Needed because the upstream NAT is in the path |
| `psk` | Shared secret. Supply your own and **never commit it** |

**Verify:**

```bash
show vpn ipsec phase1-interface ipsec-lab
diagnose vpn ike gateway list
```

**Expect:** The config matches the above. The gateway list is empty until a client connects.

**If it fails:**

| Symptom | Cause | Fix |
|---|---|---|
| IKE debug shows the proposal chosen, but the client never completes | Multiple DH groups on the FortiGate (for example `14 5`) and a different set on the client | Use `set dhgrp 14` and set FortiClient to DH 14 only |
| `no SA proposal chosen` | No common encryption or hash | Compare the FortiClient Phase 1 settings with the `proposal` line |
| PSK mismatch | Different secret on each side | Re-enter the PSK on both sides (watch for trailing spaces) |

---

## 7. Configure IPsec Phase 2

**Why:** Phase 2 negotiates the keys that protect the actual data. A mismatch here produced the error that cost the most time in the lab.

```bash
config vpn ipsec phase2-interface
    edit "ipsec-lab"
        set phase1name "ipsec-lab"
        set proposal aes128-sha1 aes256-sha1 aes128-sha256 aes256-sha256
        set pfs enable
        set dhgrp 14
        set keylifeseconds 43200
    next
end
```

- `pfs enable` + `dhgrp 14` forces a fresh Diffie-Hellman exchange for the Phase 2 keys (**PFS**). **FortiClient must match.**
- The VPN wizard creates this object for you. Check the real values on your unit:

```bash
show vpn ipsec phase2-interface
```

**Verify:** `phase1name` equals `ipsec-lab`, and you have noted the PFS and DH values. You will copy them into FortiClient in Section 9.

**If it fails.** VPN Events showed:

```text
IPsec phase 2 error
reason="peer SA proposal not match local policy"
xauthuser="vpn-test-user"  assignip=10.10.30.10  mode="quick"
```

How to read it:

- `xauthuser` and `assignip` present means **Phase 1, XAuth, and Mode Config already succeeded**. Do not touch Phase 1.
- `mode="quick"` means the failure is in **Phase 2**.
- Cause in the lab: FortiClient had **PFS disabled** while the FortiGate required it.
- Fix: FortiClient Phase 2 -> **PFS = Enabled, DH Group = 14**. The VPN connected immediately.

> `peer SA proposal not match local policy` does not only mean encryption or hash. Also check **PFS, the DH group, and the Phase 2 selectors**.

---

## 8. Configure Firewall Policies

**Why:** A tunnel only carries traffic **to** the FortiGate. The policy decides what happens next. No policy means the packet is dropped.

**Policy A: VPN to office LAN** (created by the wizard):

| Field | Value |
|---|---|
| Name | `vpn_ipsec-lab_remote` |
| Incoming | `ipsec-lab` |
| Outgoing | `port4` |
| Source | `ipsec-lab_range` |
| Destination | `lab-office-lan` |
| Service | ALL (lab only, see [Security Hardening](#14-security-hardening)) |
| Action | ACCEPT |
| NAT | Enabled |

**Policy B: VPN to upstream network** (added manually):

```bash
config firewall policy
    edit 0
        set name "vpn_ipsec-lab_remote_upstream"
        set srcintf "ipsec-lab"
        set dstintf "port3"
        set srcaddr "ipsec-lab_range"
        set dstaddr "lab-upstream-lan"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
        set logtraffic all
    next
end
```

- The destination `10.10.10.0/24` is **out of `port3`**, so the outgoing interface must be `port3`, not `port4`.
- `nat enable` rewrites the source to the FortiGate `port3` address. Hosts on that network reply to `10.10.10.2`, so **no route back to `10.10.30.0/24` is needed on the upstream router**. This is the simplest solution here.
- `logtraffic all` makes troubleshooting much easier.

**Verify:** Use the packet capture in [Section 12](#12-packet-capture-and-troubleshooting).

**If it fails.** The capture showed:

```text
ipsec-lab in   10.10.30.10 -> 10.10.10.209: icmp: echo request
```

with **no** matching `port3 out`. This means *the packet reached the FortiGate but the FortiGate refused to forward it.* No policy existed for `ipsec-lab -> port3`. Creating Policy B fixed it.

---

## 9. Configure FortiClient

**Why:** Every Phase 1 and Phase 2 setting on the client must be compatible with the FortiGate.

FortiClient -> Remote Access -> Configure VPN -> IPsec VPN:

| Setting | Value |
|---|---|
| Connection name | `ipsec-lab` |
| Remote Gateway | `<UPSTREAM_PUBLIC_IP>` or `<DDNS_HOSTNAME>` |
| Authentication Method | Pre-shared key (enter `<STRONG_LAB_PSK>`) |
| Authentication (XAuth) | Prompt on login |
| Username | `vpn-test-user` |
| IKE | **IKEv1** |
| Mode | **Aggressive** |
| Phase 1 Encryption / Hash | AES128/SHA1 and AES256/SHA256 |
| Phase 1 DH Group | **14 only** |
| NAT Traversal | **Enabled** |
| Phase 2 Encryption / Hash | AES128/SHA1 and AES256/SHA256 |
| Phase 2 Key Life | 43200 seconds |
| Replay Detection | Enabled |
| **PFS** | **Enabled, DH Group 14** |

**Verify:** Click Connect and enter the username and password.

**Expect:**

```text
VPN Connected
VPN Name:   ipsec-lab
IP Address: 10.10.30.10
Username:   vpn-test-user
```

---

## 10. Verify the VPN

**Why:** Prove each layer worked using the FortiGate's own evidence.

```bash
diagnose vpn ike gateway list
diagnose vpn tunnel list
diagnose vpn ike status
```

In the GUI: Log & Report -> VPN Events.

**Expect in VPN Events:**

```text
xauthuser="vpn-test-user"
xauthgroup="VPN-Lab-Users"
assignip=10.10.30.10
```

| Evidence | What it proves |
|---|---|
| UDP 500/4500 in the sniffer | Internet path and upstream forwarding are OK |
| IKE debug "SA proposal chosen" | Phase 1 proposal matched |
| `xauthuser` in the log | PSK and XAuth OK |
| `assignip` | Mode Config OK (client received an IP) |
| "VPN Connected" in FortiClient | Phase 2 OK |

**IKE debug** (use only when needed, then turn it off):

```bash
diagnose debug reset
diagnose vpn ike log-filter clear
diagnose vpn ike log-filter src-addr4 <REMOTE_PUBLIC_IP>
diagnose debug console timestamp enable
diagnose debug application ike -1
diagnose debug enable
```

Stop it with:

```bash
diagnose debug disable
diagnose debug reset
```

> Debug output and logs can contain real public IPs and usernames. Redact them before sharing or committing.

---

## 11. Test Network Reachability

Walk from the client outward, one hop at a time. The first failing test tells you which layer to inspect.

| Test | Command on the remote test PC | Passing means |
|---|---|---|
| 1. Tunnel up | FortiClient shows connected | Phase 1 and 2 OK |
| 2. Client IP | `ipconfig` shows `10.10.30.x` | Mode Config OK |
| 3. Routes | `route print` | Split-tunnel routes pushed |
| 4. Office LAN host | `ping <host on 10.10.20.0/24>` | Policy A OK |
| 5. Upstream-network host | `ping 10.10.10.21` | Policy B + NAT OK |
| 6. A specific server | `ping 10.10.10.209` | Host-specific path OK |

Expected `route print` output (test 3):

```text
10.10.10.0    255.255.255.0    10.10.30.11    10.10.30.10
10.10.20.0    255.255.255.0    10.10.30.11    10.10.30.10
```

This proves both networks are routed **into the tunnel**. If a network is missing here, fix the split tunnel in [Section 5](#5-create-firewall-address-objects), not the firewall.

Expected ping output (test 5):

```text
Reply from 10.10.10.21: bytes=32 time=56ms TTL=63
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

---

## 12. Packet Capture and Troubleshooting

### 12.1 The packet path

Every failure is a packet that stopped somewhere along this path. Your job is to find **where**.

```text
Client
  |
Internet
  |
UDP 500 / 4500          <- upstream forwarding
  |
FortiGate
  |
IKE Phase 1             <- proposal, DH group, PSK
  |
XAuth                   <- user, password, group
  |
Phase 2                 <- proposal, PFS, selectors
  |
VPN interface
  |
Firewall policy         <- policy for ipsec-lab -> egress interface
  |
NAT / routing
  |
Destination host        <- host firewall, ICMP, gateway
  |
Return traffic
  |
VPN client
```

### 12.2 Find the first missing packet

Work **top to bottom**. Stop at the first stage that fails and fix only that stage.

| Stage | Evidence it worked | Command | If missing, look at |
|---|---|---|---|
| UDP 500/4500 arrive | Packets in and out on `port3` | `diagnose sniffer packet port3 'udp port 500 or udp port 4500' 4 0 l` | Upstream port forward, public IP, CGNAT, client network |
| Phase 1 | "SA proposal chosen" in IKE debug | `diagnose debug application ike -1` | IKE version, encryption/hash, DH group, PSK, Aggressive Mode |
| XAuth | `xauthuser` in VPN Events | Log & Report -> VPN Events | Username, password, group membership, `xauthtype`, `authusrgrp` |
| Mode Config | `assignip` in VPN Events | Log & Report -> VPN Events | `mode-cfg`, address pool |
| Phase 2 | "VPN Connected" in FortiClient | `diagnose vpn tunnel list` | Proposal, PFS on both sides, DH group, selectors |
| Client routing | Networks in `route print` | `route print` (Windows) | Split-tunnel group |
| Policy | `ipsec-lab in` **and** egress `out` | Sniffer (below) | Firewall policy, routing, NAT |
| Destination host | `port3 in` (reply) | Sniffer (below) | Host firewall, ICMP rules, network profile, gateway |
| Return to client | `ipsec-lab out` | Sniffer (below) | Session and policy state, return route |

### 12.3 The most useful tool: sniffer on the FortiGate

```bash
diagnose sniffer packet any 'host 10.10.10.209' 4 0 l
```

`any` shows all interfaces, `4` shows interface names, `l` adds a local timestamp. `Ctrl+C` stops it.

Read it like this:

```text
ipsec-lab in     10.10.30.10  -> 10.10.10.209   packet entered the tunnel   (VPN OK)
port3 out        10.10.10.2   -> 10.10.10.209   FortiGate forwarded it      (policy OK)
port3 in         10.10.10.209 -> 10.10.10.2     server replied              (return OK)
ipsec-lab out    10.10.10.209 -> 10.10.30.10    reply back to client        (full path OK)
```

Find the **first line that is missing**. That is your problem.

| Missing line | Where to look |
|---|---|
| No `ipsec-lab in` | Client route, split tunnel, or tunnel down |
| `ipsec-lab in` but no `port3 out` | Firewall policy, routing, or NAT on the FortiGate |
| `port3 out` but no `port3 in` | The target host: firewall, ICMP rules, network profile, gateway |
| `port3 in` but no `ipsec-lab out` | Rare. Check session and policy state on the FortiGate |

### 12.4 The flow of one packet

```text
Remote PC (10.10.30.10)
   |  1. Route sends 10.10.10.0/24 into the tunnel          <- split tunnel
   v
IPsec tunnel (UDP 4500 through the upstream NAT)
   v
FortiGate interface ipsec-lab
   |  2. Policy ipsec-lab -> port3 must ACCEPT              <- policy
   |  3. NAT changes the source to 10.10.10.2               <- NAT
   v
FortiGate port3 -> upstream network -> target host 10.10.10.x
   |  4. Host replies to 10.10.10.2                         <- return path
   v
FortiGate reverses NAT and sends the reply back into the tunnel
```

### 12.5 Five questions that solve most VPN problems

```text
1. Who is the VPN client?
2. What IP did it receive?
3. Which networks are sent into the tunnel?
4. Which firewall policy allows that traffic?
5. How does the destination send the reply back?
```

---

## 13. Common Failure Scenarios

| # | Symptom | Meaning | Fix |
|---|---|---|---|
| 1 | FortiClient cannot connect, no logs on the FortiGate | UDP 500/4500 is not reaching the FortiGate | Fix the upstream forwards. Verify with the sniffer |
| 2 | Phase 1 fails with `no SA proposal chosen` | No common encryption or hash | Align the FortiClient Phase 1 settings with the `proposal` line |
| 3 | IKE debug shows a proposal chosen but negotiation stalls | DH group mismatch (for example `14 5` offered) | Use DH 14 only on both sides |
| 4 | Phase 1 never authenticates | PSK mismatch | Re-enter the PSK on both sides. Check for trailing spaces |
| 5 | Login prompt appears but login fails | XAuth failure | Check the username, password, group membership, `xauthtype`, and `authusrgrp` |
| 6 | `peer SA proposal not match local policy`, `xauthuser` and `assignip` present, `mode="quick"` | Phase 2 proposal mismatch | Align the Phase 2 encryption and hash |
| 7 | Same error, proposals already match | **PFS mismatch** (FortiGate requires PFS, client has it off) | Enable PFS with DH 14 in FortiClient |
| 8 | Connected, cannot reach a network | Split-tunnel route missing | Add the network to `ipsec-lab_split`. Confirm with `route print` |
| 9 | Route present, `ipsec-lab in` but no egress `out` | Firewall policy missing | Create the `ipsec-lab -> <egress interface>` policy |
| 10 | Packets leave, replies never return | NAT or return-path problem | Enable NAT on the policy, or add a return route on the upstream router |
| 11 | FortiGate can ping a host but the VPN client cannot | Different traffic flow | Capture the client-to-host path. The FortiGate-to-host ping proves little |
| 12 | `port3 out` but no `port3 in` | Target host firewall blocks the traffic | Check the host firewall, ICMP rules, network profile, and gateway. Test a real service port |
| 13 | Cannot reach the FortiGate's own address through the tunnel | **Local-in** traffic, not transit traffic | See [Known Limitations](#15-known-limitations) |

**Transit vs local-in:** Traffic *through* the FortiGate is controlled by firewall policies. Traffic *to* the FortiGate itself is controlled by interface `allowaccess` and any `local-in-policy`. A transit policy does not open the FortiGate itself.

---

## 14. Security Hardening

The lab used `Service: ALL` and treated the PSK casually. **Treat the lab settings as a proof of concept** and apply this checklist before real use.

```text
[ ] Generate a NEW strong PSK. Treat any PSK that appeared in a screenshot or chat as compromised
[ ] Never write the PSK or its encrypted string into documents, chats, screenshots, or Git
[ ] Replace Service ALL with only the services needed (RDP, SSH, HTTPS, ICMP, etc.)
[ ] Restrict destinations to the servers people need, not whole subnets
[ ] Keep FortiGate GUI/SSH off the WAN (port3 allowaccess is ping only)
[ ] One user per person. No shared accounts
[ ] Strong, unique passwords. Consider MFA (FortiToken, RADIUS, or LDAP) if available
[ ] Keep logging on for VPN events and VPN policies
[ ] Test a failed login and confirm it appears in the logs
[ ] Do not rely on Geo-IP blocking alone. Mobile users change IPs and carriers
[ ] Keep FortiOS and FortiClient on supported, compatible versions
[ ] Consider certificate-based authentication or IKEv2 when the platform allows it
[ ] Redact IPs, usernames, and hostnames from any logs or captures you publish
```

---

## 15. Known Limitations

**IKEv1 Aggressive Mode + PSK.** This is the combination that worked with FortiClient in this lab, but it is weaker than certificate-based or IKEv2 designs. Plan an upgrade path.

**Open items.** These are **not** VPN-establishment problems. Do not change the working Phase 1 or Phase 2 while investigating them.

### 15.1 One upstream host is unreachable through the tunnel

Observed in the lab:

- The FortiGate can ping the host (0% loss).
- The host can reach `10.10.10.2`.
- Another host in the same network (`10.10.10.21`) works from the VPN client, so tunnel, route, policy, and NAT are fine.

The focus is therefore the **host itself**. Check in order:

```text
[ ] Sniffer: does port3 out appear? Does port3 in (reply) appear?
[ ] Host firewall: is ICMP echo allowed from the FortiGate address 10.10.10.2?
[ ] Windows network profile (the Public profile blocks ping by default)
[ ] Host default gateway and subnet mask are correct
[ ] Test a real service port (RDP, HTTPS) instead of only ping
```

### 15.2 The FortiGate itself is unreachable through the tunnel

Traffic to the FortiGate is *local-in* traffic, so transit policies do not control it. Check in order:

```text
1. Is the client routing it into the tunnel? (route print)
2. Does it arrive? Sniffer:
   diagnose sniffer packet any 'host 10.10.10.2 and icmp' 4 0 l
3. Administrative access on the interface that receives the traffic (the ipsec-lab tunnel interface)
4. Any user-defined local-in policy (config firewall local-in-policy)
5. Trusted hosts on the admin account
```

**Suggested approach (to be tested, not verified in the lab):**

```bash
config system interface
    edit "ipsec-lab"
        set allowaccess ping https
    next
end
```

A safer alternative is to manage the FortiGate through its **LAN address `10.10.20.1`**, enabling the required `allowaccess` on `port4` only, so the WAN side stays closed.

Do not add a local-in policy until the sniffer proves where the packet stops.

---

## 16. Alternative Architecture

*Planned, not implemented in this lab.*

The FortiGate becomes the Internet-facing firewall, and the upstream router moves behind it:

```text
ISP modem/ONT -> FortiGate (port3 WAN) -> port4 10.10.20.1
                                            -> Router WAN 10.10.20.2 -> Router LAN 10.10.10.0/24
```

If you adopt it:

- No upstream port forwarding for UDP 500/4500 is needed.
- The FortiGate WAN is no longer `10.10.10.2`.
- Your ISP may require bridge mode on its device and PPPoE credentials. Supply those yourself and never commit them.
- The router would need a route back to `10.10.30.0/24` unless NAT stays enabled on the policy.

Do not mix steps from both designs. This guide describes the **tested** design (FortiGate behind the upstream router).

---

## 17. FortiGate Command Cheat Sheet

| Purpose | Command |
|---|---|
| Interfaces | `get system interface physical` |
| Routing table | `get router info routing-table all` |
| Phase 1 config | `show vpn ipsec phase1-interface ipsec-lab` |
| Phase 2 config | `show vpn ipsec phase2-interface` |
| Active IKE gateways | `diagnose vpn ike gateway list` |
| Active tunnels | `diagnose vpn tunnel list` |
| IKE status | `diagnose vpn ike status` |
| Ping from FortiGate | `execute ping <ip>` |
| Capture IPsec UDP | `diagnose sniffer packet port3 'udp port 500 or udp port 4500' 4 0 l` |
| Capture one host | `diagnose sniffer packet any 'host <ip>' 4 0 l` |
| IKE debug on | `diagnose debug application ike -1` then `diagnose debug enable` |
| IKE debug off | `diagnose debug disable` then `diagnose debug reset` |

On the Windows client: `ipconfig`, `route print`, `ping <ip>`.

---

## 18. Final Verification Checklist

```text
[ ] Upstream DHCP starts at 10.10.10.6. 10.10.10.2 is reserved for the FortiGate
[ ] port3 = 10.10.10.2/24, default route via 10.10.10.1
[ ] port4 = 10.10.20.1/24
[ ] Upstream router forwards UDP 500 and UDP 4500 to 10.10.10.2
[ ] Sniffer proves UDP 500/4500 in and out
[ ] Phase 1: IKEv1, Aggressive, PSK, DH 14 only, NAT-T on, Mode Config on
[ ] Phase 2: PFS enabled, DH 14, matching the client
[ ] User and group created. XAuth works (xauthuser in log)
[ ] Client receives 10.10.30.x (assignip in log)
[ ] Split tunnel contains 10.10.20.0/24 and 10.10.10.0/24
[ ] Policies ipsec-lab -> port4 and ipsec-lab -> port3 exist, logging on
[ ] route print on the client shows both networks via the VPN
[ ] Ping to an upstream-network host (10.10.10.21) works
[ ] No PSK, password, config backup, or real log is committed to Git
[ ] Before real use: services restricted, new PSK set, WAN admin access closed
```

---

## References

- [FortiGate 6.2.13 Phase 1 CLI](https://docs.fortinet.com/document/fortigate/6.2.13/cli-reference/338620/config-vpn-ipsec-phase1-interface)
- [FortiClient 7.4.3 IPsec VPN](https://docs.fortinet.com/document/forticlient/7.4.3/ems-administration-guide/952355/ipsec-vpn)
- [FortiClient 7.4.3 IKE settings](https://docs.fortinet.com/document/forticlient/7.4.3/xml-reference-guide/96295/ike-settings)
- [FortiGate IPsec diagnose commands](https://docs.fortinet.com/document/fortigate/6.2.10/cookbook/44240/ipsec-related-diagnose-command)
- [FortiGate 6.2 local-in policy](https://docs.fortinet.com/document/fortigate/6.2.12/cli-reference/295620/config-firewall-local-in-policy)
- [Troubleshooting `peer SA proposal not match local policy`](https://community.fortinet.com/fortigate-3/troubleshooting-tip-ipsec-tunnel-failure-peer-sa-proposal-not-match-local-policy-110233)
- [Aggressive Mode fails with multiple DH groups](https://community.fortinet.com/forticlient-4/troubleshooting-tip-dial-up-ipsec-vpn-fails-to-connect-in-aggressive-mode-when-multiple-dh-groups-are-selected-92668)

> **Rule to remember:** Do not troubleshoot everything at once. Identify the exact point where the packet stops, then fix only that layer.