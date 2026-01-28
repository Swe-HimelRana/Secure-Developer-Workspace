
```
Version: 1.0.0 - Tested on Ubuntu 24.04
+++++++++++++++++++++++++++++++++++++++
Although this application is open source and licensed under the MIT License, 
you must also agree to the individual licenses of all other services included in this setup.
```

# Secure Developer Workspace

A comprehensive, VPN-only development environment featuring Jenkins CI/CD, Kanban project management, collaborative whiteboarding, diagramming tools, orchestration, configuration management, and application deployment tools - all accessible securely through WireGuard VPN.

## Why This Application?

This setup provides ready to use **secure, self-hosted development environment** for teams that need:

- 🔒 **Maximum Security**: All services accessible only via WireGuard VPN - no public internet exposure
- 🚀 **CI/CD Pipeline**: Jenkins with Docker agent for building and testing applications
- 🏗️ **Infrastructure as Code**: Semaphore UI for Ansible, Terraform, OpenTofu, and Terragrunt
- 📋 **Project Management**: Kanban board for task tracking and project organization
- ✏️ **Collaboration Tools**: Excalidraw for whiteboarding and Draw.io for diagrams
- 🌐 **Split Tunneling**: VPN only routes traffic to your services, normal internet uses direct connection
- 🔐 **HTTPS Encryption**: SSL certificates for secure communication
- 🐳 **Docker-Based**: Easy to deploy and manage with Docker Compose

**Perfect for:**

- Development teams needing secure CI/CD infrastructure
- Remote teams requiring VPN-only access to development tools
- Organizations wanting self-hosted alternatives to cloud services
- Projects requiring isolated, secure development environments

## Features

- ✅ **Jenkins CI/CD** with Docker agent support
- ✅ **Semaphore UI** for Ansible, Terraform, OpenTofu, and Terragrunt
- ✅ **WireGuard VPN** for secure access
- ✅ **Kanban Board** for project management
- ✅ **Excalidraw** for collaborative whiteboarding
- ✅ **Draw.io** for technical diagrams
- ✅ **Nginx Reverse Proxy** with SSL/TLS support
- ✅ **Custom DNS** resolution via dnsmasq
- ✅ **VPN-Only Access** - services not exposed to public internet
- ✅ **Split Tunneling** - only service traffic routes through VPN

## Prerequisites

### VPS Requirements

**Minimum Configuration:**

- **CPU**: 2 cores (4 cores recommended for better performance)
- **RAM**: 2 GB (4-6 GB recommended)
- **Storage**: 20 GB SSD (50 GB recommended for Jenkins data and logs)
- **Network**: 100 Mbps connection  (1 Gbps recommended for faster experience ) 

**Why these requirements?**

- Jenkins requires significant resources (1 CPU core, 2 GB RAM allocated)
- Multiple services running simultaneously (Jenkins, Semaphore, Kanban, etc.)
- Docker overhead and system processes need additional resources
- Recommended specs provide headroom for concurrent builds and operations

**Note**: Resource limits are configured in `docker-compose.vpn.yml`. You can adjust these based on your VPS capacity, but reducing too much may cause performance issues.

### Software Requirements

- A VPS running Linux (Ubuntu/Debian recommended)
- Docker and Docker Compose installed
- Root or sudo access
- Basic knowledge of Linux commands
- WireGuard client installed on your devices

## Quick Start

### 1. Clone or Download This Repository

```bash
git clone https://github.com/Swe-HimelRana/Secure-Developer-Workspace
cd Secure-Developer-Workspace
```

### 2. Set Up VPN IP Service

The setup requires the VPN IP (`10.200.0.1`) to be available on the host. 

**Recommended: Make it persistent with systemd service:**

> **Checking for Conflicts:** Before starting, check if `10.200.0.1` is free. If you need to use a different IP, see [Changing the IP Pool](CHANGE_IP_POOL.md).

