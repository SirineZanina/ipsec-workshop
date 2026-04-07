# Step 7 — Verify the Tunnel

> ⏱ ~20 minutes | Run on: **GW-A (and GW-B for some checks)**

---

## 7.1 — Check Active Security Associations

```bash
sudo swanctl --list-sas
```

Expected output:
```
sousse-tunis: #1, ESTABLISHED, IKEv2, ...
  sousse-tunis-child: #1, reqid 1, INSTALLED, TUNNEL, ...
    local  192.168.56.10/32
    remote 192.168.56.20/32
```

Two things must be true:
- IKE SA status: **ESTABLISHED**
- Child SA status: **INSTALLED**

If the Child SA is missing, the IKE handshake succeeded but the data plane didn't come up. Run the initiate command again.

---

## 7.2 — Check the Kernel XFRM State (SAD)

The Security Association Database (SAD) lives in the kernel. StrongSwan installs SA entries here after a successful IKE_AUTH:

```bash
sudo ip xfrm state list
```

You should see **exactly 2 entries** — one for each direction of traffic (inbound and outbound):

```
src 192.168.56.10 dst 192.168.56.20
    proto esp spi 0xABCD1234 ...
    enc aes-gcm ...

src 192.168.56.20 dst 192.168.56.10
    proto esp spi 0xEF567890 ...
    enc aes-gcm ...
```

> **Why 2 SAs?** IPsec SAs are **unidirectional** by design. One SA encrypts traffic going from GW-A to GW-B. A separate SA encrypts traffic going from GW-B to GW-A. This is by design — each direction can have independent keys, algorithms, and sequence numbers.

Note the **SPI values** (Security Parameter Index) — you will see these same values in the Wireshark capture. The SPI is how the receiver looks up which key to use for decryption.

---

## 7.3 — Check XFRM Policies

```bash
sudo ip xfrm policy list
```

Policies tell the kernel **which traffic to encrypt**. You should see entries matching your traffic selectors:

```
src 192.168.56.10/32 dst 192.168.56.20/32
    dir out ...
    tmpl ... proto esp ...

src 192.168.56.20/32 dst 192.168.56.10/32
    dir fwd ...
    tmpl ... proto esp ...
```

`dir out` = outbound traffic gets encrypted. `dir fwd`/`dir in` = inbound ESP gets decrypted.

---

## 7.4 — Verify Traffic is Encrypted

Open **two terminals** on GW-A.

**Terminal 1 — capture traffic on the WAN interface:**
```bash
sudo tcpdump -i enp7s0 -n 'esp or icmp'
```

**Terminal 2 — send traffic to the peer:**
```bash
ping -c 5 192.168.56.20
```

In Terminal 1 you should see **ESP packets**, not ICMP:
```
192.168.56.10 > 192.168.56.20: ESP(spi=0xABCD1234, seq=0x1)
192.168.56.20 > 192.168.56.10: ESP(spi=0xEF567890, seq=0x1)
```

> **What you are observing:** The ICMP ping payload is completely hidden inside the ESP envelope. tcpdump can see the outer IP header (so it knows which VMs are talking) and the ESP header (SPI + sequence number), but the inner IP header and the ICMP payload are encrypted and invisible. This is ESP Tunnel Mode working correctly.

If you see raw ICMP packets instead of ESP, the XFRM policy is not matching. Check `ip xfrm policy list` and verify the traffic selectors match the IPs you are pinging.

---

## 7.5 — Capture the Full IKE Handshake

To capture a complete IKEv2 handshake from scratch, you need to tear down the tunnel and re-initiate it while tcpdump is running. However, the default config has `start_action = start` and `close_action = restart` on **both gateways** — these cause the tunnel to automatically re-establish the moment it goes down, making it impossible to catch the full handshake. You must temporarily disable them on both VMs first.

### Step 1 — Disable auto-reconnect on both GW-A and GW-B

Run this on **GW-A**:
```bash
sudo nano /etc/swanctl/conf.d/sousse-tunis.conf
```

Change:
```
start_action  = none
close_action  = none
```

Run the same on **GW-B**:
```bash
sudo nano /etc/swanctl/conf.d/sousse-tunis.conf
```

Change:
```
start_action  = none
close_action  = none
```

### Step 2 — Stop strongSwan on both VMs

