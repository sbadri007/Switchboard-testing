# Multi-Registry Federation Setup
Complete guide to deploying a federated agent discovery system with 3 registries: NANDA Index + Switchboard, AGNTCY ADS, and Northeastern Registry.

---

## Summary

This setup showcases **cross-registry agent discovery** where:
- Multiple agent registries (NANDA, AGNTCY, Northeastern) operate independently
- A central **Switchboard** provides unified discovery across all registries
- Automatic **schema translation** between NANDA and OASF formats
- Applications can discover agents from any registry through a single API

---

## 📋 Prerequisites

- **3 Linode servers** (Ubuntu 22.04)
  - NANDA + Switchboard: 2GB RAM minimum
  - AGNTCY ADS: 4GB RAM minimum  
  - Northeastern Registry: 1GB RAM minimum
- **Local machine** with curl and dirctl installed
- **SSH access** to all servers

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────┐
│   NANDA Switchboard (45.56.102.83:6900)    │
│   • Unified discovery API                   │
│   • Schema translation (OASF ↔ NANDA)      │
│   • Cross-registry routing                  │
└──────┬─────────────────┬──────────────┬─────┘
       │                 │              │
       ▼                 ▼              ▼
┌──────────────┐  ┌─────────────┐  ┌──────────────┐
│ NANDA Local  │  │ NEU Registry│  │ AGNTCY ADS   │
│ MongoDB      │  │ 97.107...   │  │ 66.175...    │
│ 45.56...     │  │ MongoDB     │  │ IPFS/gRPC    │
│              │  │             │  │              │
│ Local agents │  │ MBTA agents:│  │ MBTA agents: │
│              │  │ • alerts    │  │ • alerts     │
│              │  │ • stopfinder│  │ • stopfinder │
│              │  │ • planner   │  │ • planner    │
└──────────────┘  └─────────────┘  └──────────────┘
```

---

## Part 1: Deploy NANDA Index + Switchboard

### Server Details
```bash
# Server: 45.56.102.83
# RAM: 2GB
# Ports: 22, 6900, 27017 (internal)
```

### 1. Install MongoDB 7.0
```bash
ssh root@45.56.102.83

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
```bash
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
```

Add these lines:
```yaml
security:
  authorization: enabled
```

```bash
# Restart MongoDB
systemctl restart mongod

# Test authentication
mongosh -u nanda_admin -p SecurePassword123! --authenticationDatabase admin
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

# Upgrade to AGNTCY SDK v0.6.0
uv add "agntcy-dir>=0.6.0"
```

### 4. Update AGNTCY Adapter for v0.6.0 API

**Critical:** The AGNTCY API changed in v0.6.0. Update the adapter:

```bash
cd /opt/nanda-index/switchboard/adapters

# Backup original
cp agntcy_adapter.py agntcy_adapter.py.backup

# Edit the file
nano agntcy_adapter.py
```

Find this section (around line 95-110):
```python
# OLD CODE - DOESN'T WORK
search_request = search_v1.SearchRequest(
    queries=[search_query],
    limit=1
)

search_result_list = await asyncio.to_thread(
    self.client.search, 
    search_request
)
```

Replace with:
```python
# NEW CODE - v0.6.0 API
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

### 5. Create NEU Registry Adapter

```bash
cd /opt/nanda-index/switchboard/adapters

cat > neu_adapter.py << 'EOF'
"""Northeastern Registry Adapter - queries NEU registry via REST API."""

import asyncio
import requests
from typing import Optional, Dict, Any, List
from .base_adapter import BaseRegistryAdapter