```bash
sudo tee /etc/systemd/system/vpn-ip.service >/dev/null <<'EOF'
[Unit]
Description=Add VPN service IP 10.200.0.1 to loopback for Docker binds
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
# Check if IP exists, add if it doesn't (safe to run even if IP already exists)
ExecStart=/bin/bash -c '/sbin/ip addr show dev lo | grep -q "10.200.0.1/32" || /sbin/ip addr add 10.200.0.1/32 dev lo'
ExecStart=/sbin/ip link set lo up
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now vpn-ip.service
```

**Alternative: Quick setup (temporary, for testing only):**

> **Note:** Only use this if you want to test immediately without persistence. The systemd service above is recommended for production use.

```bash
sudo ip addr add 10.200.0.1/32 dev lo
```

> **Important:** If you use the quick setup above, you can still create the systemd service later. The service will detect that the IP already exists and won't cause conflicts. However, for a clean setup, it's better to use the systemd service from the start.

**If the service fails** (e.g., IP already exists), you can check and fix it:

```bash
# Check if IP is already assigned
ip addr show dev lo | grep 10.200.0.1

# If it's already there, the service will work on next boot
# To manually fix if needed:
sudo ip addr add 10.200.0.1/32 dev lo 2>/dev/null || echo "IP already exists (this is OK)"

# Verify the service status
sudo systemctl status vpn-ip.service
```

**To remove the VPN IP address** (if needed for testing or troubleshooting):

```bash
# Remove the IP address from loopback interface
sudo ip addr del 10.200.0.1/32 dev lo

# Verify it's removed
ip addr show dev lo | grep 10.200.0.1
# (Should return nothing if removed successfully)
```

**Note:** If you have the `vpn-ip.service` enabled, it will re-add the IP on the next boot. To permanently remove it:

```bash
# Disable and stop the service
sudo systemctl disable vpn-ip.service
sudo systemctl stop vpn-ip.service

# Then remove the IP
sudo ip addr del 10.200.0.1/32 dev lo
```
**Set up DNS port forwarding (Important!)** (required to avoid port 53 conflict with systemd-resolved):

   dnsmasq listens on port 5353, but WireGuard clients need to use standard port 53. Set up iptables port forwarding:

   ```bash
   # Redirect DNS requests from 10.200.0.1:53 to 10.200.0.1:5353
   sudo iptables -t nat -A PREROUTING -d 10.200.0.1 -p udp --dport 53 -j REDIRECT --to-port 5353
   sudo iptables -t nat -A PREROUTING -d 10.200.0.1 -p tcp --dport 53 -j REDIRECT --to-port 5353

   # Make it persistent (Ubuntu/Debian)
   sudo apt-get install -y iptables-persistent
   sudo netfilter-persistent save
   ```
 **Why this is needed:**
   - systemd-resolved uses `127.0.0.1:53` (server DNS)
   - dnsmasq uses `10.200.0.1:5353` (VPN DNS, avoids conflict)
   - iptables redirects `10.200.0.1:53` → `10.200.0.1:5353` (transparent to clients)
   - WireGuard clients use `DNS = 10.200.0.1` (standard port 53, works with macOS)

**Set up Nginx Port Forwarding (Important!)**  (required to access services via standard ports):

Since your main server likely uses ports 80 and 443, we use `iptables` to redirect traffic destined for `10.200.0.1:80/443` to the Docker containers listening on `8980/8943`.

1. **Add redirection rules:**
   ```bash
   # Redirect HTTP 10.200.0.1:80 -> 8980
   sudo iptables -t nat -A PREROUTING -d 10.200.0.1 -p tcp --dport 80 -j REDIRECT --to-port 8980

   # Redirect HTTPS 10.200.0.1:443 -> 8943
   sudo iptables -t nat -A PREROUTING -d 10.200.0.1 -p tcp --dport 443 -j REDIRECT --to-port 8943
   ```

2. **Make it persistent** (Ubuntu/Debian):
   ```bash
   sudo apt-get install -y iptables-persistent
   sudo netfilter-persistent save
   ```

3. **Verify Persistence:**
   Run this to confirm rules are saved:
   ```bash
   sudo cat /etc/iptables/rules.v4 | grep 10.200.0.1
   ```

