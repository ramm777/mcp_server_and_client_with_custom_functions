# MCP Server and Client with Custom Functions

This project sets up an N8N MCP (Model Context Protocol) server with proper SSL certificate handling for local development using Docker and Traefik reverse proxy.

## What This Solves

MCP servers require HTTPS with valid SSL certificates. When running N8N in Docker containers, you can't just use self-signed certificates for `localhost` because:

1. **Container networking** creates domain mismatches
2. **MCP libraries enforce HTTPS** at the protocol level
3. **Certificate validation fails** when containers can't resolve domains properly

## The Solution: Reverse Proxy Architecture

We use **Traefik** as a reverse proxy to solve the SSL certificate challenge:

- **Traefik** handles HTTPS termination on port 443
- **N8N** runs internally on port 5678
- **Same domain and certificate** for both external and internal access
- **No certificate validation mismatches**

## Architecture
Browser/MCP Client → Traefik (port 443) → N8N (port 5678)


↑
SSL Certificate
(mkcert generated)

## Prerequisites

- Docker and Docker Compose
- WSL (Windows Subsystem for Linux)
- mkcert for local certificate generation

## Setup Instructions

### 1. Install mkcert

```bash
# Install mkcert in WSL
sudo apt update
sudo apt install mkcert

# Install the root CA
mkcert -install
```

### 2. Generate Certificates

```bash
# Create certificates directory
mkdir -p certs

# Generate certificates for your domain
mkcert -cert-file ./certs/n8n-demo.local.pem -key-file ./certs/n8n-demo.local-key.pem n8n-demo.local localhost 127.0.0.1
```

### 3. Add Domain to Hosts File

```bash
# Add domain to WSL hosts file
echo "127.0.0.1 n8n-demo.local" | sudo tee -a /etc/hosts
```

### 4. Create Configuration Files

**traefik-tls.yml:**
```yaml
tls:
  certificates:
    - certFile: /certs/n8n-demo.local.pem
      keyFile: /certs/n8n-demo.local-key.pem
  stores:
    default:
      defaultCertificate:
        certFile: /certs/n8n-demo.local.pem
        keyFile: /certs/n8n-demo.local-key.pem
```

**.env:**
```env
DOMAIN_NAME=n8n-demo.local
GENERIC_TIMEZONE=UTC
USERPROFILE=/home/aidan
```

### 5. Start the Services

```bash
# Start all services
docker-compose up -d

# Check if containers are running
docker ps
```

### 6. Access N8N

Open your browser and go to: `https://n8n-demo.local`

## Key Configuration Details

### Why This Works

1. **`extra_hosts`**: Allows N8N to resolve `n8n-demo.local` internally
2. **`NODE_EXTRA_CA_CERTS`**: Tells Node.js to trust the mkcert CA
3. **mkcert CA mount**: Makes the certificate authority available inside the container
4. **MCP-specific routing**: Disables gzip compression for Server-Sent Events
5. **Same domain everywhere**: Eliminates certificate validation mismatches

### Critical Environment Variables

```yaml
environment:
  - N8N_FEATURE_FLAG_MCP=true          # Enable MCP support
  - NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/mkcert/rootCA.pem  # Trust mkcert CA
  - N8N_HOST=${DOMAIN_NAME}            # Use consistent domain
  - N8N_PROTOCOL=https                 # Force HTTPS
```

### MCP-Specific Traefik Configuration

```yaml
# Disable gzip for MCP endpoints (required for SSE)
- traefik.http.middlewares.nogzip.headers.customResponseHeaders.Content-Encoding=""
- traefik.http.routers.n8n_mcp.rule=Host(`${DOMAIN_NAME}`) && PathPrefix(`/mcp`)
- traefik.http.routers.n8n_mcp.middlewares=nogzip
```

## Using the MCP Server

### 1. Activate Your MCP Workflow

1. Go to `https://n8n-demo.local`
2. Find your MCP workflow (e.g., "Build an MCP Server with Google Calendar and Custom Functions")
3. Click the toggle switch to **activate** the workflow
4. The workflow should show as "Active"

### 2. MCP Endpoint URLs

- **Production URL**: `https://n8n-demo.local/mcp/my-functions/sse`
- **Test URL**: `https://n8n-demo.local/mcp-test/my-functions/sse` (temporary, one-time use)

### 3. Connect MCP Client Tools

Use the **Production URL** in your MCP Client Tool:

# Create certificates directory
mkdir -p certs

# Generate certificates for your domain
mkcert -cert-file ./certs/n8n-demo.local.pem -key-file ./certs/n8n-demo.local-key.pem n8n-demo.local localhost 127.0.0.1erify domain is in hosts file: `cat /etc/hosts | grep n8n-demo`

2. **"Bad Gateway" errors**
   - Check container logs: `docker logs mcp_server_and_client_with_custom_functions_n8n_1`
   - Ensure no port conflicts (N8N should run on 5678, Traefik on 443)

3. **Certificate validation errors**
   - Regenerate certificates: `mkcert -cert-file ./certs/n8n-demo.local.pem -key-file ./certs/n8n-demo.local-key.pem n8n-demo.local localhost 127.0.0.1`
   - Restart containers: `docker-compose down && docker-compose up -d`

### Testing the Setup

```bash
# Test the MCP endpoint
curl -k https://n8n-demo.local/mcp/my-functions/sse

# Should return SSE data like:
# event: endpoint
# data: /mcp/my-functions/messages?sessionId=...
```

# Add Domain to Hosts File

echo "127.0.0.1 n8n-demo.local" | sudo tee -a /etc/hostsmental challenge:

- **MCP requires HTTPS** with valid certificates
- **Docker networking** creates domain resolution issues
- **Self-signed certificates** don't work across container boundaries
- **Direct port access** bypasses proper SSL termination

The reverse proxy solution is the **only reliable approach** that:
- Provides consistent domain resolution
- Handles SSL termination properly
- Eliminates certificate validation mismatches
- Works with MCP Client Tools

## Credits

This setup is based on the solution described in "Running N8N with MCP Servers: The Poorly Documented SSL Certificate Challenge" and adapted for WSL/Windows environments.

# Create Configuration Files
- traefik-tls.yml
- .env

# Start the Services

docker-compose up -d

docker ps

# Access N8N
Open your browser and go to: https://n8n-demo.local

# Why This Architecture is Necessary
- MCP servers in Docker environments face a fundamental challenge:
- MCP requires HTTPS with valid certificates
- Docker networking creates domain resolution issues
- Self-signed certificates don't work across container boundaries
- Direct port access bypasses proper SSL termination
- The reverse proxy solution is the only reliable approach that:
- Provides consistent domain resolution
- Handles SSL termination properly
- Eliminates certificate validation mismatches
- Works with MCP Client Tools

# Credits
## Credits

This setup is based on the solution described in ["Running N8N with MCP Servers: The Poorly Documented SSL Certificate Challenge"](https://medium.com/@gilad437/running-n8n-with-mcp-servers-the-poorly-documented-ssl-certificate-challenge-f0aa7961a3b0) and adapted for WSL/Windows environments from the MacOS. 
