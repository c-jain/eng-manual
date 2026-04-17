# API Design — REST, GraphQL, gRPC & API Gateway

## Table of Contents

- [What is an API](#What-is-an-api)
- [REST](#rest)
- [GraphQL](#graphql)
- [gRPC](#grpc)
- [comparison](#comparison)
- [API Gateway](#api-gateway)
- [Interview lens](#Interview-lens)
- [References](#References)

---

## What is an API

An API (Application Programming Interface) is a contract between a producer and a consumer defining how they communicate. The choice of API style shapes latency, developer experience, payload size, caching behavior, and operational complexity.

Three dominant paradigms:

- **REST** — resource-oriented, HTTP-native, widely adopted
- **GraphQL** — query-oriented, client-driven, schema-first
- **gRPC** — procedure-oriented, binary, high-performance

The right choice is never universal — it depends on who the clients are, what the data access patterns look like, whether streaming is needed, and how much schema coupling you're willing to accept.

---

## REST

### Why it exists

Before REST (Roy Fielding's dissertation, 2000), distributed systems used protocols like CORBA, SOAP, and XML-RPC — all stateful, heavyweight, and tightly coupled. REST proposed using the web's existing infrastructure (HTTP, URIs, caching) as the architecture, not just the transport. The insight: the web already solved scale; model your API the same way.

### Architectural constraints

REST is not a protocol — it is an architectural *style* defined by six constraints. A system that violates any of them is not REST.

- **Client-Server** — separation of concerns; UI and data storage evolve independently
- **Stateless** — each request contains all information needed to process it; no session state stored on the server between requests
- **Cacheable** — responses must declare cacheability (`Cache-Control`, `ETag`); enables CDN and proxy caching
- **Uniform Interface** — a consistent interface simplifies the overall system architecture (see sub-constraints below)
- **Layered System** — a client cannot tell whether it is connected directly to the origin server or to an intermediary (load balancer, CDN, cache)
- **Code on Demand** *(optional)* — servers can extend client functionality by transferring executable code (e.g. JavaScript)

The **Uniform Interface** constraint has four sub-constraints:

- Resource identification in requests — URIs identify resources (`/users/42`)
- Resource manipulation through representations — clients interact via representations (JSON, XML), not the resource directly
- Self-descriptive messages — each message carries enough metadata to describe how to process it (`Content-Type`, status codes)
- HATEOAS — Hypermedia as the Engine of Application State; responses include links to valid next actions

> **Note on real-world REST:** Most APIs called "REST" are REST-ish — they use HTTP + JSON but skip HATEOAS. True REST with HATEOAS is rarely implemented in practice. What most people mean by "REST API" is HTTP-based CRUD with JSON payloads.

### HTTP method semantics

| Method | Semantics       | Idempotent | Safe |
|--------|-----------------|------------|------|
| GET    | Read            | ✓          | ✓    |
| POST   | Create          | ✗          | ✗    |
| PUT    | Full replace    | ✓          | ✗    |
| PATCH  | Partial update  | ✗*         | ✗    |
| DELETE | Delete          | ✓          | ✗    |

- **Idempotent** — calling it N times has the same effect as calling it once. PUT `/users/42` with the same body always produces the same state. POST `/users` creates a new user each time.
- **Safe** — produces no server-side side effects. GET and HEAD are safe; they should never mutate state.
- PATCH *can* be made idempotent depending on implementation (conditional patches), but the spec does not require it.

Interviewers often ask about idempotency. Know it cold — it directly affects retry logic in distributed systems.

### Status codes that matter in interviews

- `200 OK`, `201 Created`, `204 No Content`
- `301 Moved Permanently`, `304 Not Modified` (caching hit)
- `400 Bad Request`, `401 Unauthorized` (not authenticated), `403 Forbidden` (authenticated but not authorized), `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`, `429 Too Many Requests`
- `500 Internal Server Error`, `502 Bad Gateway` (upstream failed), `503 Service Unavailable`, `504 Gateway Timeout`

Distinguish 401 vs 403 — interviewers notice when you conflate them.

### versioning strategies

- **URI versioning** — `/v1/users`, `/v2/users` — most common, breaks REST's uniform interface principle but easy to test and route
- **Header versioning** — `Accept: application/vnd.myapi+json;version=2` — clean but harder to test with browsers and `curl`
- **Query parameter** — `/users?api-version=2` — simple, pollutes URLs, not cacheable by default

URI versioning wins in practice because it is the easiest to proxy, test, and document. Acknowledge the trade-off when asked.

### Problems REST brings

- **Over-fetching** — endpoint returns more fields than the client needs; wastes bandwidth
- **Under-fetching** — a single view requires data from multiple endpoints; forces N round-trips (chatty clients)
- **No strict schema** — contract drift between client and server is easy to miss; OpenAPI/Swagger helps but is not enforced at runtime
- **Versioning accumulation** — `/v1`, `/v2`, `/v3` accumulate over time; supporting multiple versions is expensive
- **Real-time limitations** — REST over HTTP/1.1 is request-response only; Server-Sent Events (SSE) provide one-way streaming, but bidirectional streaming requires WebSockets or a different paradigm

---

## GraphQL

### Why it exists

Facebook built GraphQL internally in 2012 and open-sourced it in 2015. The trigger: the Facebook News Feed required data assembled from dozens of REST endpoints, causing poor performance on mobile networks. The problem was not the REST protocol itself — it was that REST gave the server control over response shape, and different clients (mobile vs web) needed dramatically different shapes of the same data.

GraphQL moves the response shape decision to the client.

### How it works

Everything in GraphQL goes through a single endpoint (typically `POST /graphql`). The client sends a query document describing exactly the shape of data it needs.

#### Schema definition language (SDL)

The schema is the contract. It is strongly typed and defines all possible operations:

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  author: User!
  createdAt: String!
}

type Query {
  user(id: ID!): User
  posts(limit: Int, offset: Int): [Post!]!
}

type Mutation {
  createPost(title: String!, authorId: ID!): Post!
  deletePost(id: ID!): Boolean!
}

type Subscription {
  postCreated: Post!
}
```

`!` means non-nullable. `[Post!]!` means the list itself is non-null and each element is non-null.

#### Client query example

The client specifies exactly what it needs:

```graphql
query GetUserWithRecentPosts {
  user(id: "42") {
    name
    email
    posts {
      title
      createdAt
    }
  }
}
```

The server returns exactly that shape — nothing more, nothing less. No over-fetching.

#### Resolvers

Each field in the schema maps to a **resolver** — a function that fetches the data for that field. The GraphQL runtime calls resolvers recursively as it walks the query tree.

```go
// Go (using gqlgen)
func (r *queryResolver) User(ctx context.Context, id string) (*model.User, error) {
    return r.db.FindUser(ctx, id)
}

// Called for each user returned, fetching their posts
func (r *userResolver) Posts(ctx context.Context, user *model.User) ([]*model.Post, error) {
    return r.db.FindPostsByUser(ctx, user.ID)
}
```

#### The N+1 problem — the most important GraphQL pitfall

Naive resolver implementation creates the N+1 query problem:

```
Query: { users { posts { title } } }

Without batching:
  SELECT * FROM users              → returns 10 users
  SELECT * FROM posts WHERE user_id = 1
  SELECT * FROM posts WHERE user_id = 2
  SELECT * FROM posts WHERE user_id = 3
  ...                              → 10 more queries
  Total: 11 queries (1 + N)
```

Solution: **DataLoader** — batches and deduplicates resolver calls within a single request tick:

```go
// With DataLoader, the 10 individual calls are batched:
SELECT * FROM posts WHERE user_id IN (1,2,3,4,5,6,7,8,9,10)
// Total: 2 queries regardless of N
```

DataLoader collects all IDs requested within one tick of the event loop, then fires a single batched query. Every production GraphQL service needs DataLoader or an equivalent.

#### Operations

- **Query** — read data; maps to GET semantics
- **Mutation** — write data; maps to POST/PUT/DELETE semantics
- **Subscription** — real-time updates delivered over a WebSocket connection

### Problems GraphQL brings

- **N+1 queries** — naive resolver implementation hammers the database; requires DataLoader or similar batching
- **Caching is hard** — a single POST endpoint breaks HTTP caching; GET requests with persisted queries and CDN caching restore this, but it requires extra setup
- **Query depth attacks** — a malicious client can send arbitrarily deep nested queries that explode resolver chains; mitigate with depth limiting and query cost analysis (query complexity scoring)
- **Schema governance** — the schema is a shared contract; breaking changes (removing a field, changing a type) require coordination across all clients; use deprecation + migration periods
- **File uploads** — not part of the spec; requires multipart form workarounds
- **Overkill for stable, simple APIs** — if your clients' data needs are predictable and uniform, REST is simpler and better understood

---

## gRPC

### Why it exists

gRPC (Google Remote Procedure Call, open-sourced 2015) was built for Google's internal inter-service communication, where:

- REST's text overhead (JSON, HTTP headers) is wasteful at high request volumes
- Shared, strongly-typed contracts reduce interface drift between services
- Streaming is a first-class requirement (real-time feeds, large data transfers)
- Polyglot environments (Go, Java, Python, C++) need code generation from a single source of truth

gRPC uses **Protocol Buffers** as the IDL and serialization format, and **HTTP/2** as the transport.

### Protocol buffers

Protocol Buffers (protobuf) is a binary serialization format. You define messages and services in `.proto` files, then `protoc` generates type-safe client and server code in your target language.

```protobuf
// user.proto
syntax = "proto3";
package user;
option go_package = "github.com/c-jain/eng-manual/gen/user";

service UserService {
  // Unary: one request, one response
  rpc GetUser(GetUserRequest) returns (User);

  // Server streaming: one request, stream of responses
  rpc ListUsers(ListUsersRequest) returns (stream User);

  // Client streaming: stream of requests, one response
  rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchSummary);

  // Bidirectional: stream of requests, stream of responses
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message User {
  string id    = 1;  // field tag = 1
  string name  = 2;
  string email = 3;
}

message GetUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 limit = 1;
}

message BatchSummary {
  int32 created = 1;
  int32 failed  = 2;
}
```

**Why binary is smaller:** Field names are replaced by field numbers (tags `1`, `2`, `3`). Integers use varint encoding — small numbers take fewer bytes. No schema is sent on the wire; both sides have the `.proto` definition. Compared to JSON:

```
JSON:    {"id":"abc","name":"Alice","email":"alice@example.com"}  ≈ 52 bytes
Protobuf: same message                                            ≈ 20 bytes (≈60% smaller)
```

### HTTP/2 transport

gRPC runs exclusively over HTTP/2, which provides:

- **Multiplexing** — multiple RPC streams over a single TCP connection; eliminates the per-request connection overhead and HTTP/1.1's head-of-line blocking at the HTTP layer
- **Header compression** (HPACK) — repeated headers (e.g., `:authority`, `content-type`) are sent once and referenced by index after that
- **Binary framing** — data is split into typed binary frames; more efficient than HTTP/1.1 text parsing
- **Server push** — server can proactively send responses the client will need (used by gRPC for streaming)

```
HTTP/1.1:                               HTTP/2:
┌───────────────────────────────┐       ┌────────────────────────────────┐
│ TCP conn 1: RPC A (blocked)   │       │ Single TCP connection:          │
│ TCP conn 2: RPC B (blocked)   │       │   Stream 1: RPC A (concurrent)  │
│ TCP conn 3: RPC C (blocked)   │       │   Stream 2: RPC B (concurrent)  │
└───────────────────────────────┘       │   Stream 3: RPC C (concurrent)  │
  3 connections, HOL blocking           └────────────────────────────────┘
                                          1 connection, multiplexed
```

### Four streaming types

```
Unary (most common):
  Client ─── req ──────► Server
  Client ◄── res ─────── Server

Server streaming (server pushes data):
  Client ─── req ──────► Server
  Client ◄── msg1 ─────  Server
  Client ◄── msg2 ─────  Server
  Client ◄── msgN ─────  Server

Client streaming (client uploads data):
  Client ─── msg1 ─────► Server
  Client ─── msg2 ─────► Server
  Client ─── msgN ─────► Server
  Client ◄── res ─────── Server

Bidirectional (full-duplex):
  Client ─── msg1 ─────► Server
  Client ◄── msg2 ─────  Server
  Client ─── msg3 ─────► Server
  Client ◄── msg4 ─────  Server
  (interleaved, independent of each other)
```

### Generated Go code

`protoc` generates type-safe client and server interfaces. You implement the server:

```go
// Generated interface (you implement this)
type UserServiceServer interface {
    GetUser(context.Context, *GetUserRequest) (*User, error)
    ListUsers(*ListUsersRequest, UserService_ListUsersServer) error
    mustEmbedUnimplementedUserServiceServer()
}

// Your implementation
type server struct {
    pb.UnimplementedUserServiceServer
    db *sql.DB
}

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := s.db.QueryUserByID(ctx, req.Id)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "user %q not found", req.Id)
    }
    return &pb.User{Id: user.ID, Name: user.Name, Email: user.Email}, nil
}

// Server streaming: write multiple responses to the stream
func (s *server) ListUsers(req *pb.ListUsersRequest, stream pb.UserService_ListUsersServer) error {
    users, _ := s.db.QueryUsers(stream.Context(), int(req.Limit))
    for _, u := range users {
        if err := stream.Send(&pb.User{Id: u.ID, Name: u.Name}); err != nil {
            return err
        }
    }
    return nil
}

// Wire up the server
func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer()
    pb.RegisterUserServiceServer(s, &server{})
    s.Serve(lis)
}
```

### Error model

gRPC has a richer error model than HTTP status codes. The `status` package maps to HTTP/2 trailers:

```go
// Common gRPC status codes
codes.OK             // 0  — success
codes.InvalidArgument // 3  — bad input (like HTTP 400)
codes.NotFound       // 5  — resource missing (like HTTP 404)
codes.AlreadyExists  // 6  — conflict (like HTTP 409)
codes.PermissionDenied // 7 — forbidden (like HTTP 403)
codes.Unauthenticated // 16 — unauthenticated (like HTTP 401)
codes.Internal       // 13 — server error (like HTTP 500)
codes.Unavailable    // 14 — service down (like HTTP 503)
```

### Problems gRPC brings

- **Not browser-native** — browsers cannot directly use HTTP/2 trailers, which gRPC requires; `grpc-web` is a proxy-based workaround, but it adds a layer of complexity
- **Binary format** — harder to debug; `curl` doesn't work; need `grpcurl` or a dedicated GUI like Postman's gRPC support or Evans
- **Tight coupling via `.proto` files** — the schema is a shared dependency; every service change requires clients and servers to update their generated code; schema changes require careful coordination
- **Load balancing complexity** — HTTP/2 multiplexing means a single long-lived connection handles many requests; L4 load balancers (which balance at the TCP level) cannot distribute load across streams; you need L7 (application-level) load balancing or a service mesh

---

## Comparison

```
                  REST              GraphQL           gRPC
─────────────────────────────────────────────────────────────────
Protocol          HTTP/1.1+         HTTP/1.1+         HTTP/2
Format            JSON/XML          JSON              Protobuf (binary)
Schema            Optional          Required (SDL)    Required (.proto)
Typing            Weak              Strong            Strong
Runtime contract  None              Introspection     Reflection API
Caching           Native HTTP       Manual            Manual
Streaming         SSE / WebSocket   Subscriptions     Native (4 modes)
Browser support   ✓ Native          ✓ Native          Partial (grpc-web)
Tooling           Excellent         Good              Good
Code generation   Optional          Optional          Required
Best for          Public APIs       Flexible UIs      Internal services
─────────────────────────────────────────────────────────────────
```

### When to choose what

**Choose REST when:**
- Building a public API consumed by third parties or unknown clients
- High client diversity — browsers, mobile, CLI, scripts
- Operations map cleanly to CRUD on named resources
- HTTP caching (CDN, `ETag`, `Cache-Control`) is a hard requirement
- Team familiarity and tooling ecosystem (Swagger, Postman, curl) are important
- The API will be long-lived and versioned independently

**Choose GraphQL when:**
- Client data requirements vary significantly across clients (mobile vs desktop vs external)
- You want to eliminate over-fetching and under-fetching
- Building a unified API layer that aggregates multiple backend services (GraphQL as BFF/aggregation layer)
- Frontend teams need to iterate on data requirements without backend changes
- You can invest in DataLoader and caching infrastructure

**Choose gRPC when:**
- Service-to-service communication within a microservices system (not client-facing)
- Performance is critical — high throughput, low latency, high message volume
- Streaming is a first-class requirement (real-time feeds, bidirectional communication)
- Polyglot environment where you want strongly-typed, code-generated contracts across languages
- You control both client and server (tight coupling is acceptable)

---

## API Gateway

### Why it exists

In a microservices architecture, without a gateway, clients must:
- Know the address of every service
- Handle authentication independently per service call
- Deal with different protocols per service
- Accept that cross-cutting concerns (rate limiting, logging, tracing) are scattered everywhere

An **API Gateway** solves this by being the single entry point for all external traffic. Every client request flows through it.

```
                     ┌────────────────────────────────────────────┐
                     │               API Gateway                  │
                     │                                            │
Clients              │  ┌──────────┐ ┌───────────┐ ┌──────────┐   │
┌────────┐           │  │  Auth /  │ │   Rate    │ │  Router  │   │
│  Web   │──────────►│  │  AuthZ   │ │  limiting │ │  + LB    │   │
└────────┘           │  └──────────┘ └───────────┘ └──────────┘   │
                     │  ┌──────────┐ ┌───────────┐ ┌──────────┐   │
┌────────┐           │  │   SSL    │ │   Cache   │ │  Tracing │   │
│ Mobile │──────────►│  │  termin. │ │           │ │ + Logging│   │
└────────┘           │  └──────────┘ └───────────┘ └──────────┘   │
                     └────────────┬─────────────────────────┬─────┘
┌────────┐                        │                         │
│3rd Pty │──────────►  ┌──────────┴───┐            ┌────────┴────┐
└────────┘             │ User service │            │ Post service│
                       └──────────────┘            └─────────────┘
```

### Responsibilities

- **Authentication & authorization** — validates JWT tokens or introspects OAuth tokens before the request ever reaches a backend service; backends trust the gateway and read claims from injected headers
- **Rate limiting** — enforces per-user, per-IP, or per-API-key limits; common algorithms are token bucket and sliding window; protects backends from abuse and DDoS
- **Request routing** — path-based routing (`/api/users/*` → User service), header-based routing (A/B testing, canary deploys)
- **Load balancing** — distributes traffic across service instances; typically round-robin or least-connections; stateful services require sticky sessions
- **SSL termination** — handles TLS at the edge; backends communicate over the internal network unencrypted (or with lighter mTLS via a service mesh)
- **Request/response transformation** — protocol translation (REST client to gRPC backend), header injection, payload reshaping
- **Caching** — caches responses at the gateway layer for repeated identical requests
- **Observability** — injects trace IDs, aggregates access logs, emits request metrics (latency, error rate, throughput) — all in one place
- **Circuit breaking** — detects unhealthy backends and stops routing traffic to them to prevent cascade failures

### Backend for Frontend (BFF) pattern

A single gateway serving all clients becomes a least-common-denominator API — it makes compromises for every client and optimizes for none. The BFF pattern introduces a specialized gateway per client type:

```
┌────────┐    ┌──────────────────┐
│  Web   │───►│    Web BFF       │──┐
└────────┘    │ (richer datasets,│  │    ┌──────────────────────────┐
              │ batch fetching)  │  ├───►│     Internal services    │
              └──────────────────┘  │    └──────────────────────────┘
┌────────┐    ┌──────────────────┐  │
│ Mobile │───►│   Mobile BFF     │──┘
└────────┘    │ (compact payload,│
              │  battery aware)  │
              └──────────────────┘
```

Each BFF is owned by the team that owns the corresponding frontend. It can aggregate calls, reshape responses, and apply client-specific caching without affecting other clients.

### API Gateway vs service mesh

Commonly confused in interviews:

- **API Gateway** handles **north-south traffic** — external clients talking to internal services. Its concerns are business-level: authentication, rate limiting, routing, developer experience.
- **Service Mesh** (Istio, Linkerd) handles **east-west traffic** — internal service-to-service communication. Its concerns are infrastructure-level: mTLS between services, retries, circuit breaking, distributed tracing propagation.

They complement each other. In a mature architecture, you often deploy both: the API Gateway at the edge, the service mesh for inter-service reliability.

### Problems API Gateway brings

- **Single point of failure** — must be deployed in HA mode with health checks, redundant instances, and a global load balancer in front; a down gateway means all clients are down
- **Performance bottleneck** — every request passes through it; the gateway's latency adds directly to the end-to-end latency budget; keep processing lightweight
- **Configuration sprawl** — routing rules, rate limits, auth policies, and transforms accumulate into complex configuration that becomes hard to reason about; treat gateway config as code with version control and testing
- **Logic creep** — it is tempting to push business logic into the gateway (e.g., response transformations that encode domain knowledge); resist this; the gateway should handle infrastructure concerns, not business rules

### Common implementations

- **Kong** — open source, plugin-based, Lua/Go plugins, high performance, self-hosted or managed
- **AWS API Gateway** — fully managed, deep AWS integration, serverless-friendly, per-request pricing
- **nginx / Envoy** — lower-level reverse proxies commonly used as gateway building blocks; Envoy is the data plane behind most service meshes
- **Traefik** — cloud-native, automatic service discovery (reads from Docker/Kubernetes), Go-native

---

## Interview lens

### What interviewers actually look for

**The core signal they want:** Do you treat API design as a real decision with trade-offs, or do you default to REST every time without thinking?

**Justifying your choice:** When you say "I'd use REST here," immediately follow with the reasoning — public client base, HTTP caching required, CRUD maps cleanly to resources. When you say "gRPC," acknowledge the browser limitation and explain why it doesn't apply (internal service). Unprompted trade-off awareness is a strong signal.

**Key trade-offs to demonstrate:**

- REST → predictable caching, broad tooling, over/under-fetching
- GraphQL → precise data fetching, flexible for UI teams, harder caching, N+1 risk
- gRPC → low latency and streaming, poor browser support, binary debugging, tight coupling

**API Gateway questions that come up:**

- "How do you handle auth in microservices?" → Gateway validates JWT, injects user claims as headers; backends trust the gateway and don't re-validate
- "What's a BFF?" → Specialized gateway per client type; avoids one-size-fits-all API design; owned by the frontend team
- "How do you rate-limit?" → Token bucket or sliding window per user/IP/API key at the gateway; back-pressure signals propagate to clients via `429 Too Many Requests` with `Retry-After` headers
- "What happens when the gateway goes down?" → It's a SPOF; run it in HA mode with multiple instances behind a global load balancer; health checks with fast failover

**Red flags interviewers watch for:**

- Choosing gRPC for a public-facing API without acknowledging browser limitations
- Choosing GraphQL without mentioning N+1 or caching complexity
- Not mentioning HA/redundancy when describing an API Gateway
- Treating API Gateway and service mesh as interchangeable
- Using `401 Unauthorized` when you mean `403 Forbidden`

**Strong interview signals:**

- Saying "it depends" and then actually explaining on what
- Knowing the difference between idempotency and safety in HTTP verbs, and why it matters for retry logic
- Mentioning DataLoader when GraphQL comes up in a scale context
- Knowing that gRPC over HTTP/2 means L4 load balancers don't distribute load correctly, so you need L7 or a service mesh
- Distinguishing API Gateway (north-south) from service mesh (east-west) without being prompted

---

## References

- [Fielding's REST dissertation (2000)](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [GraphQL specification](https://spec.graphql.org/)
- [graphql-go/graphql — Go GraphQL library](https://github.com/graphql-go/graphql)
- [99designs/gqlgen — code-first GraphQL for Go](https://github.com/99designs/gqlgen)
- [gRPC documentation](https://grpc.io/docs/)
- [Protocol Buffers encoding](https://protobuf.dev/programming-guides/encoding/)
- [grpc/grpc-go — gRPC for Go](https://github.com/grpc/grpc-go)
- [Kong API Gateway](https://konghq.com/products/kong-gateway)
- [AWS API Gateway developer guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/)
- [Microservices patterns — BFF (Sam Newman)](https://samnewman.io/patterns/architectural/bff/)