# 11 A compact lab recipe

This is an **illustrative sequence**, not a complete production deployment or a copy-paste installer. Assume OVN NB/SB and ovn-northd are running, each host has OVS and ovn-controller, and the hosts already have transport IP reachability.

## First: one isolated logical switch

```bash
ovn-nbctl ls-add ls-lab
ovn-nbctl lsp-add ls-lab vm-a-port
ovn-nbctl lsp-set-addresses vm-a-port \
  "02:ac:10:00:00:11 10.10.10.11"
ovn-nbctl lsp-add ls-lab vm-b-port
ovn-nbctl lsp-set-addresses vm-b-port \
  "02:ac:10:00:00:12 10.10.10.12"
ovn-nbctl show
```

Plug the two VM tap devices into br-int with matching iface-ids, using a libvirt integration or a deliberate host-side script. Check the SB port bindings and test a VM-to-VM ping.

## Then: add a router and another network

```bash
ovn-nbctl ls-add ls-db
ovn-nbctl lr-add lr-lab
ovn-nbctl lrp-add lr-lab lrp-app \
  02:ac:ff:00:00:01 10.10.10.1/24
ovn-nbctl lsp-add ls-lab app-to-router
ovn-nbctl lsp-set-type app-to-router router
ovn-nbctl lsp-set-addresses app-to-router router
ovn-nbctl lsp-set-options app-to-router \
  router-port=lrp-app
```

Repeat the router-port/peer-switch-port pattern for ls-db with gateway 10.10.20.1/24. Finally, configure each guest default route. For a full working command sequence and ovn-trace example, use the official OVN sandbox tutorial. [2]

> **Safety:** Do not move a production management NIC into an OVS bridge while connected remotely without a recovery path. Keep management and packet-transport configuration explicit.
