# Cogento - AWS Deployment Guide

This guide walks you through deploying Cogento on an AWS EC2 instance with CloudFront for a custom domain endpoint like `https://cogento.edgible.com`.

---

## Prerequisites

- AWS account with EC2, CloudFront, Route 53, and ACM access
- Basic knowledge of AWS EC2, security groups, CloudFront, and Linux
- SSH access to EC2 instances
- Domain name for custom endpoint (e.g., `cogento.edgible.com`)

---

## Step 1: Launch EC2 Instance

### Recommended Instance Types

For Cogento, choose an instance based on your expected load:

| Instance Type | vCPU | RAM | Use Case |
|--------------|------|-----|----------|
| **t3.medium** | 2 | 4 GB | Light usage, development/testing, <50 members |
| **t3.large** | 2 | 8 GB | Small clubs, moderate usage (recommended minimum), 50-200 members |
| **t3.xlarge** | 4 | 16 GB | Production, larger clubs, 200-500 members |
| **m5.large** | 2 | 8 GB | Production with better CPU performance |

**Minimum requirements:**
- **RAM:** 4GB (8GB+ recommended for production)
- **CPU:** 2 vCPUs
- **Storage:** 30GB+ (SSD recommended for database performance)

**Recommended for production:** `t3.large` or `t3.xlarge` depending on member count and usage.

### Launch Instance

1. **Navigate to EC2 Console** → Launch Instance

2. **Configure instance:**
   - **Name:** `cogento-server` (or your preferred name)
   - **AMI:** Amazon Linux 2023 or Ubuntu Server 22.04 LTS (recommended)
   - **Instance type:** `t3.large` or higher
   - **Key pair:** Create or select an existing key pair (download `.pem` file)
   - **Network settings:** 
     - Create security group (see Step 2)
     - Allow SSH (port 22) from your IP
     - Allow HTTP (port 80) and HTTPS (port 443) - will be used by CloudFront/ALB
   - **Storage:** 30-100 GB gp3 SSD (depending on expected data volume)

3. **Launch the instance**

---

## Step 2: Configure Security Group

Create or update the security group with these rules:

| Type | Protocol | Port Range | Source | Description |
|------|----------|------------|--------|-------------|
| SSH | TCP | 22 | Your IP | SSH access |
| HTTP | TCP | 80 | 0.0.0.0/0 | HTTP (for CloudFront/ALB) |
| HTTPS | TCP | 443 | 0.0.0.0/0 | HTTPS (for CloudFront/ALB) |
| Custom TCP | TCP | 8000 | 127.0.0.1/32 | API (localhost only, via ALB) |
| Custom TCP | TCP | 3000 | 127.0.0.1/32 | UI (localhost only, via ALB) |

**Security Best Practices:**
- Restrict SSH access to your IP only
- Ports 8000 and 3000 should only be accessible from localhost (127.0.0.1) or the ALB security group
- For production, use an Application Load Balancer with SSL/TLS termination
- Consider using AWS Systems Manager Session Manager instead of SSH

---

## Step 3: Connect to Instance

SSH into your instance:

```bash
# Replace with your key and instance details
ssh -i /path/to/your-key.pem ec2-user@<instance-public-ip>

# For Ubuntu:
ssh -i /path/to/your-key.pem ubuntu@<instance-public-ip>
```

---

## Step 4: Install Docker and Docker Compose

### For Amazon Linux 2023

```bash
# Update system
sudo yum update -y

# Install Docker
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# Install Docker Compose plugin
sudo yum install docker-compose-plugin -y

# Log out and back in (or use newgrp) for group changes
newgrp docker
```

### For Ubuntu 22.04

```bash
# Update system
sudo apt-get update
sudo apt-get upgrade -y

# Install Docker
sudo apt-get install -y docker.io docker-compose-plugin

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group
sudo usermod -aG docker ubuntu

# Log out and back in (or use newgrp) for group changes
newgrp docker
```

### Verify Installation

```bash
docker --version
docker compose version
docker ps  # Should work without sudo
```

---

## Step 5: Clone and Prepare Cogento

### Clone Repository

