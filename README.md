# Palo Alto PA-VM IPSec VPN + BGP over IPSec (Replacing FortiGate SD-WAN)

💡 This repository documents a hands-on lab replacing FortiGate SD-WAN with Palo Alto PA-VM firewalls in an existing MPLS + SASE dual-path topology. By removing FortiGate's automated SD-WAN abstraction and rebuilding the same failover behavior manually on PA-VM, every layer — underlay static routing, IPSec tunnel establishment, BGP overlay route exchange, and export policy loop prevention — must be explicitly configured. The goal is to expose the underlay/overlay separation that FortiGate hides behind its SD-WAN engine.

---

## 🔬 Overview

This lab replaces FortiGate SD-WAN appliances with Palo Alto PA-VM (PAN-OS 10.1.0) firewalls on an existing MPLS L3VPN + Cloudflare SASE topology built in EVE-NG. It demonstrates:

* **IPSec VPN** – Dual tunnels over MPLS (tunnel.1) and SASE/WireGuard (tunnel.2) paths
* **BGP over IPSec** – Overlay route exchange through encrypted tunnels
* **MPLS-Priority Failover** – Local-preference 200 (MPLS) vs 100 (SASE) for automatic path selection
* **Underlay/Overlay Separation** – Static routes for ESP delivery, BGP for encrypted user traffic
* **Export Policy** – Preventing overlay routes from leaking into underlay BGP, avoiding routing loops
* **App-ID Firewall Policy** – Application-aware security on VPN intrazone traffic

**【日本語サマリ】**
FortiGate SD-WANをPalo Alto PA-VMに置き換え、IPSec VPN + BGP over IPSecを手動構築。
MPLS優先/SASEフェイルオーバーをlocal-preferenceで制御。
FGが自動処理していたunderlay/overlay分離を明示的に設定し、設計思想の違いを検証しました。

---

## 🏗️ Architecture & IP Addressing

### Topology

```
                    +-------------------------------------+
                    |        Internet                      |
                    |   (WireGuard UDP 4960)               |
                    +-------------------------------------+
                              |               |
                         Fixed LAN        USB Tethering
                        (Aterm)           (iPhone)
                      106.73.26.0         49.109.x.x
                              |               |
                        192.168.10.100   172.20.10.2
                              |               |
[Branch1]--PA-VM1--POP1===WireGuard===POP2--PA-VM2--[Branch2]
  AS65100    |      |    10.255.0.x    |      |    AS65200
             |    Cloudflare        Cloudflare |
             |                                 |
             +-------- MPLS (CE1-PE-CE2) ------+
             |  IPSec tunnel.1: 10.254.1.x     |
             |  BGP Local-Pref 200 (Primary)   |
             |                                 |
             +---- SASE (POP1-WG-POP2) -------+
                IPSec tunnel.2: 10.255.2.x
                BGP Local-Pref 100 (Backup)
```

### Underlay (MPLS)

| Link | Subnet | Device A | Device B |
|------|--------|----------|----------|
| PE1–PE2 | 10.200.1.0/30 | PE1 e0/1: .1 | PE2 e0/1: .2 |
| PE1–CE1 | 10.101.1.0/30 | PE1 e0/0: .1 | CE1 e0/0: .2 |
| PE2–CE2 | 10.102.1.0/30 | PE2 e0/0: .1 | CE2 e0/0: .2 |

### CE–PA-VM (Trust Zone)

| Link | Subnet | Device A | Device B |
|------|--------|----------|----------|
| CE1–PA-VM1 | 10.1.1.0/30 | CE1 e0/1: .1 | PA-VM1 e1/1: .2 |
| CE2–PA-VM2 | 10.200.2.0/24 | CE2 e0/1: .1 | PA-VM2 e1/1: .2 |

### PA-VM–POP (Untrust Zone)

| Link | Subnet | Device A | Device B |
|------|--------|----------|----------|
| PA-VM1–POP1 | 10.0.0.0/24 | PA-VM1 e1/2: .1 | POP1 ens3: .2 |
| PA-VM2–POP2 | 10.0.1.0/24 | PA-VM2 e1/2: .1 | POP2 ens3: .2 |

### IPSec Tunnel Interfaces (VPN Zone)

