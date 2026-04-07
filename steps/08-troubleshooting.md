# Step 8 — Troubleshooting

> Reference guide for all errors encountered during this workshop on Ubuntu 24.04 + StrongSwan 5.9.x

---

## Quick Diagnostic Commands

Run these first to get a full picture of the current state:

```bash
# Is the right daemon running?
sudo ss -ulnp | grep 500

# Is the config loaded?
sudo swanctl --list-conns

# Are SAs established?
sudo swanctl --list-sas

# What's in the kernel?
sudo ip xfrm state
sudo ip xfrm policy

# Live log
sudo journalctl -u strongswan -f
```

---

## Error Reference

### ❌ `no socket implementation registered`

**Symptom:**
```
no socket implementation registered, sending failed
```

**Cause:** `charon-systemd` is running but the `socket-default` plugin did not load. This happens when the plugin was installed after the daemon was already started — a full reboot is required.

**Fix:**
```bash
sudo reboot
```

After reboot, verify:
```bash
sudo journalctl -u strongswan | grep "loaded plugins"
# Confirm "socket-default" appears in the list
```

---

### ❌ `Address already in use` on port 500

**Symptom:**
```
unable to bind socket: Address already in use
could not open IPv4 socket, IPv4 disabled
could not create any sockets
```

**Cause:** The legacy `charon` daemon (from `strongswan-starter`) is running and has already claimed UDP port 500 before `charon-systemd` starts.

**Diagnosis:**
```bash
sudo ss -ulnp | grep 500
# You will see "charon" (without -systemd) holding the port
```

**Fix:**
```bash
sudo systemctl stop ipsec
sudo systemctl disable ipsec
sudo systemctl stop strongswan-starter
sudo systemctl disable strongswan-starter
sudo kill <pid of charon>
sudo systemctl restart strongswan
```

Verify only `charon-systemd` owns port 500:
```bash
sudo ss -ulnp | grep 500
# Must show charon-systemd, NOT charon
```

---

### ❌ `loading connection failed: unknown option: dpd_action`

**Symptom:**
```
loading connection 'sousse-tunis' failed: unknown option: dpd_action, config discarded
loaded 0 of 1 connections
```

**Cause:** `dpd_action` was placed at the connection level instead of inside the `children` block.

**Fix:** In `/etc/swanctl/conf.d/sousse-tunis.conf`, ensure `dpd_action` is only inside `children { ... }`, not at the top-level connection block:

```
connections {
  sousse-tunis {
    dpd_delay  = 30s        ← OK here
    dpd_action = restart    ← ❌ NOT here

    children {
      sousse-tunis-child {
        dpd_action = restart  ← ✅ only here
      }
    }
  }
}
```

Then reload:
```bash
sudo swanctl --load-all
```

---

### ❌ `NO_PROPOSAL_CHOSEN`

**Symptom:**
```
received NO_PROPOSAL_CHOSEN notify error
no IKE config found for 192.168.56.10...192.168.56.20
```

**Cause A — Config not loaded on one side:**
```bash
# On both VMs
sudo swanctl --load-all
sudo swanctl --list-conns
```

If `list-conns` shows no connections, the config failed to load. Check for syntax errors.

**Cause B — Algorithm mismatch between VMs:**
```bash
# On both VMs — must be identical
grep proposals /etc/swanctl/conf.d/sousse-tunis.conf
```

The `proposals` and `esp_proposals` lines must match exactly on both sides.

---

### ❌ `AUTHENTICATION_FAILED`

**Symptom:**
```
received AUTHENTICATION_FAILED notify error
initiate failed: establishing CHILD_SA 'sousse-tunis-child' failed
```

**Cause:** The PSK does not match between GW-A and GW-B. Even one missing character causes this.

**Diagnosis:**
```bash
# On both VMs — compare the secret value carefully
sudo cat /etc/swanctl/conf.d/secrets.conf
```

Look for any difference, especially trailing characters like `/`, `+`, or `=`.

**Fix:** Copy-paste the PSK again from the original `openssl rand -base64 48` output. Never retype it manually.

---

### ❌ Tunnel UP but ping gets no reply

**Symptom:** `swanctl --list-sas` shows ESTABLISHED + INSTALLED, but `ping 192.168.56.20` times out.

**Cause:** Traffic selectors don't match the traffic you're sending.

**Diagnosis:**
```bash
sudo ip xfrm policy list
# Check that src/dst match exactly what you're pinging
```

**Fix:** Verify `local_ts` and `remote_ts` in the config. If using `/32`, you must ping the exact IPs in those selectors.

---

### ❌ Both VMs trying to initiate simultaneously

**Symptom:** Tunnel never establishes, both sides show retransmits at the same time.

**Cause:** Both VMs have `start_action = start`.

**Fix:** Set `start_action = none` on GW-B. Only GW-A should initiate.

---

## Nuclear Reset

If everything is broken and you want a guaranteed clean state:

```bash
# On both VMs
sudo swanctl --terminate --ike sousse-tunis
sudo ip xfrm state flush
sudo ip xfrm policy flush
sudo swanctl --load-all

# Then on GW-A only
sudo swanctl --initiate --child sousse-tunis-child
```

If that still doesn't work, revert to the snapshot created in Step 0:
```bash
# On host machine
virsh snapshot-revert --domain gw-a --snapshotname "clean-install"
virsh snapshot-revert --domain gw-b --snapshotname "clean-install"
```

---

## Summary of Root Causes on Ubuntu 24.04

| Problem | Root Cause |
|---------|-----------|
| `no socket implementation registered` | Reboot not done after installing `charon-systemd` |
| Port 500 conflict | Legacy `strongswan-starter` not disabled |
| `NO_PROPOSAL_CHOSEN` | Config not loaded or proposals mismatch |
| `AUTHENTICATION_FAILED` | PSK mismatch — missing trailing character |
| `unknown option: dpd_action` | `dpd_action` at wrong level in config |
| Both sides retransmitting | Both VMs have `start_action = start` |
