# Step 6 — Bring Up the Tunnel

> ⏱ ~10 minutes | Run on: **both VMs, then GW-A**

---

## 6.1 — Start StrongSwan and Load Config

Run on **both GW-A and GW-B**:

```bash
sudo systemctl enable --now strongswan
sudo swanctl --load-all
```

Expected output:
```
loaded ike secret 'ike-sousse-tunis'
no authorities found, 0 unloaded
no pools found, 0 unloaded
loaded connection 'sousse-tunis'
successfully loaded 1 connections, 0 unloaded
```

> If you see `loading connection failed: unknown option: dpd_action`, you have `dpd_action` at the connection level instead of inside `children`. Go back to Step 4/5 and fix it.

---

## 6.2 — Open Firewall Ports

Run on **both VMs**:

```bash
sudo ufw allow 500/udp
sudo ufw allow 4500/udp
```

Port 500 is the IKEv2 negotiation port. Port 4500 is used for NAT-T (IKE over UDP when NAT is detected).

---

## 6.3 — Watch the Logs (Optional but Recommended)

Open a second terminal on GW-A and start following the log:

```bash
sudo journalctl -u strongswan -f
```

Keep this running while you initiate — you will see the IKE negotiation happen in real time.

---

## 6.4 — Initiate the Tunnel

Run on **GW-A only**:

```bash
sudo swanctl --initiate --child sousse-tunis-child
```

Expected output:
```
[IKE] initiating IKE_SA sousse-tunis[1] to 192.168.56.20
[ENC] generating IKE_SA_INIT request 0 [ SA KE No N(NATD_S_IP) ... ]
[NET] sending packet: from 192.168.56.10[500] to 192.168.56.20[500]
[NET] received packet: from 192.168.56.20[500] to 192.168.56.10[500]
[CFG] selected proposal: IKE:AES_GCM_16_256/PRF_HMAC_SHA2_384/ECP_384
[IKE] authentication of 'sousse@workshop.local' with pre-shared key
[IKE] IKE_SA sousse-tunis[1] established ...
[IKE] CHILD_SA sousse-tunis-child{1} established ...
```

### What Just Happened — The IKEv2 Handshake

The output shows the 4-packet IKEv2 handshake:

| Message | Direction | Purpose |
|---------|-----------|---------|
| `IKE_SA_INIT request` | GW-A → GW-B | Propose algorithms, start DH exchange |
| `IKE_SA_INIT response` | GW-B → GW-A | Accept proposal, complete DH exchange |
| `IKE_AUTH request` | GW-A → GW-B | Authenticate with PSK, propose Child SA |
| `IKE_AUTH response` | GW-B → GW-A | Confirm auth, install Child SA |

After this, the IKE SA is **ESTABLISHED** and the Child SA (ESP tunnel) is **INSTALLED** in the kernel.

---

## 6.5 — Clean Reset (if needed)

If the tunnel is in a bad state and you want to start fresh:

```bash
sudo swanctl --terminate --ike sousse-tunis
sudo ip xfrm state flush
sudo ip xfrm policy flush
sudo swanctl --load-all
sudo swanctl --initiate --child sousse-tunis-child
```

---

## Next Step

➡️ [Step 7 — Verify the Tunnel](07-verify-tunnel.md)