| Tunnel | Subnet | PA-VM1 | PA-VM2 | Path |
|--------|--------|--------|--------|------|
| MPLS-VPN (tunnel.1) | 10.254.1.0/30 | .1 | .2 | via MPLS (CE–PE–CE) |
| SASE-VPN (tunnel.2) | 10.255.2.0/30 | .1 | .2 | via WireGuard (POP–Internet–POP) |

### BGP AS Assignments

| Device | AS | Role |
|--------|----|------|
| PE1/PE2 | 65001 | MPLS Provider |
| CE1/CE2 | 65000 | Customer Edge |
| PA-VM1 | 65100 | Site 1 Firewall |
| PA-VM2 | 65200 | Site 2 Firewall |
| POP1 | 65300 | SASE POP 1 |
| POP2 | 65400 | SASE POP 2 |

### Measured Latency

| Path | Latency | Notes |
|------|---------|-------|
| MPLS-VPN (tunnel.1) | ~10-12ms | Direct PE–PE path |
| SASE-VPN (tunnel.2) | ~80-100ms | Internet/WireGuard path |

**【日本語サマリ】**
MPLS L3VPNとCloudflare SASE（WireGuard）の2経路構成。
PA-VM1/VM2間にIPSecトンネルを2本張り、BGPで経路交換。
MPLS経由は~10ms、SASE経由は~100msとレイテンシ差7-10倍。

---

## 🔀 Protocol Design

### Design Philosophy Comparison: FortiGate vs Palo Alto

| Aspect | FortiGate SD-WAN | Palo Alto PA-VM |
|--------|-----------------|-----------------|
| **Underlay Routing** | Automatic (SD-WAN engine manages ESP path) | Manual static routes required for ESP delivery |
| **Overlay Routing** | BGP/Static with SD-WAN member abstraction | BGP over IPSec with explicit peer-group config |
| **Path Priority** | SD-WAN SLA rules + health-check | BGP local-preference (200=MPLS, 100=SASE) |
| **Failover** | Automatic via SLA threshold breach | Automatic via BGP peer down detection |
| **Tunnel Ping** | Allowed by default on tunnel interface | Requires management-profile on each tunnel IF |
| **Intrazone Traffic** | Allowed by default | Denied by default (explicit rule needed) |
| **Route Leak Prevention** | Implicit (SD-WAN isolates overlay/underlay) | Explicit export policy required |
| **Configuration Style** | Integrated (phase1/phase2/SD-WAN member) | Layered (IKE crypto → gateway → tunnel → BGP → policy) |

> **Key Insight:** FortiGate's SD-WAN engine abstracts three critical layers into a single configuration: underlay ESP routing, overlay user traffic routing, and route leak prevention. On PA-VM, each layer must be configured independently. This separation provides greater visibility and control, but requires deeper understanding of the underlay/overlay relationship.

### Three-Layer Architecture

| Layer | Purpose | Configuration |
|-------|---------|---------------|
| **Underlay Static** | Deliver ESP packets to correct physical path | `MPLS-UL`: 10.200.2.0/24 → e1/1, `SASE-UNDERLAY`: 10.0.1.0/24 → e1/2 |
| **IPSec Tunnel** | Encrypt/decrypt traffic between sites | IKEv2 + ESP (AES-256-CBC, SHA256, DH14) |
| **Overlay BGP** | Route user traffic through preferred tunnel | Local-pref 200 (MPLS) / 100 (SASE) via import policy |

### Data Flow

```
[User Traffic - Overlay]
Branch1 → PA-VM1 → BGP selects tunnel.1 (LocPrf 200)
  → encrypt → ESP packet

[ESP Delivery - Underlay]
ESP → e1/1 → CE1 → PE1 → MPLS → PE2 → CE2 → e1/1 → PA-VM2
  → decrypt → tunnel.1 → Branch2

[Failover - MPLS down]
BGP peer PA-VM2-MPLS goes down → route withdrawn
  → BGP automatically selects tunnel.2 (LocPrf 100)
  → ESP via e1/2 → POP1 → WireGuard → POP2 → e1/2 → PA-VM2
```

