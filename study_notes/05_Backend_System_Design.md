# Backend & System Design — From Code to Architecture

> **Target**: Knows basic Python. No production backend experience.
> **Goal**: Understand every backend pattern needed for the platform-level system design.

---

# PART 1: REST APIs & HTTP

## 1. What is a REST API

### Plain Explanation

REST = Representational State Transfer. It's a standard way for computers to talk to each other over HTTP. Your frontend sends requests to a URL, the backend processes it, and sends back a response.

### Simple FastAPI Example

```python
from fastapi import FastAPI

app = FastAPI()

# Data store (in real life: database)
users = {"1": {"name": "Alice", "email": "alice@example.com"}}

@app.get("/users/{user_id}")
def get_user(user_id: str):
    """GET /users/1 → returns Alice's data"""
    if user_id in users:
        return users[user_id]
    return {"error": "User not found"}, 404
```

### Core Principles

- **Resources** (nouns): /users, /orders, /products
- **HTTP methods** (verbs): GET, POST, PUT, DELETE
- **Stateless**: each request is independent, no server-side session
- **Standard status codes**: tell the client what happened

### Why This Matters
Every API you build (ROAST, SYNAPSE, SuperOwl) is a REST API. This is the foundation of backend engineering.

---

## 2. Resources and URLs

### Naming Conventions

```python
# GOOD (nouns, plural, consistent):
GET    /users              # list all users
GET    /users/{id}         # get one user
POST   /users              # create user
PUT    /users/{id}         # replace user
DELETE /users/{id}         # delete user

# GOOD (nested resources):
GET    /users/{id}/orders  # orders for user 1

# BAD (verbs, inconsistent):
GET    /getUserData        # use GET /users/{id}
POST   /createUserRecord   # use POST /users
```

### Why This Matters
Consistent URL patterns make your API predictable. the platform APIs follow REST conventions. Interviewers check if you know RESTful design.

---

## 3. HTTP Methods

### What Each Method Does

| Method | Action | Idempotent? | Example |
|--------|--------|-------------|---------|
| **GET** | Read data | Yes | Get user profile |
| **POST** | Create new resource | No | Create new order |
| **PUT** | Replace entire resource | Yes | Update all fields |
| **PATCH** | Partial update | Yes | Update one field |
| **DELETE** | Remove resource | Yes | Delete account |

### Idempotency Explained

**Idempotent** = calling it 10 times has the same effect as calling it once.

```
GET /users/1 → returns Alice. Call it 100 times → same result. ✅
DELETE /users/1 → deletes Alice. Call again → Alice already gone, no error. ✅
POST /orders → creates new order. Call 3 times → 3 orders! ❌ (Not idempotent)
```

### FastAPI Implementation

```python
@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}

@app.post("/items")
def create_item(name: str, price: float):
    item = {"id": next_id(), "name": name, "price": price}
    db.insert(item)
    return item, 201

@app.put("/items/{item_id}")
def replace_item(item_id: int, name: str, price: float):
    db[item_id] = {"id": item_id, "name": name, "price": price}
    return db[item_id]

@app.delete("/items/{item_id}")
def delete_item(item_id: int):
    del db[item_id]
    return 204
```

### Why This Matters
Understanding idempotency is critical for payment systems. If the client retries a POST (network timeout), you might charge the customer twice. That's why POST is NOT idempotent and you need idempotency keys.

---

## 4. HTTP Status Codes

### The Codes You MUST Know

| Code | Name | Meaning | When to Use |
|------|------|---------|-------------|
| **200** | OK | Success | GET, PUT, PATCH |
| **201** | Created | Resource created | POST (successful creation) |
| **204** | No Content | Success, no body | DELETE |
| **400** | Bad Request | Client sent bad data | Validation error |
| **401** | Unauthorized | Not logged in | Missing/invalid auth token |
| **403** | Forbidden | Logged in but no permission | User can't access this resource |
| **404** | Not Found | Resource doesn't exist | Wrong URL or ID |
| **422** | Unprocessable Entity | Valid format, invalid content | Email format wrong |
| **429** | Too Many Requests | Rate limited | Exceeded API quota |
| **500** | Internal Server Error | Something broke on server | Unexpected error |

### Why This Matters
Every API you build uses these codes. The frontend reads these codes to decide what to show the user (error page, retry button, redirect to login). Using the wrong code confuses clients.

---

## 5. Request/Response Structure

### Anatomy of an HTTP Request

```
POST /api/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
Content-Type: application/json

{
    "user_id": 123,
    "items": [{"product_id": 456, "quantity": 2}]
}
```

| Part | Example | Purpose |
|------|---------|---------|
| Method + Path | `POST /api/orders` | What to do |
| Headers | `Authorization: Bearer...` | Metadata (auth, format) |
| Body | `{"user_id": 123, ...}` | The actual data |

