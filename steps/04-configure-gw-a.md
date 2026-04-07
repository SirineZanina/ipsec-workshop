# Step 4 — Configure GW-A (Sousse)

> ⏱ ~10 minutes | Run on: **GW-A only**

---

## 4.1 — Generate a Pre-Shared Key

Generate a strong random PSK. Run this **on GW-A only**:

```bash
openssl rand -base64 48
```

Example output (yours will be different):
```
MAIEb3DcGfHcGz0aX11ytJgrvXii51Ih5SVcugD0ugLwwRnCNMflKJSp2e5796o/
```

> ⚠️ **Copy the full output including any trailing characters like `/` or `+`.** A single missing character will cause `AUTHENTICATION_FAILED` when the tunnel tries to come up. Use copy-paste — never retype it manually.

You will paste this same value on **both VMs** in the secrets files.

---

## 4.2 — Create the Connection Config

```bash
sudo tee /etc/swanctl/conf.d/sousse-tunis.conf << 'EOF'
connections {
  sousse-tunis {
    version = 2
    local_addrs  = 192.168.56.10
    remote_addrs = 192.168.56.20

    proposals = aes256gcm16-prfsha384-ecp384

    dpd_delay = 30s

    local {
      auth = psk
      id   = sousse@workshop.local
    }
    remote {
      auth = psk
      id   = tunis@workshop.local
    }

    children {
      sousse-tunis-child {
        local_ts  = 192.168.56.10/32
        remote_ts = 192.168.56.20/32

        esp_proposals = aes256gcm16-ecp384
        dpd_action    = restart
        start_action  = start
        close_action  = restart
        mode          = tunnel
      }
    }
  }
}
EOF
```

### Config Explained

| Field | Value | Why |
|-------|-------|-----|
| `version = 2` | IKEv2 | IKEv1 is deprecated — always use IKEv2 |
| `proposals` | `aes256gcm16-prfsha384-ecp384` | IKE SA algorithm suite — AES-256-GCM + SHA-384 PRF + ECP-384 DH |
| `dpd_delay` | `30s` | Send a keepalive probe every 30s to detect dead peers |
| `auth = psk` | Pre-Shared Key | Simpler than certificates for a workshop |
| `local_ts` / `remote_ts` | `/32` | Single-host traffic selectors — we use WAN IPs directly, no separate LAN |
| `esp_proposals` | `aes256gcm16-ecp384` | ESP (data plane) algorithm — GCM is AEAD so no separate integrity needed |
| `dpd_action = restart` | restart | If peer dies, restart the Child SA automatically |
| `start_action = start` | start | Auto-initiate the tunnel when StrongSwan starts |
| `mode = tunnel` | tunnel | ESP Tunnel Mode — encrypts the entire original IP packet |

> ⚠️ **Common mistake:** `dpd_action` is only valid inside the `children` block — **not** at the connection level. Placing it at the connection level causes `loading connection failed: unknown option: dpd_action` and the entire config is discarded.

> ⚠️ **Traffic selectors:** We use `/32` (single host), not `/24`. Using `/24` here would mean "encrypt any traffic from anyone in `192.168.56.0/24`" — which overlaps with the peer's IP and causes routing loops.

---

## 4.3 — Create the Secrets File

Replace `<PASTE_YOUR_PSK_HERE>` with the key generated in Step 4.1:

```bash
sudo tee /etc/swanctl/conf.d/secrets.conf << 'EOF'
secrets {
  ike-sousse-tunis {
    id-sousse = sousse@workshop.local
    id-tunis  = tunis@workshop.local
    secret    = <PASTE_YOUR_PSK_HERE>
  }
}
EOF
```

Secure the file — it contains the PSK in plaintext:
```bash
sudo chmod 600 /etc/swanctl/conf.d/secrets.conf
sudo chown root:root /etc/swanctl/conf.d/secrets.conf
```

Verify permissions:
```bash
ls -la /etc/swanctl/conf.d/secrets.conf
# Expected: -rw------- 1 root root ...
```

> `chmod 600` means only the owner can read/write. `chown root:root` makes root the owner. Together: only root can read this file — a regular user cannot `cat` it and steal the key.

---

## 4.4 — Verify the Config File

```bash
cat /etc/swanctl/conf.d/sousse-tunis.conf
```

Check it looks correct before moving on.

---

## Next Step

➡️ [Step 5 — Configure GW-B](05-configure-gw-b.md)
