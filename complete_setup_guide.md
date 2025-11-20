# Complete Pentest Lab Setup Guide

## Prerequisites Checklist
- ✅ Docker Desktop installed and running
- ✅ PowerShell open as Administrator
- ✅ Project directory created: `C:\Windows\system32\security_version2\pen-test_automation`

---

## STEP 1: Install Node.js (for MCP Server)

1. Download Node.js from: https://nodejs.org/
2. Install the LTS version
3. Verify installation:
```powershell
node --version
npm --version
```

---

## STEP 2: Set Up MCP Server

Navigate to your project directory:
```powershell
cd C:\Windows\system32\security_version2\pen-test_automation
```

Create MCP server directory and files:
```powershell
New-Item -ItemType Directory -Path ".\mcp-server" -Force
cd mcp-server
```

Create `package.json` (copy from artifact "MCP Server Package Configuration")

Create `kali-mcp-server.js` (copy from artifact "MCP Server for Kali Container")

Install dependencies:
```powershell
npm install
```

---

## STEP 3: Update Docker Compose

Go back to main directory:
```powershell
cd ..
```

Update your `docker-compose.yml` to include the MCP server:

```yaml
version: '3.8'

services:
  kali:
    image: kalilinux/kali-rolling
    container_name: kali-pentest
    networks:
      - pentest-network
    volumes:
      - ./data:/data
    stdin_open: true
    tty: true
    command: /bin/bash
    restart: unless-stopped

  n8n:
    image: n8nio/n8n
    container_name: n8n-automation
    networks:
      - pentest-network
    ports:
      - "5678:5678"
    volumes:
      - ./n8n:/home/node/.n8n
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=changeme123
      - WEBHOOK_URL=http://localhost:5678/
    restart: unless-stopped
    extra_hosts:
      - "host.docker.internal:host-gateway"

  mcp-server:
    build: ./mcp-server
    container_name: mcp-server
    networks:
      - pentest-network
    ports:
      - "3000:3000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    depends_on:
      - kali

networks:
  pentest-network:
    driver: bridge
```

---

## STEP 4: Create Dockerfile for MCP Server

Create a Dockerfile in the `mcp-server` directory:

```powershell
cd mcp-server
New-Item -ItemType File -Path ".\Dockerfile" -Force
```

Add this content to `Dockerfile`:
```dockerfile
FROM node:18-alpine

# Install Docker CLI to communicate with host Docker
RUN apk add --no-cache docker-cli

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY kali-mcp-server.js ./

EXPOSE 3000

CMD ["node", "kali-mcp-server.js"]
```

Go back to main directory:
```powershell
cd ..
```

---

## STEP 5: Launch Everything

Make sure Docker Desktop is running, then:

```powershell
# Recreate network if needed
docker network create pentest-network

# Build and start all containers
docker-compose up -d --build

# Verify all containers are running
docker ps
```

You should see 3 containers:
- `kali-pentest`
- `n8n-automation`
- `mcp-server`

---

## STEP 6: Install Kali Tools

Access the Kali container and install tools:

```powershell
docker exec -it kali-pentest bash
```

Inside Kali, run:
```bash
apt update
apt install -y kali-linux-headless nmap nikto sqlmap hydra metasploit-framework whois dnsutils curl wget
exit
```

---

## STEP 7: Verify MCP Server

Test the MCP server:

```powershell
# Test if MCP server is responding
curl http://localhost:3000/tools
```

You should see a JSON response with available tools.

Test a simple command:
```powershell
curl -X POST http://localhost:3000/execute `
  -H "Content-Type: application/json" `
  -d '{\"tool\":\"execute_kali_command\",\"arguments\":{\"command\":\"nmap --version\"}}'
```

---

## STEP 8: Set Up n8n Workflows

1. Open browser and go to: `http://localhost:5678`
2. Login with:
   - Username: `admin`
   - Password: `changeme123`

3. Create a new workflow:
   - Click "Create New Workflow"
   - Click the three dots (⋮) in top right
   - Select "Import from File"
   - Import the "n8n Reconnaissance Workflow" JSON
   - Click "Save"
   - Click "Activate" toggle

4. Repeat for "n8n Vulnerability Scanning Workflow"

---

## STEP 9: Test Your Setup

### Test Reconnaissance Workflow:

Get the webhook URL from n8n (click on the Webhook node to see it), then:

```powershell
curl -X POST http://localhost:5678/webhook/start-recon `
  -H "Content-Type: application/json" `
  -d '{\"target\":\"scanme.nmap.org\"}'
```

### Test Vulnerability Scan Workflow:

```powershell
curl -X POST http://localhost:5678/webhook/vuln-scan `
  -H "Content-Type: application/json" `
  -d '{\"target\":\"scanme.nmap.org\",\"port\":80}'
```

---

## STEP 10: Using MCP with Claude

To integrate with Claude (this chat), you would:

1. **Share the MCP server URL** with Claude: `http://localhost:3000`
2. **Claude can then call your tools** through the MCP protocol
3. **Automate pentesting** by asking Claude to:
   - Plan attack strategies
   - Execute scans via MCP
   - Analyze results
   - Generate reports

Example conversation with Claude:
> "Claude, scan scanme.nmap.org using the MCP server and analyze the results for vulnerabilities"

---

## Architecture Overview

```
┌─────────────────┐
│   You (User)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Claude      │◄──── Analyzes results, plans strategy
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   MCP Server    │◄──── Translates commands
│  (localhost:3000)│
└────────┬────────┘
         │
         ├──────────────┐
         ▼              ▼
┌─────────────┐  ┌──────────────┐
│     n8n     │  │ Kali Linux   │
│  Workflows  │  │  Container   │
│(orchestrate)│  │(tools: nmap, │
│             │  │nikto, etc.)  │
└─────────────┘  └──────────────┘
```

---

## Troubleshooting

### Container won't start:
```powershell
docker-compose logs [container-name]
```

### MCP server can't reach Kali:
```powershell
docker network inspect pentest-network
```

### Reset everything:
```powershell
docker-compose down
docker volume prune
docker-compose up -d --build
```

### Check container logs:
```powershell
docker logs kali-pentest
docker logs mcp-server
docker logs n8n-automation
```

---

## Security Notes

⚠️ **IMPORTANT**: This is for learning/testing only!

- Change default n8n password immediately
- Only scan systems you own or have permission to test
- Never use on production networks without authorization
- Keep this lab isolated from your main network
- Use legally and ethically

---

## Next Steps

1. ✅ Get everything running
2. Create custom n8n workflows for specific pentest scenarios
3. Build a report generation workflow
4. Add notification integrations (Discord, Slack, email)
5. Create scheduled scans
6. Build a web dashboard for results

---

## Useful Commands

```powershell
# View all containers
docker ps -a

# Stop everything
docker-compose down

# Start everything
docker-compose up -d

# View logs
docker-compose logs -f

# Access Kali shell
docker exec -it kali-pentest bash

# Restart MCP server
docker-compose restart mcp-server

# Check MCP server status
curl http://localhost:3000/tools
```