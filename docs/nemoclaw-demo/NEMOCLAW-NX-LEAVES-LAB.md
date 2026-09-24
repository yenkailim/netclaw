# NemoClaw Nexus leaves lab handoff

Copy this document to the lab owner. It is the CML / NX-OS spec for the
NemoClaw egress demo (two Nexus leaves, one IP later allowlisted).

Do **not** copy passwords from `testbed/testbed.yaml`. That file currently
has credentials in git. New lab logins go in environment variables only.

NemoClaw OpenShell policy, the HUD, and the sandbox are **out of scope**
for this handoff. Return the IP table and golden lab; policy is applied
later from those numbers.

## Purpose

Always-on CML lab for a NemoClaw egress demo.

- **NX-1** = in-scope (SSH will be allowlisted later)
- **NX-2** = same role, same testbed, **omitted from policy** later
- Hero change = **Loopback99** (additive, reversible, does not break OSPF)

This is not a VXLAN / vPC / BGP showcase. If it takes more than one glance
to explain the topology, it is too big.

The demo needs three things to be true:

1. The boxes look like a Nexus leaf pair (NX-OS CLI, OSPF FULL).
2. Both are SSH-able from pyATS on the NemoClaw host.
3. Mgmt IPs are static, so later policy can list NX-1 and omit NX-2.

## Topology (exactly this)

```text
        mgmt LAN (static, reachable from NemoClaw host/sandbox)
              |                         |
           mgmt0                     mgmt0
            NX-1                      NX-2
         Ethernet1/1 --------------- Ethernet1/1
              10.1.12.1/30            10.1.12.2/30
                    OSPF 1, area 0, point-to-point
```

- **2 nodes only.** No spine, no IOS-XE core, no Linux jumphost, no extra
  unmanaged switch except whatever CML needs for mgmt.
- Node labels, hostnames, and pyATS device names: **`NX-1`** and **`NX-2`**
  (exact).
- Lab title: `nemoclaw-nx-leaves`.

## Image

| Want | Why |
| --- | --- |
| Cisco **Nexus 9000v** (CML `nxosv9000` / NX-OS 9k) | Prompt and CLI are real Nexus (`feature ospf`, `Ethernet1/1`, `#`) |
| Fallback: any CML **NX-OS** image that does OSPF + Loopback + SSH | Still a Nexus story |
| **Do not use** IOSv / IOL / CAT / CSR and label them Nexus | The CCIE audience will call it |

9000v is RAM-heavy (often ~8 GB each). If the CML VM cannot hold two, take
the **lightest NX-OS** image, not a different OS. Confirm RAM **before**
building.

First boot 10–15 minutes is normal. Do not build this lab live in the
session.

## Addressing (this is the demo contract)

This dCloud pod’s data plane is **VLAN-PRIMARY** (`198.18.128.0/18` via
`vPodGW`). `oc-admin-alma` (NemoClaw host) is on that VLAN at
`198.18.134.13/24`, default gateway `198.18.128.1`.

**Yes: NX-1 and NX-2 mgmt0 sit on VLAN-PRIMARY as well.** If they live on
a CML-internal-only NAT, pyATS from Alma never reaches them, and a later
NemoClaw deny looks like “no route,” not policy.

What does **not** go on VLAN-PRIMARY: the Ethernet1/1 OSPF `/30` and the
Loopbacks. Those stay on the lab fabric inside CML.

Mgmt IPs must be **static** and **stable across reboot**. DHCP breaks the
NemoClaw YAML. Prefer unused addresses on the same `/24` as Alma so they
are on-link (no extra hop through `vPodGW`):

| Device | Interface | Address | Notes |
| --- | --- | --- | --- |
| oc-admin-alma | ens192 | `198.18.134.13/24` | Already present; do not collide |
| NX-1 | mgmt0 (VRF `management`) | `198.18.134.11/24` | Allowlisted later; pick another free `.134.x` if `.11` is taken |
| NX-2 | mgmt0 (VRF `management`) | `198.18.134.12/24` | Same VLAN; **not** listed in policy later |
| NX-1 | Ethernet1/1 | `10.1.12.1/30` | P2P inside CML only |
| NX-2 | Ethernet1/1 | `10.1.12.2/30` | P2P inside CML only |
| NX-1 | Loopback0 | `10.0.0.1/32` | OSPF router-id |
| NX-2 | Loopback0 | `10.0.0.2/32` | OSPF router-id |
| both | Loopback99 | **absent** | Demo delta |

CML’s **external connector** for mgmt0 must bridge onto VLAN-PRIMARY, not
a private CML NAT. If CML itself is a VM, put that VM on VLAN-PRIMARY too
(sandbox must reach CML HTTPS for inventory). It is missing from the
current pod drawing; add it or the lab has nowhere to run.

If `.11` / `.12` are already used, pick two free addresses in
`198.18.134.0/24` and write them in the return packet. Do not use
`198.18.134.13` or `198.18.128.1`.

**Handoff must include the real mgmt IPs.** Policy is written from those
numbers.

Reachability required **from the NemoClaw host**, before policy:

- SSH `:22` to NX-1 works
- SSH `:22` to NX-2 works
- ICMP optional

If NX-2 is not reachable from the host, the later deny is “no route,” not
NemoClaw.

## Device config (golden, both boxes)

Keep it this thin:

- `hostname NX-1` / `NX-2`
- `feature ssh`, `feature ospf`
- **Off:** `nxapi`, `telnet`, `grpc`, `netconf`, `bash-shell` if you can.
  Scenario 1 is SSH-only on purpose.
- `no system default switchport` (or `no switchport` on Ethernet1/1) so
  the link is L3
- Ethernet1/1: `ip ospf network point-to-point`, `no shutdown`
- OSPF 1, area 0, `network` for Lo0 + the /30
- SSH on mgmt0, `network-admin` user for pyATS
- `ip route 0.0.0.0/0 198.18.128.1` in `vrf context management` (vPodGW)
- `copy running-config startup-config`

Sketch (passwords via env / CML secrets, never in git):

```text
hostname NX-1
feature ssh
feature ospf
no system default switchport

vrf context management
  ip route 0.0.0.0/0 198.18.128.1

interface mgmt0
  vrf member management
  ip address 198.18.134.11/24

interface Ethernet1/1
  no switchport
  ip address 10.1.12.1/30
  ip ospf network point-to-point
  no shutdown

interface Loopback0
  ip address 10.0.0.1/32

router ospf 1
  router-id 10.0.0.1
  network 10.0.0.1/32 area 0
  network 10.1.12.0/30 area 0
```

NX-2 is the same with hostname `NX-2`, mgmt `198.18.134.12/24`, Ethernet1/1
`10.1.12.2/30`, Loopback0 `10.0.0.2/32`, router-id `10.0.0.2`.

Do **not** pre-create Loopback99. Do **not** plant a hello-interval
mismatch. OSPF must be **FULL** at baseline so the only demo delta is
Lo99.

NTP / interface descriptions are optional and weaker than Lo99. Do not
plant a fault that dies when only one leaf changes.

## pyATS testbed

Both devices, `os: nxos`, SSH to the **mgmt IPs**, Unicon.

- Device names: `NX-1`, `NX-2`
- Credentials: `%ENV{NXOS_USERNAME}` / `%ENV{NXOS_PASSWORD}` (or
  equivalent). **Never commit the password.**
- `StrictHostKeyChecking=no` is fine for lab
- **Both devices stay in the testbed.** Removing NX-2 makes the named
  retry impossible — the agent must be able to *name* it and try SSH.

Shape (fill IPs; keep secrets in env):

```yaml
testbed:
  name: nemoclaw-nx-leaves
  credentials:
    default:
      username: "%ENV{NXOS_USERNAME}"
      password: "%ENV{NXOS_PASSWORD}"
  devices:
    NX-1:
      os: nxos
      connections:
        defaults:
          class: unicon.Unicon
        ssh:
          protocol: ssh
          ip: 198.18.134.11
          port: 22
          ssh_options: "-F /dev/null -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
    NX-2:
      os: nxos
      connections:
        defaults:
          class: unicon.Unicon
        ssh:
          protocol: ssh
          ip: 198.18.134.12
          port: 22
          ssh_options: "-F /dev/null -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
```

## Hero change (do not apply it now)

The session will ask NetClaw to put this on “the Nexus leaves.” Only NX-1
should ever get it.

```text
interface Loopback99
  description NEMOCLAW-DEMO
  ip address 10.99.0.1/32
  no shutdown
```

Prove-it: `show ip interface brief` and
`show running-config interface loopback99`.

Revert: delete Lo99 on NX-1, or restore the CML snapshot. Snapshot
**after** golden (OSPF FULL, no Lo99).

## CML packaging

- One lab, left **STARTED**
- External connector + static mgmt, not “whatever DHCP gave us this
  morning”
- Export topology YAML
- Take a **CML snapshot/checkpoint** named `golden-no-lo99`
- Do not publish node serial/console ports onto the lab LAN (console must
  stay on the CML controller)

## Acceptance (lab owner signs this)

From the NemoClaw **host** (not yet the sandbox policy):

1. `show hostname` → `NX-1` / `NX-2`
2. `show ip ospf neighbor` on both → **FULL** on Ethernet1/1
3. `show ip interface brief` → Lo0 up, **no Lo99**
4. pyATS SSH to **both** names succeeds
5. Mgmt IPs survive a node reboot (still `.11` and `.12`)
6. `feature nxapi` / grpc / netconf are off
7. Return packet: topology export, startup configs, testbed YAML (no
   secrets), IP table, pasted baseline shows, snapshot name

## Out of scope

Push back if any of these appear:

- VXLAN, vPC, BGP
- Extra switches or an IOS core
- A TFTP server
- Enabling NX-API or gNMI “for later”
- DHCP mgmt
- IOSv / IOL / CAT labeled as Nexus

## What to return

| Item | Notes |
| --- | --- |
| CML lab title | `nemoclaw-nx-leaves` |
| Topology export | YAML |
| Startup configs | NX-1 and NX-2, no Lo99 |
| pyATS testbed | Both devices; credentials via env only |
| IP table | Real NX-1 / NX-2 mgmt (default `198.18.134.11` / `.12`) |
| Baseline shows | hostname, OSPF neighbor, `show ip interface brief` |
| Snapshot | `golden-no-lo99` |

They own the boxes and IPs. NemoClaw policy (allow NX-1 `:22`, omit NX-2,
CML GET-only) is a later step using the IP table from this return packet.