**【日本語サマリ】**
FGのSD-WANはunderlay/overlay/ループ防止を1つのエンジンで自動処理。
PA-VMでは3層を個別に設定: underlay static route、IPSecトンネル、overlay BGP。
この分離構造を理解していることがマルチベンダー設計の鍵。

---

## ⚙️ Configuration Summary

### IPSec VPN

Both tunnels use identical crypto parameters with IKEv2:

| Parameter | Value |
|-----------|-------|
| IKE Version | IKEv2 |
| Encryption | AES-256-CBC |
| Hash | SHA256 |
| DH Group | Group 14 |
| IKE Lifetime | 8 hours |
| IPSec Lifetime | 1 hour |
| PSK | paloalto123 |
| Anti-Replay | Enabled |

### BGP Peer Groups (PA-VM1)

| Peer Group | Peer | Remote AS | Local IF | Purpose |
|------------|------|-----------|----------|---------|
| CE | CE1 | 65000 | e1/1 (10.1.1.2) | MPLS underlay routes |
| POP | POP1 | 65300 | e1/2 (10.0.0.1) | SASE underlay routes |
| VPN-MPLS | PA-VM2-MPLS | 65200 | tunnel.1 (10.254.1.1) | Overlay via MPLS |
| VPN-SASE | PA-VM2-SASE | 65200 | tunnel.2 (10.255.2.1) | Overlay via SASE |

### BGP Import Policy

| Rule | Applied To | Action | Local-Preference |
|------|-----------|--------|-----------------|
| MPLS-PREFER | VPN-MPLS | Allow | 200 (Primary) |
| SASE-BACKUP | VPN-SASE | Allow | 100 (Backup) |

### BGP Export Policy (Loop Prevention)

| Rule | Applied To | Match | Action |
|------|-----------|-------|--------|
| NO-VPN-TO-CE | CE | from-peer VPN-MPLS | Deny |
| NO-VPN-TO-CE2 | CE | from-peer VPN-SASE | Deny |
| NO-VPN-TO-POP | POP | from-peer VPN-MPLS | Deny |
| NO-VPN-TO-POP2 | POP | from-peer VPN-SASE | Deny |
| ALLOW-REST-CE | CE | 0.0.0.0/0 (any) | Allow |
| ALLOW-REST-POP | POP | 0.0.0.0/0 (any) | Allow |

### Security Policy (App-ID)

| Rule | From | To | Application | Action |
|------|------|----|-------------|--------|
| Trust-to-VPN | Trust | VPN | any | Allow |
| VPN-to-Trust | VPN | Trust | any | Allow |
| Untrust-to-VPN | Untrust | VPN | any | Allow |
| VPN-to-Untrust | VPN | Untrust | any | Allow |
| **VPN-Intrazone** | **VPN** | **VPN** | **bgp, ping** | **Allow** |
| Trust-to-Untrust | Trust | Untrust | any | Allow |
| Untrust-to-Trust | Untrust | Trust | any | Allow |

> The VPN-Intrazone rule uses App-ID to restrict tunnel-internal traffic to BGP and ping only, demonstrating Palo Alto's application-aware firewall capability.

**【日本語サマリ】**
IPSec: IKEv2/AES-256/SHA256/DH14の2トンネル構成。
BGP: Import PolicyでMPLS優先(LocPrf 200)、Export PolicyでVPN経路のunderlay漏洩防止。
FW: VPN Intrazone にApp-ID適用（bgp + pingのみ許可）。

---

## ✅ Verification Results

### 1. IPSec Tunnel Status

Both IKEv2 SAs established with ESP child SAs in "Mature" state.

```
admin@PA-VM1> show vpn ike-sa
IKEv2 SAs
Gateway ID   Peer-Address    Gateway Name
1            10.200.2.2      MPLS-VPN-GW
2            10.0.1.1        SASE-VPN-GW
```

### 2. Tunnel Ping (Bidirectional)

```
admin@PA-VM1> ping source 10.254.1.1 host 10.254.1.2    → 0% loss, ~10ms
admin@PA-VM1> ping source 10.255.2.1 host 10.255.2.2    → 0% loss, ~70ms
admin@PA-VM2> ping source 10.254.1.2 host 10.254.1.1    → 0% loss, ~12ms
admin@PA-VM2> ping source 10.255.2.2 host 10.255.2.1    → 0% loss, ~97ms
```

