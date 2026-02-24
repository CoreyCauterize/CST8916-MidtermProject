# CST8916-MidtermProject for Group 7

[Presention Video](url)

Technical Report by 

## Section 1: REST and GraphQL for Data Requests and Updates


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