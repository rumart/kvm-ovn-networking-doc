# 10 Five rules to remember
- **OVS forwards; OVN decides the intended forwarding.** They are related, not interchangeable.
- **A VM connects to `br-int`, but belongs to an OVN logical switch through its port binding.**
- **A logical switch does not imply a physical VLAN.** In overlay mode, the physical fabric transports tunnel packets between chassis IPs.
- **A logical router is normally distributed.** Routing between internal logical networks does not require a central router VM.
- **The external boundary is different.** A localnet port bridges into a physical L2 network; shared SNAT typically uses one active gateway chassis per relevant gateway port.

## Two designs at a glance

| **Overlay-first design**                                       | **VLAN-first design**                                                 |
|----------------------------------------------------------------|-----------------------------------------------------------------------|
| Tenant L2 is carried by Geneve between KVM hosts.              | Tenant L2 is carried by configured physical VLANs.                    |
| Physical fabric needs routed transport reachability.           | Physical fabric must carry each used VLAN.                            |
| More flexibility for tenant isolation and addressing.          | Simple connection to existing L2, but dependent on VLAN availability. |
| External access normally goes through localnet gateway design. | External routing can be performed by the physical fabric.             |

**Caution on “VLAN-only”:** attaching two hosts to the same localnet/VLAN avoids a Geneve tunnel for that L2 path. It does **not** automatically mean that all OVN logical routing between different VLAN-backed networks is tunnel-free; OVN may still use tunnels for distributed routing. For a strictly tunnel-free design, decide deliberately where inter-VLAN routing should happen. [1]
