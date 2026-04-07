# Step 1 — Network Setup

> ⏱ ~15 minutes | Run on: **host machine**

Each VM needs **two network interfaces**:

| Interface | Network Type | Purpose |
|-----------|-------------|---------|
| Interface 1 | NAT (`default`) | Internet access — for `apt install` |
| Interface 2 | Isolated (`ipsec-lab`) | VPN WAN link between the two gateways |

The isolated network simulates the "public internet" link between the two offices. It has no connection to the real internet or the host machine — traffic only flows between GW-A and GW-B.

---

## 1.1 — Create the Isolated Network

### Method A: virt-manager GUI

1. Open **virt-manager**
2. Go to **Edit → Connection Details**
3. Click the **Virtual Networks** tab
4. Click **"+"** at the bottom left
5. Fill in:
   - **Name:** `ipsec-lab`
   - **Mode:** `Isolated` ← critical, prevents any routing to host/internet
   - **IPv4 / DHCP:** disable entirely (we use static IPs)
6. Click **Finish**

### Method B: virsh CLI

Create the network definition:
```bash
cat > /tmp/ipsec-net.xml << 'EOF'
<network>
  <name>ipsec-lab</name>
  <bridge name="virbr-ipsec" stp="on" delay="0"/>
  <!-- No <forward> tag = fully isolated, no NAT, no routing out -->
</network>
EOF
```

Define, start, and autostart it:
```bash
virsh net-define /tmp/ipsec-net.xml
virsh net-start ipsec-lab
virsh net-autostart ipsec-lab
```

Verify:
```bash
virsh net-list --all
```

Expected output:
```
 Name        State    Autostart   Persistent
----------------------------------------------
 default     active   yes         yes
 ipsec-lab   active   yes         yes
```

---

## 1.2 — Attach Both VMs to the Isolated Network

Each VM needs a second NIC connected to `ipsec-lab`.

### Method A: virt-manager GUI

1. Open the VM details (click the **"i" info button** for the VM)
2. Click **"Add Hardware" → Network**
3. Set **Network source** to `ipsec-lab`
4. Leave model as `virtio`
5. Click **Finish**
6. Repeat for the second VM

### Method B: virsh CLI

```bash
# Attach ipsec-lab to gw-a
virsh attach-interface --domain gw-a \
  --type network \
  --source ipsec-lab \
  --model virtio \
  --config --persistent

# Attach ipsec-lab to gw-b
virsh attach-interface --domain gw-b \
  --type network \
  --source ipsec-lab \
  --model virtio \
  --config --persistent
```

---

## 1.3 — Windows Host (VirtualBox)

If you are using **VirtualBox on Windows** instead of KVM:

**Create the Host-Only network:**
1. Open VirtualBox → **File → Tools → Network Manager**
2. Click **Create** to add a new host-only adapter
3. Set IPv4 address to `192.168.100.1`, mask `255.255.255.0`
4. **Disable the DHCP server** (we assign static IPs manually)

**Configure GW-A adapters:**
- Adapter 1: Host-only → select the adapter you just created (`192.168.100.0/24`)
- Adapter 2: Internal Network → Name: `site-a`

**Configure GW-B adapters:**
- Adapter 1: same Host-only adapter as GW-A
- Adapter 2: Internal Network → Name: `site-b` (different from GW-A)

---

## Next Step

➡️ [Step 2 — Static IP Configuration](02-static-ip.md)
