# Physical-to-logical architecture

*Generic KVM + OVS + OVN*

![Physical switch through host NICs, OVS and OVN to VMs](images/physical-to-logical-architecture-new.png)

Physical IPs are underlay addresses; OVN logical switches and VM IPs form the guest network. Diagram is conceptual, not a literal packet trace.

![Host network constructs](images/host-logical-networking.png)
