# Step 0 — Create VM Snapshots

> ⏱ ~5 minutes | Run on: **host machine**

Create snapshots of both VMs **before** making any configuration changes. This gives you a guaranteed clean restore point if anything goes wrong.

---

## Why Snapshots?

IPsec configuration involves changes across multiple files, network settings, and daemons. If you misconfigure something and can't recover, a snapshot lets you revert both VMs to a clean state in seconds rather than reinstalling from scratch.

---

## Commands

Run on your **host machine** (not inside the VMs):

```bash
# List all VMs to confirm both are visible
virsh list --all
```

Expected output:
```
 Id   Name   State
--------------------
 1    gw-a   running
 2    gw-b   running
```

Create a snapshot for GW-A:
```bash
virsh snapshot-create-as --domain gw-a \
  --name "clean-install" \
  --description "Before IPsec config" \
  --atomic
```

Create a snapshot for GW-B:
```bash
virsh snapshot-create-as --domain gw-b \
  --name "clean-install" \
  --description "Before IPsec config" \
  --atomic
```

Verify snapshots were created:
```bash
virsh snapshot-list gw-a
virsh snapshot-list gw-b
```

---

## How to Restore a Snapshot

If you need to start over at any point:

```bash
# Revert GW-A to the clean snapshot
virsh snapshot-revert --domain gw-a --snapshotname "clean-install"

# Revert GW-B to the clean snapshot
virsh snapshot-revert --domain gw-b --snapshotname "clean-install"
```

---

## Next Step

➡️ [Step 1 — Network Setup](01-network-setup.md)
