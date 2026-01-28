# VPN IP Service Setup

This guide explains how to set up the VPN IP (`10.200.0.1`) on your host system, which is required for Docker containers to bind to the VPN interface.

## Why This Is Needed

Docker port bindings like `10.200.0.1:9080:8080` require the host to already have the IP `10.200.0.1` assigned. Without this setup, you'll see errors like:

```
failed to bind host port 10.200.0.1:9080/tcp: cannot assign requested address
```

> **Checking for Conflicts:** Before starting, check if `10.200.0.1` is free. If you need to use a different IP, see [Changing the IP Pool](CHANGE_IP_POOL.md).

## Quick Setup

### Step 1: Add VPN IP to Loopback Interface

```bash
sudo ip addr add 10.200.0.1/32 dev lo
```

**Verify it's added:**
```bash
ip a show dev lo | grep 10.200.0.1
```

You should see:
```
inet 10.200.0.1/32 scope global lo
```

### Step 2: Make It Persistent (Systemd Service)

The IP will be lost after reboot. Create a systemd service to automatically add it on boot:

```bash
sudo tee /etc/systemd/system/vpn-ip.service >/dev/null <<'EOF'
[Unit]
Description=Add VPN service IP 10.200.0.1 to loopback for Docker binds
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
# Check if IP exists, add if it doesn't
ExecStart=/bin/bash -c '/sbin/ip addr show dev lo | grep -q "10.200.0.1/32" || /sbin/ip addr add 10.200.0.1/32 dev lo'
ExecStart=/sbin/ip link set lo up
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now vpn-ip.service
```

**Verify service is running:**
```bash
sudo systemctl status vpn-ip.service
```

You should see:
```
Active: active (exited) since [timestamp]
```

### Step 3: Set Up Port Forwarding (Important!)

Since your main server likely uses ports 80 and 443, we use `iptables` to redirect traffic destined for `10.200.0.1:80/443` to the Docker containers listening on `8980/8943`.

1. **Add redirection rules:**
   ```bash
   # Redirect HTTP 10.200.0.1:80 -> 8980
   sudo iptables -t nat -A PREROUTING -d 10.200.0.1 -p tcp --dport 80 -j REDIRECT --to-port 8980

   # Redirect HTTPS 10.200.0.1:443 -> 8943
   sudo iptables -t nat -A PREROUTING -d 10.200.0.1 -p tcp --dport 443 -j REDIRECT --to-port 8943
   
   # Redirect DNS 10.200.0.1:53 -> 5353
   sudo iptables -t nat -A PREROUTING -d 10.200.0.1 -p udp --dport 53 -j REDIRECT --to-port 5353
   sudo iptables -t nat -A PREROUTING -d 10.200.0.1 -p tcp --dport 53 -j REDIRECT --to-port 5353
   ```

2. **Make it persistent** (Ubuntu/Debian):
   ```bash
   sudo apt-get install -y iptables-persistent
   sudo netfilter-persistent save
   ```


## Verification

After setting up, verify the IP is available:

```bash
# Check if IP exists
ip a show dev lo | grep 10.200.0.1

# Test binding (should not error)
sudo ss -tulpn | grep 10.200.0.1

# Try starting a container (should work now)
docker compose -f docker-compose.vpn.yml up -d jenkins
```

## Troubleshooting

### IP Already Exists

If you see "File exists" error:
```bash
# Remove existing IP first
sudo ip addr del 10.200.0.1/32 dev lo

# Then add it again
sudo ip addr add 10.200.0.1/32 dev lo
```

### Service Not Starting

**Check service status:**
```bash
sudo systemctl status vpn-ip.service
sudo journalctl -u vpn-ip.service
```

**Common issues:**
- Network not ready: Ensure `network-online.target` is available
- Permission issues: Ensure running with sudo

**Manual start:**
```bash
sudo systemctl start vpn-ip.service
```

### IP Disappears After Reboot

**Verify service is enabled:**
```bash
sudo systemctl is-enabled vpn-ip.service
# Should output: enabled
```

**If not enabled:**
```bash
sudo systemctl enable vpn-ip.service
```

### Docker Still Can't Bind

**Check if IP exists:**
```bash
ip a show dev lo | grep 10.200.0.1
```

**If missing, add manually:**
```bash
sudo ip addr add 10.200.0.1/32 dev lo
```

**Restart Docker containers:**
```bash
docker compose -f docker-compose.vpn.yml restart
```

## Alternative: Using Network Interface

If you prefer not to use loopback, you can add the IP to a network interface:

```bash
# Find your network interface
ip a

# Add IP to interface (replace eth0 with your interface)
sudo ip addr add 10.200.0.1/32 dev eth0
```

However, using loopback (`lo`) is recommended as it's always available and doesn't depend on network interfaces.

## Removing the VPN IP

If you need to remove the VPN IP:

```bash
# Remove IP
sudo ip addr del 10.200.0.1/32 dev lo

# Disable and remove systemd service
sudo systemctl disable vpn-ip.service
sudo systemctl stop vpn-ip.service
sudo rm /etc/systemd/system/vpn-ip.service
sudo systemctl daemon-reload
```

## Technical Details

### Why Loopback Interface?

- **Always available**: Loopback interface (`lo`) is always present, even if network interfaces are down
- **No routing conflicts**: IPs on loopback don't interfere with network routing
- **Simple**: Easy to manage and doesn't require network interface configuration

### IP Address Choice

The IP `10.200.0.1` is chosen because:
- It's the WireGuard VPN server IP (first IP in the VPN subnet)
- It's in the private IP range (RFC 1918)
- It matches our dedicated VPN subnet `10.200.0.0/24`

### Security Considerations

- The IP is only accessible from the VPN network (`10.200.0.0/24`)
- Services binding to this IP are not exposed to the public internet
- This is a security feature, not a vulnerability

## Related Documentation

- [Main README](README.md) - Complete setup guide
- [SSL Certificate Setup](SSL_CERTIFICATE.md) - SSL/TLS configuration
- [Changing IP Pool](CHANGE_IP_POOL.md) - Conflict resolution guide
