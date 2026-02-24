# CST8916-MidtermProject for Group 7

[Presention Video](url)

Technical Report by 

## Section 1: REST and GraphQL for Data Requests and Updates

### 1.1 Overview

The stock market monitoring application requires two distinct categories of data interaction: structured, resource-based operations such as creating portfolios, managing alert configurations, and authenticating users; and flexible, query-heavy reads where clients need to retrieve deeply nested combinations of portfolio positions, market snapshots, and historical price data in a single round-trip. REST and GraphQL address these two categories differently and, as this section will demonstrate, complement each other well when used together.

---

### 1.2 REST API for Stock Market Operations

REST (Representational State Transfer) exposes resources via a fixed set of HTTP endpoints and is well-suited to the predictable, resource-oriented actions in the application. Its cache-friendly GET semantics, universal client support, and straightforward error handling via HTTP status codes make it the natural choice for the following operations:

#### 1.2.1 Key REST Endpoints

| HTTP Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/auth/login` | User authentication — returns JWT |
| GET | `/api/portfolio/{userId}` | Retrieve full portfolio summary |
| POST | `/api/portfolio/{userId}/positions` | Add a new stock position |
| PUT | `/api/portfolio/{userId}/positions/{id}` | Update quantity / cost basis |
| DELETE | `/api/portfolio/{userId}/positions/{id}` | Remove a position |
| POST | `/api/alerts` | Create a price or P&L threshold alert |
| PUT | `/api/alerts/{alertId}` | Update alert threshold or action |
| DELETE | `/api/alerts/{alertId}` | Delete an alert |
| GET | `/api/market/quote/{symbol}` | Snapshot price for a ticker |
| GET | `/api/market/history/{symbol}?range=1m` | Historical OHLCV data |

#### 1.2.2 REST Advantages for This System

- **Caching:** HTTP caching with ETags and Cache-Control headers reduces database load for frequently requested quotes and portfolio snapshots.
- **Simplicity:** Clear resource semantics (GET, POST, PUT, DELETE) map directly to CRUD operations on portfolios and alerts.
- **Interoperability:** REST is universally understood, has mature tooling, and integrates trivially with existing auth middleware (JWT Bearer headers).
- **Scalability:** Straightforward horizontal scaling using standard load balancers — no specialised infrastructure required.

#### 1.2.3 REST Limitations

REST suffers from over-fetching and under-fetching when clients have complex, variable data needs. For example, a dashboard that must simultaneously display a user's portfolio positions enriched with live quotes, sector breakdowns, and 30-day P&L trends would require multiple sequential REST calls. This increases latency and unnecessary data transfer — a problem GraphQL resolves.

---

### 1.3 GraphQL API for Flexible Data Queries

GraphQL allows clients to describe precisely the shape of the data they need in a single request. This is highly valuable in a stock monitoring application where different views (mobile vs. desktop, overview vs. deep-dive) require vastly different data projections.

#### 1.3.1 Example GraphQL Query: Dashboard Load

```graphql
query DashboardData($userId: ID!) {
  portfolio(userId: $userId) {
    totalValue
    dailyGainLoss
    positions {
      symbol
      quantity
      averageCost
      quote { price change changePercent }
      history(range: "7d") { date close }
    }
  }
  alerts(userId: $userId, status: ACTIVE) {
    id symbol condition threshold
  }
}
```

This single query replaces at least four separate REST calls, eliminates over-fetching, and allows the mobile client to request a lightweight subset of the same data by simply omitting the `history` field.

#### 1.3.2 Mutations for Write Operations

GraphQL mutations handle state changes that require complex, related updates. For example, adding a position while simultaneously setting an alert can be expressed in a single mutation that the server executes atomically, reducing the risk of partial failure states that would require client-side rollback logic.

#### 1.3.3 GraphQL Advantages for This System

- **No over-fetching:** Clients fetch exactly the fields required — critical for low-bandwidth mobile users.
- **Single round-trip:** Multiple data sources (positions, quotes, history, alerts) resolved in one network request.
- **Strongly typed schema:** The GraphQL schema acts as a typed contract between client and server, enabling automatic validation and IDE tooling.
- **Client-side caching:** Apollo Client's normalised cache automatically reuses query results across components, reducing redundant network calls.

#### 1.3.4 GraphQL Limitations

GraphQL queries with unbounded depth (e.g., positions → history → dividends → company → ...) can cause N+1 database queries. This must be mitigated with DataLoader batching. Additionally, HTTP-level caching is harder with GraphQL because all queries typically use POST to the same endpoint.

---

### 1.4 REST vs GraphQL: Comparative Summary

| Criterion | REST | GraphQL |
|---|---|---|
| Data Fetching | Fixed endpoint, may over-fetch | Client-defined, exact fields only |
| Multiple Resources | Multiple HTTP requests | Single query across related data |
| HTTP Caching | ✅ Native GET caching | ⚠️ Requires persisted queries or CDN config |
| Type Safety | ⚠️ Via OpenAPI spec only | ✅ Built-in schema validation |
| Tooling Maturity | ✅ Excellent (Postman, Swagger) | ✅ Good (Apollo Studio, GraphiQL) |
| Best Fit | Auth, CRUD, alert management | Dashboard queries, portfolio data |

---

## Section 2: WebSockets for Real-time Communication


## Section 3: Technology Recommendation and Justification


## Section 4: System Architecture Diagram

```mermaid
flowchart LR
    FE[Frontend<br/>Web App]:::frontend
    WS[WebSocket Service<br/>Client Alert & Live Price]:::ws
    BE[REST API<br/>User Auth & Portfolios]:::backend
    MI[Market Ingestion<br/> Webscoket]:::ms
    BUS[(Event Bus<br/>Redis / Kafka)]:::bus
    DB[(Database<br/>Portfolios & Users Auth)]:::database
    MARKET[External Market Data]:::external
    GQL[GraphQL<br/> Portfolios and Queries]:::gra

    FE -- WSS --> WS
    FE -- HTTPS --> BE
    FE --  Graph --> GQL
    GQL -- collects -->BE
    GQL -- collects -->DB

    %%MARKET -- Live Price Stream --> BE
    MARKET -- Streams --> MI
    MI -- Normalize --> BUS
    BE -- Publish Price Events --> BUS
    WS -- Subscribe --> BUS
    BE -- Read / Write --> DB

    classDef frontend stroke:#1E88E5,stroke-width:2px;
    classDef backend stroke:#43A047,stroke-width:2px;
    classDef ws stroke:#0288D1,stroke-width:2px;
    classDef bus stroke:#8E24AA,stroke-width:2px;
    classDef database stroke:#F9A825,stroke-width:2px;
    classDef external stroke:#D81B60,stroke-width:2px;
    classDef gra stroke:#D81B00,stroke-width:2px;
    classDef ms stroke:#D81FF0,stroke-width:2px;

````