### Anatomy of an HTTP Response

```
HTTP/1.1 201 Created
Content-Type: application/json

{
    "order_id": 789,
    "status": "confirmed",
    "total": 49.99
}
```

### Why This Matters
Debugging API issues starts with checking the raw request/response. Your browser's dev tools, curl, or Postman show this exact structure.

---

## 6. Pagination — Offset vs Cursor

### Offset Pagination (Common but Flawed)

```python
# /api/users?page=2&limit=20
# page=2 means: skip first 20 users, return next 20

@app.get("/api/users")
def list_users(page: int = 1, limit: int = 20):
    offset = (page - 1) * limit
    users = db.execute(
        "SELECT * FROM users LIMIT ? OFFSET ?",
        (limit, offset)
    )
    return {
        "data": users,
        "page": page,
        "has_more": len(users) == limit
    }
```

**Problem**: If a user is inserted on page 1 while you're viewing page 2, you see the same user twice (items shift).

### Cursor Pagination (Better)

```python
# /api/users?cursor=2024-01-15T00:00:00Z&limit=20
# cursor = timestamp of last item you saw

@app.get("/api/users")
def list_users(cursor: str = None, limit: int = 20):
    if cursor:
        users = db.execute(
            "SELECT * FROM users WHERE created_at > ? ORDER BY created_at LIMIT ?",
            (cursor, limit)
        )
    else:
        users = db.execute(
            "SELECT * FROM users ORDER BY created_at LIMIT ?",
            (limit,)
        )
    
    next_cursor = users[-1]["created_at"] if len(users) == limit else None
    return {"data": users, "next_cursor": next_cursor}
```

**Advantage**: New items on page 1 don't affect page 2. Stable, consistent view.

### Why This Matters
ROAST doesn't paginate (single analysis). SYNAPSE does (search results). Choosing the right pagination affects user experience and backend performance.

---

## 7. API Versioning

### Why Version?

```python
# v1 — original
@app.get("/v1/analyze")

# v2 — improved (breaking change)
@app.get("/v2/analyze")
```

Clients upgrade at their own pace. Old clients keep using v1. New clients use v2. No breaking changes for anyone.

### Approaches

| Method | Example | Pros | Cons |
|--------|---------|------|------|
| URL path | `/v1/users` | Simple, explicit | URL changes |
| Header | `Accept: version=2` | Clean URLs | Harder to test |
| Query param | `/users?version=2` | Simple | Clutters URL |

### Why This Matters
ROAST has one version. SYNAPSE uses `/api/v1/`. the platform the platform API versions for enterprise clients who need stability.

---

## 8. Idempotency

### Why It Matters for Payments

```
1. User clicks "Pay $50"
2. POST /api/charge → network timeout
3. User sees error, clicks again
4. POST /api/charge again

Without idempotency: charged $100!
With idempotency: second request returns same result, no extra charge.
```

### Implementation

```python
# Client sends idempotency key (UUID) with every POST
POST /api/charge
Idempotency-Key: a1b2c3d4-e5f6-7890-abcd-ef1234567890

{
    "amount": 5000,
    "currency": "USD"
}
```

```python
# Server checks if it already processed this key
@app.post("/api/charge")
def charge(idempotency_key: str = Header(None)):
    # Check if we've seen this key before
    existing = redis.get(f"idempotency:{idempotency_key}")
    if existing:
        return existing  # Return same result, no duplicate charge
    
    # Process normally
    result = payment_service.charge(amount)
    
    # Cache result with key (24h TTL)
    redis.set(f"idempotency:{idempotency_key}", result, ex=86400)
    return result
```

### Why This Matters
Every payment system needs idempotency. Stripe's API made this standard. Interviewers love this topic because it shows you think about failure modes.

---

# PART 2: AUTHENTICATION

## 9. What is JWT (JSON Web Token)

### Plain Explanation

JWT is a self-contained token that proves WHO you are. It contains user info (user_id, role, expiration) in an encoded format the server can verify without a database lookup.

### Why JWT?

| Aspect | Session (cookie) | JWT |
|--------|-----------------|-----|
| Storage | Server memory/DB | Client (browser/app) |
| Lookup | DB query on every request | Decode and verify signature |
| Scaling | Need shared session store | Stateless — any server can verify |
| Revocation | Can delete session | Can't revoke (until expiry) |

---

## 10. JWT Structure

### The Three Parts

```
header.payload.signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFsaWNlIn0.
B2gxqVJw9H_K5vZ5YQ3cGZuX1mGXvhqZHxZsVXpOj8o
```

### Part 1: Header

```json
{
  "alg": "HS256",    // signing algorithm
  "typ": "JWT"       // token type
}
```

