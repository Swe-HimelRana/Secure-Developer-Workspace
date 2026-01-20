# SSL Certificate Setup with mkcert

This guide explains how to set up SSL certificates for your VPN-only services using `mkcert`, a tool for making locally-trusted development certificates.

## Prerequisites

- `mkcert` installed on your local machine
- Access to the server where certificates will be deployed
- WireGuard VPN connection configured

## Step 1: Install mkcert (Local Machine)

**On macOS:**
```bash
brew install mkcert
```

**On Linux:**
```bash
# Download from https://github.com/FiloSottile/mkcert/releases
# Or use package manager
sudo apt install mkcert  # Ubuntu/Debian
```

**On Windows:**
```bash
choco install mkcert
# Or download from GitHub releases
```

## Step 2: Install Local CA (One-time Setup)

**Important:** This installs a Certificate Authority (CA) on your local machine that will be trusted by your browser.

```bash
mkcert -install
```

This creates a local CA and adds it to your system's trust store. You'll see:
```
Created a new local CA at "/Users/yourname/Library/Application Support/mkcert" 
The local CA is now installed in the system trust store! ⚠️
The local CA is now installed in the Firefox trust store! ⚠️
```

## Step 3: Generate Certificates

Navigate to your project directory and create a `certs` folder:

```bash
cd /path/to/myJenkinsSetup
mkdir -p certs
cd certs
```

Generate certificates for all your services:

```bash
mkcert jenkins.hs kanban.hs draw.hs diagram.hs
```

**Output:**
```
Created a new certificate valid for the following names 📜
 - "jenkins.hs"
 - "kanban.hs"
 - "draw.hs"
 - "diagram.hs"

The certificate is at "./jenkins.hs+3.pem" and the key at "./jenkins.hs+3-key.pem" ✅

It will expire on [date] 🗓
```

**Note:** The certificate name includes `+3` because it covers 3 additional domains (4 total: jenkins.hs + 3 others).

## Step 4: Deploy Certificates to Server

Copy the certificates to your server:

```bash
# From your local machine
scp certs/jenkins.hs+3.pem user@your-server:/path/to/app/certs/
scp certs/jenkins.hs+3-key.pem user@your-server:/path/to/app/certs/
```

**Or if you're already on the server:**
```bash
# Create certs directory on server
mkdir -p /root/app/certs

# Copy certificates (adjust path as needed)
# Certificates should be in: /root/app/certs/
```

## Step 5: Verify Docker Compose Configuration

Ensure `docker-compose.vpn.yml` has the certificate volume mounted:

```yaml
nginx:
  volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf:ro
    - ./certs:/etc/nginx/certs:ro  # Certificates directory
  ports:
    - "10.0.0.1:80:80"
    - "10.0.0.1:443:443"  # HTTPS port
```

## Step 6: Verify Nginx Configuration

The `nginx.conf` should reference the certificates:

```nginx
server {
    listen 443 ssl;
    server_name jenkins.hs;
    
    ssl_certificate     /etc/nginx/certs/jenkins.hs+3.pem;
    ssl_certificate_key /etc/nginx/certs/jenkins.hs+3-key.pem;
    # ... rest of config
}
```

## Step 7: Restart Services

On your server:

```bash
cd /root/app
docker compose up -d --force-recreate nginx
```

## Step 8: Install CA Certificate on Client Devices

**Important:** For browsers to trust the certificates, you need to install the local CA certificate on each device that will access the services.

### Find CA Certificate Location

**macOS:**
```bash
mkcert -CAROOT
# Output: /Users/yourname/Library/Application Support/mkcert
# CA files: rootCA.pem and rootCA-key.pem
```

**Linux:**
```bash
mkcert -CAROOT
# Usually: ~/.local/share/mkcert
```

**Windows:**
```bash
mkcert -CAROOT
# Usually: %LOCALAPPDATA%\mkcert
```

### Install CA on macOS

1. Copy `rootCA.pem` to your device
2. Double-click the file
3. Open Keychain Access
4. Find "mkcert" certificate
5. Double-click and set "Trust" to "Always Trust"