```bash
# Install git if not present
sudo yum install git -y  # Amazon Linux
# OR
sudo apt-get install git -y  # Ubuntu

# Clone repository
cd ~
git clone https://github.com/edgible/Cogento.git
cd Cogento
```

### Create Required Directories

```bash
# Create volume directories for PostgreSQL data
mkdir -p ./volumes/postgres/data
mkdir -p ./volumes/pgadmin/data

# Fix pgAdmin permissions (pgAdmin runs as UID 5050)
# This prevents permission errors when pgAdmin tries to create sessions directory
sudo chown -R 5050:5050 ./volumes/pgadmin/data
# Or if using EBS volume:
# sudo chown -R 5050:5050 /mnt/cogento-data/pgadmin

# Set proper permissions
sudo chown -R $USER:$USER ./volumes
```

---

## Step 6: Configure Environment Variables

### Create `.env` File

Docker Compose automatically reads variables from a `.env` file in the same directory as `docker-compose.yml`.

```bash
cd ~/Cogento
nano .env
# OR
vi .env
```

**Add your configuration:**

```bash
# PostgreSQL Configuration
POSTGRES_USER=cogento
POSTGRES_PASSWORD=your-secure-password-here
POSTGRES_PORT=5432

# Shared Superuser (optional - for multi-tenant management)
SHARED_SUPERUSER_EMAIL=admin@yourdomain.com

# pgAdmin Configuration
PGADMIN_EMAIL=admin@yourdomain.com
PGADMIN_PASSWORD=your-pgadmin-password
PGADMIN_PORT=5050

# API Configuration
API_PORT=8000
POSTGRES_HOST=cogento-postgres
POSTGRES_USER=cogento
POSTGRES_PASSWORD=your-secure-password-here

# Stripe Key Encryption (generate a secure key)
STRIPE_KEY_ENCRYPTION_KEY=your-32-character-encryption-key-here

# Email Configuration (for sending login codes, etc.)
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password
MAIL_FROM=noreply@yourdomain.com
MAIL_FROM_NAME=Cogento
MAIL_PORT=587
MAIL_SERVER=smtp.gmail.com
MAIL_STARTTLS=true

# UI Configuration
UI_PORT=3000
UI_BASE_URL=https://cogento.edgible.com  # Your CloudFront domain
```

**Generate secure keys:**

```bash
# Generate PostgreSQL password
openssl rand -base64 32

# Generate Stripe encryption key (32 characters)
openssl rand -hex 16
```

**Secure the .env file:**

```bash
chmod 600 .env
```

---

## Step 7: Build and Start Services

### Build Images

```bash
cd ~/Cogento
docker compose build
```

### Start Services

```bash
docker compose up -d
```

### Check Logs

```bash
# Follow logs
docker compose logs -f

# Check specific service
docker compose logs -f cogento-api
docker compose logs -f cogento-ui

# Check container status
docker compose ps

# Check if services are responding
curl http://localhost:8000/health
curl http://localhost:3000
```

---

## Step 8: Simplest Option - CloudFront Direct to EC2 (No Nginx, No ALB)

**This is the simplest option!** CloudFront terminates SSL, so your EC2 instance only needs to serve HTTP. No Nginx or ALB required.

### Update Security Group

Your EC2 security group needs to allow:
1. **HTTP (port 3000)** from CloudFront (for UI)
2. **HTTP (port 5050)** from your IP or CloudFront (for pgAdmin, optional)

1. **EC2 Console** → Security Groups → Select your instance's security group
2. **Edit inbound rules:**
   
   **Rule 1: UI (required)**
   - **Type:** Custom TCP
   - **Port:** 3000
   - **Source:** 
     - Option A: `0.0.0.0/0` (allows from anywhere - CloudFront will still protect it)
     - Option B: Restrict to CloudFront IP ranges (more secure, see below)
   - **Description:** "Cogento UI for CloudFront"
   
   **Rule 2: pgAdmin (optional)**
   - **Type:** Custom TCP
   - **Port:** 5050
   - **Source:** 
     - **Recommended:** Your IP address (e.g., `123.45.67.89/32`) for security
     - Or `0.0.0.0/0` if you want to access via CloudFront (see Step 10 for pgAdmin CloudFront setup)
   - **Description:** "pgAdmin database management"