### Part 2: Payload (the data)

```json
{
  "sub": "user_123",           // subject (user ID)
  "name": "Alice",             // custom data
  "role": "admin",             // access level
  "iat": 1516239022,           // issued at (timestamp)
  "exp": 1516242622            // expiration (timestamp)
}
```

### Part 3: Signature (prevents tampering)

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret_key
)
```

If anyone modifies the payload, the signature won't match. The server rejects it.

### Code

```python
import jwt

# Create token
payload = {"user_id": "123", "role": "admin", "exp": time() + 3600}
token = jwt.encode(payload, "secret_key", algorithm="HS256")

# Verify token
try:
    decoded = jwt.decode(token, "secret_key", algorithms=["HS256"])
    user_id = decoded["user_id"]  # safe to use
except jwt.ExpiredSignatureError:
    return "Token expired"
except jwt.InvalidTokenError:
    return "Invalid token"
```

---

## 11. Access Token vs Refresh Token

### The Problem

Short-lived tokens (15 min) = more secure (if stolen, limited damage).
But users don't want to log in every 15 minutes.

### The Solution: Two Tokens

```
Login: username + password → server returns:
  - Access token (15 min): sent with every request
  - Refresh token (7 days): stored safely, used only to get new access tokens

When access token expires:
  - Client sends refresh token → server returns new access token
  - Refresh token can be revoked (logged out on all devices)
```

### Flow

```python
# Login
@app.post("/login")
def login(username, password):
    if not verify_password(password, stored_hash):
        return 401
    
    access_token = create_access_token({"user_id": user.id}, expires=15*60)
    refresh_token = create_refresh_token({"user_id": user.id}, expires=7*24*60*60)
    
    return {"access_token": access_token, "refresh_token": refresh_token}

# Refresh
@app.post("/refresh")
def refresh(refresh_token: str):
    try:
        payload = jwt.decode(refresh_token, REFRESH_SECRET, algorithms=["HS256"])
        new_access = create_access_token({"user_id": payload["user_id"]})
        return {"access_token": new_access}
    except:
        return 401  # must log in again
```

### Why This Matters
ROAST uses token-based auth (email tokens for extra analysis). SYNAPSE uses admin key auth for sensitive endpoints. SuperOwl uses Firebase Auth.

---

# PART 3: DATABASES & CACHING

## 27. What is Redis

### Plain Explanation

Redis is an in-memory data store. It keeps ALL data in RAM (not disk), making it extremely fast (microsecond latency). It's NOT a replacement for your main database.

### Common Uses

| Use Case | Why Redis | Example |
|----------|-----------|---------|
| Cache | Fast reads, reduces DB load | Cache API responses |
| Sessions | Fast, auto-expiry | User login sessions |
| Rate Limiting | Atomic counters | 100 requests/hour |
| Real-time | Pub/sub, sorted sets | Leaderboards, chat |
| Queues | Lists with blocking pop | Background job queue |

### Why This Matters
ROAST uses Redis for EVERYTHING: rate limits, sessions, market data cache, breaking signals, corpus, budget tracking. SYNAPSE uses it. SuperOwl uses Redis-compatible Upstash.

---

## 28. Redis Data Types

### Strings

```python
# Simplest type — a key with a string value
redis.set("user:123:name", "Alice")
redis.get("user:123:name")  # → "Alice"

# With TTL (auto-expire after 1 hour)
redis.setex("session:abc123", 3600, user_data)

# Atomic counter
redis.incr("page_views")  # → 1, 2, 3...
```

### Hashes

```python
# Like a Python dict — group related fields
redis.hset("user:123", mapping={
    "name": "Alice",
    "email": "alice@example.com",
    "role": "admin"
})

redis.hget("user:123", "name")        # → "Alice"
redis.hgetall("user:123")              # → {"name": "Alice", ...}
redis.hincrby("user:123", "logins", 1) # atomic increment a field
```

### Lists

```python
# Ordered collection, good for queues
redis.lpush("my_queue", "job1")     # add to front
redis.lpush("my_queue", "job2")
redis.rpop("my_queue")              # remove from back
```

### Sets

```python
# Unordered, no duplicates. Great for membership checks
redis.sadd("online_users", "user:123")
redis.smembers("online_users")       # → {"user:123"}
redis.sismember("online_users", "user:123")  # → True
```

### Sorted Sets

```python
# Like sets but with scores (ordered by score)
redis.zadd("leaderboard", {"Alice": 100, "Bob": 85, "Charlie": 95})
redis.zrevrange("leaderboard", 0, 2)  # → ["Alice", "Charlie", "Bob"]
redis.zincrby("leaderboard", 10, "Bob")  # Bob now has 95
```

### Why This Matters
ROAST uses sorted sets for rate limiting, strings for cache, hashes for sessions, lists for job queues. The right data type makes the implementation trivial.

---

## 29. Cache-Aside Pattern

### The Most Common Caching Pattern

```python
def get_user(user_id):
    # 1. Check cache first
    cached = redis.get(f"user:{user_id}")
    if cached is not None:
        return json.loads(cached)
    
    # 2. Cache miss — read from database
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. Write to cache (with TTL)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    
    return user
