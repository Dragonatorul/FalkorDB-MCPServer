# Quick Start Guide

Get up and running with FalkorDB MCP Server in under 5 minutes.

## Prerequisites

- **FalkorDB Instance**: A running FalkorDB server (local or remote)
- **Docker** (recommended) or **Node.js 16+** for local installation
- **Basic familiarity** with REST APIs and Cypher query language

## Option 1: Docker Deployment (Recommended)

### 1. Start FalkorDB (if needed)

```bash
# Start FalkorDB with Docker
docker run -d \
  --name falkordb \
  -p 6379:6379 \
  falkordb/falkordb:latest
```

### 2. Run FalkorDB MCP Server

```bash
# Run the MCP server
docker run -d \
  --name falkordb-mcp \
  -p 3000:3000 \
  -e FALKORDB_HOST=localhost \
  -e FALKORDB_PORT=6379 \
  -e MCP_API_KEY=your-secure-api-key \
  --network host \
  ghcr.io/dragonatorul/falkordb-mcpserver:1.1.0
```

### 3. Verify Installation

```bash
# Check server health
curl http://localhost:3000/health

# Get server metadata
curl -H "Authorization: Bearer your-secure-api-key" \
     http://localhost:3000/api/mcp/metadata
```

**Expected Response:**
```json
{
  "provider": "falkordb-mcp-server",
  "version": "1.1.0",
  "capabilities": {
    "graphOperations": true,
    "queryExecution": true,
    "multiTenancy": true
  }
}
```

## Option 2: Local Development Setup

### 1. Clone and Install

```bash
# Clone the repository
git clone https://github.com/Dragonatorul/FalkorDB-MCPServer.git
cd FalkorDB-MCPServer

# Install dependencies
npm install

# Build the project
npm run build
```

### 2. Configure Environment

```bash
# Create environment file
cp .env.example .env

# Edit configuration
cat > .env << EOF
NODE_ENV=development
PORT=3000

# FalkorDB Configuration
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
FALKORDB_USERNAME=
FALKORDB_PASSWORD=

# Authentication
MCP_API_KEY=your-secure-api-key

# Multi-tenancy (optional)
ENABLE_MULTI_TENANCY=false
EOF
```

### 3. Start the Server

```bash
# Start in development mode
npm run dev

# Or start in production mode
npm start
```

## Basic Usage Examples

### 1. Create a Graph and Add Data

```bash
# Create a simple graph
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-secure-api-key" \
  -d '{
    "graphName": "social_network",
    "query": "CREATE (alice:Person {name: \"Alice\", age: 30}) RETURN alice"
  }' \
  http://localhost:3000/api/mcp/context
```

### 2. Query Graph Data

```bash
# Find all persons
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-secure-api-key" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person) RETURN p.name, p.age"
  }' \
  http://localhost:3000/api/mcp/context
```

### 3. List Available Graphs

```bash
# Get all graphs
curl -H "Authorization: Bearer your-secure-api-key" \
     http://localhost:3000/api/mcp/graphs
```

**Expected Response:**
```json
{
  "graphs": ["social_network"],
  "total": 1
}
```

## Testing Multi-Tenancy (Optional)

### 1. Enable Multi-Tenancy

```bash
# Update environment configuration
cat >> .env << EOF

# Multi-tenancy Configuration
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true

# JWT Configuration (example - replace with your values)
BEARER_JWKS_URI=https://your-auth-provider/.well-known/jwks.json
BEARER_ISSUER=https://your-auth-provider
BEARER_ALGORITHM=RS256
BEARER_AUDIENCE=falkordb-mcp-server
EOF
```

### 2. Test with JWT Tokens

```bash
# Query with tenant context (requires valid JWT token)
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-jwt-token" \
  -d '{
    "graphName": "memories",
    "query": "CREATE (m:Memory {content: \"Tenant-specific memory\"}) RETURN m"
  }' \
  http://localhost:3000/api/mcp/context
```

This will create a graph named `tenant123_memories` (where `tenant123` is extracted from the JWT token).

## Common Configuration Patterns

### Single-Tenant Setup (Default)

```env
# Basic configuration - no multi-tenancy
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
MCP_API_KEY=your-api-key
ENABLE_MULTI_TENANCY=false
```

### Multi-Tenant Setup

```env
# Multi-tenant configuration
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true

# JWT validation
BEARER_JWKS_URI=https://auth.example.com/.well-known/jwks.json
BEARER_ISSUER=https://auth.example.com
```

### Production Setup

```env
# Production configuration
NODE_ENV=production
FALKORDB_HOST=falkordb.internal.company.com
FALKORDB_PORT=6379
FALKORDB_USERNAME=mcp_user
FALKORDB_PASSWORD=secure_password
MCP_API_KEY=production-secure-api-key

# SSL/TLS (if supported by FalkorDB)
FALKORDB_TLS=true
```

## Verification Checklist

After setup, verify these endpoints work:

- [ ] **Health Check**: `GET /health` returns `200 OK`
- [ ] **Metadata**: `GET /api/mcp/metadata` returns server info
- [ ] **Graph List**: `GET /api/mcp/graphs` returns empty array initially
- [ ] **Query Execution**: `POST /api/mcp/context` can create and query data
- [ ] **Authentication**: Requests without API key return `401 Unauthorized`

## Next Steps

1. **Read the [Architecture Guide](architecture.md)** to understand the system design
2. **Explore [API Reference](api-reference.md)** for complete endpoint documentation
3. **Review [Configuration Guide](configuration.md)** for advanced options
4. **Check [Examples](examples/)** for real-world usage patterns
5. **Set up [Monitoring](monitoring.md)** for production deployments

## Troubleshooting

### Server Won't Start
- Check FalkorDB connection: `telnet falkordb-host 6379`
- Verify environment variables are set correctly
- Check logs: `docker logs falkordb-mcp` or `npm run dev`

### Authentication Errors
- Verify API key is correctly set in environment
- Check Authorization header format: `Bearer your-api-key`
- For JWT tokens, verify JWKS URL is accessible

### Query Failures
- Test FalkorDB connectivity independently
- Verify Cypher query syntax
- Check graph name doesn't contain special characters

For detailed troubleshooting, see [Troubleshooting Guide](troubleshooting.md).