**Why is this needed?**
- Prevents conflict with your main Nginx, Traefik or Apache server binding to `0.0.0.0:80`.
- Allows you to access apps via standard URLs (e.g., `https://jenkins.hs`) without typing ports.


For more info about vpn ip configuration see [VPN IP Service Setup](VPN_IP_SETUP.md).

### 3. Configuration Files (`.env`, `tempmail.yml`, `startpage.yml`)

Before starting services, prepare the key configuration files:

- **`.env`** (Jenkins agent + WireGuard server settings):

  - Copy the example file and edit:

    ```bash
    cp env.example .env
    nano .env
    ```
  - Set at least:

    ```bash
    JENKINS_SECRET=your_secret_here           # From Jenkins UI after creating docker-agent node
    WIREGUARD_SERVER_URL=auto                # Or your VPS public IP
    WIREGUARD_PEERS=1                        # Number of client configs to generate
    SEMAPHORE_ADMIN_PASSWORD=changeme        # Change after first login
    SEMAPHORE_ADMIN_EMAIL=admin@localhost
    ```
- **`tempmail.yml`** (TempMail service configuration):

  - Use the provided sample as a starting point:

    ```bash
    cp tempmail.sample.yml tempmail.yml
    nano tempmail.yml
    ```
  - Ensure `tempmail.yml` is a **file**, not a directory, and update:

    - IMAP host
    - catch-all email
    - app-specific password
    - domains you want to use
- **`startpage.yml`** (Glance dashboard configuration):

  - Edit `startpage.yml` in this repo to configure your dashboard sections, links, and widgets.
  - This file is mounted into the `dashboard` container as `glance.yml`.

> These files must exist and be valid **before** you start the containers, otherwise some services may fail or use defaults.

### 4. Start Services (without Jenkins Docker agent)

> The Jenkins Docker agent (`docker-agent`) needs a valid secret from the Jenkins UI.
> **Do not** start it until you’ve created the node and updated `.env`.

**Start all core services (excluding Jenkins `docker-agent`):**

```bash
docker compose -f docker-compose.vpn.yml up -d --scale docker-agent=0
```

**Or start individual core services: (Optional)**

```bash
# Start WireGuard VPN (required first)
docker compose -f docker-compose.vpn.yml up -d wireguard

# Start DNS server
docker compose -f docker-compose.vpn.yml up -d dnsmasq

# Start Jenkins
docker compose -f docker-compose.vpn.yml up -d jenkins

# Start Kanban board
docker compose -f docker-compose.vpn.yml up -d kanban

# Start Excalidraw (draw.hs)
docker compose -f docker-compose.vpn.yml up -d excalidraw

# Start Draw.io (diagram.hs)
docker compose -f docker-compose.vpn.yml up -d drawio

# Start Startpage (Dashboard)
docker compose -f docker-compose.vpn.yml up -d dashboard

# Start TempMail
docker compose -f docker-compose.vpn.yml up -d tempmail

# Start Ansible Semaphore (Ansible, Terraform, OpenTofu, Terragrunt)
docker compose -f docker-compose.vpn.yml up -d semaphore

# Start Nginx reverse proxy
docker compose -f docker-compose.vpn.yml up -d nginx
```

**Verify services are running:**

```bash
docker compose -f docker-compose.vpn.yml ps
```
You should see all core services with status "Up" or "Healthy".

**Get Wireguard client configuration:**

   ```bash
   docker compose -f docker-compose.vpn.yml exec wireguard cat /config/peer1/peer1.conf
   ```
  
After connection you should be able to access all services.

### 5. Install SSL Certificate Authority (CA) on Your Devices

**Important:** Before accessing services via HTTPS, you must install the mkcert root CA certificate on each device. This allows your browser to trust the SSL certificates used by the services.

The repository includes the CA certificate files.
- `mkcert-root-ca.crt` - Certificate file (use this for installation)
- `mkcert-root-ca.pem` - Alternative format
- `rootCA-key.pem` - Private key (keep secure, not needed for client installation)
- `rootCA.pem` - Certificate (keep secure, not needed for client installation, Used for creating certificate)