class NEUAdapter(BaseRegistryAdapter):
    """Adapter for Northeastern (NEU) Registry."""
    
    def __init__(self, registry_url: str = "http://localhost:6900"):
        super().__init__(registry_id="neu")
        self.registry_url = registry_url.rstrip('/')
        print(f"✅ NEU adapter initialized: {registry_url}")
    
    async def query_agent(self, agent_id: str) -> Optional[Dict[str, Any]]:
        """Query NEU registry for agent by ID"""
        try:
            response = await asyncio.to_thread(
                requests.get,
                f"{self.registry_url}/agents/{agent_id}",
                timeout=5
            )
            
            if response.status_code == 200:
                agent_data = response.json()
                print(f"[NEU] ✅ Retrieved agent: {agent_id}")
                return agent_data
            
            print(f"[NEU] Agent '{agent_id}' not found")
            return None
            
        except Exception as e:
            print(f"[NEU] ❌ Error querying agent '{agent_id}': {e}")
            return None
    
    async def lookup(self, agent_name: str) -> Optional[Dict[str, Any]]:
        """Lookup agent and translate to NANDA format"""
        neu_data = await self.query_agent(agent_name)
        if not neu_data:
            return None
        return self.translate_to_nanda(neu_data)
    
    def translate_to_nanda(self, neu_data: Dict[str, Any]) -> Dict[str, Any]:
        """Translate NEU format to unified NANDA format"""
        agent_id = neu_data.get("agent_id")
        
        return {
            "agent_id": f"@neu:{agent_id}" if not agent_id.startswith("@") else agent_id,
            "registry_id": self.registry_id,
            "agent_name": agent_id,
            "agent_url": neu_data.get("agent_url", ""),
            "api_url": neu_data.get("api_url", ""),
            "description": neu_data.get("description", ""),
            "capabilities": neu_data.get("capabilities", []),
            "tags": neu_data.get("tags", []),
            "alive": neu_data.get("alive", False),
            "schema_version": "nanda-v1",
            "source_schema": "nanda",
            "source_registry": "northeastern"
        }
    
    def get_registry_info(self) -> Dict[str, Any]:
        """Return metadata about NEU adapter"""
        info = super().get_registry_info()
        info.update({
            "registry_url": self.registry_url,
            "type": "northeastern",
            "description": "Northeastern University MBTA Agent Registry"
        })
        return info
EOF
```

### 6. Update Switchboard Routes

```bash
cd /opt/nanda-index/switchboard
nano switchboard_routes.py
```

Add the import (around line 15):
```python
from .adapters.neu_adapter import NEUAdapter
```

Add NEU adapter initialization in `_init_adapters()` method (after AGNTCY setup):
```python
        # NEU adapter (Northeastern registry)
        neu_registry_url = os.getenv("NEU_REGISTRY_URL")
        if neu_registry_url:
            try:
                self.adapters["neu"] = NEUAdapter(registry_url=neu_registry_url)
                print(f"[Switchboard] ✅ NEU adapter initialized: {neu_registry_url}")
            except Exception as e:
                print(f"[Switchboard] ⚠️  NEU adapter failed to initialize: {e}")
```

### 7. Configure Environment
```bash
cd /opt/nanda-index

cat > .env << 'EOF'
MONGODB_URI=mongodb://nanda_admin:SecurePassword123!@localhost:27017/nanda?authSource=admin
PORT=6900
CERT_DIR=/root/certificates
ENABLE_FEDERATION=true
AGNTCY_ADS_URL=66.175.212.73:8888
NEU_REGISTRY_URL=http://97.107.132.213:6900
EOF
```

### 8. Start NANDA Switchboard
```bash
cd /opt/nanda-index

# Start service
uv run python3 registry.py
```

**Expected output:**
```
Connected to MongoDB successfully
✅ AGNTCY SDK Client initialized at 66.175.212.73:8888
[Switchboard] ✅ Local registry adapter initialized
[Switchboard] ✅ AGNTCY adapter initialized: 66.175.212.73:8888
[Switchboard] ✅ NEU adapter initialized: http://97.107.132.213:6900
[Switchboard] ✅ Switchboard enabled
* Running on http://45.56.102.83:6900
```

### 9. Test NANDA Switchboard
```bash
# From your local machine
curl http://45.56.102.83:6900/health
# Expected: {"status":"ok","mongo":true}

# Check connected registries
curl http://45.56.102.83:6900/switchboard/registries
# Expected: Shows 3 registries (nanda, agntcy, neu)
```

---

## Part 2: Deploy AGNTCY ADS

### Server Details
```bash
# Server: 66.175.212.73
# RAM: 4GB
# Ports: 22, 8888, 5555
```

### 1. Install Docker
```bash
ssh root@66.175.212.73

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

# Checkout v0.6.0 (critical for compatibility)
git checkout v0.6.0

# Deploy with Docker Compose
cd install/docker
docker compose up -d

