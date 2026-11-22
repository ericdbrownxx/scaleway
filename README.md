# Scaleway Apple Silicon Mac mini: Connection Guide

Replace `51.159.121.59` with your server's IP if it ever changes. Mac username: `m1`.

## Part A: One-time setup

### 1. Block public VNC so only the SSH tunnel can reach it

Run on the Mac, over SSH:

```bash
echo 'block drop in quick on ! lo0 proto tcp from any to any port 5900' | sudo tee /etc/pf.vnc.conf
sudo pfctl -E
sudo pfctl -a com.apple/200.vncblock -f /etc/pf.vnc.conf
```

Make it survive reboots:

```bash
sudo tee /Library/LaunchDaemons/local.vnc-block.plist >/dev/null <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
<key>Label</key><string>local.vnc-block</string>
<key>ProgramArguments</key><array><string>/bin/sh</string><string>-c</string><string>/sbin/pfctl -E; /sbin/pfctl -a com.apple/200.vncblock -f /etc/pf.vnc.conf</string></array>
<key>RunAtLoad</key><true/>
</dict></plist>
EOF
sudo launchctl bootstrap system /Library/LaunchDaemons/local.vnc-block.plist
```

### 2. Verify the block works

From Windows PowerShell:

```powershell
Test-NetConnection 51.159.121.59 -Port 59010
```

`TcpTestSucceeded : False` is the result you want. If it says `True`, the rule is not active, so repeat step 1.

## Part B: Every time you want to connect

### Step 1. Open the SSH tunnel (Windows PowerShell)

```powershell
ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -L 127.0.0.1:5901:127.0.0.1:5900 m1@51.159.121.59
```

Enter the `m1` password and **leave this window open** the whole time you use VNC.

### Step 2. Connect with a VNC client to 127.0.0.1 port 5901

Never use port 59010 or the public IP.

**TigerVNC**
- VNC server: `127.0.0.1::5901`
- Username `m1` and the Mac account password when prompted.

**Remote Desktop Manager (RDM)**
- Host: `127.0.0.1` (not `localhost`)
- Port: `5901`
- Username: `m1`
- Password: the `m1` account password
- If the ARD entry type fails, use a plain **VNC** entry with the same settings.

### Step 3. When finished

Close the VNC client, then type `exit` in the PowerShell window.

## Part C: If it stops working

### 1. Is the tunnel alive?

In a second PowerShell window:

```powershell
Test-NetConnection 127.0.0.1 -Port 5901
```

If `False`, the SSH window died. Reopen it (Part B, step 1).

### 2. SSH window shows `channel 2: open failed: connect failed: Operation timed out`

Screen Sharing is hung. Run on the Mac:

```bash
sudo killall screensharingd
sleep 3
nc -v -w 5 127.0.0.1 5900
```

You should see `RFB 003.889`. Then reconnect your VNC client.

### 3. Service keeps crashing

```bash
sudo log show --last 30m --predicate 'process == "screensharingd"' | grep -iE "Authentication:|error" | grep -v errno:32 | tail -20
```

`Authentication: FAILED` from IPs that are not yours means the public port is exposed again. Redo Part A.

### 4. Last resort

```bash
sudo reboot
```

Wait 2 to 3 minutes, then start again from Part B.

## Part D: Good habits

Check for logins that are not yours:

```bash
sudo log show --last 24h --predicate 'process == "screensharingd"' | grep "Authentication:" | grep -v FAILED | tail
```

- Use a strong `m1` password, since SSH on port 22 is also exposed to the internet.
- Keep the Mac awake:

```bash
sudo pmset -a sleep 0 displaysleep 0 disablesleep 1
```

- Scaleway bills the Mac mini while it exists. Back up what you need and delete the server from the Scaleway console when you are done.

## Quick reference

| Item | Value |
|---|---|
| SSH | `ssh m1@51.159.121.59` |
| Tunnel | `ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -L 127.0.0.1:5901:127.0.0.1:5900 m1@51.159.121.59` |
| VNC target | `127.0.0.1` port `5901` |
| Restart Screen Sharing | `sudo killall screensharingd` |
| Public VNC port (keep closed) | `59010` |
