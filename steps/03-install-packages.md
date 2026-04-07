# Step 3 — Install StrongSwan and Required Packages

> ⏱ ~10 minutes | Run on: **both VMs**

---

## 3.1 — Install All Packages

Run on **both GW-A and GW-B**:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
  strongswan \
  strongswan-swanctl \
  strongswan-pki \
  charon-systemd \
  libcharon-extra-plugins \
  libcharon-extauth-plugins \
  libstrongswan-extra-plugins \
  libstrongswan-standard-plugins \
  tcpdump \
  wireshark-common \
  net-tools
```

### Package Reference

| Package | Purpose |
|---------|---------|
| `strongswan` | Meta-package — pulls in all core StrongSwan components |
| `strongswan-swanctl` | Modern `swanctl` CLI for managing the running daemon |
| `strongswan-pki` | PKI tools — generates keys and certificates (needed for Bonus A) |
| `charon-systemd` | **The IKE daemon** — runs IKEv2 negotiations, manages SAs. Without this, no tunnel |
| `libcharon-extra-plugins` | EAP authentication + VICI interface support |
| `libcharon-extauth-plugins` | PAM and external authentication plugins for charon |
| `libstrongswan-standard-plugins` | Core crypto: x509, pem, pkcs1/pkcs8, sha2 — needed to load keys and certs |
| `libstrongswan-extra-plugins` | Extra crypto: `ecp` (required for `ecp384` DH group), AES-GCM — without this, `NO_PROPOSAL_CHOSEN` |
| `tcpdump` | Capture ESP and IKE packets on the WAN interface |
| `wireshark-common` | IKEv2/ESP dissectors — open `.pcap` files on your host with Wireshark |
| `net-tools` | `ifconfig`, `netstat` — useful for troubleshooting |

---

## 3.2 — Disable the Legacy IPsec Daemon

> ⚠️ **Critical step — do not skip.**

The `strongswan` metapackage also installs the legacy `strongswan-starter` daemon. If left enabled, it launches the old `charon` binary which **claims UDP port 500 first**, preventing `charon-systemd` from binding. This causes the silent `no socket implementation registered` error.

Disable it permanently on **both VMs**:

```bash
sudo systemctl stop ipsec
sudo systemctl disable ipsec
sudo systemctl stop strongswan-starter
sudo systemctl disable strongswan-starter
```

---

## 3.3 — ⚠️ Reboot Both VMs

```bash
sudo reboot
```

> **This step is mandatory.** Even after installing `charon-systemd`, a reboot is required for it to pick up the `socket-default` plugin correctly on Ubuntu 24.04 with StrongSwan 5.9.x. A `systemctl restart` alone is not sufficient.

---

## 3.4 — Verify After Reboot

After rebooting, SSH back in and run on **both VMs**:

**Check the correct daemon is running:**
```bash
sudo ss -ulnp | grep 500
```

You must see `charon-systemd` — **not** `charon`:
```
UNCONN 0  0  0.0.0.0:500   0.0.0.0:*  users:(("charon-systemd",...))
UNCONN 0  0  0.0.0.0:4500  0.0.0.0:*  users:(("charon-systemd",...))
```

If you see `charon` (without `-systemd`), the legacy daemon is still running:
```bash
sudo systemctl stop ipsec
sudo kill <pid shown in ss output>
sudo systemctl restart strongswan
```

**Check the socket plugin is loaded:**
```bash
sudo journalctl -u strongswan | grep "loaded plugins"
```

Confirm `socket-default` appears in the list. If it does not appear, reboot again.

**Check StrongSwan version:**
```bash
swanctl --version
# or
ipsec version
```

---

## Next Step

➡️ [Step 4 — Configure GW-A](04-configure-gw-a.md)