**Optional - Restrict to CloudFront IPs only:**
- CloudFront publishes IP ranges: https://d7uri8nf7uskq.cloudfront.net/tools/list-cloudfront-ips
- You can use AWS WAF or restrict security group to CloudFront IPs for extra security
- For simplicity, `0.0.0.0/0` is fine since CloudFront will be the only public entry point

### Get EC2 Instance Public IP or Elastic IP

1. **EC2 Console** → Instances → Select your instance
2. Copy the **Public IPv4 address** (e.g., `54.123.45.67`)
3. **Recommended:** Allocate an **Elastic IP** so the IP doesn't change:
   - EC2 Console → Elastic IPs → Allocate Elastic IP address
   - Actions → Associate Elastic IP address → Select your instance
   - Note the Elastic IP address

### Create Route 53 Record for Origin Hostname

**Important:** CloudFront requires a hostname (not an IP address) for the origin. Create a subdomain that points to your EC2 instance:

1. **Route 53 Console** → Hosted zones → Select `edgible.com`
2. **Create record:**
   - **Record name:** `cogento-origin` (creates `cogento-origin.edgible.com`)
   - **Record type:** A
   - **Alias:** No (disabled)
   - **Value:** Your EC2's **Elastic IP** or **Public IP** (e.g., `54.66.23.34`)
   - **TTL:** 300 (5 minutes) or 60 (1 minute for testing)
   - **Create record**

**Note:** This hostname is only used internally by CloudFront to reach your EC2 instance. Users will still access `https://cogento.edgible.com`.

**That's it!** No Nginx, no ALB setup needed. Proceed to Step 9 for SSL certificate and Step 10 for CloudFront setup.

---

## Step 8 (Alternative): Set Up Nginx Reverse Proxy

**Note:** Only needed if you want SSL termination on the EC2 instance itself. For the simplest setup, skip this and use CloudFront direct to EC2 (Step 8 above).

**Note:** You can skip the ALB setup and use a simpler Nginx reverse proxy on the EC2 instance. This saves costs but requires managing SSL on the instance.

### Install Nginx

```bash
# Amazon Linux
sudo yum install nginx -y

# Ubuntu
sudo apt-get install nginx -y
```

### Configure Nginx

```bash
sudo nano /etc/nginx/conf.d/cogento.conf
```

Add configuration:

```nginx
server {
    listen 80;
    server_name cogento.edgible.com;

    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name cogento.edgible.com;

    # SSL certificates (will be configured with Certbot)
    ssl_certificate /etc/letsencrypt/live/cogento.edgible.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cogento.edgible.com/privkey.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Proxy to UI
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Proxy API requests
    location /api {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Install Certbot for SSL

```bash
# Amazon Linux
sudo yum install certbot python3-certbot-nginx -y

# Ubuntu
sudo apt-get install certbot python3-certbot-nginx -y

# Request certificate
sudo certbot --nginx -d cogento.edgible.com
```

### Start Nginx

```bash
sudo systemctl start nginx
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

### Update Security Group

