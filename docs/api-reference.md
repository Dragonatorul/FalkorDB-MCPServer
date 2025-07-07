# API Reference

Complete REST API documentation for FalkorDB MCP Server with request/response examples, authentication details, and error handling.

## Base URL

```
http://localhost:3000
```

## Authentication

All API endpoints (except `/health`) require authentication.

### API Key Authentication

**Header Format:**
```
Authorization: Bearer your-api-key
```

**Example:**
```bash
curl -H "Authorization: Bearer your-api-key" \
     http://localhost:3000/api/mcp/metadata
```

### JWT Bearer Token Authentication

**Header Format:**
```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Token Requirements:**
- Valid JWT token with required claims
- Token must be signed by configured JWKS provider
- Must include `tenant_id` claim for multi-tenant operations

## Core Endpoints

### Health Check

#### `GET /health`

Returns server health status.

**Authentication:** None required

**Response:**
```json
{
  "status": "ok",
  "timestamp": "2025-01-07T20:00:00.000Z",
  "uptime": 3600,
  "database": {
    "connected": true,
    "responseTime": "5ms"
  }
}
```

**Status Codes:**
- `200 OK` - Server is healthy
- `503 Service Unavailable` - Server or database issues

---

### Server Metadata

#### `GET /api/mcp/metadata`

Returns server capabilities and configuration information.

**Authentication:** Required

**Response:**
```json
{
  "provider": "falkordb-mcp-server",
  "version": "1.1.0",
  "capabilities": {
    "graphOperations": true,
    "queryExecution": true,
    "multiTenancy": true,
    "parameterBinding": true,
    "healthChecking": true
  },
  "configuration": {
    "authMode": "bearer",
    "multiTenancy": {
      "enabled": true,
      "graphPrefixing": true,
      "authMode": "bearer"
    }
  },
  "limits": {
    "maxQueryLength": 1000000,
    "maxParameterSize": 10485760,
    "queryTimeout": 30000
  }
}
```

**Status Codes:**
- `200 OK` - Success
- `401 Unauthorized` - Invalid or missing authentication

---

### List Graphs

#### `GET /api/mcp/graphs`

Returns list of available graphs for the authenticated user/tenant.

**Authentication:** Required

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `filter` | string | - | Filter graphs by name pattern |
| `limit` | number | 100 | Maximum number of graphs to return |
| `offset` | number | 0 | Number of graphs to skip |

**Single-Tenant Response:**
```json
{
  "graphs": [
    "memories",
    "analytics",
    "user_data"
  ],
  "total": 3,
  "filter": null,
  "pagination": {
    "limit": 100,
    "offset": 0,
    "hasMore": false
  }
}
```

**Multi-Tenant Response:**
```json
{
  "graphs": [
    "memories",
    "analytics"
  ],
  "total": 2,
  "tenant": "tenant123",
  "filter": null,
  "pagination": {
    "limit": 100,
    "offset": 0,
    "hasMore": false
  }
}
```

**Status Codes:**
- `200 OK` - Success
- `401 Unauthorized` - Invalid authentication
- `403 Forbidden` - Insufficient permissions

---

### Execute Graph Query

#### `POST /api/mcp/context`

Executes a Cypher query against the specified graph.

**Authentication:** Required

**Request Body:**
```json
{
  "graphName": "string",
  "query": "string",
  "parameters": {
    "key": "value"
  }
}
```

**Request Schema:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `graphName` | string | ✅ | Name of the graph to query |
| `query` | string | ✅ | Cypher query to execute |
| `parameters` | object | ❌ | Query parameters for substitution |

**Example Request:**
```json
{
  "graphName": "social_network",
  "query": "CREATE (p:Person {name: $name, age: $age}) RETURN p",
  "parameters": {
    "name": "Alice",
    "age": 30
  }
}
```

**Success Response:**
```json
{
  "success": true,
  "data": [
    {
      "p": {
        "identity": 1,
        "labels": ["Person"],
        "properties": {
          "name": "Alice",
          "age": 30
        }
      }
    }
  ],
  "statistics": {
    "nodesCreated": 1,
    "propertiesSet": 2,
    "labelsAdded": 1,
    "queryExecutionTime": "2.5ms"
  },
  "query": "CREATE (p:Person {name: $name, age: $age}) RETURN p",
  "graphName": "social_network",
  "tenant": "tenant123"
}
```

**Query Response:**
```json
{
  "success": true,
  "data": [
    {
      "name": "Alice",
      "age": 30
    },
    {
      "name": "Bob", 
      "age": 25
    }
  ],
  "statistics": {
    "nodesReturned": 2,
    "queryExecutionTime": "1.2ms"
  },
  "query": "MATCH (p:Person) RETURN p.name as name, p.age as age",
  "graphName": "social_network"
}
```

**Status Codes:**
- `200 OK` - Query executed successfully
- `400 Bad Request` - Invalid query or parameters
- `401 Unauthorized` - Invalid authentication
- `403 Forbidden` - Access denied to graph
- `404 Not Found` - Graph does not exist
- `500 Internal Server Error` - Query execution error

---

## Multi-Tenant Specific Endpoints

### Tenant Graph List

#### `GET /api/mcp/graphs?tenant=true`

Returns tenant-specific graph information with resolved names.

**Authentication:** Required (JWT with tenant context)

**Response:**
```json
{
  "graphs": [
    {
      "name": "memories",
      "resolvedName": "tenant123_memories",
      "tenant": "tenant123"
    },
    {
      "name": "analytics", 
      "resolvedName": "tenant123_analytics",
      "tenant": "tenant123"
    }
  ],
  "total": 2,
  "tenant": "tenant123",
  "tenantInfo": {
    "id": "tenant123",
    "extractedFrom": "jwt.tenant_id"
  }
}
```

## Request/Response Examples

### Creating and Querying Data

**1. Create a Graph with Nodes and Relationships:**

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -d '{
    "graphName": "company",
    "query": "CREATE (c:Company {name: $companyName}) CREATE (p:Person {name: $personName}) CREATE (p)-[:WORKS_FOR]->(c) RETURN c, p",
    "parameters": {
      "companyName": "TechCorp",
      "personName": "John Doe"
    }
  }' \
  http://localhost:3000/api/mcp/context
```

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "c": {
        "identity": 1,
        "labels": ["Company"],
        "properties": {
          "name": "TechCorp"
        }
      },
      "p": {
        "identity": 2,
        "labels": ["Person"],
        "properties": {
          "name": "John Doe"
        }
      }
    }
  ],
  "statistics": {
    "nodesCreated": 2,
    "relationshipsCreated": 1,
    "propertiesSet": 2,
    "labelsAdded": 2,
    "queryExecutionTime": "3.1ms"
  }
}
```

**2. Complex Query with Pattern Matching:**

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -d '{
    "graphName": "company",
    "query": "MATCH (p:Person)-[r:WORKS_FOR]->(c:Company) WHERE c.name = $company RETURN p.name as employee, r, c.name as company",
    "parameters": {
      "company": "TechCorp"
    }
  }' \
  http://localhost:3000/api/mcp/context
```

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "employee": "John Doe",
      "r": {
        "identity": 1,
        "type": "WORKS_FOR",
        "start": 2,
        "end": 1,
        "properties": {}
      },
      "company": "TechCorp"
    }
  ],
  "statistics": {
    "nodesReturned": 2,
    "relationshipsReturned": 1,
    "queryExecutionTime": "1.8ms"
  }
}
```

### Multi-Tenant Operations

**3. Multi-Tenant Graph Creation:**

```bash
# Request with JWT token containing tenant_id: "org_456"
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -d '{
    "graphName": "customer_data",
    "query": "CREATE (c:Customer {id: $customerId, name: $name}) RETURN c",
    "parameters": {
      "customerId": "cust_001",
      "name": "Acme Corp"
    }
  }' \
  http://localhost:3000/api/mcp/context
