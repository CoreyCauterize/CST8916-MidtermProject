# CST8916-MidtermProject for Group 7

# Use case: Real-Time Stock Market Monitoring Application

[Presention Video](url)

## Group Members 
- **Corey Markstewart**
- **Faiza Boudehane**
- **Idris Jovial Sop Nwabo**

## Section 1: REST and GraphQL for Data Requests and Updates

### 1.1 Overview

The Real-time stock market monitoring application requires two distinct categories of data interaction: 
- Resource-based operations such as creating portfolios, managing alert configurations, and authenticating users.
- Flexible, query-heavy reads where clients need to retrieve deeply nested combinations of portfolio positions, market snapshots, and historical price data in a single round-trip.
REST and GraphQL address these two categories differently and, as this section will demonstrate, complement each other well when used together.

### 1.2 REST API for Stock Market Operations

REST is ideal for **stable, resource-bases operations** where endpoints map cleanly to resources (users, portfolios, alert rules).  
In this system, REST works well for:

- **User Authentication**
- **Portfolio CRUD** (create/update positions, watchlists)
- **Alert rule CRUD** (create/update/delete alert conditions)
- **Admin/ops endpoints** (health checks, configuration, support tooling)

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

#### 1.2.2 REST Advantages for This System

- **Caching:** HTTP caching with ETags and Cache-Control headers reduces database load for frequently requested quotes and portfolio snapshots.
- **Simplicity:** Clear resource semantics (GET, POST, PUT, DELETE) map directly to CRUD operations on portfolios and alerts.
- **Interoperability:** REST is universally understood, has mature tooling, and integrates trivially with existing auth middleware (JWT Bearer headers).
- **Scalability:** Straightforward horizontal scaling using standard load balancers — no specialised infrastructure required.

#### 1.2.3 REST Limitations

REST suffers from over-fetching and under-fetching when clients have complex, variable data needs. For example, a dashboard that must simultaneously display a user's portfolio positions enriched with live quotes, sector breakdowns, and 30-day P&L trends would require multiple sequential REST calls. This increases latency and unnecessary data transfer — a problem GraphQL resolves.


### 1.3 GraphQL API for Flexible Data Queries

GraphQL is best for **complex reads** where the UI needs **different combinations of data** depending on screen and device.  
In a trading dashboard, a client often needs portfolio, quotes, and derived metrics together. GraphQL can return all of that in **one round-trip**.

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

This single query replaces at least four separate REST calls, eliminates over-fetching, and allows the client to request a lightweight subset history of the same data by simply omitting the `history` field.

#### 1.3.2 GraphQL Advantages for This System

- **No over-fetching:** Clients fetch exactly the fields required — critical for low-bandwidth mobile users.
- **Single round-trip:** Multiple data sources (positions, quotes, history, alerts) resolved in one network request.
- **Strongly typed schema:** The GraphQL schema acts as a typed contract between client and server, enabling automatic validation and IDE tooling.
- **Client-side caching:** Client's normalised cache automatically reuses query results across components, reducing redundant network calls.

#### 1.3.3 GraphQL Limitations

GraphQL queries with unbounded depth (e.g., positions → history → dividends → company → ...) can cause N+1 database queries. This must be mitigated with DataLoader batching. Additionally, HTTP-level caching is harder with GraphQL because all queries typically use POST to the same endpoint.

---

## Section 2: WebSockets for Real-time Communication

### 2.1 Why WebSockets Are Essential

REST and GraphQL are request-response protocols: the client must initiate every data exchange. For a live stock market application, this is fundamentally insufficient. Price ticks arrive multiple times per second during market hours; and portfolio P&L fluctuates continuously; and time-sensitive alerts must reach users instantly. Polling REST endpoints frequently enough to appear "real-time" is expensive; it wastes bandwidth, increases server load, and still introduces latency equal to the polling interval. 
WebSockets eliminate this inefficiency by providing a **persistent, full-duplex TCP connection, low-latency, bidirectional channel** between clients and the server over which the server can push data the moment it becomes available.

### 2.2 WebSocket Connection Lifecycle

1. Client opens WebSocket connection.
2. Client authenticates by sending a JWT.
3. Server validates the token and registers the client with its user ID, then subscribes the client to their portfolio's ticker symbols.
4. Price events flow from the Market Data Ingestion Service through Kafka into the WebSocket Server, which fans out price ticks to all subscribed clients.
5. Clients may send subscription management messages (`subscribe`, `unsubscribe`) to add or remove tickers without reconnecting.
6. Heartbeat ping/pong frames are exchanged every 30 seconds to detect stale connections and trigger reconnection logic on the client.

### 2.3 Scaling WebSocket Connections

A common scalable design is to keep the WebSocket layer **stateless**, and feed it from an internal event stream.

- Market data ingestion publishes tick events to a **stream**
- WebSocket gateway consumes tick topics and fans out to connected clients
- Multiple gateway instances scale horizontally behind a load balancer  
  (often with “sticky sessions” OR a shared pub/sub layer)

