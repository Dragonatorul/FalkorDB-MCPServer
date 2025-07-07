# Basic Usage Examples

This guide provides practical examples for common FalkorDB MCP Server operations, from simple queries to complex graph operations.

## Getting Started

### Prerequisites

- FalkorDB MCP Server running (see [Quick Start Guide](../quick-start.md))
- Valid API key or JWT token for authentication
- Basic understanding of Cypher query language

### Basic Setup

```bash
# Set your API endpoint and key
export MCP_BASE_URL="http://localhost:3000"
export MCP_API_KEY="your-api-key"

# Test server connectivity
curl -H "Authorization: Bearer $MCP_API_KEY" \
     $MCP_BASE_URL/api/mcp/metadata
```

## Simple Graph Operations

### 1. Creating Nodes

**Create a single person:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "CREATE (p:Person {name: $name, age: $age, email: $email}) RETURN p",
    "parameters": {
      "name": "Alice Johnson",
      "age": 30,
      "email": "alice@example.com"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Expected Response:**
```json
{
  "success": true,
  "data": [
    {
      "p": {
        "identity": 1,
        "labels": ["Person"],
        "properties": {
          "name": "Alice Johnson",
          "age": 30,
          "email": "alice@example.com"
        }
      }
    }
  ],
  "statistics": {
    "nodesCreated": 1,
    "propertiesSet": 3,
    "labelsAdded": 1,
    "queryExecutionTime": "2.1ms"
  }
}
```

**Create multiple nodes:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "UNWIND $people AS person CREATE (p:Person {name: person.name, age: person.age}) RETURN p",
    "parameters": {
      "people": [
        {"name": "Bob Smith", "age": 25},
        {"name": "Carol Davis", "age": 35},
        {"name": "David Wilson", "age": 28}
      ]
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 2. Creating Relationships

**Connect people with friendships:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (a:Person {name: $person1}), (b:Person {name: $person2}) CREATE (a)-[r:FRIENDS_WITH {since: $since}]->(b) RETURN a, r, b",
    "parameters": {
      "person1": "Alice Johnson",
      "person2": "Bob Smith",
      "since": "2024-01-15"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Create bidirectional friendship:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (a:Person {name: $person1}), (b:Person {name: $person2}) CREATE (a)-[:FRIENDS_WITH {since: $since}]->(b), (b)-[:FRIENDS_WITH {since: $since}]->(a) RETURN a, b",
    "parameters": {
      "person1": "Alice Johnson",
      "person2": "Carol Davis",
      "since": "2023-06-10"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 3. Querying Data

**Find all people:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person) RETURN p.name as name, p.age as age ORDER BY p.age"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Find friends of a specific person:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person {name: $name})-[:FRIENDS_WITH]->(friend:Person) RETURN friend.name as friendName, friend.age as friendAge",
    "parameters": {
      "name": "Alice Johnson"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Find mutual friends:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (a:Person {name: $person1})-[:FRIENDS_WITH]->(mutual:Person)<-[:FRIENDS_WITH]-(b:Person {name: $person2}) RETURN mutual.name as mutualFriend",
    "parameters": {
      "person1": "Alice Johnson",
      "person2": "David Wilson"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

## Advanced Graph Operations

### 1. Company and Employee Network

**Create a company structure:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "company_org",
    "query": "CREATE (company:Company {name: $companyName, industry: $industry}) CREATE (ceo:Person {name: $ceoName, title: \"CEO\"}) CREATE (cto:Person {name: $ctoName, title: \"CTO\"}) CREATE (dev1:Person {name: $dev1Name, title: \"Senior Developer\"}) CREATE (dev2:Person {name: $dev2Name, title: \"Junior Developer\"}) CREATE (ceo)-[:WORKS_FOR]->(company) CREATE (cto)-[:WORKS_FOR]->(company) CREATE (dev1)-[:WORKS_FOR]->(company) CREATE (dev2)-[:WORKS_FOR]->(company) CREATE (cto)-[:REPORTS_TO]->(ceo) CREATE (dev1)-[:REPORTS_TO]->(cto) CREATE (dev2)-[:REPORTS_TO]->(cto) RETURN company, ceo, cto, dev1, dev2",
    "parameters": {
      "companyName": "TechCorp Solutions",
      "industry": "Software Development",
      "ceoName": "Sarah Chen",
      "ctoName": "Michael Rodriguez",
      "dev1Name": "Emma Thompson",
      "dev2Name": "James Kumar"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Query organizational hierarchy:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "company_org",
    "query": "MATCH (boss:Person)<-[:REPORTS_TO]-(employee:Person) RETURN boss.name as manager, boss.title as managerTitle, collect(employee.name) as directReports ORDER BY boss.title"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 2. Product Recommendation System

**Create products and categories:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "ecommerce",
    "query": "CREATE (electronics:Category {name: \"Electronics\"}) CREATE (books:Category {name: \"Books\"}) CREATE (clothing:Category {name: \"Clothing\"}) CREATE (laptop:Product {name: \"Gaming Laptop\", price: 1299.99}) CREATE (phone:Product {name: \"Smartphone\", price: 799.99}) CREATE (novel:Product {name: \"Sci-Fi Novel\", price: 15.99}) CREATE (tshirt:Product {name: \"Cotton T-Shirt\", price: 24.99}) CREATE (laptop)-[:BELONGS_TO]->(electronics) CREATE (phone)-[:BELONGS_TO]->(electronics) CREATE (novel)-[:BELONGS_TO]->(books) CREATE (tshirt)-[:BELONGS_TO]->(clothing) RETURN electronics, books, clothing, laptop, phone, novel, tshirt"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Add customer purchase history:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "ecommerce",
    "query": "CREATE (customer:Customer {name: $customerName, email: $email}) WITH customer MATCH (product:Product {name: $productName}) CREATE (customer)-[:PURCHASED {date: $purchaseDate, rating: $rating}]->(product) RETURN customer, product",
    "parameters": {
      "customerName": "John Doe",
      "email": "john.doe@email.com",
      "productName": "Gaming Laptop",
      "purchaseDate": "2024-12-15",
      "rating": 5
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Find product recommendations:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "ecommerce",
    "query": "MATCH (customer:Customer {name: $customerName})-[:PURCHASED]->(purchased:Product)-[:BELONGS_TO]->(category:Category)<-[:BELONGS_TO]-(recommended:Product) WHERE NOT (customer)-[:PURCHASED]->(recommended) RETURN DISTINCT recommended.name as recommendation, recommended.price as price, category.name as category",
    "parameters": {
      "customerName": "John Doe"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 3. Knowledge Graph

**Create knowledge entities:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "knowledge_base",
    "query": "CREATE (ai:Topic {name: \"Artificial Intelligence\", type: \"Technology\"}) CREATE (ml:Topic {name: \"Machine Learning\", type: \"Technology\"}) CREATE (python:Topic {name: \"Python\", type: \"Programming Language\"}) CREATE (tensorflow:Topic {name: \"TensorFlow\", type: \"Framework\"}) CREATE (ml)-[:PART_OF]->(ai) CREATE (tensorflow)-[:USED_FOR]->(ml) CREATE (python)-[:IMPLEMENTS]->(tensorflow) RETURN ai, ml, python, tensorflow"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Query knowledge relationships:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "knowledge_base",
    "query": "MATCH path = (start:Topic {name: $startTopic})-[*1..3]-(end:Topic {name: $endTopic}) RETURN path, length(path) as pathLength ORDER BY pathLength LIMIT 5",
    "parameters": {
      "startTopic": "Python",
      "endTopic": "Artificial Intelligence"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

## Graph Analytics Examples

### 1. Centrality Analysis

**Find most connected people (degree centrality):**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person) OPTIONAL MATCH (p)-[r:FRIENDS_WITH]-() RETURN p.name as person, count(r) as connections ORDER BY connections DESC LIMIT 10"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Find shortest path between two people:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH path = shortestPath((start:Person {name: $startPerson})-[:FRIENDS_WITH*]-(end:Person {name: $endPerson})) RETURN [node in nodes(path) | node.name] as pathNames, length(path) as pathLength",
    "parameters": {
      "startPerson": "Alice Johnson",
      "endPerson": "David Wilson"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 2. Aggregation Queries

**Get statistics by age group:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person) WITH CASE WHEN p.age < 25 THEN \"Young\" WHEN p.age < 35 THEN \"Adult\" ELSE \"Senior\" END as ageGroup, p RETURN ageGroup, count(p) as count, avg(p.age) as averageAge ORDER BY ageGroup"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Find popular products by purchase count:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "ecommerce",
    "query": "MATCH (customer:Customer)-[purchase:PURCHASED]->(product:Product) RETURN product.name as productName, count(purchase) as purchaseCount, avg(purchase.rating) as averageRating ORDER BY purchaseCount DESC LIMIT 10"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

## Data Management Examples

### 1. Updating Existing Data

**Update person information:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person {name: $name}) SET p.age = $newAge, p.city = $city RETURN p",
    "parameters": {
      "name": "Alice Johnson",
      "newAge": 31,
      "city": "San Francisco"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Add new properties to existing nodes:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person) WHERE p.email IS NOT NULL SET p.hasEmail = true SET p.contactMethod = \"email\" RETURN count(p) as updatedCount"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 2. Deleting Data

**Delete specific relationships:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (a:Person {name: $person1})-[r:FRIENDS_WITH]-(b:Person {name: $person2}) DELETE r RETURN count(r) as deletedRelationships",
    "parameters": {
      "person1": "Alice Johnson",
      "person2": "Bob Smith"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

**Delete nodes and their relationships:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person {name: $name}) DETACH DELETE p RETURN count(p) as deletedNodes",
    "parameters": {
      "name": "David Wilson"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

## Batch Operations

### 1. Bulk Data Import

**Import CSV-like data:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "company_org",
    "query": "UNWIND $employees AS emp MERGE (p:Person {email: emp.email}) SET p.name = emp.name, p.department = emp.department, p.salary = emp.salary RETURN count(p) as processedEmployees",
    "parameters": {
      "employees": [
        {"name": "Alice Williams", "email": "alice.w@company.com", "department": "Engineering", "salary": 95000},
        {"name": "Bob Johnson", "email": "bob.j@company.com", "department": "Marketing", "salary": 75000},
        {"name": "Carol Smith", "email": "carol.s@company.com", "department": "Sales", "salary": 85000}
      ]
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 2. Complex Aggregation

**Multi-level aggregation query:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "company_org",
    "query": "MATCH (p:Person)-[:WORKS_FOR]->(c:Company) WITH c, p.department as dept, count(p) as employeeCount, avg(p.salary) as avgSalary RETURN c.name as company, dept as department, employeeCount, round(avgSalary) as averageSalary ORDER BY dept, avgSalary DESC"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

## Error Handling Examples

### 1. Handling Missing Data

**Safe query with OPTIONAL MATCH:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person {name: $name}) OPTIONAL MATCH (p)-[:FRIENDS_WITH]->(friend:Person) RETURN p.name as person, collect(friend.name) as friends",
    "parameters": {
      "name": "NonExistent Person"
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 2. Conditional Logic

**Using CASE statements:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "ecommerce",
    "query": "MATCH (p:Product) RETURN p.name as product, p.price as price, CASE WHEN p.price < 50 THEN \"Budget\" WHEN p.price < 200 THEN \"Mid-range\" ELSE \"Premium\" END as priceCategory ORDER BY p.price"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

## Graph Schema Exploration

### 1. Discover Node Types

**Get all node labels:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "CALL db.labels() YIELD label RETURN label ORDER BY label"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 2. Discover Relationship Types

**Get all relationship types:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "CALL db.relationshipTypes() YIELD relationshipType RETURN relationshipType ORDER BY relationshipType"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 3. Schema Overview

**Get graph schema summary:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (n) WITH labels(n) as nodeLabels, count(n) as nodeCount UNWIND nodeLabels as label RETURN label, sum(nodeCount) as totalNodes ORDER BY totalNodes DESC"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

## Performance Tips

### 1. Use Indexes for Better Performance

**Create an index (if FalkorDB supports it):**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "CREATE INDEX ON :Person(name)"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 2. Limit Large Result Sets

**Use LIMIT and SKIP for pagination:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "MATCH (p:Person) RETURN p.name as name, p.age as age ORDER BY p.name SKIP $skip LIMIT $limit",
    "parameters": {
      "skip": 0,
      "limit": 10
    }
  }' \
  $MCP_BASE_URL/api/mcp/context
```

### 3. Profile Query Performance

**Use PROFILE to analyze query performance:**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MCP_API_KEY" \
  -d '{
    "graphName": "social_network",
    "query": "PROFILE MATCH (p:Person)-[:FRIENDS_WITH]->(friend:Person) RETURN p.name, count(friend) as friendCount ORDER BY friendCount DESC LIMIT 5"
  }' \
  $MCP_BASE_URL/api/mcp/context
```

These examples demonstrate the core functionality of FalkorDB MCP Server and provide a foundation for building more complex graph-based applications.

---

*This documentation was generated by Claude Code, an AI assistant, to provide comprehensive usage examples for the FalkorDB MCP Server.*