### Install CA on Windows

1. Copy `rootCA.pem` to your device
2. Double-click the file
3. Click "Install Certificate"
4. Choose "Local Machine" → "Place all certificates in the following store"
5. Select "Trusted Root Certification Authorities"
6. Click "Finish"

### Install CA on Linux

```bash
# Copy rootCA.pem to device
sudo cp rootCA.pem /usr/local/share/ca-certificates/mkcert.crt
sudo update-ca-certificates
```

### Install CA on Mobile Devices (iOS/Android)

1. Copy `rootCA.pem` to your device
2. **iOS:** Settings → General → VPN & Device Management → Install Profile
3. **Android:** Settings → Security → Install from storage → Select rootCA.pem

## Step 9: Access Services via HTTPS

Once connected to WireGuard VPN and CA certificate installed:

- **Jenkins**: `https://jenkins.hs`
- **Kanban**: `https://kanban.hs`
- **Excalidraw**: `https://draw.hs`
- **Draw.io**: `https://diagram.hs`

**Note:** HTTP requests automatically redirect to HTTPS.

## Troubleshooting

### Certificate Not Trusted

**Symptom:** Browser shows "Not Secure" or certificate warning

**Solution:**
- Ensure CA certificate (`rootCA.pem`) is installed on your device
- Verify CA is in system trust store: `mkcert -install` (if not already done)
- Clear browser cache and restart browser

### Certificate Not Found Error

**Symptom:** Nginx error: `SSL_CTX_use_certificate_file() failed`

**Solution:**
```bash
# Verify certificates exist on server
ls -la /root/app/certs/

# Check file permissions (should be readable)
chmod 644 /root/app/certs/jenkins.hs+3.pem
chmod 600 /root/app/certs/jenkins.hs+3-key.pem

# Verify docker volume mount
docker compose exec nginx ls -la /etc/nginx/certs/
```

### Port 443 Not Accessible

**Symptom:** Cannot connect to HTTPS

**Solution:**
```bash
# Verify port is bound correctly
sudo ss -tulpn | grep :443

# Check docker-compose ports configuration
# Should have: "10.0.0.1:443:443"

# Restart nginx
docker compose restart nginx
```

### Certificate Expired

**Symptom:** Certificate expired error

**Solution:**
```bash
# Check expiration date
openssl x509 -in certs/jenkins.hs+3.pem -noout -dates

# Regenerate certificates
cd certs
mkcert jenkins.hs kanban.hs draw.hs diagram.hs

# Restart nginx
docker compose restart nginx
```

## Certificate Expiration

Certificates generated by mkcert are valid for **up to 825 days** (approximately 2.25 years). The expiration date is shown when generating:

```
It will expire on 20 April 2028 🗓
```

**To renew before expiration:**
1. Regenerate certificates using the same command
2. Copy new certificates to server
3. Restart nginx: `docker compose restart nginx`

## Security Notes

⚠️ **Important Security Considerations:**

1. **Local CA Only:** These certificates are for **development/internal use only**
2. **VPN-Only Access:** Services are only accessible via WireGuard VPN
3. **Private Key Security:** Keep `*-key.pem` files secure and never commit them to git
4. **CA Certificate:** The `rootCA.pem` can sign any certificate - protect it
5. **Not for Production:** Use Let's Encrypt or proper CA certificates for public-facing production services

## File Structure

```
myJenkinsSetup/
├── certs/
│   ├── jenkins.hs+3.pem          # Certificate (public)
│   ├── jenkins.hs+3-key.pem     # Private key (keep secure!)
│   └── rootCA.pem                # CA certificate (for client installation)
├── nginx.conf                    # Nginx config with SSL
└── docker-compose.vpn.yml       # Docker compose with cert volume mount
```

## Additional Resources

- [mkcert GitHub](https://github.com/FiloSottile/mkcert)
- [mkcert Documentation](https://github.com/FiloSottile/mkcert#readme)
- [Nginx SSL Configuration](https://nginx.org/en/docs/http/configuring_https_servers.html)