Update EC2 security group to allow:
- **HTTP (80)** from anywhere (for Let's Encrypt validation)
- **HTTPS (443)** from anywhere (or restrict to CloudFront IPs if preferred)

---

## Step 8 (Alternative): Set Up Application Load Balancer (ALB)

**Note:** ALB is optional but recommended for production. It provides better health checks, SSL termination, and scalability. You can skip this if using Nginx reverse proxy (Step 8 above).

### Create Application Load Balancer

1. **Navigate to EC2 Console** → Load Balancers → Create Load Balancer
2. **Choose Application Load Balancer**
3. **Configure:**
   - **Name:** `cogento-alb`
   - **Scheme:** Internet-facing
   - **IP address type:** IPv4
   - **VPC:** Select your EC2 instance's VPC
   - **Availability Zones:** Select at least 2 zones
   - **Security group:** Create new or use existing (allow HTTP 80 and HTTPS 443)

### Create Target Group

1. **Target Groups** → Create target group
2. **Configure:**
   - **Target type:** Instances
   - **Name:** `cogento-targets`
   - **Protocol:** HTTP
   - **Port:** 3000 (UI port) or 80 (if using Nginx)
   - **VPC:** Same as ALB
   - **Health check:**
     - **Path:** `/`
     - **Protocol:** HTTP
     - **Port:** 3000 or 80
     - **Healthy threshold:** 2
     - **Unhealthy threshold:** 2
     - **Timeout:** 5 seconds
     - **Interval:** 30 seconds

3. **Register targets:**
   - Select your EC2 instance
   - Port: 3000 or 80
   - Click "Include as pending below"

### Configure ALB Listeners

1. **Select your ALB** → Listeners tab
2. **Add listener:**
   - **Protocol:** HTTPS
   - **Port:** 443
   - **Default action:** Forward to `cogento-targets`
   - **SSL certificate:** Select or request ACM certificate (see Step 9)

3. **Add HTTP listener:**
   - **Protocol:** HTTP
   - **Port:** 80
   - **Default action:** Redirect to HTTPS (port 443)

---

## Step 9: Set Up SSL Certificate (ACM)

### Request Certificate

1. **Navigate to ACM Console** → Request certificate
   - **Important:** Request the certificate in the `us-east-1` region (required for CloudFront)
2. **Configure:**
   - **Domain name:** `cogento.edgible.com`
   - **Validation method:** DNS validation (recommended)
3. **DNS Validation:**
   
   **If using Route 53 (your case):**
   - ACM will automatically detect your Route 53 hosted zone
   - Click **"Create record in Route 53"** button - this automatically adds the validation record
   - Wait for validation (usually 2-5 minutes)
   
   **If using external DNS provider:**
   - Copy the CNAME record name and value from ACM
   - Add it to your DNS provider
   - Wait for validation (usually 5-30 minutes)

**Note:** The certificate will be used by CloudFront (Step 10), not ALB. If you're using the simplest setup (CloudFront direct to EC2), you don't need to attach the certificate to anything else.

### Attach Certificate to ALB (Only if using ALB)

If you set up an ALB in Step 8:
1. **Select your ALB** → Listeners → Edit HTTPS listener
2. **Select certificate** from ACM
3. **Save**

---

## Step 10: Set Up CloudFront Distribution

### Create CloudFront Distribution

1. **Navigate to CloudFront Console** → Create distribution
2. **Configure origin:**
   
   **Option A: Simplest - Direct to EC2 on HTTP (Recommended)**
   - **Origin domain:** `cogento-origin.edgible.com` (the Route 53 A record you created in Step 8)
     - **Note:** CloudFront requires a hostname, not an IP address. Use the subdomain that points to your EC2 IP.
   - **Origin path:** Leave empty
   - **Name:** Auto-generated
   - **Origin protocol policy:** **HTTP Only** (CloudFront will terminate SSL)
   - **HTTP port:** 
     - Expand the **"Protocol (custom origins only)"** section
     - Set **HTTP port:** `3000` (defaults to 80, change to 3000 for your UI port)
     - Leave HTTPS port as default (443) - not used since protocol is HTTP Only
   - **Note:** This is the simplest option - no Nginx or ALB needed!
   
   **Important:** Users will access `https://cogento.edgible.com` on port **443** (standard HTTPS). The port 3000 is only used internally between CloudFront and your EC2 instance. Users never need to specify a port in the URL.
   
   **Option B: Using ALB (if you set up ALB in Step 8)**
   - **Origin domain:** Select your ALB (e.g., `cogento-alb-123456789.us-east-1.elb.amazonaws.com`)
   - **Origin path:** Leave empty
   - **Name:** Auto-generated
   - **Origin protocol policy:** HTTPS Only
   - **Origin SSL protocols:** TLSv1.2
   
   **Option C: Direct to EC2 with Nginx (if you set up Nginx in Step 8)**
   - **Origin domain:** `cogento-origin.edgible.com` (the Route 53 A record pointing to your EC2)
   - **Origin path:** Leave empty
   - **Name:** Auto-generated
   - **Origin protocol policy:** HTTPS Only (Nginx terminates SSL)
   - **Origin SSL protocols:** TLSv1.2
   - **HTTPS port:** 443

3. **Configure default cache behavior:**
   - **Viewer protocol policy:** Redirect HTTP to HTTPS
   - **Allowed HTTP methods:** GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
   - **Cache policy:** CachingDisabled (for dynamic content)
   - **Origin request policy:** AllViewer (forward all headers and query strings)

4. **Configure distribution settings:**
   - **Price class:** Use all edge locations (or select based on your needs)
   - **Alternate domain names (CNAMEs):** `cogento.edgible.com`
   - **SSL certificate:** Select your ACM certificate (must be in `us-east-1` for CloudFront)
   - **Default root object:** Leave empty
   - **Custom SSL client support:** TLSv1.2 and newer (recommended)

5. **Create distribution**

### Update Route 53 DNS

Since you're using Route 53 for `edgible.com`, this is straightforward:

1. **Navigate to Route 53 Console** → Hosted zones → Select `edgible.com`
2. **Create record:**
   - **Record name:** `cogento` (creates `cogento.edgible.com`)
   - **Record type:** A
   - **Alias:** Yes (enabled by default for Route 53)
   - **Route traffic to:** 
     - **Alias to CloudFront distribution**
     - **Choose distribution:** Select your CloudFront distribution from the dropdown
   - **Routing policy:** Simple routing
   - **Evaluate target health:** No (optional)
   - **Create record**

**Note:** 
- Route 53 will automatically resolve to the CloudFront distribution's IP addresses
- DNS propagation typically takes a few minutes
- You can verify with: `dig cogento.edgible.com` or `nslookup cogento.edgible.com`

**If using external DNS provider (not Route 53):**
- Create a CNAME record: `cogento.edgible.com` → `d1234567890.cloudfront.net` (your CloudFront domain)

---

## Step 11: Update UI Base URL

Update the `.env` file to use your CloudFront domain:

```bash
cd ~/Cogento
nano .env
```

Update:
```bash
UI_BASE_URL=https://cogento.edgible.com
```

Restart services:
```bash
docker compose down
docker compose up -d
```

---

## Step 12: Set Up Persistent Storage (EBS)

For production, attach a dedicated EBS volume for database persistence:

1. **Create EBS Volume:**
   - EC2 → Volumes → Create Volume
   - Size: 50GB+ (based on expected data)
   - Volume type: gp3 (SSD)
   - Availability Zone: Same as your EC2 instance

2. **Attach Volume to Instance:**
   - Select volume → Actions → Attach Volume
   - Select your EC2 instance

3. **Format and Mount Volume:**

```bash
# Find the new volume
lsblk

# Format (replace xvdf with your device name)
sudo mkfs -t ext4 /dev/xvdf

# Create mount point
sudo mkdir /mnt/cogento-data

# Mount volume
sudo mount /dev/xvdf /mnt/cogento-data

# Update fstab for auto-mount on reboot
echo '/dev/xvdf /mnt/cogento-data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab

# Update permissions
sudo chown -R $USER:$USER /mnt/cogento-data

# Fix pgAdmin permissions (pgAdmin container runs as UID 5050)
sudo chown -R 5050:5050 /mnt/cogento-data/pgadmin

# Update docker-compose.yml to use mounted volume
cd ~/Cogento
nano docker-compose.yml
```

Update volumes section in `docker-compose.yml`:

```yaml
volumes:
  - /mnt/cogento-data/postgres:/var/lib/postgresql/data
  - /mnt/cogento-data/pgadmin:/var/lib/pgadmin
```

Create directories:
```bash
sudo mkdir -p /mnt/cogento-data/postgres
sudo mkdir -p /mnt/cogento-data/pgadmin
sudo chown -R $USER:$USER /mnt/cogento-data
```

Restart containers:
```bash
cd ~/Cogento
docker compose down
docker compose up -d
```

---

## Step 13: Set Up Auto-Start on Boot

### Using systemd (Recommended)

Create systemd service:

```bash
sudo nano /etc/systemd/system/cogento.service
```

Add configuration:

```ini
[Unit]
Description=Cogento Docker Compose Application
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ec2-user/Cogento
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
User=ec2-user
Group=docker

[Install]
WantedBy=multi-user.target
```

**For Ubuntu, replace `ec2-user` with `ubuntu`:**

```bash
sudo sed -i 's/ec2-user/ubuntu/g' /etc/systemd/system/cogento.service
```

Enable and start service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable cogento.service
sudo systemctl start cogento.service

# Check status
sudo systemctl status cogento.service
```

---

## Step 14: Access Cogento

### Via CloudFront Domain

Once CloudFront distribution is active (usually 5-15 minutes), access Cogento at:

```
https://cogento.edgible.com
```

### Access pgAdmin

**Option 1: Direct Access (Recommended for Security)**

Access pgAdmin directly via EC2 IP (restrict security group to your IP):

```
http://YOUR_EC2_IP:5050
```

**Option 2: Via CloudFront (Optional)**

If you want to access pgAdmin via HTTPS through CloudFront, you'll need to:
1. Create a second CloudFront distribution for pgAdmin
2. Point it to `cogento-origin.edgible.com:5050` (or create another origin hostname)
3. Create Route 53 record: `pgadmin.edgible.com` → CloudFront distribution

**Note:** For security, it's recommended to access pgAdmin directly via EC2 IP and restrict the security group to your IP address only.

**pgAdmin Login:**
- **Email:** Value from `PGADMIN_EMAIL` in your `.env` file (default: `admin@cogento.com`)
- **Password:** Value from `PGADMIN_PASSWORD` in your `.env` file (default: `admin`)

### Verify Services

```bash
# Check all containers are running
docker compose ps

# Check API health
curl http://localhost:8000/health

# Check UI
curl http://localhost:3000

# Check CloudFront endpoint
curl https://cogento.edgible.com
```

---

## Monitoring and Maintenance

### View Logs

```bash
# Application logs
docker compose logs -f cogento-api
docker compose logs -f cogento-ui

# System logs
sudo journalctl -u cogento.service -f
```

### Check Container Health

```bash
# Container status
docker compose ps

# Resource usage
docker stats

# Health check endpoint
curl http://localhost:8000/health
```

### Backup Data

```bash
# Backup PostgreSQL database
docker exec cogento-postgres pg_dump -U cogento postgres > backup-$(date +%Y%m%d).sql

# Backup volumes
tar -czf cogento-backup-$(date +%Y%m%d).tar.gz \
  /mnt/cogento-data/postgres \
  /mnt/cogento-data/pgadmin

# Upload to S3 (if configured)
aws s3 cp cogento-backup-*.tar.gz s3://your-backup-bucket/
```

### Update Cogento

```bash
cd ~/Cogento

# Pull latest changes
git pull

# Rebuild and restart
docker compose down
docker compose build
docker compose up -d

# Check logs
docker compose logs -f
```

---

## Troubleshooting

### 504 Gateway Timeout Error

If you see a **504 Gateway Timeout** error from CloudFront, it means CloudFront can't connect to your EC2 instance. Check these in order:

#### 1. Verify Services Are Running on EC2

SSH into your EC2 instance and check:

```bash
# Check if Docker containers are running
docker compose ps

# Check if UI is responding locally
curl http://localhost:3000

# Check if API is responding locally
curl http://localhost:8000/health

# Check what's listening on port 3000
sudo ss -tlnp | grep 3000
# Alternative (if ss not available):
sudo lsof -i :3000
# Or check Docker port mapping:
docker compose ps
```

**If services aren't running:**
```bash
cd ~/Cogento
docker compose up -d
docker compose logs -f
```

**If pgAdmin shows permission errors:**
```bash
# Fix pgAdmin directory permissions
sudo chown -R 5050:5050 ./volumes/pgadmin/data
# Or if using EBS volume:
sudo chown -R 5050:5050 /mnt/cogento-data/pgadmin

# Restart pgAdmin
docker compose restart cogento-pgadmin

# Check logs
docker compose logs -f cogento-pgadmin
```

#### 2. Check Security Group Configuration

The security group must allow HTTP (port 3000) from CloudFront:

1. **EC2 Console** → Instances → Select your instance → Security tab
2. **Click on the security group** → Edit inbound rules
3. **Verify you have a rule:**
   - **Type:** Custom TCP
   - **Port:** 3000
   - **Source:** `0.0.0.0/0` (or CloudFront IP ranges)
   - **Description:** "Cogento UI for CloudFront"

**If missing, add the rule:**
- Click "Add rule"
- Type: Custom TCP
- Port: 3000
- Source: `0.0.0.0/0`
- Save rules

#### 3. Verify CloudFront Origin Configuration

1. **CloudFront Console** → Distributions → Select your distribution
2. **Origins tab** → Edit your origin
3. **Verify:**
   - **Origin domain:** `cogento-origin.edgible.com` (hostname, not IP address)
     - CloudFront requires a hostname, not an IP address
     - This should be the Route 53 A record you created pointing to your EC2 IP
   - **HTTP port:** `3000` (not 80)
   - **Origin protocol policy:** HTTP Only

**Common mistakes:**
- Using IP address instead of hostname (CloudFront doesn't allow IPs)
- Using port 80 instead of 3000
- Using HTTPS protocol when EC2 only serves HTTP
- Using wrong hostname or hostname that doesn't resolve to EC2 IP

#### 4. Test Direct Connection to EC2

From your local machine (not from EC2 itself):

```bash
# Replace with your EC2's public IP
curl http://YOUR_EC2_PUBLIC_IP:3000

# Should return HTML (not connection refused or timeout)
```

**If this fails:**
- Security group is blocking (check step 2)
- Services aren't running (check step 1)
- Firewall on EC2 might be blocking (check with `sudo ufw status`)

#### 5. Check CloudFront Distribution Status

1. **CloudFront Console** → Distributions → Select your distribution
2. **Status** should be "Deployed" (not "In Progress")
3. **General tab** → Check **State:** Enabled
4. **Origins tab** → Check origin shows as available

**If distribution is still deploying:**
- Wait 5-15 minutes for CloudFront to propagate changes
- Check CloudWatch metrics for origin errors

#### 6. Check CloudFront Logs and Metrics

1. **CloudFront Console** → Distributions → Select your distribution
2. **Monitoring tab** → Check:
   - **4xx Error Rate** (should be low)
   - **5xx Error Rate** (should be 0)
   - **Origin Latency** (should be reasonable)

3. **Enable CloudFront access logs** (optional, for detailed debugging):
   - General tab → Edit
   - Enable "Standard logging"
   - Create S3 bucket for logs

#### 7. Verify Elastic IP (If Using)

If you're using an Elastic IP, make sure it's associated:

1. **EC2 Console** → Elastic IPs
2. **Verify** your Elastic IP is associated with your instance
3. **In CloudFront**, make sure the origin domain uses the Elastic IP (not the public IP that might change)

#### Quick Fix Checklist

Run through these commands on your EC2 instance:

```bash
# 1. Verify services
docker compose ps
curl http://localhost:3000

# 2. Check security group allows port 3000
# (Do this in AWS Console - EC2 → Security Groups)

# 3. Test from outside (replace with your IP)
# Run this from your local machine:
curl http://YOUR_EC2_PUBLIC_IP:3000

# 4. If all above work, wait for CloudFront to propagate (5-15 min)
# Then test:
curl https://cogento.edgible.com
```

---

## Troubleshooting (General)

### Container Won't Start

1. **Check logs:**
   ```bash
   docker compose logs cogento-api
   docker compose logs cogento-ui
   ```

2. **Verify directories exist:**
   ```bash
   ls -la ~/Cogento/volumes/
   # OR if using EBS:
   ls -la /mnt/cogento-data/
   ```

3. **Check Docker daemon:**
   ```bash
   sudo systemctl status docker
   ```

### Can't Access via CloudFront

1. **Check CloudFront distribution status:**
   - CloudFront Console → Distributions → Your distribution
   - Status should be "Deployed"

2. **Check ALB health:**
   - EC2 Console → Target Groups → Select your target group
   - Verify instance is healthy

3. **Check security groups:**
   - ALB security group should allow HTTP/HTTPS from CloudFront
   - EC2 security group should allow traffic from ALB security group

4. **Check DNS:**
   ```bash
   dig cogento.edgible.com
   # Should resolve to CloudFront distribution
   ```

### Database Connection Issues

1. **Check PostgreSQL is running:**
   ```bash
   docker compose ps cogento-postgres
   docker compose logs cogento-postgres
   ```

2. **Verify credentials in .env:**
   ```bash
   cat .env | grep POSTGRES
   ```

3. **Test connection:**
   ```bash
   docker exec -it cogento-postgres psql -U cogento -d postgres
   ```

### Out of Disk Space

1. **Check disk usage:**
   ```bash
   df -h
   docker system df
   ```

2. **Clean up Docker:**
   ```bash
   docker system prune -a
   ```

3. **Increase EBS volume size:**
   - EC2 Console → Volumes → Select volume → Modify
   - Extend filesystem: `sudo resize2fs /dev/xvdf`

---

## Cost Optimization Tips

1. **Use Reserved Instances** for predictable workloads (save ~40%)
2. **Stop instance when not in use** (development/testing)
3. **Use gp3 EBS volumes** (cheaper than gp2 with better performance)
4. **Enable EBS volume snapshots** for backups
5. **Monitor CloudWatch** for resource usage and optimize instance size
6. **Use CloudFront caching** for static assets (reduces origin load)

---

## Security Recommendations

1. **Restrict Security Groups:**
   - Limit SSH access to your IP only
   - ALB should only allow traffic from CloudFront
   - EC2 should only allow traffic from ALB security group

2. **Use IAM Roles:**
   - Attach IAM role to EC2 instance instead of storing credentials in .env

3. **Enable AWS Systems Manager:**
   - Use Session Manager instead of SSH when possible

4. **Regular Updates:**
   - Keep system and Docker updated
   - Regularly update Cogento codebase

5. **SSL/TLS:**
   - Always use HTTPS in production (CloudFront + ACM certificate)

6. **Secrets Management:**
   - Consider AWS Secrets Manager for sensitive keys
   - Never commit .env file to git

7. **Database Security:**
   - Use strong PostgreSQL passwords
   - Restrict pgAdmin access (only allow from your IP)

---

## Architecture Summary

### Option A: With ALB (Recommended for Production)

```
Internet
   ↓
CloudFront (https://cogento.edgible.com)
   ↓
Application Load Balancer (HTTPS)
   ↓
EC2 Instance (t3.large or larger)
   ├── Docker Compose
   │   ├── cogento-ui (port 3000) → Nginx serving React app
   │   ├── cogento-api (port 8000) → FastAPI backend
   │   ├── cogento-postgres (port 5432) → PostgreSQL database
   │   └── cogento-pgadmin (port 5050) → Database admin (optional)
   └── EBS Volume → Persistent database storage
```

### Option B: Without ALB (Simpler, Lower Cost)

```
Internet
   ↓
CloudFront (https://cogento.edgible.com)
   ↓
EC2 Instance (t3.large or larger)
   ├── Nginx Reverse Proxy (port 443) → SSL termination
   ├── Docker Compose
   │   ├── cogento-ui (port 3000) → Nginx serving React app
   │   ├── cogento-api (port 8000) → FastAPI backend
   │   ├── cogento-postgres (port 5432) → PostgreSQL database
   │   └── cogento-pgadmin (port 5050) → Database admin (optional)
   └── EBS Volume → Persistent database storage
```

**Benefits of skipping ALB:**
- Lower cost (~$16/month savings)
- Simpler setup
- Good for single-instance deployments

**When to use ALB:**
- Multiple instances (high availability)
- Better health checks and auto-scaling
- Easier SSL certificate management
- Production environments with high traffic

---

## Next Steps

- Configure Stripe webhooks to sync customer data
- Set up your first tenant
- Configure email settings for login codes
- Set up automated backups
- Configure CloudWatch monitoring and alarms
- Review [GETTING_STARTED.md](../GETTING_STARTED.md) for application setup
- Review [docs/](.) for detailed architecture documentation

---

## Support

For issues or questions:
- Check [GETTING_STARTED.md](../GETTING_STARTED.md) for application configuration
- Review application logs: `docker compose logs -f`
- Check system logs: `sudo journalctl -u cogento.service`
- Review [TROUBLESHOOTING.md](../api/TROUBLESHOOTING.md) for common issues
