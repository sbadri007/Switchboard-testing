# Switchboard-testing
Testing of switchboard logic, and cross communication across multiple registry networks
# NANDA Index Federation Setup

Complete step-by-step guide to deploy NANDA Index + AGNTCY ADS with working federation.

---

## Prerequisites

- 2 Linode servers (Ubuntu 22.04)
- Domain name (optional, for SSL)
- Basic Linux knowledge

---

## Part 1: Deploy NANDA Index Registry

### Server Setup
```
# Linode specs: 1GB RAM, Ubuntu 22.04
# Note your public IP (example: 45.56.102.83)
# Configure firewall: Ports 22, 6900, 27017 (internal only)
```

### 1. Install MongoDB 7.0
```
ssh root@YOUR_NANDA_IP

# Add MongoDB repository
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  tee /etc/apt/sources.list.d/mongodb-org-7.0.list

apt update
apt install -y mongodb-org

# Start MongoDB
systemctl start mongod
systemctl enable mongod
systemctl status mongod
```

### 2. Configure MongoDB Authentication

# Connect to MongoDB
mongosh

# Create admin user (in MongoDB shell)
use admin
db.createUser({
  user: "nanda_admin",
  pwd: "SecurePassword123!",
  roles: ["userAdminAnyDatabase", "readWriteAnyDatabase"]
})
exit

# Enable authentication
nano /etc/mongod.conf
# Add these lines (remove # before security:):
security:
  authorization: enabled

# Restart MongoDB
systemctl restart mongod
```

### 3. Install NANDA Index
```bash
# Install uv package manager
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

# Clone NANDA Index
cd /opt
git clone https://github.com/projnanda/nanda-index.git
cd nanda-index

# Install dependencies
uv sync
```

### 4. Configure Environment
```bash
# Create .env file
cat > .env << 'EOF'
MONGODB_URI=mongodb://nanda_admin:SecurePassword123!@localhost:27017/nanda?authSource=admin
PORT=6900
CERT_DIR=/root/certificates
ENABLE_FEDERATION=true
AGNTCY_ADS_URL=AGNTCY_IP:8888
EOF

# Replace AGNTCY_IP with your AGNTCY server IP later
```

### 5. Start NANDA Index
```bash
cd /opt/nanda-index

# Load environment
export MONGODB_URI='mongodb://nanda_admin:SecurePassword123!@localhost:27017/nanda?authSource=admin'
export PORT=6900
export ENABLE_FEDERATION=false  # Enable after AGNTCY is ready

# Start service
uv run python3 registry.py

# Should see:
# * Running on http://0.0.0.0:6900
```

### 6. Test NANDA Registry
```bash
# From your local machine
curl http://YOUR_NANDA_IP:6900/health
# Expected: {"status":"ok","mongo":true}

# Register test agent
curl -X POST http://YOUR_NANDA_IP:6900/register \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "test-agent-001",
    "agent_url": "http://example.com",
    "api_url": "http://example.com/api",
    "name": "Test Agent",
    "capabilities": ["testing"]
  }'

# List agents
curl http://YOUR_NANDA_IP:6900/list
# Expected: {"test-agent-001":"http://example.com"}
```

---

## Part 2: Deploy AGNTCY ADS

### Server Setup
```bash
# Linode specs: 4GB RAM, Ubuntu 22.04
# Note your public IP (example: 66.175.212.73)
# Configure firewall: Ports 22, 8888, 5555
```

### 1. Install Docker
```bash
ssh root@YOUR_AGNTCY_IP

apt update
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
systemctl enable docker
systemctl start docker

# Verify
docker --version
docker compose version
```

### 2. Clone and Deploy AGNTCY
```bash
cd /opt
git clone https://github.com/agntcy/dir.git
cd dir

# Checkout v0.6.0 (important for version compatibility)
git checkout v0.6.0

# Deploy with Docker Compose
cd install/docker
docker compose up -d