#### Install CA Certificate

**On macOS:**
1. Copy `mkcert-root-ca.crt` to your Mac
2. Double-click the file to open it in Keychain Access
3. Find "mkcert" or "mkcert root CA" in the login keychain
4. Double-click the certificate
5. Expand "Trust" section
6. Set "When using this certificate" to **"Always Trust"**
7. Close the window and enter your password when prompted

**On Windows:**
1. Copy `mkcert-root-ca.crt` to your Windows computer
2. Double-click the file
3. Click **"Install Certificate"**
4. Choose **"Local Machine"** (requires admin rights)
5. Select **"Place all certificates in the following store"**
6. Click **"Browse"** and select **"Trusted Root Certification Authorities"**
7. Click **"Next"** → **"Finish"**
8. Click **"Yes"** on the security warning

**On Linux (Ubuntu/Debian):**
```bash
# Copy the certificate to the system certificates directory
sudo cp mkcert-root-ca.crt /usr/local/share/ca-certificates/mkcert-root-ca.crt

# Update the certificate store
sudo update-ca-certificates
```

**On iOS:**
1. Copy `mkcert-root-ca.crt` to your iPhone/iPad (via AirDrop, email, or cloud storage)
2. Open the file on your device
3. Go to **Settings** → **General** → **VPN & Device Management** (or **Profiles & Device Management**)
4. Tap on the downloaded profile
5. Tap **"Install"** in the top right
6. Enter your passcode
7. Tap **"Install"** again to confirm
8. Go to **Settings** → **General** → **About** → **Certificate Trust Settings**
9. Enable trust for the "mkcert root CA" certificate

**On Android:**
1. Copy `mkcert-root-ca.crt` to your Android device
2. Go to **Settings** → **Security** → **Encryption & credentials** (or **Install from storage**)
3. Tap **"Install a certificate"** → **"CA certificate"**
4. Select the `mkcert-root-ca.crt` file
5. Enter a name for the certificate (e.g., "mkcert Root CA")
6. Tap **"OK"**

> **Note:** After installing the CA certificate, you may need to restart your browser or device for the changes to take effect.

### 6. Access Services

1. **Connect to WireGuard VPN** (see [WireGuard Setup](VPN_IP_SETUP.md))
2. Access services via:
   - **Jenkins**: `https://jenkins.hs`
   - **Kanban**: `https://kanban.hs`
   - **Excalidraw**: `https://draw.hs`
   - **Draw.io**: `https://diagram.hs`
   - **Startpage (Dashboard)**: `https://startpage.hs`
   - **TempMail**: `https://tempmail.hs`
   - **Ansible Semaphore** (Ansible, Terraform, OpenTofu, Terragrunt): `https://semaphoreui.hs`

#### Jenkins Initial Setup

1. **Access Jenkins:**

   - Open `https://jenkins.hs` in your browser
   	For the very first Jenkins login, you need the **initial admin password**:

		```bash
		docker compose -f docker-compose.vpn.yml exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
		```
   - Complete initial setup wizard
2. **Configure Docker Agent:**

   - Go to `Manage Jenkins` → `Manage Nodes and Clouds` → `New Node`
   - Node name: `docker-agent`
   - Type: `Permanent Agent`
   - Remote root directory: `/home/jenkins/workspace`
   - Launch method: `Launch agent by connecting it to the controller` (JNLP)
   - Copy the agent secret
3. **Update `.env` with agent secret:**

   ```bash
   JENKINS_SECRET=your_secret_from_jenkins_ui
   ```
4. **Start Docker agent:**

   ```bash
   docker compose -f docker-compose.vpn.yml up -d docker-agent
   ```


**Check individual service status:**

```bash
# Check if specific service is running
docker compose -f docker-compose.vpn.yml ps jenkins
docker compose -f docker-compose.vpn.yml ps excalidraw
docker compose -f docker-compose.vpn.yml ps drawio
docker compose -f docker-compose.vpn.yml ps kanban
docker compose -f docker-compose.vpn.yml ps dashboard
docker compose -f docker-compose.vpn.yml ps tempmail

# View logs
docker compose -f docker-compose.vpn.yml logs excalidraw
docker compose -f docker-compose.vpn.yml logs drawio
docker compose -f docker-compose.vpn.yml logs dashboard
docker compose -f docker-compose.vpn.yml logs tempmail
docker compose -f docker-compose.vpn.yml logs semaphore
```


