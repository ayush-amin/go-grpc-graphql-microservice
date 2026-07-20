# Go gRPC GraphQL Microservice

A microservices-based application demonstrating gRPC inter-service communication with a GraphQL API gateway.

## Architecture

```
                    ┌─────────────────────────────────────┐
                    │          GraphQL Gateway            │
                    │         (Port 8000)                 │
                    └───────────────┬─────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       ▼                       ▼
    ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
    │    Account    │       │    Catalog    │       │     Order     │
    │   Service     │       │   Service     │       │   Service     │
    │  (Port 8080)  │       │  (Port 8080)  │       │  (Port 8080)  │
    └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
            │                       │                       │
            ▼                       ▼                       ▼
    ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
    │  PostgreSQL   │       │ Elasticsearch │       │  PostgreSQL   │
    │  (account_db) │       │ (catalog_db)  │       │  (order_db)   │
    └───────────────┘       └───────────────┘       └───────────────┘
```

## Services

### Account Service

Manages user accounts

| Method                    | Description                   |
| ------------------------- | ----------------------------- |
| `PostAccount(name)`       | Create a new account          |
| `GetAccount(id)`          | Get account by ID             |
| `GetAccounts(skip, take)` | List accounts with pagination |

### Catalog Service

Manages product catalog with full-text search.

| Method                                  | Description                   |
| --------------------------------------- | ----------------------------- |
| `PostProduct(name, description, price)` | Create a new product          |
| `GetProduct(id)`                        | Get product by ID             |
| `GetProducts(skip, take)`               | List products with pagination |
| `GetProductsByIDs(ids)`                 | Get products by multiple IDs  |
| `SearchProducts(query, skip, take)`     | Full-text search              |

### Order Service

Manages orders with cross-service communication.

| Method                           | Description               |
| -------------------------------- | ------------------------- |
| `PostOrder(accountID, products)` | Create an order           |
| `GetOrdersForAccount(accountID)` | Get orders for an account |

## Technologies

| Technology       | Version         | Purpose                     |
| ---------------- | --------------- | --------------------------- |
| Go               | 1.25+           | Primary language            |
| gRPC             | v1.79.1         | Inter-service communication |
| Protocol Buffers | v1.36.11        | Service contracts           |
| GraphQL          | gqlgen v0.17.86 | API Gateway                 |
| PostgreSQL       | 10.3            | Account & Order databases   |
| Elasticsearch    | 6.2.4           | Catalog database (search)   |
| Docker           | Compose v3.7    | Containerization            |

## Project Structure

```
.
├── account/                 # Account microservice
│   ├── account.proto        # gRPC protocol definition
│   ├── up.sql               # Database schema
│   ├── app.dockerfile       # Application Dockerfile
│   ├── db.dockerfile        # Database Dockerfile
│   ├── cmd/account/main.go  # Entry point
│   ├── service.go           # Business logic
│   ├── repository.go        # Data access layer
│   ├── server.go            # gRPC server
│   ├── client.go            # gRPC client
│   └── pb/                  # Generated protobuf files
├── catalog/                 # Catalog microservice
│   └── ...
├── order/                   # Order microservice
│   └── ...
├── graphql/                 # GraphQL Gateway
│   ├── schema.graphql       # GraphQL schema
│   ├── gqlgen.yml           # gqlgen configuration
│   ├── main.go              # Entry point
│   └── *_resolver.go        # GraphQL resolvers
├── docker-compose.yaml      # Docker orchestration
├── go.mod
└── go.sum
```

## Prerequisites

- Go 1.25+
- Docker & Docker Compose
- protobuf compiler (`protoc`)
- Go gRPC plugins:
  ```bash
  go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
  go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
  ```

## Getting Started

### 1. Clone the repository

```bash
git clone git@github.com:ayush-amin/go-grpc-graphql-microservice.git
cd go-grpc-graphql-microservice
```

### 2. Start databases

```bash
docker-compose up -d account_db catalog_db order_db
```

### 3. Generate protobuf code

```bash
cd account
protoc --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  account.proto

cd ../catalog
protoc --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  catalog.proto

cd ../order
protoc --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  order.proto
```

### 4. Run services

```bash
cd account && go run cmd/account/main.go
cd catalog && go run cmd/catalog/main.go
cd order && go run cmd/order/main.go
cd graphql && go run main.go
```

### 5. Access GraphQL Playground

Open http://localhost:8000/playground

## Environment Variables

| Service | Variable              | Description                  |
| ------- | --------------------- | ---------------------------- |
| Account | `DATABASE_URL`        | PostgreSQL connection string |
| Catalog | `DATABASE_URL`        | Elasticsearch URL            |
| Order   | `DATABASE_URL`        | PostgreSQL connection string |
| Order   | `ACCOUNT_SERVICE_URL` | Account gRPC endpoint        |
| Order   | `CATALOG_SERVICE_URL` | Catalog gRPC endpoint        |
| GraphQL | `ACCOUNT_SERVICE_URL` | Account gRPC endpoint        |
| GraphQL | `CATALOG_SERVICE_URL` | Catalog gRPC endpoint        |
| GraphQL | `ORDER_SERVICE_URL`   | Order gRPC endpoint          |

## GraphQL API

### Queries

```graphql
query {
  accounts(pagination: { skip: 0, take: 10 }) {
    id
    name
    orders {
      id
      totalPrice
    }
  }

  products(query: "laptop", pagination: { skip: 0, take: 10 }) {
    id
    name
    description
    price
  }
}
```

### Mutations

```graphql
mutation {
  createAccount(account: { name: "John Doe" }) {
    id
    name
  }

  createProduct(
    product: { name: "Laptop", description: "Gaming laptop", price: 999.99 }
  ) {
    id
    name
  }

  createOrder(
    order: { accountId: "...", products: [{ id: "...", quantity: 2 }] }
  ) {
    id
    totalPrice
    products {
      name
      quantity
      price
    }
  }
}
```