```

**For writes**:
```python
def update_user(user_id, data):
    # Update database
    db.execute("UPDATE users SET name = ? WHERE id = ?", data["name"], user_id)
    
    # Invalidate cache (next read will repopulate)
    redis.delete(f"user:{user_id}")
```

### Why This Works

- Frequent reads hit cache (fast, <5ms)
- Infrequent reads hit DB (slow, ~50ms)
- Cache automatically refreshes on miss
- Cache invalidation on write keeps data fresh

---

## 30. Cache Invalidation

### The Hard Problem

Cache invalidation is famously difficult. The cached data is a COPY of the DB — when the DB changes, the cache becomes stale.

### Strategies

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **TTL** | Auto-expire after N seconds | Simple | Data may be stale within TTL |
| **Write-through** | Update DB + cache together | Always fresh | Slower writes |
| **Write-behind** | Update DB, async update cache | Fast writes | Brief inconsistency |
| **Event-based** | DB change event → invalidate cache | Precise | Complex (event bus needed) |

### Why This Matters
ROAST uses TTL extensively (15-day DIVE snapshot, 24h breaking signal, 1h session). When you see stale data, it's usually a cache invalidation problem.

---

## 31. TTL — Time to Live

### Why TTL is Your Friend

```python
# Every cached item should have a TTL
redis.setex("key", 3600, value)  # expires in 1 hour

# Without TTL:
# - You might serve stale data forever
# - Redis might run out of memory
# - You need manual cleanup logic
```

### Common TTL Values

| Data Type | TTL | Rationale |
|-----------|-----|-----------|
| User session | 1 hour | Users re-login |
| API response | 5 min | Data changes moderately |
| Weather data | 30 min | Changes hourly |
| Static config | 24 hours | Rarely changes |
| Leaderboard | 1 hour | Updated periodically |
| Market data | 15 days | ROAST's DIVE cache |

### Why This Matters
Every redis.set() in your system should have a TTL unless you have a specific reason not to. Memory leaks in Redis happen because of missing TTLs.

---

## 32. Rate Limiting

### Why Rate Limit

- Prevent abuse (DDoS, brute force)
- Protect downstream APIs (they have limits too)
- Ensure fair resource allocation
- Control costs (pay-per-API-call services)

### Token Bucket (Simple)

```python
# Allow 100 requests per hour per user
# Key: rate_limit:{user_ip}

def check_rate_limit(user_ip: str, max_requests=100, window=3600):
    key = f"rate_limit:{user_ip}"
    current = redis.get(key)
    
    if current is None:
        # First request — set counter = 1
        redis.setex(key, window, 1)
        return True  # allowed
    
    current = int(current)
    if current >= max_requests:
        return False  # blocked
    
    redis.incr(key)  # increment counter
    return True
