# WSL2 → Raspberry Pi SSH Setup via Windows ProxyJump

Connect to a Raspberry Pi (or any LAN device) from WSL2 using Windows as a jump host.

## The Problem

WSL2 runs in a virtual network (`172.x.x.x`) isolated from your local LAN (`192.168.x.x`), so it can't directly reach devices on your network without extra configuration.

## Solution: Windows as a ProxyJump Host

```
WSL2 Ubuntu  →  Windows Host  →  Raspberry Pi
172.x.x.x       172.x.x.1        192.168.x.x
```

---

## Prerequisites

- WSL2 running Ubuntu on Windows
- Raspberry Pi (or any device) with SSH enabled on your local network
- Windows and the target device on the same LAN subnet
- Windows user account with administrator privileges

---

## Step 1 — Install OpenSSH Server on Windows

Windows needs an SSH server so WSL2 can use it as a jump host.

Open **PowerShell as Administrator** and run:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

Verify it is running:

```powershell
Get-Service -Name sshd
```

> ✅ Status should show `Running`. The service will now start automatically on every boot.

---

## Step 2 — Configure Windows Firewall

Allow inbound SSH connections from WSL2. Run in **PowerShell as Administrator**:

```powershell
# Find your WSL adapter name (look for vEthernet with a 172.x.x.x address)
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress
```

Then allow traffic on that adapter (replace the adapter name if yours differs):

```powershell
New-NetFirewallRule -DisplayName "WSL2 Inbound" -Direction Inbound -InterfaceAlias "vEthernet (WSL (Hyper-V firewall))" -Action Allow
New-NetFirewallRule -DisplayName "OpenSSH Inbound" -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow
```

> ⚠️ If you see `Access Denied`, make sure PowerShell is running as Administrator (right-click → Run as administrator).

---

## Step 3 — Find Your IPs

In your WSL2 terminal, run:

```bash
# Your WSL2 IP
hostname -I

# Your Windows host IP (the gateway)
ip route | grep default | awk '{print $3}'
```

In PowerShell on Windows, find your Raspberry Pi IP:

```powershell
# Scan your LAN or check your router's device list
arp -a
```

---

## Step 4 — Configure SSH in WSL2

Create or edit the SSH config file:

```bash
nano ~/.ssh/config
```

Add the following block:

```
Host rpi
    HostName <PI_IP>           # e.g. 192.168.1.100
    User <PI_USERNAME>         # e.g. ubuntu or pi
    ProxyJump <WIN_USER>@<WIN_GATEWAY_IP>  # e.g. john@172.27.176.1
```

Save with `Ctrl+O`, `Enter`, then `Ctrl+X`.

Fix file permissions (required — SSH ignores config files with wrong permissions):

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chown $USER:$USER ~/.ssh/config
```

---

## Step 5 — Test the Connection

```bash
ssh rpi
```

SSH will authenticate with Windows first, then tunnel through to your Raspberry Pi. You will be prompted for passwords on both machines.

---

## Step 6 — Fix WSL2 DNS (if you get "Name or service not known")

WSL2's default DNS can be unreliable. Fix it permanently:

**Prevent WSL from overwriting DNS config:**

```bash
sudo nano /etc/wsl.conf
```

Add:

```ini
[network]
generateResolvConf = false
```

**Set a reliable DNS server:**

```bash
sudo rm /etc/resolv.conf
sudo nano /etc/resolv.conf
```

Add:

```
nameserver 8.8.8.8
nameserver 1.1.1.1
```

**Restart WSL** from PowerShell:

```powershell
wsl --shutdown
```

Reopen your WSL terminal and try again.

---

## Optional — Passwordless SSH with Keys

Avoid typing passwords on every connection:

```bash
# Generate a key if you don't have one
ssh-keygen -t ed25519

# Copy key to the Windows jump host
ssh-copy-id <WIN_USER>@<WIN_GATEWAY_IP>

# Copy key to the Raspberry Pi
ssh-copy-id rpi
```

After this, `ssh rpi` connects instantly with no password prompts.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Name or service not known` | Fix DNS (Step 6) or check `~/.ssh/config` permissions (Step 4) |
| Connection hangs silently | Windows SSH server not running or firewall blocking port 22 (Steps 1 & 2) |
| `Permission denied` on config | Run `chmod 600 ~/.ssh/config` |
| `Access Denied` in PowerShell | Open PowerShell as Administrator |
| Ping to Windows gateway fails | Add firewall rule for WSL adapter (Step 2) |
| `Cannot find any service 'sshd'` | OpenSSH Server not installed — run Step 1 |
| `vEthernet (WSL)` adapter not found | Run `Get-NetIPAddress` to find the exact adapter name |

---

## Quick Reference

| Item | Command |
|---|---|
| Find WSL2 IP | `hostname -I` |
| Find Windows gateway IP | `ip route \| grep default \| awk '{print $3}'` |
| Find adapter name | `Get-NetIPAddress -AddressFamily IPv4` (PowerShell) |
| Check sshd status | `Get-Service -Name sshd` (PowerShell) |
| Test SSH port reachable | `nc -zv <WIN_IP> 22` |
| SSH with verbose output | `ssh -v rpi` |