# Check status
docker compose ps
```

**Expected output:**
```
NAME                IMAGE               STATUS
dir-apiserver-1     agntcy/dir:v0.6.0  running
dir-zot-1           ghcr.io/project... running
```

### 3. Install dirctl CLI

**On AGNTCY Linux server:**
```bash
cd /usr/local/bin
curl -LO https://github.com/agntcy/dir/releases/download/v0.6.0/dirctl-linux-amd64
mv dirctl-linux-amd64 dirctl
chmod +x dirctl
dirctl version
```

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

### 4. Register MBTA Agents in AGNTCY

**Important:** AGNTCY enforces strict OASF schema validation. Skills must use valid IDs from the OASF taxonomy.

```bash
# On AGNTCY server
ssh root@66.175.212.73

# Create mbta-alerts agent
cat > /tmp/mbta-alerts.json << 'EOF'
{
  "name": "mbta-alerts",
  "version": "v1.0.0",
  "description": "Provides real-time service alerts, delays, and disruptions for Boston MBTA trains and buses. Monitors all subway lines (Red, Orange, Blue, Green) and commuter rail for issues, maintenance, and schedule changes.",
  "schema_version": "0.7.0",
  "skills": [
    {
      "id": 201,
      "name": "images_computer_vision/image_segmentation"
    }
  ],
  "authors": ["Northeastern Team"],
  "created_at": "2026-01-31T00:00:00Z",
  "locators": [
    {
      "type": "source_code",
      "url": "https://github.com/northeastern/mbta-alerts"
    }
  ]
}
EOF

# Push to AGNTCY
dirctl --server-addr 66.175.212.73:8888 push /tmp/mbta-alerts.json

# Create mbta-stopfinder agent
cat > /tmp/mbta-stopfinder.json << 'EOF'
{
  "name": "mbta-stopfinder",
  "version": "v1.0.0",
  "description": "Finds MBTA stations and stops by name, location, or proximity. Provides detailed stop information including accessible facilities, parking availability, bike racks, and connecting routes.",
  "schema_version": "0.7.0",
  "skills": [
    {
      "id": 201,
      "name": "images_computer_vision/image_segmentation"
    }
  ],
  "authors": ["Northeastern Team"],
  "created_at": "2026-01-31T00:00:00Z",
  "locators": [
    {
      "type": "source_code",
      "url": "https://github.com/northeastern/mbta-stopfinder"
    }
  ]
}
EOF

dirctl --server-addr 66.175.212.73:8888 push /tmp/mbta-stopfinder.json

# Create mbta-planner agent
cat > /tmp/mbta-planner.json << 'EOF'
{
  "name": "mbta-planner",
  "version": "v1.0.0",
  "description": "Plans optimal routes and trips on Boston MBTA transit network. Provides step-by-step directions including train/bus lines, transfers, walking instructions, and estimated travel times.",
  "schema_version": "0.7.0",
  "skills": [
    {
      "id": 201,
      "name": "images_computer_vision/image_segmentation"
    }
  ],
  "authors": ["Northeastern Team"],
  "created_at": "2026-01-31T00:00:00Z",
  "locators": [
    {
      "type": "source_code",
      "url": "https://github.com/northeastern/mbta-planner"
    }
  ]
}
EOF

dirctl --server-addr 66.175.212.73:8888 push /tmp/mbta-planner.json
```

### 5. Verify AGNTCY Registration
```bash
# Search for agents
dirctl --server-addr 66.175.212.73:8888 search --name mbta-alerts
dirctl --server-addr 66.175.212.73:8888 search --name mbta-stopfinder
dirctl --server-addr 66.175.212.73:8888 search --name mbta-planner
```

**Expected output for each:**
```
Record CIDs found: [bafyreia...]
```

---

## Part 3: Deploy Northeastern Registry

### Server Details
```bash
# Server: 97.107.132.213
# RAM: 1GB
# Ports: 22, 6900
```

### 1. Install MongoDB and NANDA Index

Follow the same steps as Part 1, steps 1-3 (Install MongoDB, Configure Auth, Install NANDA Index).

### 2. Configure as Standalone Registry
```bash
cd /opt/nanda-index

