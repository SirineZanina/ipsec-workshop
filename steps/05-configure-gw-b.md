# Step 5 — Configure GW-B (Tunis)

> ⏱ ~5 minutes | Run on: **GW-B only**

---

## 5.1 — Create the Connection Config

```bash
sudo tee /etc/swanctl/conf.d/sousse-tunis.conf << 'EOF'
connections {
  sousse-tunis {
    version = 2
    local_addrs  = 192.168.56.20
    remote_addrs = 192.168.56.10

    proposals = aes256gcm16-prfsha384-ecp384

    dpd_delay = 30s

    local {
      auth = psk
      id   = tunis@workshop.local
    }
    remote {
      auth = psk
      id   = sousse@workshop.local
    }

    children {
      sousse-tunis-child {
        local_ts  = 192.168.56.20/32
        remote_ts = 192.168.56.10/32

        esp_proposals = aes256gcm16-ecp384
        dpd_action    = restart
        start_action  = none
        close_action  = restart
        mode          = tunnel
      }
    }
  }
}
EOF
```

### What Changed from GW-A

| Field | GW-A | GW-B |
|-------|------|------|
| `local_addrs` | `192.168.56.10` | `192.168.56.20` |
| `remote_addrs` | `192.168.56.20` | `192.168.56.10` |
| `local id` | `sousse@workshop.local` | `tunis@workshop.local` |
| `remote id` | `tunis@workshop.local` | `sousse@workshop.local` |
| `local_ts` | `192.168.56.10/32` | `192.168.56.20/32` |
| `remote_ts` | `192.168.56.20/32` | `192.168.56.10/32` |
| `start_action` | `start` | `none` ← important |

> ⚠️ **`start_action = none` on GW-B is required.** If both VMs have `start_action = start`, both sides attempt to initiate simultaneously after boot, causing a race condition where both keep rejecting each other's initiation. Only GW-A should initiate.

---

## 5.2 — Create the Secrets File

Use the **exact same PSK** you generated on GW-A. The IDs are listed in mirrored order (local first):

```bash
sudo tee /etc/swanctl/conf.d/secrets.conf << 'EOF'
secrets {
  ike-sousse-tunis {
    id-tunis  = tunis@workshop.local
    id-sousse = sousse@workshop.local
    secret    = <PASTE_SAME_PSK_HERE>
  }
}
EOF
sudo chmod 600 /etc/swanctl/conf.d/secrets.conf
sudo chown root:root /etc/swanctl/conf.d/secrets.conf
```

### Why Are the IDs Listed in a Different Order?

Each VM lists its own identity first as a readability convention. StrongSwan doesn't care about the order — it just needs to find a secret entry where **both IDs match**. What matters is that the `secret` value is **character-for-character identical** on both sides.

| | GW-A secrets.conf | GW-B secrets.conf |
|---|---|---|
| First ID | `id-sousse` (I am Sousse) | `id-tunis` (I am Tunis) |
| Second ID | `id-tunis` (peer is Tunis) | `id-sousse` (peer is Sousse) |
| secret | **must match exactly** | **must match exactly** |

---

## 5.3 — Verify the Config File

```bash
cat /etc/swanctl/conf.d/sousse-tunis.conf
cat /etc/swanctl/conf.d/secrets.conf
```

---

## Next Step

➡️ [Step 6 — Bring Up the Tunnel](06-bring-up-tunnel.md)