# Check status
docker compose ps
# Should see: dir-apiserver-1 (running), dir-zot-1 (running)
```

### 3. Install dirctl CLI

**On your local Mac (Apple Silicon):**
```bash
curl -LO https://github.com/agntcy/dir/releases/download/v0.6.0/dirctl-darwin-arm64
sudo mv dirctl-darwin-arm64 /usr/local/bin/dirctl
sudo chmod +x /usr/local/bin/dirctl
dirctl version
```

**On your local Mac (Intel):**
```bash
curl -LO https://github.com/agntcy/dir/releases/download/v0.6.0/dirctl-darwin-amd64
sudo mv dirctl-darwin-amd64 /usr/local/bin/dirctl
sudo chmod +x /usr/local/bin/dirctl
dirctl version
```

**On AGNTCY Linux server:**
```bash
cd /usr/local/bin
curl -LO https://github.com/agntcy/dir/releases/download/v0.6.0/dirctl-linux-amd64
mv dirctl-linux-amd64 dirctl
chmod +x dirctl
dirctl version
```

### 4. Register Test Agent in AGNTCY
```bash
# Create OASF format agent (on your local machine)
cat > /tmp/test-vision-agent.json << 'EOF'
{
  "name": "test-vision-agent",
  "version": "v1.0.0",
  "description": "Test computer vision agent for federation demo",
  "schema_version": "0.7.0",
  "skills": [
    {
      "id": 201,
      "name": "images_computer_vision/image_segmentation"
    }
  ],
  "authors": ["NANDA Team"],
  "created_at": "2026-01-12T00:00:00Z",
  "locators": [
    {
      "type": "source_code",
      "url": "https://github.com/example/vision-agent"
    },
    {
      "type": "docker_image",
      "url": "docker.io/example/vision-agent:v1.0.0"
    }
  ]
}
EOF

# Push to AGNTCY (replace with your AGNTCY IP)
dirctl --server-addr YOUR_AGNTCY_IP:8888 push /tmp/test-vision-agent.json

# Should output: Pushed record with CID: baeareih...

# Verify
dirctl --server-addr YOUR_AGNTCY_IP:8888 search --name test-vision-agent
# Should output: Record CIDs found: [baeareih...]
```

---

## Part 3: Enable Federation

### 1. Upgrade NANDA SDK to v0.6.0
```bash
# On NANDA server
ssh root@YOUR_NANDA_IP
cd /opt/nanda-index

# Upgrade SDK
uv remove agntcy-dir
uv add "agntcy-dir>=0.5.0"
# Should install v0.6.0

# Verify
ls .venv/lib/python3.13/site-packages/ | grep agntcy
```

### 2. Update Adapter Code

**File:** `/opt/nanda-index/switchboard/adapters/agntcy_adapter.py`

**Find this section (around line 95-110):**
```python
# OLD CODE - DOESN'T WORK
search_query = search_v1.RecordQuery(
    type=search_v1.RecordQueryType.RECORD_QUERY_TYPE_NAME,
    value=agent_name
)

search_request = search_v1.SearchRequest(
    queries=[search_query],
    limit=1
)

search_result_list = await asyncio.to_thread(
    self.client.search, 
    search_request
)
```

**Replace with:**
```python
# NEW CODE - v0.6.0 API
search_query = search_v1.RecordQuery(
    type=search_v1.RecordQueryType.RECORD_QUERY_TYPE_NAME,
    value=agent_name
)

search_request = search_v1.SearchCIDsRequest(
    queries=[search_query],
    limit=1
)

search_result_list = await asyncio.to_thread(
    self.client.search_cids, 
    search_request
)
```

**Key changes:**
- `SearchRequest` → `SearchCIDsRequest`
- `self.client.search` → `self.client.search_cids`

**To make the change:**
```bash
cd /opt/nanda-index/switchboard/adapters

# Backup original
cp agntcy_adapter.py agntcy_adapter.py.backup

# Edit the file
nano agntcy_adapter.py
# Make the changes above, save with Ctrl+X, Y, Enter
```

### 3. Update NANDA Environment
```bash
cd /opt/nanda-index