```

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "c": {
        "identity": 1,
        "labels": ["Customer"],
        "properties": {
          "id": "cust_001",
          "name": "Acme Corp"
        }
      }
    }
  ],
  "statistics": {
    "nodesCreated": 1,
    "propertiesSet": 2,
    "labelsAdded": 1,
    "queryExecutionTime": "2.3ms"
  },
  "tenant": "org_456",
  "resolvedGraphName": "org_456_customer_data"
}
```

## Error Responses

### Error Response Format

```json
{
  "error": {
    "type": "string",
    "message": "string",
    "code": "string",
    "details": {}
  },
  "timestamp": "2025-01-07T20:00:00.000Z",
  "path": "/api/mcp/context",
  "tenant": "string"
}
```

### Common Error Types

**1. Authentication Error (401):**
```json
{
  "error": {
    "type": "AuthenticationError",
    "message": "Invalid or missing API key",
    "code": "INVALID_API_KEY"
  },
  "timestamp": "2025-01-07T20:00:00.000Z",
  "path": "/api/mcp/metadata"
}
```

**2. Validation Error (400):**
```json
{
  "error": {
    "type": "ValidationError",
    "message": "Request validation failed",
    "code": "VALIDATION_FAILED",
    "details": {
      "field": "graphName",
      "message": "Graph name is required",
      "value": null
    }
  },
  "timestamp": "2025-01-07T20:00:00.000Z",
  "path": "/api/mcp/context"
}
```

**3. Query Error (400):**
```json
{
  "error": {
    "type": "QueryError",
    "message": "Invalid Cypher syntax",
    "code": "CYPHER_SYNTAX_ERROR",
    "details": {
      "query": "INVALID CYPHER QUERY",
      "position": {
        "line": 1,
        "column": 1
      }
    }
  },
  "timestamp": "2025-01-07T20:00:00.000Z",
  "path": "/api/mcp/context"
}
```

