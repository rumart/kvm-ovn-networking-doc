# 11a A three-node test plan

This turns the lab recipe into an **acceptance test plan** for a three-host cluster with **no shared storage**. VMs live on local disks, are deliberately spread across hosts and logical networks, east-west traffic must flow **only where policy permits**, and a **second tenant** on its own logical router must stay isolated from the first. Work through the phases in order and follow the packet in the same direction chapter 09 does: underlay first, then control plane, then dataplane, then policy.

## Assumptions and scope

| **Item** | **Assumption** |
|----------|----------------|
| Hosts | kvm-a, kvm-b, kvm-c — each running OVS and ovn-controller (three chassis) |
| OVN central | NB/SB databases and ovn-northd on one of the three hosts, or on a separate machine |
| Underlay | Chassis IPs mutually reachable, Geneve (UDP 6081) permitted, clocks in sync |
| Storage | Local disks only, so **live migration is out of scope**; cold shared-nothing migration is an optional extra |
| Guests | Small test VMs with `ping` and one TCP tool (`nc` or `iperf3`) available |

## Test topology

Extend the lab recipe with a third network, ls-web (10.10.20.0/24), and connect ls-app (10.10.10.0/24), ls-web and ls-db (10.10.30.0/24) to one logical router, lr-tenant. A second tenant gets its own router, lr-tenant2, with ls-app2 (10.20.10.0/24) and ls-db2 (10.20.20.0/24), and **no transit link between the two routers** — the tenants are isolated by construction. Seven VMs cover every interesting path:

| **VM** | **Host** | **Logical switch** | **IP** | **What it exercises** |
|--------|----------|--------------------|--------|-----------------------|
| vm-app1 | kvm-a | ls-app | 10.10.10.11 | anchor for cross-host L2 and routed paths |
| vm-app2 | kvm-b | ls-app | 10.10.10.12 | cross-host L2 on the *same* logical switch |
| vm-web1 | kvm-c | ls-web | 10.10.20.21 | cross-host, cross-network routed path |
| vm-db1 | kvm-c | ls-db | 10.10.30.31 | routed policy target; same-host routing with vm-web1 |
| vm-db2 | kvm-a | ls-db | 10.10.30.32 | same-host routing with vm-app1 — no tunnel needed |
| vm2-app1 | kvm-b | ls-app2 | 10.20.10.11 | tenant 2 routing through its own router, lr-tenant2 |
| vm2-db1 | kvm-c | ls-db2 | 10.20.20.21 | tenant 2 routed + tunneled path; cross-tenant isolation target |

## Phase 0 — Pre-flight

Prove the underlay before OVN is blamed for anything.

- `ping` between all chassis IP pairs (a→b, a→c, b→c and reverse).
- Test full-size frames: `ping -M do -s 1472` for a 1500-byte underlay, between all pairs.
- Confirm NTP/chrony sync on all three hosts.
- Confirm the host firewall permits UDP 6081 between chassis IPs.
- Allow for Geneve overhead (roughly 58 bytes per tenant frame): prefer an underlay MTU of 1600+, or lower guest MTUs to match.

**Pass:** every pair reaches every other, full-size frames survive, clocks agree.

## Phase 1 — Control plane

- `ovn-sbctl show` lists all three chassis with the expected hostnames and tunnel IPs.
- `ovn-nbctl show` matches the intended topology; every VM port reports up (`ovn-nbctl lsp-get-up`).
- Each port binding sits on the correct chassis in the SB database.
- No errors in the ovn-controller logs on any host.

**Pass:** three chassis registered, all ports bound where the VMs actually run.

## Phase 2 — Cross-host L2 (same logical switch)

- Ping vm-app1 ↔ vm-app2 and vm-db2 ↔ vm-db1 in both directions.
- While pinging, capture on the transport NIC (`tcpdump -ni <nic> udp port 6081`): encapsulated packets appear only toward the remote chassis.
- Contrast: during a vm-app1 ↔ vm-db2 exchange (same host kvm-a, different logical networks) no tunnel packets appear for that pair — routing happened locally.
- Send full-size frames with the DF bit set between VMs on different hosts.

**Pass:** cross-host ping works, Geneve is visible on the underlay only for remote paths, and full-size frames survive.

## Phase 3 — Routing between logical networks

- vm-app1 ↔ vm-db2 and vm-web1 ↔ vm-db1: routed on one host, no tunnel.
- vm-app1 ↔ vm-web1 and vm-app2 ↔ vm-db1: routed *and* tunneled.
- vm2-app1 ↔ vm2-db1: the same routed + tunneled mechanics through lr-tenant2 — a second router must behave identically.
- Confirm TTL decrements across each router and that every guest uses its own logical-router gateway (10.10.x.1 for tenant 1, 10.20.x.1 for tenant 2) as its default route.

**Pass:** routed traffic works locally and across hosts with no compulsory detour through a central gateway, as described in chapter 05.

## Phase 4 — Policy enforcement

> **Default allow:** a logical switch with no ACLs forwards everything. Install an explicit low-priority **drop-all** rule on each logical switch *before* adding allow rules, or policy testing silently tests nothing.

