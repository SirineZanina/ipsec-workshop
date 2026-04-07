# Step 2 — Static IP Configuration
> ⏱ ~10 minutes | Run on: **both VMs**

Assign static IPs to the `ipsec-lab` interface on each VM.

---

## Why Static IPs?

DHCP assigns IPs dynamically — the IP can change every time the VM reboots or the lease expires. IPsec requires you to explicitly specify each peer's IP address in the config:

```
local_addrs  = 192.168.56.10   # must be exact
remote_addrs = 192.168.56.20   # must be exact
```

If GW-B's IP changes from `192.168.56.20` to something else after a reboot, the tunnel breaks immediately. Static IPs give you **guaranteed, predictable endpoints** — a requirement for any VPN.

> In production, VPN gateways **always** have static IPs or DNS names that resolve to fixed IPs.

---

## 2.1 — Identify the New Interface

After attaching the second NIC in Step 1, open each VM and find the new interface:

```bash
ip link show
```

You will see two interfaces (ignoring `lo`). The one **without** an IP assigned yet is the `ipsec-lab` interface. It will be named something like `enp2s0`, `ens4`, or `eth1` depending on your hypervisor.

> ⚠️ The interface name varies by hypervisor and VM. Always check with `ip link show` — do not assume the name.

---

## 2.2 — Configure GW-A (Sousse)

Edit the Netplan config:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Replace the contents with (adjust interface names to match your system):

```yaml
network:
  version: 2
  ethernets:
    enp1s0:          # NAT adapter — keep DHCP for internet access
      dhcp4: true
    enp2s0:          # ipsec-lab adapter — static IP
      dhcp4: false
      addresses:
        - 192.168.56.10/24
```

Apply and secure:

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

If Netplan did not bring the interface up, bring it up manually:

```bash
sudo ip link set enp2s0 up
```

> Replace `enp2s0` with your actual interface name if different.

---

## 2.3 — Configure GW-B (Tunis)

Edit the Netplan config:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp1s0:          # NAT adapter
      dhcp4: true
    enp2s0:          # ipsec-lab adapter — static IP
      dhcp4: false
      addresses:
        - 192.168.56.20/24
```

Apply and secure:

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

If Netplan did not bring the interface up, bring it up manually:

```bash
sudo ip link set enp2s0 up
```

> Replace `enp2s0` with your actual interface name if different.

---

## 2.4 — Verify Connectivity

From **GW-A**, ping GW-B:

```bash
ping -c 4 192.168.56.20
```

From **GW-B**, ping GW-A:

```bash
ping -c 4 192.168.56.10
```

Expected: `0% packet loss` on both sides. If ping fails, double-check:

- Both VMs are attached to the same `ipsec-lab` network
- Interface names in Netplan match your actual interface names (`ip link show`)
- `sudo netplan apply` was run after editing
- The interface state is UP — run `ip link show <interface>` and confirm `state UP`

---

## Next Step

➡️ [Step 3 — Install Packages](03-install-packages.md)