### 3. BGP Peer Status (All Established)

```
admin@PA-VM1> show routing protocol bgp summary
  peer POP1:          AS 65300, Established
  peer CE1:           AS 65000, Established
  peer PA-VM2-MPLS:   AS 65200, Established   ← Over tunnel.1
  peer PA-VM2-SASE:   AS 65200, Established   ← Over tunnel.2
```

### 4. BGP Local-Preference Verification

PA-VM2's loc-rib shows MPLS routes preferred (LocPrf 200) over SASE routes (LocPrf 100):

```
admin@PA-VM2> show routing protocol bgp loc-rib
  Prefix             Nexthop       Peer            LocPrf
  10.0.0.0/24        10.254.1.1    PA-VM1-MPLS     200  *   ← Best (MPLS)
  10.1.1.0/30        10.254.1.1    PA-VM1-MPLS     200  *   ← Best (MPLS)
  192.168.1.0/24     10.254.1.1    PA-VM1-MPLS     200  *   ← Best (MPLS)
  10.0.0.0/24        10.255.2.1    PA-VM1-SASE     100      ← Backup (SASE)
  10.1.1.0/30        10.255.2.1    PA-VM1-SASE     100      ← Backup (SASE)
  192.168.1.0/24     10.255.2.1    PA-VM1-SASE     100      ← Backup (SASE)
```

### 5. Export Policy (Clean CE BGP Table)

After applying export policy and clearing CE BGP, overlay routes no longer appear on CE2:

```
CE2# show ip bgp
     Network          Next Hop       Path
 *   10.1.1.0/30      10.102.1.1     65001 65001 i     ← MPLS only
 *   10.101.1.0/30    10.102.1.1     65001 ?           ← MPLS only
 *   10.102.1.0/30    10.102.1.1     65001 ?           ← MPLS only
 *   192.168.1.0      10.102.1.1     65001 65001 i     ← MPLS only
```

> No routes from PA-VM2 (AS 65200) visible — export policy successfully prevents overlay route leakage.

**【日本語サマリ】**
IPSec SA確立 → トンネルping疎通 → 全BGPピアEstablished →
LocPrf 200/100でMPLS優先動作確認 → Export PolicyでCEへのルート漏洩防止確認。

---

## 🔧 Troubleshooting (10 Issues Encountered)

### Issue 1: IKE SA Not Automatically Established

|  |  |
|--|--|
| **Symptom** | `show vpn ike-sa` → No SAs found |
| **Cause** | No traffic-generating route (static/BGP) pointing through the tunnel; PAN-OS IPSec is on-demand |
| **Fix** | `test vpn ike-sa gateway MPLS-VPN-GW` to manually trigger; resolved permanently after BGP config |

### Issue 2: Tunnel Interface Not Responding to Ping

|  |  |
|--|--|
| **Symptom** | IPSec SA established, ESP packets flowing, but tunnel IP ping 100% loss |
| **Cause** | Missing management-profile on tunnel interfaces; PA-VM requires explicit ping permission per interface |
| **Fix** | `set network interface tunnel units tunnel.1 interface-management-profile allow-ping` |
| **Contrast** | FortiGate: `set allowaccess ping` on physical IF covers tunnels too; PA-VM requires per-interface config |

### Issue 3: SASE-VPN Underlay Asymmetric Routing

|  |  |
|--|--|
| **Symptom** | MPLS-VPN works, SASE-VPN fails; ESP packets routed via MPLS path instead of SASE |
| **Cause** | PA-VM learned 10.0.1.0/24 via BGP (CE→MPLS path) instead of direct SASE path |
| **Fix** | Static route `SASE-UNDERLAY`: 10.0.1.0/24 → ethernet1/2 via POP |
| **Lesson** | IPSec underlay must be pinned with static routes; BGP can override ESP delivery path |

### Issue 4: WireGuard Not Running

|  |  |
|--|--|
| **Symptom** | POP1→PA-VM2 unreachable; `wg show` returns empty |
| **Cause** | WireGuard service not started after EVE-NG reboot |
| **Fix** | `wg-quick up wg0` on both POP1 and POP2 |