# Edit .env file
nano .env

# Update this line (replace with your actual AGNTCY IP):
AGNTCY_ADS_URL=YOUR_AGNTCY_IP:8888

# Save and exit
```

### 4. Restart NANDA with Federation
```bash
cd /opt/nanda-index

# Stop current process if running
pkill -f "python3 registry.py"

# Load environment
export MONGODB_URI='mongodb://nanda_admin:SecurePassword123!@localhost:27017/nanda?authSource=admin'
export ENABLE_FEDERATION=true
export AGNTCY_ADS_URL="YOUR_AGNTCY_IP:8888"

# Start service
uv run python3 registry.py

# Look for these success messages:
# ✅ AGNTCY SDK Client initialized at YOUR_AGNTCY_IP:8888
# [Switchboard] ✅ AGNTCY adapter initialized: YOUR_AGNTCY_IP:8888
# [Switchboard] ✅ Switchboard enabled
```

---

## Part 4: Testing & Validation

### Test 1: Check Connected Registries
```bash
curl http://YOUR_NANDA_IP:6900/switchboard/registries

# Expected output:
{
  "count": 2,
  "registries": [
    {"registry_id": "nanda", "status": "active", "type": "local"},
    {"registry_id": "agntcy", "status": "active", "server_address": "YOUR_AGNTCY_IP:8888"}
  ]
}
```

### Test 2: Lookup NANDA Agent via Switchboard
```bash
curl "http://YOUR_NANDA_IP:6900/switchboard/lookup/test-agent-001"

# Expected: Returns agent data in NANDA format
```

### Test 3: Lookup AGNTCY Agent via Switchboard (Cross-Registry!)
```bash
curl "http://YOUR_NANDA_IP:6900/switchboard/lookup/@agntcy:test-vision-agent"

# Expected output (OASF → NANDA translated):
{
  "agent_id": "@agntcy:test-vision-agent",
  "agent_name": "test-vision-agent",
  "registry_id": "agntcy",
  "capabilities": ["image_segmentation"],
  "agent_url": "https://github.com/example/vision-agent",
  "schema_version": "nanda-v1",
  "source_schema": "oasf",
  ...
}
```

### Test 4: Search Across Both Registries
```bash
# Search in NANDA
curl "http://YOUR_NANDA_IP:6900/search?q=test"

# Should return: test-agent-001

# Search via dirctl in AGNTCY
dirctl --server-addr YOUR_AGNTCY_IP:8888 search --name "test-*"

# Should return: test-vision-agent CID
```

---

## Complete Test Script

Save this as `test-federation.sh`:

```bash
#!/bin/bash

NANDA_IP="YOUR_NANDA_IP"
AGNTCY_IP="YOUR_AGNTCY_IP"

echo "=== Testing NANDA Index Federation ==="

echo -e "\n1. Testing NANDA Registry health..."
curl http://$NANDA_IP:6900/health

echo -e "\n\n2. Checking connected registries..."
curl http://$NANDA_IP:6900/switchboard/registries

echo -e "\n\n3. Looking up NANDA agent..."
curl "http://$NANDA_IP:6900/switchboard/lookup/test-agent-001"

echo -e "\n\n4. Looking up AGNTCY agent (cross-registry)..."
curl "http://$NANDA_IP:6900/switchboard/lookup/@agntcy:test-vision-agent"

echo -e "\n\n=== Tests Complete ==="
```

Run it:
```bash
chmod +x test-federation.sh
./test-federation.sh
```

---

## Creating More Test Agents

### NANDA Agent
```bash
curl -X POST http://YOUR_NANDA_IP:6900/register \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "my-custom-agent",
    "agent_url": "http://myagent.com",
    "api_url": "http://myagent.com/api",
    "name": "My Custom Agent",
    "capabilities": ["custom_capability"]
  }'
