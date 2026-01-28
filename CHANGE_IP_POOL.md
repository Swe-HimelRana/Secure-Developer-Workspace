# Changing the VPN IP Pool

This guide explains how to check if the default IP pool (`10.200.0.1`) is already in use on your system, and how to change it to a different subnet (e.g., `10.201.0.1`) if necessary.

## Phase 1: Check for IP Conflicts

Before starting, verify that the subnet we intend to use (`10.200.0.0/24`) is free.

### 1. Check Host Routes
Run this command to see if the system already routes traffic for this subnet:

```bash
ip route | grep "10.200.0"
```
*   **Result (Empty):** ✅ Safe to use.
*   **Result (Shows output):** ❌ Conflict found. Choose a different IP (e.g., `10.201.0.1`).

### 2. Check Docker Networks
Run this command to see if any Docker network is already using this subnet:

```bash
docker network inspect $(docker network ls -q) | grep "10.200.0"
```
*   **Result (Empty):** ✅ Safe to use.
*   **Result (Shows output):** ❌ Conflict found. Choose a different IP.

---

## Phase 2: Changing the IP Pool

If you need to change the IP (e.g., from `10.200.0.1` to `10.201.0.1`), follow these steps.

### Step 1: Clean Up Old State
If you already set up the old IP, remove it first.

```bash
# Stop containers
docker compose -f docker-compose.vpn.yml down

# Stop the service
sudo systemctl stop vpn-ip.service

# Remove old IP manually
sudo ip addr del 10.200.0.1/32 dev lo

# Remove old iptables rules (replace IP with your old one)
sudo iptables -t nat -D PREROUTING -d 10.200.0.1 -p tcp --dport 80 -j REDIRECT --to-port 8980
sudo iptables -t nat -D PREROUTING -d 10.200.0.1 -p tcp --dport 443 -j REDIRECT --to-port 8943
sudo iptables -t nat -D PREROUTING -d 10.200.0.1 -p udp --dport 53 -j REDIRECT --to-port 5353
sudo iptables -t nat -D PREROUTING -d 10.200.0.1 -p tcp --dport 53 -j REDIRECT --to-port 5353
sudo netfilter-persistent save
```

### Step 2: Update Configuration Files

You need to find and replace the old IP (`10.200.0.1`) with your new IP (`10.201.0.1`) in the following files:

1.  **`docker-compose.vpn.yml`**
    *   Update `ports` mapping (e.g., `10.201.0.1:9080:8080`)
    *   Update `dnsmasq` command args (`--listen-address`, `--address`)
    *   Update `wireguard` environment (`PEERDNS`, `ALLOWEDIPS`)
    *   **Note:** Also update the subnet `10.200.0.0` -> `10.201.0.0` in `ALLOWEDIPS` and `INTERNAL_SUBNET`.

2.  **`VPN_IP_SETUP.md`**
    *   Update the systemd service definition.
    *   Update the iptables commands.

3.  **`README.md`** & **`SSL_CERTIFICATE.md`**
    *   Update documentation references to avoid confusion later.

### Step 3: Apply the New Setup

**1. Update the Systemd Service**
Create the service with the **NEW** IP.

```bash
sudo tee /etc/systemd/system/vpn-ip.service >/dev/null <<'EOF'
[Unit]
Description=Add VPN service IP 10.201.0.1 to loopback for Docker binds
After=network-online.target

[Service]
Type=oneshot
# Check if IP exists, add if it doesn't
ExecStart=/bin/bash -c '/sbin/ip addr show dev lo | grep -q "10.201.0.1/32" || /sbin/ip addr add 10.201.0.1/32 dev lo'
ExecStart=/sbin/ip link set lo up
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now vpn-ip.service
```

**2. Apply New Port Forwarding**
Use your NEW IP for the rules.

```bash
# HTTP / HTTPS
sudo iptables -t nat -A PREROUTING -d 10.201.0.1 -p tcp --dport 80 -j REDIRECT --to-port 8980
sudo iptables -t nat -A PREROUTING -d 10.201.0.1 -p tcp --dport 443 -j REDIRECT --to-port 8943

# DNS
sudo iptables -t nat -A PREROUTING -d 10.201.0.1 -p udp --dport 53 -j REDIRECT --to-port 5353
sudo iptables -t nat -A PREROUTING -d 10.201.0.1 -p tcp --dport 53 -j REDIRECT --to-port 5353

# Save
sudo netfilter-persistent save
```

**3. Restart Docker**
```bash
docker compose -f docker-compose.vpn.yml up -d
```
