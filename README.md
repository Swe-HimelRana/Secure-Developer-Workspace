# Secure Developer Workspace

A comprehensive, VPN-only development environment featuring Jenkins CI/CD, Kanban project management, collaborative whiteboarding, and diagramming tools - all accessible securely through WireGuard VPN.

## Why This Application?

This setup provides a **secure, self-hosted development environment** for teams that need:

- 🔒 **Maximum Security**: All services accessible only via WireGuard VPN - no public internet exposure
- 🚀 **CI/CD Pipeline**: Jenkins with Docker agent for building and testing applications
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
- ✅ **WireGuard VPN** for secure access
- ✅ **Kanban Board** for project management
- ✅ **Excalidraw** for collaborative whiteboarding
- ✅ **Draw.io** for technical diagrams
- ✅ **Nginx Reverse Proxy** with SSL/TLS support
- ✅ **Custom DNS** resolution via dnsmasq
- ✅ **VPN-Only Access** - services not exposed to public internet
- ✅ **Split Tunneling** - only service traffic routes through VPN

## Prerequisites

- A VPS running Linux (Ubuntu/Debian recommended)
- Docker and Docker Compose installed
- Root or sudo access
- Basic knowledge of Linux commands
- WireGuard client installed on your devices

## Quick Start

### 1. Clone or Download This Repository

```bash
git clone <repository-url>
cd myJenkinsSetup
```

### 2. Set Up VPN IP Service