**4. Database Error (500):**
```json
{
  "error": {
    "type": "DatabaseError", 
    "message": "Failed to connect to FalkorDB",
    "code": "CONNECTION_FAILED",
    "details": {
      "host": "localhost",
      "port": 6379,
      "timeout": 5000
    }
  },
  "timestamp": "2025-01-07T20:00:00.000Z",
  "path": "/api/mcp/context"
}
```

**5. Multi-Tenant Error (403):**
```json
{
  "error": {
    "type": "TenantError",
    "message": "Access denied to graph",
    "code": "TENANT_ACCESS_DENIED",
    "details": {
      "requestedGraph": "other_tenant_data",
      "tenant": "tenant123",
      "reason": "Graph belongs to different tenant"
    }
  },
  "timestamp": "2025-01-07T20:00:00.000Z",
  "path": "/api/mcp/context",
  "tenant": "tenant123"
}
```

## Parameter Binding

### Supported Parameter Types

| Type | Example | Description |
|------|---------|-------------|
| String | `"Alice"` | Text values (automatically quoted) |
| Number | `42`, `3.14` | Integer and floating-point numbers |
| Boolean | `true`, `false` | Boolean values |
| Array | `[1, 2, 3]` | Arrays of primitive types |
| Object | `{"key": "value"}` | JSON objects (converted to strings) |
| Null | `null` | Null values |

### Parameter Examples

**String Parameters:**
```json
{
  "query": "CREATE (p:Person {name: $name}) RETURN p",
  "parameters": {
    "name": "Alice Smith"
  }
}
```

**Number Parameters:**
```json
{
  "query": "MATCH (p:Person) WHERE p.age > $minAge RETURN p",
  "parameters": {
    "minAge": 25
  }
}
```

**Array Parameters:**
```json
{
  "query": "MATCH (p:Person) WHERE p.id IN $ids RETURN p",
  "parameters": {
    "ids": [1, 2, 3, 4, 5]
  }
}
```

**Complex Parameters:**
```json
{
  "query": "CREATE (p:Person {name: $name, metadata: $metadata}) RETURN p",
  "parameters": {
    "name": "Bob",
    "metadata": {
      "department": "Engineering",
      "level": "Senior",
      "skills": ["Python", "JavaScript"]
    }
  }
}
```

## Rate Limits and Constraints

### Default Limits

| Constraint | Default Value | Description |
|------------|---------------|-------------|
| Request Size | 10 MB | Maximum request body size |
| Query Length | 1,000,000 chars | Maximum Cypher query length |
| Parameter Size | 10 MB | Maximum size of parameters object |
| Query Timeout | 30 seconds | Maximum query execution time |
| Concurrent Requests | 100 | Maximum concurrent requests per client |

### Custom Headers

**Request ID Tracking:**
```bash
curl -H "X-Request-ID: req_12345" \
     -H "Authorization: Bearer your-api-key" \
     http://localhost:3000/api/mcp/metadata
```

**Response includes request ID:**
```json
{
  "provider": "falkordb-mcp-server",
  "requestId": "req_12345"
}
```

## SDKs and Client Libraries

### JavaScript/Node.js Example

```javascript
const axios = require('axios');

class FalkorDBMCPClient {
  constructor(baseURL, apiKey) {
    this.client = axios.create({
      baseURL,
      headers: {
        'Authorization': `Bearer ${apiKey}`,
        'Content-Type': 'application/json'
      }
    });
  }

  async query(graphName, query, parameters = {}) {
    const response = await this.client.post('/api/mcp/context', {
      graphName,
      query,
      parameters
    });
    return response.data;
  }

  async listGraphs() {
    const response = await this.client.get('/api/mcp/graphs');
    return response.data;
  }

  async getMetadata() {
    const response = await this.client.get('/api/mcp/metadata');
    return response.data;
  }
}

// Usage
const client = new FalkorDBMCPClient('http://localhost:3000', 'your-api-key');

async function example() {
  // Create a node
  const result = await client.query(
    'social_network',
    'CREATE (p:Person {name: $name}) RETURN p',
    { name: 'Alice' }
  );
  
  console.log('Created:', result.data);
}
```

### Python Example

```python
import requests
import json

class FalkorDBMCPClient:
    def __init__(self, base_url, api_key):
        self.base_url = base_url
        self.headers = {
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json'
        }
    
    def query(self, graph_name, query, parameters=None):
        url = f'{self.base_url}/api/mcp/context'
        data = {
            'graphName': graph_name,
            'query': query,
            'parameters': parameters or {}
        }
        response = requests.post(url, headers=self.headers, json=data)
        return response.json()
    
    def list_graphs(self):
        url = f'{self.base_url}/api/mcp/graphs'
        response = requests.get(url, headers=self.headers)
        return response.json()

# Usage
client = FalkorDBMCPClient('http://localhost:3000', 'your-api-key')

result = client.query(
    'social_network',
    'CREATE (p:Person {name: $name}) RETURN p',
    {'name': 'Alice'}
)

print(f"Created: {result['data']}")
```

---

*This documentation was generated by Claude Code, an AI assistant, to provide comprehensive API reference for the FalkorDB MCP Server.*