```

### AGNTCY Agent
```bash
cat > /tmp/my-agent.json << 'EOF'
{
  "name": "my-agntcy-agent",
  "version": "v1.0.0",
  "description": "My custom AGNTCY agent",
  "schema_version": "0.7.0",
  "skills": [
    {"id": 100, "name": "custom/skill"}
  ],
  "authors": ["Your Name"],
  "created_at": "2026-01-15T00:00:00Z",
  "locators": [
    {"type": "source_code", "url": "https://github.com/you/agent"}
  ]
}
EOF

dirctl --server-addr YOUR_AGNTCY_IP:8888 push /tmp/my-agent.json
```

---

## Troubleshooting

### MongoDB Authentication Error
```bash
# Error: Command find requires authentication
# Solution: Check MongoDB URI
export MONGODB_URI='mongodb://nanda_admin:SecurePassword123!@localhost:27017/nanda?authSource=admin'
```

### Switchboard Not Enabled
```bash
# Check logs for: "Switchboard disabled"
# Solution: Set environment variable
export ENABLE_FEDERATION=true
```

### AGNTCY Connection Failed
```bash
# Error: gRPC connection refused
# Check: Is AGNTCY ADS running?
ssh root@YOUR_AGNTCY_IP
docker compose ps

# Should show dir-apiserver-1 as "running"
```

### Version Mismatch Error
```bash
# Error: "unknown method Search"
# Solution: Make sure you're using v0.6.0 for both:

# On NANDA server
cd /opt/nanda-index
uv run pip list | grep agntcy
# Should show: agntcy-dir 0.6.0

# On AGNTCY server
cd /opt/dir
git describe --tags
# Should show: v0.6.0
```

---

## Quick Reference

### NANDA Index
- **URL:** `http://YOUR_NANDA_IP:6900`
- **Health:** `/health`
- **Register:** `POST /register`
- **Lookup:** `GET /lookup/{agent_id}`
- **Switchboard:** `GET /switchboard/lookup/{agent_id}`

### AGNTCY ADS
- **URL:** `YOUR_AGNTCY_IP:8888` (gRPC)
- **Push:** `dirctl --server-addr IP:8888 push agent.json`
- **Search:** `dirctl --server-addr IP:8888 search --name "name"`

### Agent ID Formats
- **NANDA agent:** `my-agent-id`
- **AGNTCY agent:** `@agntcy:agent-name`

---

## Architecture Diagram

```
┌────────────────────────────────────┐
│     Your Application               │
│     (Exchange Agent, etc.)         │
└─────────────┬──────────────────────┘
              │ Query for agents
              ▼
┌────────────────────────────────────┐
│  NANDA Index + Switchboard         │
│  YOUR_NANDA_IP:6900                │
│                                    │
│  Routes queries to registries      │
│  Translates schemas automatically  │
└──────┬──────────────────┬──────────┘
       │                  │
       ▼                  ▼
┌─────────────┐    ┌──────────────┐
│   NANDA     │    │   AGNTCY     │
│  Registry   │    │     ADS      │
│             │    │              │
│  MongoDB    │    │  P2P Network │
│  Local DB   │    │  Docker      │
│             │    │              │
│  Agents:    │    │  Agents:     │
│  • test-001 │    │  • vision    │
└─────────────┘    └──────────────┘
```

---

## Success Criteria

✅ NANDA Index running and accessible  
✅ MongoDB connected with authentication  
✅ AGNTCY ADS deployed and running  
✅ Test agents registered in both registries  
✅ Switchboard shows 2 connected registries  
✅ Cross-registry lookup working  
✅ Schema translation working (OASF → NANDA)

**You're done!** You now have a working federated agent discovery system.

---

## What You Built

You successfully deployed:
- **NANDA Index Registry** - Central agent directory with MongoDB
- **AGNTCY ADS** - External P2P agent network
- **Switchboard** - Cross-registry discovery with automatic schema translation
- **Federation** - Single query searches multiple independent registries

**Next steps:** Register your real agents (MBTA agents, etc.) and integrate with your Exchange Agent for dynamic discovery!