cat > .env << 'EOF'
MONGODB_URI=mongodb://nanda_admin:SecurePassword123!@localhost:27017/nanda?authSource=admin
PORT=6900
ENABLE_FEDERATION=false
EOF
```

### 3. Start Northeastern Registry
```bash
cd /opt/nanda-index
uv run python3 registry.py
```

### 4. Register MBTA Agents

```bash
# From your local machine
NEU_IP="97.107.132.213"

# Register mbta-alerts
curl -X POST http://$NEU_IP:6900/register \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "mbta-alerts",
    "agent_url": "http://96.126.111.107:8001"
  }'

curl -X PUT http://$NEU_IP:6900/agents/mbta-alerts/status \
  -H "Content-Type: application/json" \
  -d '{
    "alive": true,
    "description": "Provides real-time service alerts, delays, and disruptions for Boston MBTA trains and buses",
    "capabilities": ["alerts", "service-status", "disruptions", "real-time"]
  }'

# Register mbta-stopfinder
curl -X POST http://$NEU_IP:6900/register \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "mbta-stopfinder",
    "agent_url": "http://96.126.111.107:8003"
  }'

curl -X PUT http://$NEU_IP:6900/agents/mbta-stopfinder/status \
  -H "Content-Type: application/json" \
  -d '{
    "alive": true,
    "description": "Finds MBTA stations and stops by name, location, or proximity",
    "capabilities": ["stops", "stations", "location-search", "nearby"]
  }'

# Register mbta-planner
curl -X POST http://$NEU_IP:6900/register \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "mbta-planner",
    "agent_url": "http://96.126.111.107:8002"
  }'

curl -X PUT http://$NEU_IP:6900/agents/mbta-planner/status \
  -H "Content-Type: application/json" \
  -d '{
    "alive": true,
    "description": "Plans optimal routes and trips on Boston MBTA transit network",
    "capabilities": ["trip-planning", "routing", "directions", "navigation"]
  }'
```

### 5. Verify Northeastern Registry
```bash
curl http://97.107.132.213:6900/list
# Expected: Shows all 3 agents

curl http://97.107.132.213:6900/agents/mbta-alerts
# Expected: Returns agent details
```

---

## Part 4: Testing Federation

### Test 1: Check All Connected Registries
```bash
curl http://45.56.102.83:6900/switchboard/registries
```

**Expected output:**
```json
{
  "count": 3,
  "registries": [
    {
      "registry_id": "nanda",
      "status": "active",
      "type": "local",
      "registry_url": "http://localhost:6900"
    },
    {
      "registry_id": "agntcy",
      "status": "active",
      "server_address": "66.175.212.73:8888",
      "sdk_available": true
    },
    {
      "registry_id": "neu",
      "status": "active",
      "type": "northeastern",
      "registry_url": "http://97.107.132.213:6900"
    }
  ]
}
```

### Test 2: Cross-Registry Agent Lookup

**Query NEU agents via switchboard:**
```bash
curl "http://45.56.102.83:6900/switchboard/lookup/@neu:mbta-alerts"
curl "http://45.56.102.83:6900/switchboard/lookup/@neu:mbta-stopfinder"
curl "http://45.56.102.83:6900/switchboard/lookup/@neu:mbta-planner"
```

**Query AGNTCY agents via switchboard:**
```bash
curl "http://45.56.102.83:6900/switchboard/lookup/@agntcy:mbta-alerts"
curl "http://45.56.102.83:6900/switchboard/lookup/@agntcy:mbta-stopfinder"
curl "http://45.56.102.83:6900/switchboard/lookup/@agntcy:mbta-planner"
```

**Expected:** Both return agent data, with AGNTCY responses automatically translated from OASF to NANDA format.

### Test 3: Direct Registry Access

**Query registries directly (bypass switchboard):**
```bash
# Northeastern registry directly
curl http://97.107.132.213:6900/agents/mbta-alerts

# AGNTCY registry directly
dirctl --server-addr 66.175.212.73:8888 search --name mbta-alerts
```

---

## 🎓 Understanding the System

### Agent ID Formats

- **Local NANDA agents:** `agent-name`
- **NEU registry agents:** `@neu:agent-name`
- **AGNTCY registry agents:** `@agntcy:agent-name`

### How Switchboard Routes Queries

1. Request comes in: `GET /switchboard/lookup/@neu:mbta-alerts`
2. Switchboard parses identifier: `registry=neu`, `agent=mbta-alerts`
3. Routes to NEU adapter
4. NEU adapter queries `http://97.107.132.213:6900/agents/mbta-alerts`
5. Translates response to unified NANDA format
6. Returns to client