### 2.5 WebSocket vs. Alternatives

| Technology | Latency | Server Push | Bi-directional | Scale Complexity |
|---|---|---|---|---|
| Short Polling (REST) | High (interval-dependent) | ❌ | ❌ | Low |
| Long Polling | Medium | ✅ Simulated | ❌ | Medium |
| Server-Sent Events | Low | ✅ Native | ❌ (one-way) | Low |
| **WebSockets** | **Very Low (<50ms)** | **✅ Native** | **✅** | Medium (requires pub/sub) |
| WebTransport (HTTP/3) | Very Low | ✅ | ✅ | High (emerging) |

---

## Section 3: Technology Recommendation and Justification

### 3.1 Recommended Architecture: REST + GraphQL + WebSockets

Use a **combination**:

- **REST (HTTP)** for authentication + writes (portfolio & alert rule management)
- **GraphQL (HTTP)** for flexible, multi-entity reads (dashboard views)
- **WebSockets** for real-time streaming (ticks + alerts)

### Justification
- **REST for writes**: simplest operationally, easiest to audit/version, great tooling
- **GraphQL for reads**: reduces round-trips and avoids over/under-fetching for complex UI
- **WebSockets for realtime**: lowest latency, avoids expensive polling, natural subscription model

**Separation of concerns**
- HTTP APIs = configuration + state (portfolio, rules, user prefs)
- WebSockets = high-frequency ephemeral events (ticks, triggers)


### 3.2 Technology Stack Recommendation

| Layer | Technology | Rationale |
|---|---|---|
| API Gateway | Kong / AWS API Gateway | Centralised auth, rate limiting, routing |
| REST API | Node.js (Express) or Python (FastAPI) | High throughput, async I/O, easy OpenAPI gen |
| GraphQL API | Apollo Server + DataLoader | Mature ecosystem, N+1 prevention, schema federation |
| WebSocket Server | Node.js + Socket.io or `ws` library | Event-loop optimised for concurrent connections |
| Message Queue | Apache Kafka | High-throughput, durable pub/sub for price events |
| Relational DB | SQL or PostgreSQL | ACID transactions for portfolios and user data |
| Cache | Redis | Sub-millisecond price cache, session storage, alert state |
| Time-Series DB | TimescaleDB (PostgreSQL extension) | Efficient storage and query of OHLCV price history |
| Client (Web) | React + Apollo Client + Socket.io-client | GraphQL caching + WebSocket subscription management |
| Client (Mobile) | React Native + same libraries | Code sharing between web and mobile |

---
## Section 4: System Architecture Diagram

Below is an Azure-oriented architecture showing how clients, REST/GraphQL APIs, WebSockets, and backend services interact.

### 4.1 Architecture Diagram

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

### 4.2 Key Data Flows Explained

#### 4.2.1 Portfolio Dashboard Load (GraphQL)

When a user opens their dashboard, the client sends a GraphQL POST request through the API Gateway to the Apollo Server. Apollo resolves the query by calling the Portfolio Service for position data and batching price lookups against the Redis cache via DataLoader. The single network response contains all data needed to render the full dashboard view.

#### 4.2.2 Alert Creation (REST)

A user configures a price alert via a REST `POST` request to `/api/alerts`. The REST API writes the alert configuration to PostgreSQL and stores the active threshold in Redis for sub-millisecond lookup during the high-frequency price evaluation loop in the Alert Service.

#### 4.2.3 Live Price Feed (WebSocket)

Market Data Providers stream price ticks to the Market Data Ingestion Service via their own WebSocket or REST feeds. The ingestion service normalises the data, writes time-series records to TimescaleDB, updates the Redis price cache, and publishes events to a Kafka topic. The WebSocket Server consumes from Kafka and pushes each tick to all subscribed clients within milliseconds of the exchange price being reported. Simultaneously, the Alert Service consumes the same Kafka stream and evaluates alert conditions; if a threshold is crossed, it publishes an alert event that the WebSocket Server delivers to the affected user's client.

### 4.3 Notes on key Azure services used

- **Azure API Management**: single front door for REST + GraphQL (policies, auth, rate limits).
- **Azure Web PubSub**: managed WebSocket service for scalable real-time fan-out.
- **Azure Event Hubs**: high-throughput ingestion stream for price ticks.
- **Azure Cache for Redis**: fast access to *latest quotes* for dashboard reads.
- **Azure SQL / Cosmos DB**: transactional and flexible stores for portfolios, rules, watchlists.
- **Azure Monitor + Application Insights**: logs, metrics, distributed tracing.

---

## Conclusion

The hybrid API architecture proposed in this report assigns each communication technology to the workloads it is most efficient at handling. REST provides a stable, cacheable, semantically clear interface for authentication and resource management. GraphQL eliminates over-fetching and multiple round-trips for the complex dashboard queries that define the user experience. WebSockets make real-time delivery of price ticks and alerts possible with latencies that no polling strategy can match. Together, these three architectural styles form a production-grade foundation capable of serving thousands of concurrent users with sub-100ms data freshness during live market hours.