The setup requires the VPN IP (`10.0.0.1`) to be available on the host. See [VPN IP Service Setup](#vpn-ip-service-setup) below.

### 3. Configure Environment Variables

Create a `.env` file:

```bash
cp env.example .env  # Copy example file
# Or create manually:
nano .env
```

Add your configuration:

```bash
JENKINS_SECRET=your_secure_secret_here
WIREGUARD_SERVER_URL=auto  # or your VPS public IP
WIREGUARD_PEERS=1  # Number of WireGuard client configs to generate
```

### 4. Start Services

**Start all services:**
```bash
docker compose -f docker-compose.vpn.yml up -d
```

**Or start individual services:**

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

# Start Nginx reverse proxy
docker compose -f docker-compose.vpn.yml up -d nginx
```

**Verify services are running:**
```bash
docker compose -f docker-compose.vpn.yml ps
```

You should see all services with status "Up" or "Healthy".

### 5. Access Services

1. **Connect to WireGuard VPN** (see [WireGuard Setup](#wireguard-setup))
2. Access services via:
   - **Jenkins**: `https://jenkins.hs`
   - **Kanban**: `https://kanban.hs`
   - **Excalidraw**: `https://draw.hs`
   - **Draw.io**: `https://diagram.hs`

## Detailed Setup Guide

### VPN IP Service Setup

Docker containers need the VPN IP (`10.0.0.1`) to be available on the host before starting.

**Quick setup:**

```bash
sudo ip addr add 10.0.0.1/32 dev lo
```

**Make it persistent:**

```bash
sudo tee /etc/systemd/system/vpn-ip.service >/dev/null <<'EOF'
[Unit]
Description=Add VPN service IP 10.0.0.1 to loopback for Docker binds
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/ip addr add 10.0.0.1/32 dev lo
ExecStart=/sbin/ip link set lo up
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now vpn-ip.service
```

For detailed instructions, see the [VPN IP Service Setup Guide](VPN_IP_SETUP.md).

### WireGuard Setup

1. **Set up DNS port forwarding** (required to avoid port 53 conflict with systemd-resolved):
   
   dnsmasq listens on port 5353, but WireGuard clients need to use standard port 53. Set up iptables port forwarding:
   
   ```bash
   # Redirect DNS requests from 10.0.0.1:53 to 10.0.0.1:5353
   sudo iptables -t nat -A PREROUTING -d 10.0.0.1 -p udp --dport 53 -j REDIRECT --to-port 5353
   sudo iptables -t nat -A PREROUTING -d 10.0.0.1 -p tcp --dport 53 -j REDIRECT --to-port 5353
   
   # Make it persistent (Ubuntu/Debian)
   sudo apt-get install -y iptables-persistent
   sudo netfilter-persistent save
   ```
   
   **Why this is needed:**
   - systemd-resolved uses `127.0.0.1:53` (server DNS)
   - dnsmasq uses `10.0.0.1:5353` (VPN DNS, avoids conflict)
   - iptables redirects `10.0.0.1:53` → `10.0.0.1:5353` (transparent to clients)
   - WireGuard clients use `DNS = 10.0.0.1` (standard port 53, works with macOS)

2. **Start WireGuard container:**
   ```bash
   docker compose -f docker-compose.vpn.yml up -d wireguard
   ```

3. **Start dnsmasq:**
   ```bash
   docker compose -f docker-compose.vpn.yml up -d dnsmasq
   ```

4. **Get client configuration:**
   ```bash
   docker compose -f docker-compose.vpn.yml exec wireguard cat /config/peer1/peer1.conf
   ```

5. **Import config to WireGuard client** on your device

6. **Connect to VPN**

7. **Verify DNS resolution:**
   ```bash
   nslookup jenkins.hs 10.0.0.1
   ```

### Jenkins Setup

1. **Access Jenkins:**
   - Connect to WireGuard VPN
   - Open `https://jenkins.hs` in your browser
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

### Starting Individual Services

**Excalidraw (draw.hs):**
```bash
docker compose -f docker-compose.vpn.yml up -d excalidraw
# Access at: https://draw.hs
```

**Draw.io (diagram.hs):**
```bash
docker compose -f docker-compose.vpn.yml up -d drawio
# Access at: https://diagram.hs
```

**Kanban Board:**
```bash
docker compose -f docker-compose.vpn.yml up -d kanban
# Access at: https://kanban.hs
```

**Check service status:**
```bash
# Check if specific service is running
docker compose -f docker-compose.vpn.yml ps jenkins
docker compose -f docker-compose.vpn.yml ps excalidraw
docker compose -f docker-compose.vpn.yml ps drawio
docker compose -f docker-compose.vpn.yml ps kanban

# View logs
docker compose -f docker-compose.vpn.yml logs excalidraw
docker compose -f docker-compose.vpn.yml logs drawio
```

### SSL Certificate Setup

For HTTPS access, you need SSL certificates. See the [SSL Certificate Guide](SSL_CERTIFICATE.md) for detailed instructions.

**Quick summary:**
1. Install `mkcert` on your local machine
2. Generate certificates: `mkcert jenkins.hs kanban.hs draw.hs diagram.hs`
3. Copy certificates to server: `scp certs/*.pem user@server:/path/to/app/certs/`
4. Install CA certificate on your devices
5. Restart nginx: `docker compose restart nginx`

## Configuration

### Service Ports

All services bind to `10.0.0.1` (VPN interface only):

- **Jenkins**: `10.0.0.1:9080` → `https://jenkins.hs`
- **Jenkins Agent**: `10.0.0.1:9500`
- **Kanban**: `10.0.0.1:9090` → `https://kanban.hs`
- **Excalidraw**: `10.0.0.1:8081` → `https://draw.hs`
- **Draw.io**: `10.0.0.1:8082` → `https://diagram.hs`
- **Nginx**: `10.0.0.1:80` (HTTP redirect) and `10.0.0.1:443` (HTTPS)

### Environment Variables

Edit `.env` file:

```bash
# Jenkins Agent Secret (get from Jenkins UI after creating node)
JENKINS_SECRET=your_secret_here

# WireGuard Configuration
WIREGUARD_SERVER_URL=auto  # Auto-detect or set to your VPS IP
WIREGUARD_PEERS=1  # Number of client configs to generate
```

### Customizing Services

You can modify `docker-compose.vpn.yml` to:
- Adjust resource limits (CPU/memory)
- Change port mappings
- Add environment variables
- Modify service configurations

## Adding New Services

You can add additional services to this setup. However, **keep it focused on essential developer tools**. 

**Good additions:**
- Code review tools (e.g., Gitea, GitLab)
- Documentation servers (e.g., Wiki.js, BookStack)
- Monitoring tools (e.g., Grafana, Prometheus)
- Development databases (e.g., PostgreSQL, Redis)

**Avoid:**
- Entertainment services (media servers, game servers)
- Non-development tools
- Services that don't fit the developer workflow

**To add a service:**

1. Add service to `docker-compose.vpn.yml`
2. Add DNS entry in dnsmasq command: `--address=/newservice.hs/10.0.0.1`
3. Add nginx reverse proxy configuration
4. Update this README with the new service

## Troubleshooting

### Services Not Accessible

- **Verify VPN connection**: `ping 10.0.0.1`
- **Check DNS resolution**: `nslookup jenkins.hs 10.0.0.1`
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
docker run --rm -v app_jenkins_home:/data -v $(pwd):/backup alpine tar czf /backup/jenkins-backup.tar.gz /data
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

Created by **Himel Rana**

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

### Docker Images Used

- `jenkins/jenkins:latest-jdk25` - Jenkins CI/CD server
- `kanboard/kanboard:latest` - Kanban board
- `excalidraw/excalidraw:latest` - Excalidraw whiteboard
- `jgraph/drawio:latest` - Draw.io diagrams
- `linuxserver/wireguard:latest` - WireGuard VPN server
- `strm/dnsmasq:latest` - DNS server
- `nginx:alpine` - Reverse proxy

## Additional Documentation

- [SSL Certificate Setup](SSL_CERTIFICATE.md) - Detailed SSL/TLS configuration guide
- [VPN IP Service Setup](VPN_IP_SETUP.md) - VPN IP configuration guide

## Support

For issues, questions, or contributions:
- Open an issue on GitHub
- Contact: contact[at]himelrana.com

---

**Note**: This setup is designed for development/internal use. For production deployments, consider additional security hardening, monitoring, and backup strategies.