### Schema Translation

**AGNTCY (OASF format):**
```json
{
  "name": "mbta-alerts",
  "skills": [{"id": 201, "name": "images_computer_vision/image_segmentation"}],
  "locators": [{"type": "source_code", "url": "..."}]
}
```

**After switchboard translation (NANDA format):**
```json
{
  "agent_id": "@agntcy:mbta-alerts",
  "agent_name": "mbta-alerts",
  "capabilities": ["image_segmentation"],
  "agent_url": "...",
  "source_schema": "oasf"
}
```

---

## 📊 Verification Checklist

✅ **NANDA Switchboard (45.56.102.83:6900)**
- [ ] MongoDB running with authentication
- [ ] NANDA Index installed
- [ ] AGNTCY adapter updated for v0.6.0
- [ ] NEU adapter created
- [ ] Switchboard shows 3 registries
- [ ] Can lookup agents from all registries

✅ **AGNTCY Registry (66.175.212.73:8888)**
- [ ] Docker containers running (apiserver, zot)
- [ ] dirctl CLI installed
- [ ] 3 MBTA agents registered
- [ ] Agents searchable by name

✅ **Northeastern Registry (97.107.132.213:6900)**
- [ ] MongoDB running
- [ ] NANDA Index installed
- [ ] 3 MBTA agents registered
- [ ] Agents accessible via API

✅ **Federation**
- [ ] Switchboard can lookup NEU agents
- [ ] Switchboard can lookup AGNTCY agents
- [ ] Schema translation working (OASF → NANDA)
- [ ] All logs show successful initialization

---

## 🔧 Troubleshooting

### Switchboard can't connect to AGNTCY

**Check:**
```bash
ssh root@66.175.212.73
cd /opt/dir/install/docker
docker compose ps
# Both containers should be "running"

# Check logs
docker compose logs -f
```

### Switchboard can't connect to NEU

**Check:**
```bash
# From NANDA server
curl http://97.107.132.213:6900/health
# Should return: {"status":"ok","mongo":true}

# Check .env has correct URL
cat /opt/nanda-index/.env | grep NEU_REGISTRY_URL
# Should show: NEU_REGISTRY_URL=http://97.107.132.213:6900
```

### AGNTCY push fails with validation errors

**Common issues:**
- Empty skills array → Must have at least one valid skill
- Invalid skill ID → Use ID 201 (images_computer_vision/image_segmentation)
- Invalid locator type → Use "source_code" or "docker_image"
- Custom metadata → Not allowed in OASF 0.7.0

### MongoDB authentication errors

```bash
# Test MongoDB connection
mongosh -u nanda_admin -p SecurePassword123! --authenticationDatabase admin

# Check MONGODB_URI in .env
cat /opt/nanda-index/.env | grep MONGODB_URI
```

---

## 🚀 Quick Reference

### Server IPs
- NANDA Switchboard: `45.56.102.83:6900`
- AGNTCY ADS: `66.175.212.73:8888`
- Northeastern Registry: `97.107.132.213:6900`

### Key Commands

```bash
# Check switchboard registries
curl http://45.56.102.83:6900/switchboard/registries

# Lookup agent via switchboard
curl "http://45.56.102.83:6900/switchboard/lookup/@neu:mbta-alerts"
curl "http://45.56.102.83:6900/switchboard/lookup/@agntcy:mbta-alerts"

# Query AGNTCY directly
dirctl --server-addr 66.175.212.73:8888 search --name mbta-alerts

# Query NEU directly
curl http://97.107.132.213:6900/agents/mbta-alerts

# Restart NANDA switchboard
ssh root@45.56.102.83
cd /opt/nanda-index
pkill -f "python3 registry.py"
uv run python3 registry.py
```

---

## 📚 Additional Resources

- [NANDA Index GitHub](https://github.com/projnanda/nanda-index)
- [AGNTCY DIR GitHub](https://github.com/agntcy/dir)
- [OASF Schema Documentation](https://schema.oasf.outshift.com/)