### Issue 5: One-Way IPSec SA Activation

|  |  |
|--|--|
| **Symptom** | PA-VM2→PA-VM1 ping works, PA-VM1→PA-VM2 fails; ESP bidirectional on wire |
| **Cause** | SA activation timing issue after rekey; responder side not fully initialized |
| **Fix** | Ping from both directions to activate SA bidirectionally |

### Issue 6: Nested Tunnel Encapsulation (Critical)

|  |  |
|--|--|
| **Symptom** | MPLS-VPN ping fails; `flow_tunnel_encap_nested` counter incrementing |
| **Cause** | BGP overlay route for 10.200.2.0/24 (MPLS-VPN ESP destination) pointed through SASE tunnel → tunnel-in-tunnel |
| **Fix** | Static route `MPLS-UL`: 10.200.2.0/24 → ethernet1/1 via CE |
| **Lesson** | Overlay BGP must never override underlay ESP destination routes; static pinning is mandatory |

### Issue 7: Routing Loop via CE Re-advertisement (Critical)

|  |  |
|--|--|
| **Symptom** | PA-VM2→CE1 ping returns ICMP Redirect + TTL exceeded; `flow_fwd_l3_ttl_zero` counter incrementing |
| **Cause** | PA-VM2 re-advertised IPSec overlay routes (10.1.1.0/30 etc.) to CE2 via BGP; CE2 preferred these over MPLS routes and sent ESP back to PA-VM2 → loop |
| **Fix** | BGP export policy: deny VPN-MPLS/VPN-SASE routes to CE and POP peer groups, with allow-all default |
| **Lesson** | Without export policy, overlay routes leak into underlay and create loops. FortiGate handles this implicitly; PA-VM requires explicit configuration |

### Issue 8: PAN-OS CLI Command Truncation

|  |  |
|--|--|
| **Symptom** | Long `set` commands cut off in EVE-NG console/SSH |
| **Cause** | Terminal width limitation in EVE-NG console and SSH client |
| **Fix** | Use hierarchical mode (`edit`/`top`) to break commands into short lines |
| **Lesson** | Production PA-VM uses WebGUI/Panorama/XML API; CLI hierarchical mode is essential for lab work |

### Issue 9: Static Route Missing Destination (Commit Error)

|  |  |
|--|--|
| **Symptom** | `Validation Error: static-route is missing 'destination'` |
| **Cause** | `edit` creates the object immediately but required parameter (destination) was not set before commit |
| **Fix** | Add destination to incomplete object, or `delete` and recreate |

### Issue 10: BGP Import Policy Not Taking Effect

|  |  |
|--|--|
| **Symptom** | Local-preference shows 100 for all routes despite import policy set to 200 |
| **Cause** | Existing BGP sessions don't re-evaluate routes when policy is added |
| **Fix** | Enable `soft-reset-with-stored-info` on peer-group; commit triggers BGP flap which re-applies policy |

**【日本語サマリ】**
10件のトラブルを経験。特に重要なのはIssue 6/7: underlay/overlay分離の問題。
Issue 6: overlay BGPがunderlayのESP宛先を上書き→nested tunnel発生。
Issue 7: overlay経路がCEに漏洩→ルーティングループ発生。
両方ともstatic route固定+export policyで解決。FGが暗黙的に処理していた部分。

---

## 🛠️ Lab Environment

| Component | Detail |
|-----------|--------|
| **Platform** | EVE-NG Pro on PC (32GB RAM) |
| **PA-VM** | PAN-OS 10.1.0 (eval license) |
| **CE/PE** | Cisco IOL (IOS 15.x) |
| **POP** | Debian Linux + FRR + WireGuard |
| **SASE** | Cloudflare Zero Trust (Gateway/WARP) |
| **EVE-NG Host** | 192.168.133.133 |

---

## 🔍 Quick Validation Sequence

After deployment, verify in this order. Each step depends on the previous one succeeding.

