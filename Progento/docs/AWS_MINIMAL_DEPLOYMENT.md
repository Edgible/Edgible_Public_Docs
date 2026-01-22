# AWS Deployment Guide

## EC2 Instance Requirements

**Recommended Instance**: `t3.xlarge`
- **vCPUs**: 4
- **RAM**: 16GB
- **Cost**: ~$0.1664/hour (~$120/month if running 24/7)
- **Storage**: 30GB gp3 (minimum) - ~$3/month
- **Total**: ~$123/month for continuous operation

## Memory Allocation

With the optimized docker-compose configuration:
- **Ollama**: 3GB (phi model ~1.6GB + overhead for inference)
- **Embedding Service**: 1GB
- **Weaviate**: 2GB (optimized for vector search performance)
- **PostgreSQL**: 1GB (optimized for database performance)
- **Progento API**: 2GB (optimized for performance)
- **UI**: 2GB (Vite/esbuild needs significant memory for TypeScript compilation)
- **OS Overhead**: ~2GB
- **Total**: ~11GB (comfortable headroom on 16GB instance)

## Setup Instructions

### 1. Launch EC2 Instance

1. Go to AWS EC2 Console
2. Launch Instance:
   - **AMI**: Amazon Linux 2023 or Ubuntu 22.04 LTS
   - **Instance Type**: t3.xlarge
   - **Storage**: 30GB gp3 (minimum)
   - **Security Group**: 
     - Port 22 (SSH) - Required for initial access
     - Port 3000 (UI) - Optional, for public UI access (can use SSH tunnel instead)
     - Port 8000 (API) - Optional, for public API access (can use SSH tunnel instead)
   - **Key Pair**: Create/download a key pair for SSH access

### 2. Connect and Install Dependencies

**First, determine your OS and default user:**
- **Amazon Linux**: Default user is `ec2-user`
- **Ubuntu**: Default user is `ubuntu`

**For Ubuntu (Recommended):**
```bash
# SSH into your instance
ssh -i your-key.pem ubuntu@your-instance-ip

# Update system
sudo apt update && sudo apt upgrade -y

# Install prerequisites for Docker
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker and Docker Compose V2 (plugin)
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group
sudo usermod -aG docker ubuntu

# Log out and back in for group changes to take effect
exit
```

**For Amazon Linux:**
```bash
# SSH into your instance
ssh -i your-key.pem ec2-user@your-instance-ip

# Update system
sudo yum update -y

# Install Docker (Amazon Linux uses yum)
sudo yum install -y docker

# Install Docker Compose V2 plugin
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group
sudo usermod -aG docker ec2-user

# Log out and back in for group changes to take effect
exit
```

**Verify installation:**
```bash
# SSH back in
ssh -i your-key.pem ubuntu@your-instance-ip  # or ec2-user for Amazon Linux

# Verify Docker and Docker Compose work without sudo
docker ps  # Should work without sudo
docker compose version  # Should show Docker Compose V2
```

### 3. Clone and Deploy Progento

**Option A: Clone using SSH (Recommended if you have SSH keys set up)**

```bash
# SSH back in
ssh -i your-key.pem ubuntu@your-instance-ip  # or ec2-user for Amazon Linux

# Clone using SSH URL (if you have SSH keys configured in GitHub)
git clone git@github.com:Edgible/Progento.git
cd Progento
```

**Option B: Clone using Personal Access Token (PAT)**

1. **Create a Personal Access Token on GitHub:**
   - Go to: https://github.com/settings/tokens/new
   - Click "Generate new token (classic)"
   - Give it a name (e.g., "EC2 Deployment")
   - Set expiration (e.g., 90 days or No expiration for testing)
   - Select scopes: Check **`repo`** (full control of private repositories)
   - Click "Generate token"
   - **Copy the token immediately** (starts with `ghp_`) - you won't see it again!