Stop on **GW-B first**, then **GW-A** — this prevents GW-B from re-initiating while you're resetting GW-A:

**GW-B:**
```bash
sudo systemctl stop strongswan
```

**GW-A:**
```bash
sudo systemctl stop strongswan
sudo ip xfrm state flush
sudo ip xfrm policy flush
```

What these three commands do:

- **`sudo systemctl stop strongswan`** — stops the strongSwan daemon (charon). This tears down all active IKE and Child SAs and sends DELETE notifications to the peer.
- **`sudo ip xfrm state flush`** — clears the kernel's Security Association Database (SAD). Even after strongSwan stops, the kernel may still hold leftover SA entries with encryption keys. This wipes them completely.
- **`sudo ip xfrm policy flush`** — clears the kernel's Security Policy Database (SPD). These are the rules that tell the kernel which traffic to encrypt/decrypt. Flushing this ensures no stale policies match traffic after the tunnel is gone.

> Together, the three commands give you a guaranteed clean slate — no daemon, no kernel SAs, no kernel policies. Without all three, the tunnel can appear gone from strongSwan's perspective while the kernel is still silently processing ESP packets using old keys.

### Step 3 — Start strongSwan back up on both VMs

**GW-B:**
```bash
sudo systemctl start strongswan
sudo swanctl --load-all
```

**GW-A:**
```bash
sudo systemctl start strongswan
sudo swanctl --load-all
```

Confirm the tunnel is not up yet:
```bash
sudo swanctl --list-sas
# Should return nothing
```

### Step 4 — Start the capture (Terminal 1 on GW-A)

```bash
sudo tcpdump -i enp7s0 -w ~/ike_capture.pcap \
  'udp port 500 or udp port 4500 or esp'
```

### Step 5 — Initiate the tunnel (Terminal 2 on GW-A)

```bash
sudo swanctl --initiate --child sousse-tunis-child
sleep 3
ping -c 5 192.168.56.20
```

### Step 6 — Stop the capture

`Ctrl+C` in Terminal 1.

### Step 7 — Restore the config on both VMs

Run on **GW-A** and **GW-B**:
```bash
sudo nano /etc/swanctl/conf.d/sousse-tunis.conf
```

Change back:
```
start_action  = start
close_action  = restart
```

Reload on both:
```bash
sudo swanctl --load-all
```

### Step 8 — Copy the pcap to your host machine

First, find GW-A's IP address:
```bash
ip a show enp7s0
```

Then run this **on your host machine** (not on GW-A), replacing `<GW-A-IP>` with the IP you just noted:

```bash
scp <user>@<GW-A-IP>:~/ike_capture.pcap ~/ike_capture.pcap
```

On Windows (PowerShell):
```powershell
scp <user>@<GW-A-IP>:~/ike_capture.pcap C:\Users\YourUsername\Desktop\ike_capture.pcap
```

---

## 7.6 — Wireshark Analysis

Open `ike_capture.pcap` in Wireshark and apply these filters:

| Filter | What to look for |
|--------|-----------------|
| `isakmp` | IKEv2 handshake — exactly 4 packets (IKE_SA_INIT x2 + IKE_AUTH x2) |
| `esp` | ESP data packets — completely opaque payload |
| `udp.port == 500` | IKE negotiation — which VM sent the first packet? |
| `udp.port == 4500` | NAT-T — does it appear? (yes if NAT is detected between the VMs) |

**Key observations:**
- IKE packets: you can see algorithm proposals and DH values in `IKE_SA_INIT`, but identities are encrypted in `IKE_AUTH`
- ESP packets: only the outer IP header + SPI are visible — inner IP, ICMP, payload = completely invisible
- Match the SPI from `ip xfrm state` with the SPI visible in ESP packets in Wireshark

---

## ✅ Tunnel is Working When:

- `swanctl --list-sas` → **ESTABLISHED** + **INSTALLED** ✅
- `ip xfrm state` → **exactly 2 entries** with correct SPIs ✅
- `tcpdump` → **ESP packets** when pinging, no raw ICMP ✅
- Wireshark → **4 IKE packets** (IKE_SA_INIT × 2, IKE_AUTH × 2) ✅
- `journalctl -u strongswan` → **no errors** ✅

---

## Next Step

➡️ [Step 8 — Troubleshooting](08-troubleshooting.md) (if needed)