## Configuration

### Service Ports

All services bind to `10.200.0.1` (VPN interface only):

- **Jenkins**: `10.200.0.1:9080` → `https://jenkins.hs`
- **Jenkins Agent**: `10.200.0.1:9500`
- **Kanban**: `10.200.0.1:9090` → `https://kanban.hs`
- **Excalidraw**: `10.200.0.1:8081` → `https://draw.hs`
- **Draw.io**: `10.200.0.1:8082` → `https://diagram.hs`
- **Startpage (Dashboard)**: `10.200.0.1:8083` → `https://startpage.hs`
- **TempMail**: `10.200.0.1:8084` → `https://tempmail.hs`
- **Ansible Semaphore** (Ansible, Terraform, OpenTofu, Terragrunt): `10.200.0.1:8085` → `https://semaphoreui.hs`
- **Nginx**: `10.200.0.1:80` (HTTP redirect) and `10.200.0.1:443` (HTTPS)


### Customizing Services

You can modify `docker-compose.vpn.yml` to:

- Adjust resource limits (CPU/memory)
- Change port mappings
- Add environment variables
- Modify service configurations

## Authentication & Default Credentials

**Jenkins**

- Initial admin password (first login):
  - `docker compose -f docker-compose.vpn.yml exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`
- Jenkins Docker agent:
  - Node name: `docker-agent`
  - Agent secret: from Jenkins UI (`Manage Jenkins` → `Manage Nodes and Clouds` → `docker-agent`)
  - Store in `.env` as `JENKINS_SECRET=...`

**WireGuard**

- Peer configs:
  - `docker compose -f docker-compose.vpn.yml exec wireguard cat /config/peer1/peer1.conf`
- Import `peer1.conf` into your WireGuard client.

**TempMail**

- Config file: `tempmail.yml` (host) → `/var/www/html/tempmail.yml` (container)
- Contains:
  - IMAP host, catch-all email, app password, and allowed domains.
- **Do not commit real credentials**; keep them only in your private `tempmail.yml`.

**Semaphore UI**

- Admin user (on fresh DB initialization):
  - Username: `admin`
  - Password: from `.env` → `SEMAPHORE_ADMIN_PASSWORD` (default `changeme`, change after first login)
  - Email: from `.env` → `SEMAPHORE_ADMIN_EMAIL`
- Credentials are only applied when Semaphore initializes a new BoltDB database.

## Adding New Services

You can add additional services to this setup. However, **keep it focused on essential developer tools**.

**Good additions:**

> **Note**: These services were not included as they may not be required for all developers or development servers. However, if you believe they are required, you can add them and we recommend creating a pull request.

- Code review tools (e.g., Gitea, GitLab)
- Documentation servers (e.g., Wiki.js, BookStack)
- Monitoring tools (e.g., Grafana, Prometheus)
- Development databases (e.g., PostgreSQL, Redis)
- Kubernetes cluster (e.g., Minikube) 

**Avoid:**

- Entertainment services (media servers, game servers)
- Non-development tools
- Services that don't fit the developer workflow

**To add a service:**

1. Add service to `docker-compose.vpn.yml`
2. Add DNS entry in dnsmasq command: `--address=/newservice.hs/10.200.0.1`
3. Add nginx reverse proxy configuration
4. Update this README with the new service

## Troubleshooting

### Services Not Accessible

- **Verify VPN connection**: `ping 10.200.0.1`
- **Check DNS resolution**: `nslookup jenkins.hs 10.200.0.1`
- **Verify containers are running**: `docker compose ps`
- **Check logs**: `docker compose logs [service-name]`

### Docker Agent Won't Connect

- Verify node name matches: `docker-agent`
- Check secret in `.env` matches Jenkins UI
- View agent logs: `docker compose logs docker-agent`