2. **Clone using the token:**
```bash
# SSH back in
ssh -i your-key.pem ubuntu@your-instance-ip

# Clone using token (replace YOUR_TOKEN with your actual token)
git clone https://YOUR_TOKEN@github.com/Edgible/Progento.git
cd Progento
```

**Deploy Progento:**
```bash
# Use the AWS-optimized docker-compose configuration
cd docker
docker compose -f docker-compose.aws-minimal.yml up -d --build
```

### 4. Initialize Ollama Model

**Important**: You must pull a model before you can use Ollama for queries.

```bash
# Pull the phi model (smallest, ~1.6GB, recommended for testing)
docker exec progento-ollama ollama pull phi

# Verify the model was pulled
docker exec progento-ollama ollama list

# Should show: phi:latest
```

**Note**: This download takes a few minutes (~1.6GB). The model is stored in the `./ollama_data` volume.

### 5. Verify Services

```bash
# Check all services are running
docker compose -f docker-compose.aws-minimal.yml ps

# Check health endpoints
curl http://localhost:8000/api/health
curl http://localhost:8001/health
curl http://localhost:11434/api/tags
```

### 6. Test Progento via Command Line

**Complete end-to-end test:**

This test creates a knowledge base called "PROGENTO" based on `/mnt/edgible/Progento`, scans it, and queries it using the smallest model (phi).

```bash
# 1. Check API health
echo "=== Checking API Health ==="
curl http://localhost:8000/api/health
echo -e "\n"

# 2. Create Progento knowledge base
echo "=== Creating Progento Knowledge Base ==="
REPO_RESPONSE=$(curl -s -X POST http://localhost:8000/api/repos \
  -H "Content-Type: application/json" \
  -d '{
    "name": "PROGENTO",
    "path": "/mnt/edgible/Progento"
  }')

echo "$REPO_RESPONSE" | python3 -m json.tool

# Extract repo_id from response
REPO_ID=$(echo "$REPO_RESPONSE" | python3 -c "import sys, json; print(json.load(sys.stdin)['id'])")
echo "Repository ID: $REPO_ID"
echo -e "\n"

# 3. Scan the knowledge base
echo "=== Starting Scan ==="
curl -X POST http://localhost:8000/api/repos/$REPO_ID/scan \
  -H "Content-Type: application/json"
echo -e "\n\n"

# 4. Monitor scan progress (wait for completion)
echo "=== Monitoring Scan Progress ==="
echo "Waiting for scan to complete (this may take several minutes)..."
while true; do
  STATUS=$(curl -s http://localhost:8000/api/repos/$REPO_ID/status | python3 -c "import sys, json; print(json.load(sys.stdin)['status'])")
  FILES=$(curl -s http://localhost:8000/api/repos/$REPO_ID/status | python3 -c "import sys, json; print(json.load(sys.stdin).get('files_scanned', 0))")
  CHUNKS=$(curl -s http://localhost:8000/api/repos/$REPO_ID/status | python3 -c "import sys, json; print(json.load(sys.stdin).get('chunks_created', 0))")
  
  echo "Status: $STATUS | Files: $FILES | Chunks: $CHUNKS"
  
  if [ "$STATUS" = "completed" ]; then
    echo "Scan completed!"
    break
  elif [ "$STATUS" = "error" ]; then
    echo "Scan failed!"
    break
  fi
  
  sleep 5
done
echo -e "\n"

# 5. Query the knowledge base
echo "=== Querying Knowledge Base ==="
echo "Query: 'What features does Progento have?'"
echo "This may take 30-120 seconds on CPU (using phi model)..."
echo -e "\n"

curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What features does Progento have?",
    "repo_id": "'$REPO_ID'",
    "top_k": 5,
    "model": "phi"
  }' | python3 -m json.tool

echo -e "\n"
echo "=== Test Complete ==="
```