Example allow set (stateful, using `allow-related` so established return traffic flows):

| **Rule** | **Intent** |
|----------|------------|
| A1 | ls-app → ls-web: ICMP and TCP 443 only |
| A2 | ls-web → ls-db: TCP 5432 only |
| A3 | intra-switch proof: deny vm-app2 → vm-app1 TCP 22 |
| A4 | ls-app2 → ls-db2 (tenant 2): TCP 5432 only — same method, second router |

Predict before you send: run `ovn-trace` for each row below and record the verdict. Then send real traffic and compare. The matrix:

| **From** | **To** | **Path** | **Traffic** | **Expected** |
|----------|--------|----------|-------------|--------------|
| vm-app1 (kvm-a) | vm-web1 (kvm-c) | routed + tunnel | ICMP | pass |
| vm-app1 | vm-web1 | routed + tunnel | TCP 443 | pass |
| vm-app1 | vm-web1 | routed + tunnel | TCP 22 | **drop** |
| vm-app2 (kvm-b) | vm-db1 (kvm-c) | routed + tunnel | TCP 5432 | pass |
| vm-app2 | vm-db1 | routed + tunnel | ICMP | **drop** |
| vm-app1 | vm-db1 | routed + tunnel | any | **drop** (no rule exists) |
| vm-db1 | vm-app1 | new connection | any | **drop** (only established return may flow) |
| vm-app2 | vm-app1 (kvm-a) | L2 tunnel | ICMP | pass |
| vm-app2 | vm-app1 | L2 tunnel | TCP 22 | **drop** (intra-switch ACL) |
| vm2-app1 (kvm-b) | vm2-db1 (kvm-c) | routed + tunnel | TCP 5432 | pass |
| vm2-app1 | vm-app1 (kvm-a) | cross-tenant | any | **drop** (no route between tenants) |
| vm-app1 | vm2-app1 (kvm-b) | cross-tenant | any | **drop** (no route between tenants) |

> **Isolation by construction:** the cross-tenant rows fail because lr-tenant and lr-tenant2 have no route to each other — not because an ACL dropped them. The moment a transit link is added (Phase 6), ACLs become the only enforcement, so keep the drop-all baseline on every switch.

Statefulness check: a TCP session initiated from the allowed side must succeed while the same session initiated in reverse must fail. Confirm drops with evidence, not silence: drop counters in `ovs-ofctl dump-flows br-int` should increment for the denied rows.

**Pass:** actual traffic matches the matrix in both directions, and every result matches the ovn-trace prediction.

## Phase 5 — Resilience

Verify the availability implication from chapter 02 by experiment:

| **Failure** | **During** | **After recovery** |
|-------------|------------|--------------------|
| Stop ovn-controller on kvm-b | existing VM traffic on all hosts keeps flowing | new configuration propagates; kvm-b rejoins SB |
| Restart OVS on kvm-b | VMs on kvm-b briefly lose connectivity | flows reprogrammed, all phases re-pass |
| Reboot kvm-c | vm-web1, vm-db1 and vm2-db1 gone; kvm-a and kvm-b traffic — including policy — unaffected | port bindings return; policy still enforced |
| Restart OVN central | dataplane unaffected; configuration changes fail briefly | changes propagate again |

Re-run the policy matrix after each recovery. The most common regression is forwarding that survives while policy state drifts.

**Pass:** dataplane independence from the control plane demonstrated; no policy regressions after any recovery.

## Phase 6 — Optional extras

- **Cross-tenant peering:** link lr-tenant and lr-tenant2 through a transit logical switch, then permit exactly one flow with ACLs on it (for example vm2-app1 → vm-db1 TCP 5432). Everything else must stay dropped — now the ACL, not the missing route, is the enforcement.
- **Overlapping tenant addresses:** re-address tenant 2 to reuse tenant 1's subnets and re-run the isolation rows. Isolation must still hold, proving it is logical rather than IP-range based. Expect tcpdump to become confusing — that is why this stays optional.
- **Throughput baseline:** iperf3 vm-app1 → vm-app2 (tunneled) versus vm-app1 → vm-db2 (local routed). Record the numbers for future comparison.
- **Cold shared-nothing migration:** `virsh migrate --copy-storage-all` a VM between hosts, then verify the port binding follows in `ovn-sbctl` and policy still holds. Treat as exploratory — this is the roughest edge in OVN + libvirt setups.
- **North-south:** if the gateway and SNAT design from chapter 07 is deployed, apply the same predict-send-compare method to external traffic.

## Exit criteria

The cluster passes when:

1. All three chassis are registered and stable.
2. Cross-host L2 and routed paths work, with tunnel traffic visible only where it should be.
3. The policy matrix passes completely: no leaks between tiers, no over-blocking.
4. Tenant 2 is unreachable from tenant 1 and vice versa — no cross-tenant traffic of any kind.
5. ovn-trace predictions match observed behavior for every matrix row.
6. Resilience tests complete without policy regression.

> **Keep the matrix:** it becomes the regression suite. Re-run it after every OVS/OVN upgrade, kernel change, or logical-topology edit.