```

### Sliding Window (More Accurate)

```python
def sliding_window_rate_limit(user_id: str, max_requests=10, window_ms=60000):
    """Sliding window using sorted set"""
    key = f"ratelimit:{user_id}"
    now = time() * 1000  # milliseconds
    window_start = now - window_ms
    
    # Remove old entries
    redis.zremrangebyscore(key, 0, window_start)
    
    # Count current entries
    count = redis.zcard(key)
    
    if count >= max_requests:
        return False  # blocked
    
    # Add current request
    redis.zadd(key, {str(now): now})
    redis.expire(key, window_ms // 1000)  # TTL the key
    return True
```

### Why This Matters
ROAST uses rate limiting (3 free analyses/day). SYNAPSE uses 30 req/min per IP. SuperOwl uses Redis-based rate limiting for VAPI calls. Every production API needs this.

---

# PART 4: REAL-TIME & ASYNC

## 33. What is a WebSocket

### Plain Explanation

WebSocket is a persistent, bidirectional connection between client and server. Unlike HTTP (request → response → close), WebSocket stays open so either side can send data anytime.

### HTTP vs WebSocket

```
HTTP:
Client → "GET /data" → Server
Server → "{...data...}" → Client
Connection: CLOSED

HTTP again:
Client → "GET /data" → Server (new connection!)
Server → "{...data...}" → Client

WebSocket:
Client → "connect" → Server
Connection: OPEN (stays open)
Server → "new data!" → Client (push, no request needed!)
Client → "ok" → Server
Server → "more data!" → Client
...
(continues until one side disconnects)
```

### When to Use WebSocket

| Use Case | Why WebSocket |
|----------|---------------|
| Live chat | Both sides send messages anytime |
| Real-time updates | Server pushes data (sports scores, stock prices) |
| Streaming | Progress updates, logs, AI responses |
| Gaming | Low-latency state sync |
| Voice AI | Bidirectional streaming |

---

## 34. WebSocket vs HTTP

### FastAPI WebSocket Example

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await websocket.accept()
    try:
        while True:
            # Receive message from client
            data = await websocket.receive_text()
            
            # Process (e.g., get AI response)
            response = await get_ai_response(data)
            
            # Send response back
            await websocket.send_text(response)
            
            # Optional: send progress updates
            await websocket.send_json({"type": "progress", "step": 2})
    except WebSocketDisconnect:
        print(f"Client {client_id} disconnected")
```

### ROAST WebSocket Flow

```
Client → POST /api/analyse (upload resume)
Server → 202 Accepted (session created)

Client → WS /api/ws/{session_id} (connect for updates)
Server → {"type": "section_complete", "section": "red_flags", ...}
Server → {"type": "section_complete", "section": "market_pulse", ...}
Server → {"type": "complete", ...}

If WS disconnects:
Client → GET /api/session/{id}/state (poll every 5s until complete)
```

### Why This Matters
ROAST uses WebSocket for real-time resume analysis streaming. SuperOwl uses it for live call transcripts. the platform voice AI uses it for bidirectional audio streaming.

---

## 35. Async Python — async/await

### The Problem

```python
# Synchronous — blocks until complete
def process_user(user_id):
    user = db.fetch(user_id)    # waits... 100ms
    payment = stripe.charge()   # waits... 200ms
    email = send_email()        # waits... 100ms
    return user
# Total: 400ms (all sequential)
```

### The Solution

```python
async def process_user(user_id):
    # All three start immediately, run concurrently
    user_task = asyncio.create_task(db.fetch(user_id))
    payment_task = asyncio.create_task(stripe.charge())
    email_task = asyncio.create_task(send_email())
    
    # Wait for all to complete
    user = await user_task
    payment = await payment_task
    email = await email_task
    return user
# Total: ~200ms (the slowest single operation, not sum)
```

### How It Works

```
Event loop runs... runs task until await → pauses → runs other tasks → returns to await

This is NOT threads (parallel execution).
This is concurrency (efficient waiting).

Threads: 2 cooks working on different meals (context switching overhead)
Async: 1 cook starts rice, while rice cooks starts chopping vegetables (efficient)
```

### Why This Matters
ROAST runs multiple agents in parallel using `asyncio.gather`. SuperOwl handles multiple VAPI webhooks concurrently. AI agents need async to manage many tool calls.

---

## 36. asyncio.gather

```python
async def run_parallel_agents(context):
    # All 4 agents start at the same time
    results = await asyncio.gather(
        red_flag_agent.run(context),
        six_second_agent.run(context),
        competitive_agent.run(context),
        technical_depth_agent.run(context),
        return_exceptions=True  # don't crash if one fails
    )
    
    # results[0] = red_flag output
    # results[1] = six_second output
    # etc.
    
    return results
```

### Error Handling

```python
# Without return_exceptions: if one agent crashes, ALL crash
# With return_exceptions=True: failed tasks return Exception objects

results = await asyncio.gather(task1(), task2(), return_exceptions=True)
if isinstance(results[0], Exception):
    print(f"Task 1 failed: {results[0]}")
    results[0] = default_value  # use fallback
```

### Why This Matters
This IS ROAST's architecture — parallel agents using asyncio.gather. Understanding this gives you the mental model for how multi-agent systems work at the code level.

---

# PART 5: INFRASTRUCTURE & ARCHITECTURE

## 37. What is Docker

### Plain Explanation

Docker packages your application + all its dependencies into a portable **image**. You run the image as a **container**. Works the same everywhere — your laptop, the cloud, any server.

### Simple Dockerfile

```dockerfile
# FROM: start from a base Python image
FROM python:3.12-slim

# WORKDIR: set working directory inside container
WORKDIR /app

# COPY: copy your code into the container
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

# CMD: what to run when container starts
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Key Concepts

| Concept | Analogy | What it is |
|---------|---------|------------|
| **Image** | A recipe | Blueprint for a container (read-only) |
| **Container** | A cooked meal | Running instance of an image |
| **Dockerfile** | Recipe book | Instructions to build an image |
| **Registry** | Cookbook library | Where images are stored (Docker Hub) |

### Why This Matters
ROAST, SYNAPSE, and SuperOwl all run in Docker containers on Cloud Run. Docker is the standard deployment unit. Every backend engineer must understand it.

---

## 38. Event-Driven Architecture

### The Core Idea

Instead of services calling each other directly (tightly coupled), they communicate through EVENTS.

```python
# Tight coupling (bad):
class OrderService:
    def create_order(self, order):
        db.save(order)
        email_service.send_confirmation(order)       # direct call
        inventory_service.update_stock(order)        # direct call
        analytics_service.track(order)               # direct call
        # If email service is down, order creation fails!
```

```python
# Event-driven (good):
class OrderService:
    def create_order(self, order):
        db.save(order)
        event_bus.publish("order.created", order)
        # Done! Other services pick up the event
        # If email is down, order still created

# Separate services listen for events:
@event_bus.on("order.created")
def send_confirmation(order):
    email_service.send(order)  # runs independently

@event_bus.on("order.created")
def update_inventory(order):
    inventory_service.update(order)  # runs independently
```

### Why This Matters
SuperOwl is event-driven: VAPI webhook → FastAPI processes → FCM push, Slack notification, Firebase update — all triggered by the "call ended" event.

---

## 39. Pub/Sub Pattern

### How It Works

```
Publisher → Topic → [Subscriber A, Subscriber B, Subscriber C]

Publisher: emits message to topic (doesn't know who's listening)
Topic: receives message, delivers to all subscribers
Subscribers: each processes independently
```

### Code with Redis Pub/Sub

```python
# Publisher (Order Service)
import redis
r = redis.Redis()
r.publish("order_events", json.dumps({"type": "order.created", "order_id": 123}))

# Subscriber 1 (Email Service)
pubsub = r.pubsub()
pubsub.subscribe("order_events")
for message in pubsub.listen():
    event = json.loads(message["data"])
    if event["type"] == "order.created":
        send_email(event["order_id"])

# Subscriber 2 (Inventory Service) — runs separately
pubsub = r.pubsub()
pubsub.subscribe("order_events")
for message in pubsub.listen():
    event = json.loads(message["data"])
    if event["type"] == "order.created":
        update_inventory(event["order_id"])
```

### Why This Matters
the platform architecture uses Pub/Sub extensively (Google Pub/Sub is the backbone). Understanding this pattern is essential for designing scalable voice AI systems.

---

## 40. Message Queues

### Pub/Sub vs Queues

| | Pub/Sub | Queue |
|---|---------|-------|
| Delivery | To ALL subscribers | To ONE consumer |
| Use case | Broadcast event | Distribute work |
| Example | Email + SMS + Analytics all triggered by order | Background job: one worker processes it |
| Scaling | Add subscribers | Add more workers |

### Queue Example: Background Jobs

```python
# Producer (API endpoint)
@app.post("/api/analyse")
def submit_analysis(resume_data):
    # Enqueue the job, return immediately
    redis.lpush("analysis_queue", json.dumps(resume_data))
    return {"status": "queued", "id": job_id}

# Consumer (background worker — runs separately)
def worker():
    while True:
        # Block until a job is available
        _, job_data = redis.brpop("analysis_queue")
        resume = json.loads(job_data)
        
        # Run the heavy processing
        result = run_full_pipeline(resume)
        
        # Store result
        redis.set(f"result:{job_id}", json.dumps(result))
```

### Why This Matters
This pattern is EXACTLY ROAST's architecture. The `/api/analyse` endpoint enqueues work and returns immediately. The analysis runs as a background task. WebSocket streams progress.

---

## 41. Multi-Tenancy

### Plain Explanation

Multi-tenancy = one application serving MULTIPLE customers (tenants), with each tenant's data isolated from others.

### Approaches

| Approach | Isolation | Complexity | Cost | Example |
|----------|-----------|------------|------|---------|
| **Separate DB** | Strong | High | High | Banks, healthcare |
| **Separate Schema** | Medium | Medium | Medium | Enterprise SaaS |
| **Shared DB (tenant_id)** | Weak | Low | Low | Most SaaS |

### Shared DB Approach (Most Common)

```python
# Every table has a tenant_id column
class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    tenant_id = Column(Integer, nullable=False, index=True)  # ← tenant isolation
    customer_name = Column(String)
    amount = Column(Float)

# Every query ENFORCES tenant isolation
def get_orders(tenant_id, current_user):
    return db.query(Order).filter(
        Order.tenant_id == tenant_id,  # ← always filter by tenant!
        Order.created_by == current_user.id
    ).all()

# NEVER:
# db.query(Order).all()  ← exposes ALL tenants' data!
```

### Why This Matters
SuperOwl is multi-tenant (each business is a tenant). SYNAPSE could be multi-tenant (different domains). the platform the platform serves multiple enterprise customers with strict data isolation.

---

## 42. Circuit Breaker Pattern

### The Problem

One failing service can cascade into a system-wide failure:
- Service A calls Service B
- Service B is slow/failing
- Service A's threads get blocked waiting
- Service A runs out of resources
- Service C (that calls A) also fails
- Cascade...

### The Solution

**Three states:**
```
CLOSED (normal): requests pass through
    ↓ (failures exceed threshold)
OPEN (failing): requests fail immediately (fast-fail)
    ↓ (after cooldown period)
HALF-OPEN (testing recovery): allow one test request
    ↓ success → CLOSED
    ↓ failure → OPEN again
```

### Code

```python
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=5, cooldown=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.state = "CLOSED"
        self.last_failure_time = 0
        self.cooldown = cooldown
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.cooldown:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpen("Service unavailable")
        
        try:
            result = func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"  # recovery confirmed
            self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise
```

### Why This Matters
ROAST uses circuit breakers for each LLM provider. If Groq fails 3 times, it stops calling Groq for 5 minutes and uses fallbacks. This prevents cascading failures.

---

## 43. Fallback Chain

### The Pattern

Try primary provider. If it fails, try secondary. If that fails, try tertiary.

```python
async def call_llm_with_fallback(prompt):
    providers = [
        ("NVIDIA NIM", call_nim),
        ("Groq 70B", call_groq_70b),
        ("Groq 8B", call_groq_8b),
        ("OpenRouter", call_openrouter),
        ("Local model", call_local_model),  # always works
    ]
    
    for name, provider in providers:
        try:
            result = await provider(prompt)
            if result is not None:
                log_llm_call(name, success=True)
                return result
        except Exception as e:
            log_llm_call(name, success=False, error=str(e))
            continue  # try next provider
    
    raise RuntimeError("All LLM providers failed")
```

### Why This Matters
ROAST has EXACTLY this pattern for the ReviewAgent. Every agent has a fallback chain. ACARE uses this for the planner (NIM → Groq → deterministic). This is the standard pattern for LLM-dependent production systems.

---

## 44. Load Balancing

### Plain Explanation

Distributing incoming requests across multiple backend servers.

### Algorithms

| Algorithm | How | Best For |
|-----------|-----|----------|
| **Round-robin** | Server 1, 2, 3, 1, 2, 3... | Similar servers |
| **Least connections** | Send to server with fewest active connections | Uneven request durations |
| **IP hash** | Same client always goes to same server | Session affinity |
| **Weighted** | More powerful servers get more requests | Uneven server capacity |

### Why This Matters
Cloud Run handles this automatically for ROAST/SYNAPSE/SuperOwl. But understanding load balancing is essential for designing scalable systems.

---

## 45. CAP Theorem

### The Theory

A distributed system can have only TWO of three properties:

- **Consistency** (C): Every read gets the most recent write. All nodes see the same data.
- **Availability** (A): Every request gets a response (even if stale).
- **Partition Tolerance** (P): System continues working despite network failures between nodes.

### The Tradeoffs

```
CP (Consistency + Partition Tolerance):
  - Prefer when accuracy matters more than uptime
  - Example: Banking — I'd rather the ATM be down than show wrong balance
  - What happens: if nodes can't sync, system rejects writes

AP (Availability + Partition Tolerance):
  - Prefer when uptime matters more than perfect accuracy  
  - Example: Social media — better to show stale "likes" than show nothing
  - What happens: if nodes can't sync, system accepts writes anyway

CA (Consistency + Availability):
  - CANNOT exist in distributed systems (partitions WILL happen)
  - Only works on SINGLE machines
```

### Why This Matters
ROAST uses Redis = AP (available, may serve stale cache).
PostgreSQL = CP (strong consistency, may reject writes under partition).
When you choose between eventual consistency and strong consistency, you're applying CAP.

---

# PART 6: ENTERPRISE INTEGRATION PATTERNS (For the platform / Implementation)

## 46. Webhook Pattern

### Plain Explanation

Instead of polling for updates, the system sends an HTTP request to a URL when something happens. "I'll call you when I have news."

### Flow

```
the platform detects a customer wants to check order status
      ↓
the platform calls your webhook: POST /webhooks/order-status
Body: {"customer_id": "123", "order_id": "ORD-456"}
      ↓
Your system looks up the order in the enterprise DB
      ↓
Your system responds: {"status": "shipped", "eta": "2026-06-25"}
      ↓
the platform tells the customer: "Your order has been shipped and will arrive June 25th."
```

### Code Example

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.post("/webhooks/order-status")
async def handle_order_status(request: Request):
    """Called by the platform when a customer asks about their order"""
    data = await request.json()
    customer_id = data["customer_id"]
    order_id = data["order_id"]
    
    # Look up in enterprise system
    order = await enterprise_db.query(
        "SELECT status, eta FROM orders WHERE id = ?", order_id
    )
    
    # Return structured response — the platform will use it to reply
    return {
        "status": order["status"],
        "eta": order["eta"],
        "carrier": "FedEx",
        "tracking_url": f"https://fedex.com/track/{order['tracking']}"
    }


@app.post("/webhooks/ticket-create")
async def create_ticket(request: Request):
    """Called by the platform when a customer requests human support"""
    data = await request.json()
    # Create ticket in ServiceNow / Jira / Zendesk
    ticket_id = await servicenow.create_ticket(
        summary=data["issue"],
        customer_id=data["customer_id"],
        priority="medium"
    )
    return {"ticket_id": ticket_id, "status": "created"}
```

### Why This Matters
This is EXACTLY what implementation engineers build — webhooks that connect the platform to enterprise systems (CRM, ticketing, billing, order management).

---

## 47. API Gateway / Proxy Pattern

### Why Needed

Enterprise clients have existing systems (Salesforce, ServiceNow, custom databases). the platform can't call them all directly. You build a **translation layer**.

```
the platform → Your Integration API → Enterprise System
```

```python
# Your integration service translates the platform standard requests
# into enterprise-specific API calls

@app.post("/integrations/servicenow/create-ticket")
async def sn_create_ticket(customer_name: str, issue: str, severity: str):
    """
    Translates the platform request into ServiceNow's specific API format
    """
    # Map the platform severity to ServiceNow's priority
    priority_map = {"low": 3, "medium": 2, "high": 1}
    
    payload = {
        "caller_id": customer_name,
        "short_description": issue,
        "priority": priority_map.get(severity, 3),
        "category": "the platform Generated",
    }
    
    # Call ServiceNow's REST API
    headers = {"Authorization": f"Bearer {SERVICENOW_TOKEN}"}
    resp = requests.post(
        f"{SERVICENOW_URL}/api/now/table/incident",
        json=payload,
        headers=headers
    )
    return {"ticket_number": resp.json()["result"]["number"]}
```

### Why This Matters
This is the core of the an implementation engineer's integration work. You don't build the platform — you build the glue between the platform and the client's systems.

---

## 48. State Machine Pattern (the platform Automatas)

### Why State Machines for Conversation

A conversation has states:
```
GREETING → ASK_ISSUE → RESOLVING → CONFIRM → CLOSE
                         ↓ (if fails)
                   ESCALATE_TO_HUMAN
```

Each state has:
- What triggers entry to this state
- What the system does in this state
- What transitions are possible
- What data is collected

### Code Concept

```python
class ConversationStateMachine:
    def __init__(self):
        self.state = "GREETING"
        self.context = {}
    
    def transition(self, user_input):
        if self.state == "GREETING":
            self.context["intent"] = self.detect_intent(user_input)
            if self.context["intent"] == "billing":
                self.state = "COLLECT_ACCOUNT"
            else:
                self.state = "ROUTE_TO_DEPARTMENT"
        
        elif self.state == "COLLECT_ACCOUNT":
            account = self.extract_account(user_input)
            if account:
                self.context["account"] = account
                self.state = "LOOKUP_BILLING"
            else:
                self.state = "ASK_ACCOUNT_AGAIN"
        
        elif self.state == "LOOKUP_BILLING":
            result = self.billing_api.lookup(self.context["account"])
            if result["success"]:
                self.context["bill_info"] = result
                self.state = "PRESENT_BILL"
            else:
                self.state = "ESCALATE"
    
    def get_response(self):
        """What should the system SAY in current state"""
        responses = {
            "GREETING": "Hi! How can I help you today?",
            "COLLECT_ACCOUNT": "Could you provide your account number?",
            "LOOKUP_BILLING": "Let me check that for you...",
            "PRESENT_BILL": f"Your current balance is {self.context['bill_info']['amount']}",
            "ESCALATE": "Let me connect you to a specialist.",
        }
        return responses[self.state]
```

### Why This Matters
the platform automatas ARE state machines. Understanding state machines = understanding how the platform works under the hood. Your LangGraph experience (SYNAPSE) IS state machine expertise — mention it.

---

## Quick Reference

```python
# FastAPI patterns
@app.get("/items/{id}")
@app.post("/items", status_code=201)
@app.put("/items/{id}")
@app.delete("/items/{id}", status_code=204)

# Async patterns
async def parallel_tasks():
    results = await asyncio.gather(
        task_a(), task_b(), return_exceptions=True
    )

# Redis patterns
redis.setex("key", 3600, value)  # cache with TTL
redis.incr("counter")            # atomic counter
redis.lpush("queue", job)        # job queue

# Circuit breaker + fallback
for name, fn in fallback_chain:
    try:
        return await fn()
    except:
        continue
```