```
1. WireGuard       : POP1# wg show                          → peer handshake recent
                     POP1# ping -c 3 10.255.0.2             → 0% loss
2. MPLS Underlay   : CE2# show ip bgp summary               → PfxRcd > 0
3. IPSec SA        : PA-VM1> show vpn ike-sa                 → 2 gateways found
4. Tunnel Ping     : PA-VM1> ping source 10.254.1.1 host 10.254.1.2  → 0% loss
                     PA-VM1> ping source 10.255.2.1 host 10.255.2.2  → 0% loss
5. BGP Established : PA-VM1> show routing protocol bgp summary       → all peers Established
6. Local-Pref      : PA-VM1> show routing protocol bgp loc-rib       → MPLS=200, SASE=100
7. Export Policy   : CE2# show ip bgp → no AS65200 routes from PA-VM2
```

**【日本語サマリ】**
WireGuard起動確認→MPLS underlay→IPSec SA→Tunnel ping→BGP→LocPrf→Export Policyの順で検証。

---

## 📋 Remaining Tasks

| Task | Status | Notes |
|------|--------|-------|
| IPSec VPN (dual tunnel) | ✅ Done | MPLS-VPN + SASE-VPN |
| BGP over IPSec | ✅ Done | VPN-MPLS / VPN-SASE peer-groups |
| MPLS/SASE Priority | ✅ Done | Local-preference 200/100 |
| Export Policy | ✅ Done | VPN route leak prevention |
| App-ID Firewall | ✅ Done | VPN Intrazone: bgp + ping |
| NAT | ⬜ Not needed | Cloudflare SASE handles internet NAT; no branch PCs in lab |
| WebGUI Access | ⬜ Pending | Management IF down; POP SSH tunnel setup incomplete |
| Brownout Failover Test | ⬜ Pending | `tc qdisc add dev ens5 root netem delay 300ms` on POP1 |
| Full App-ID Policy | ⬜ Pending | Other zone-pair rules still use `application any` |

---

## 📚 Key Takeaways

1. **Underlay/Overlay Separation is Everything.** FortiGate's SD-WAN engine hides the most critical design decision — keeping ESP delivery routes separate from encrypted user traffic routes. On PA-VM, forgetting a single static route causes nested tunnels (Issue 6) or routing loops (Issue 7). Understanding this separation is the foundation of multi-vendor SD-WAN/VPN design.

2. **PA-VM's "Deny by Default" Philosophy.** Unlike FortiGate, PA-VM denies intrazone traffic, requires explicit management-profiles per interface, and doesn't auto-establish IPSec SAs without traffic. Every permission must be explicitly granted. This is more secure but increases configuration complexity.

3. **BGP Export Policy is Mandatory.** When running BGP over IPSec alongside underlay BGP (CE/PE), overlay routes will leak into the underlay without export policy. This creates loops that are difficult to diagnose — the symptom (ICMP Redirect + TTL exceeded) doesn't immediately point to BGP route leakage as the root cause.

4. **Multi-Vendor Experience Reveals Hidden Assumptions.** Each vendor makes different assumptions about what should be automatic vs explicit. FortiGate automates underlay routing; Viptela centralizes it via vSmart/OMP; PA-VM leaves it entirely to the operator. Knowing all three approaches enables vendor-neutral network design.

**【日本語サマリ】**
1. Underlay/Overlay分離が最重要設計ポイント。FGは暗黙的、PA-VMは明示的。
2. PA-VMは「明示的に許可しないものは全て拒否」が徹底されている。
3. BGP over IPSec構成ではexport policyが必須。なければルーティングループ発生。
4. マルチベンダー経験により、各社の暗黙の前提が見えるようになる。

---

## 🔗 Related Repositories

* [Enterprise-SP](https://github.com/mikio-abe/Enterprise-SP) – MPLS L3VPN underlay configuration
* [SASE-ZeroTrust](https://github.com/mikio-abe/SASE-ZeroTrust) – Cloudflare Zero Trust integration
* [SD-WAN (FortiGate)](https://github.com/mikio-abe/SD-WAN) – FortiGate SD-WAN with brownout testing
* [Cisco Viptela SD-WAN](https://github.com/mikio-abe/Cisco-Viptela-SD-WAN-CLI-Only-No-vManage) – Viptela CLI-only lab (no vManage)
* [Troubleshooting](https://github.com/mikio-abe/Troubleshooting) – Network troubleshooting methodology