### SSL Certificate Issues

See [SSL Certificate Guide](SSL_CERTIFICATE.md) for troubleshooting.

## Maintenance

### Update Services

```bash
docker compose -f docker-compose.vpn.yml pull
docker compose -f docker-compose.vpn.yml up -d
```

### Backup Jenkins Data

```bash
# Find the actual volume name first
docker volume ls | grep jenkins_home

# Backup (replace 'secure-developer-workspace_jenkins_home' with your actual volume name)
docker run --rm -v secure-developer-workspace_jenkins_home:/data -v $(pwd):/backup alpine tar czf /backup/jenkins-backup.tar.gz /data
```

### View Logs

```bash
# All services
docker compose -f docker-compose.vpn.yml logs -f

# Specific service
docker compose -f docker-compose.vpn.yml logs -f jenkins
```

## Security Considerations

- ✅ **VPN-Only Access**: Services not exposed to public internet
- ✅ **HTTPS Encryption**: SSL/TLS for all connections
- ✅ **Split Tunneling**: Only service traffic routes through VPN
- ✅ **Firewall**: Only WireGuard port (51820/udp) open publicly
- ⚠️ **Keep secrets secure**: Never commit `.env` file
- ⚠️ **Regular updates**: Keep Docker images updated
- ⚠️ **Backup regularly**: Especially Jenkins data

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Author

Created by **Himel**

Contact: contact[at]himelrana.com

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

**Guidelines:**

- Keep services focused on developer tools
- Maintain security best practices
- Update documentation for any changes
- Test changes thoroughly before submitting

## References

### Official Application URLs

- **Jenkins**: [https://www.jenkins.io/](https://www.jenkins.io/)
- **Kanban (Kanboard)**: [https://kanboard.org/](https://kanboard.org/)
- **Excalidraw**: [https://excalidraw.com/](https://excalidraw.com/)
- **Draw.io**: [https://www.diagrams.net/](https://www.diagrams.net/)
- **WireGuard**: [https://www.wireguard.com/](https://www.wireguard.com/)
- **Docker**: [https://www.docker.com/](https://www.docker.com/)
- **Nginx**: [https://nginx.org/](https://nginx.org/)
- **Glance Dashboard**: [https://github.com/glanceapp/glance](https://github.com/glanceapp/glance)
- **TempMail Service**: [https://github.com/swe-himelrana/tempmail](https://github.com/swe-himelrana/tempmail)
- **Ansible Semaphore**: [https://semaphoreui.com/](https://semaphoreui.com/) | [https://docs.ansible-semaphore.com/](https://docs.ansible-semaphore.com/) | [Docker Hub](https://hub.docker.com/r/semaphoreui/semaphore) - Modern UI for Ansible, Terraform, OpenTofu, and Terragrunt

### Docker Images Used

- `jenkins/jenkins:latest-jdk25` - Jenkins CI/CD server
- `kanboard/kanboard:latest` - Kanban board
- `excalidraw/excalidraw:latest` - Excalidraw whiteboard
- `jgraph/drawio:latest` - Draw.io diagrams
- `linuxserver/wireguard:latest` - WireGuard VPN server
- `strm/dnsmasq:latest` - DNS server
- `nginx:alpine` - Reverse proxy
- `ghcr.io/swe-himelrana/glance-dashboard:latest` - Startpage dashboard
- `ghcr.io/swe-himelrana/tempmail:latest` - Temp mail service
- `semaphoreui/semaphore:latest` - Semaphore UI for Ansible, Terraform, OpenTofu, and Terragrunt

## Additional Documentation

- [SSL Certificate Setup](SSL_CERTIFICATE.md) - Detailed SSL/TLS configuration guide
- [VPN IP Service Setup](VPN_IP_SETUP.md) - VPN IP configuration guide

## Support

For issues, questions, or contributions:

- Open an issue on GitHub
- Contact: contact[at]himelrana.com

---

**Note**: This setup is designed for development/internal use. For production deployments, consider additional production-grade database, security hardening (although current security is not bad at all), monitoring, and backup strategies.