**Notes:**
- The `/mnt/edgible/Progento` path is mounted in the docker-compose.aws-minimal.yml file
- Scan may take 5-15 minutes depending on repository size
- Queries take 30-120 seconds on CPU (phi model is the smallest/fastest)
- The `model: "phi"` parameter ensures the smallest model is used (can be omitted as phi is the default)

### 7. Access the UI

**Option A: SSH Tunneling (More Secure - Recommended for Testing)**

This creates a secure tunnel through SSH, so you don't need to expose the port publicly:

```bash
# Forward both UI (3000) and API (8000) ports
# Amazon Linux
ssh -L 3000:localhost:3000 -L 8000:localhost:8000 -i your-key.pem ec2-user@your-instance-ip

# Ubuntu
ssh -L 3000:localhost:3000 -L 8000:localhost:8000 -i your-key.pem ubuntu@your-instance-ip
```

Then open your browser to: `http://localhost:3000`

**Option B: Public Access (Requires Security Group Configuration)**

To access the UI directly from the internet:

1. **Get your instance's public IP:**
   - In AWS Console: EC2 → Instances → Select your instance
   - Copy the "Public IPv4 address" (e.g., `54.123.45.67`)

2. **Update UI API URL:**
   ```bash
   # Stop UI container
   docker compose -f docker-compose.aws-minimal.yml stop progento-ui
   
   # Edit docker-compose.aws-minimal.yml and change VITE_API_URL:
   # From: VITE_API_URL=http://localhost:8000
   # To: VITE_API_URL=http://YOUR_INSTANCE_IP:8000
   
   # Restart UI (will rebuild with new API URL)
   docker compose -f docker-compose.aws-minimal.yml up -d --build progento-ui
   ```

3. **Configure Security Group:**
   - In AWS Console: EC2 → Instances → Select your instance
   - Click on the "Security" tab
   - Click on the Security Group name (e.g., `sg-xxxxxxxxx`)
   - Click "Edit inbound rules"
   - Click "Add rule"
   - Configure:
     - **Type**: Custom TCP
     - **Port range**: 3000
     - **Source**: 
       - For testing: `0.0.0.0/0` (allows from anywhere - **not secure for production**)
       - For production: Your IP address (e.g., `123.45.67.89/32`)
     - **Description**: "Progento UI"
   - Click "Save rules"
   
   **Also add rule for API (if needed):**
   - Port range: 8000
   - Same source settings
   - Description: "Progento API"

4. **Access the UI:**
   - Open browser to: `http://your-instance-ip:3000`
   - Replace `your-instance-ip` with your actual public IP

**Security Note**: 
- For testing, `0.0.0.0/0` is fine but allows anyone to access your UI
- For production, restrict to your IP address or use SSH tunneling
- Consider setting up authentication/HTTPS for production use

## Performance Expectations

With t3.xlarge and optimized configuration:
- **Startup**: 2-3 minutes for all services
- **Repository Scanning**: Moderate speed (CPU-only, no GPU)
- **Query Response**: 30-120 seconds per query (phi model on CPU, can be longer for complex queries)
- **Embedding Generation**: 1-2 seconds per chunk (CPU-only)

**Timeout Settings:**
- **Ollama Service**: 300 seconds (5 minutes) - sufficient for CPU inference
- **UI Axios**: 600 seconds (10 minutes) - prevents browser timeout
- **FastAPI**: No timeout (relies on service timeouts)

## Cost Optimization Tips

1. **Stop the instance when not testing**: Only pay for compute time
2. **Use Spot Instances**: Can save 50-90% but may be interrupted
3. **Reserved Instances**: If running 24/7, save ~40% with 1-year commitment
4. **Use appropriate storage**: 30GB minimum recommended

## Next Steps

Once you've verified deployment works:
1. **Upgrade to GPU instance** (g4dn.xlarge) for better performance
2. **Use larger models** (llama2:7b, codellama:13b) for better quality
3. **Set up proper security** (HTTPS, authentication, etc.)
4. **Configure backups** for databases and volumes
