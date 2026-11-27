# 00.01 How the Web Works

**Running example:** Chai & Co (online tea shop). **Goal:** explain exactly what happens between tapping "Buy" and seeing "Order placed".

## 1. Client and Server

- **Client:** the program that asks (phone app, browser). **Server:** an always-on computer (usually in a data centre, rented from AWS / Google Cloud / Azure) that answers. The word also means the program doing the answering.
- **Request** = message out ("give me the teas"). **Response** = message back (the tea list).
- The client always speaks first. The server can't reach out on its own (push needs tricks: polling, WebSockets, mobile push).
- One server serves thousands of clients at once. The role is per conversation: the owner's laptop is a client when it sends a request.
- Servers can be clients too: Chai & Co's server calling a payment provider is that provider's client.
- **HTTP is stateless:** the server remembers nothing between requests. Login or cart state has to be re-sent (cookie/token) or stored server-side.

```
  +-----------+   request    +-----------+
  |  client   | -----------> |  server   |
  | (phone)   | <----------- | (Chai&Co) |
  +-----------+   response   +-----------+
```

**Why it matters later:** databases, caches and extra servers all sit behind the server. The phone still sees one request and one response.

## 2. What Happens When You Open the App

Five steps, always in this order (about 0.5 s in total):

| # | Step | What happens |
| --- | --- | --- |
| 1 | **DNS lookup** | chaiandco.com → 203.0.113.10 (name → IP address). The answer is cached. |
| 2 | **TCP connect** | Both sides agree to talk. TCP = reliable, ordered delivery (lost pieces are re-sent). |
| 3 | **Request** | `GET /teas` over the open connection. |
| 4 | **Server works** | Reads the request, queries the database, builds the answer. |
| 5 | **Response** | The tea list comes back on the same connection. The app draws it. |

- Most of the time is spent waiting for messages to cross distance. The server's own work is often the shortest part.
- Slow page? Ask which of the 5 steps is slow. A slow DNS, a far-away server and a slow database are three different problems with three different fixes.
- **DNS gotcha:** answers are cached for a TTL (time to live). If you change a server's IP, some users keep hitting the old one until the TTL expires. Low TTL = faster change but more lookups.
- **DNS layers:** phone → resolver (ISP, 8.8.8.8) → root → .com → the domain's own server. Record types: A (IPv4), AAAA (IPv6), CNAME (alias).
- **TCP gotchas:** the handshake costs a round trip, a new connection starts sending slowly (slow start), and one lost packet stalls the packets behind it (head-of-line blocking).

## 3. Round Trips and RTT

**RTT** (round-trip time) = how long one message there and back takes, in ms. It's set mainly by distance, so a faster server barely helps.

| Customer location | RTT |
| --- | --- |
| Same city | \~5–10 ms |
| Same country, far side | \~30–60 ms |
| Another continent | \~100–150 ms |
| Other side of the world | \~200–300 ms |

**Fresh HTTPS visit ≈ 4 round trips:** DNS (1) + TCP connect (1) + TLS secure setup (1; older versions 2) + request/response (1).

- RTT 80 ms → 4 × 80 = **320 ms** before data arrives, even if the server answers instantly. RTT 10 ms → about 40 ms.
- **Fewer round trips:** reuse the open connection (keep-alive) so the next request costs 1 RTT, not 4.
- **Shorter round trips:** put content closer to the customer (CDNs, regional servers).
- **Anti-pattern:** 10 sequential calls = 10 × RTT. Batch them or run them in parallel.
- **Physics floor:** light in fibre is roughly 5 ms one way per 1,000 km. No server upgrade beats that.
- TLS 1.3 needs 1 RTT, and HTTP/3 (QUIC over UDP) merges transport and encryption setup. Mobile networks add extra jitter.
- Total latency = RTTs + server time + transfer time (size ÷ bandwidth) + rendering. Latency and bandwidth are different things.

## 4. HTTP, HTTPS and Methods

- **HTTP** = the shared rules for writing requests and responses. Every request has a **method** (verb) and a **path** (the thing, e.g. `/orders/981`), and may carry a **body**.
- **HTTPS** = HTTP + encryption. Others on the café Wi-Fi can see who you talk to, but not what is said. Cost: +1 RTT, plus certificate management (an expired certificate causes an outage).
- HTTPS only protects data in transit. It doesn't make the app itself trustworthy.

| Method | Meaning | Chai & Co example | Safe | Idempotent |
| --- | --- | --- | --- | --- |
| GET | read | `GET /teas` | yes | yes |
| POST | create | `POST /orders` | no | **no** |
| PUT | replace whole thing | `PUT /teas/42` | no | yes |
| PATCH | change some fields | `PATCH /orders/981` (address only) | no | depends |
| DELETE | remove | `DELETE /cart/items/7` | no | yes |

- **Golden rule:** GET must never change data. Crawlers (Google bots) and link-preview tools call GET links automatically, so a "cancel order" GET link could cancel orders by accident.
- **Safe** = changes nothing. **Idempotent** = doing it N times has the same effect as doing it once. This decides which requests are safe to retry.
- **The lost-reply problem:** the app can't tell if the order went through. Retrying a GET is always safe. Retrying a POST can double-charge. The fix is an **idempotency key**:

```
 tap "Buy" -> app generates key K-7f3a (once per tap)
 POST /orders (Idempotency-Key: K-7f3a) --> server: new key, so place order 981
                                            and STORE the result under K-7f3a
 (response lost)
 app retries with the SAME key K-7f3a   --> server: key already seen, so return
                                            stored result (order 981). No 2nd order.
```

- Generate the key once per user intent and reuse it on every retry. The server stores key → result (with a TTL, e.g. 24 h). It must also handle two concurrent requests with the same key (use a unique constraint or lock).
- Requests and responses also carry **headers** (Content-Type, Authorization, Cache-Control). Filters and pagination go in the query string: `GET /teas?category=green&page=2`.
- Never put secrets in URLs: they end up in logs and browser history.

## 5. Status Codes

The first digit tells you whose problem it is:

| Family | Meaning | Whose fault |
| --- | --- | --- |
| 2xx | worked | nobody's |
| 3xx | look somewhere else | nobody's, follow the new address |
| 4xx | the request was wrong | client's, fix the request |
| 5xx | the server failed | server's, the request was fine |

| Code | Name | Example |
| --- | --- | --- |
| 200 | OK | `GET /teas` worked |
| 201 | Created | `POST /orders` made an order |
| 301 / 302 | Moved permanently / temporarily | `/shop` now lives at `/teas` |
| 400 | Bad request | quantity of −3 |
| 401 | Unauthorized | not logged in (server doesn't know who you are) |
| 403 | Forbidden | logged in, but customers can't delete teas |
| 404 | Not found | `GET /orders/99999` |
| 429 | Too many requests | rate limiting, slow down |
| 500 | Internal server error | a bug crashed the server mid-order |
| 503 | Service unavailable | overloaded or in maintenance, try later |

- **401 vs 403:** 401 = log in. 403 = I know you, and you may not do this. (The names are slightly off: 401 really means unauthenticated.)
- **500 vs 503:** 500 = something broke on this request (usually a bug). 503 = the server can't take requests right now.
- **4xx:** repeating the same request fails again. Show the user what to fix. **503:** wait and retry.
- **Also useful:** 204 No Content, 304 Not Modified (cache still valid), 409 Conflict, 422 Unprocessable. Behind a load balancer you'll see 502 (bad reply from the server behind it) and 504 (the server behind it timed out).
- **Retry properly:** only on transient failures (503, 504, network errors), and only if the call is idempotent or has a key. Use **exponential backoff with jitter** (1 s, 2 s, 4 s… plus random variation) and honour `Retry-After`. Otherwise you cause a **retry storm** that keeps an overloaded server down.
- Monitor the **error rate by family**. A spike in 5xx, especially 503 during a sale, is the classic sign that one server can no longer keep up. That is where Phase 01 starts.

## 6. APIs, Endpoints and JSON

- **API** = the published menu of requests a server understands and how to ask for them. The app, website and owner's laptop all use the same server through one API.
- **Endpoint** = method + path. A full description answers five questions: method, path, input, output, errors.

| Part | `POST /orders` |
| --- | --- |
| Method / Path | POST `/orders` |
| Input | body: which teas and how many, delivery address |
| Output | `201 Created` with order number and total |
| Errors | 400 empty cart · 401 not logged in · 503 overloaded |

- `GET /orders/{id}`: `{id}` is a placeholder for a real number such as `/orders/1042`. Errors: 404 if it doesn't exist.
- **JSON** = plain-text format for bodies. `{}` holds names and values, `[]` holds lists. Any language can read it, so iPhone, Android and web share one API.

```
POST /orders
{ "items": [{ "teaId": 42, "quantity": 2 }], "address": "12 Park Road" }

201 Created
{ "orderId": 981, "status": "placed", "total": 18.5 }
```

- **The API is a promise.** Renaming `quantity` to `qty` breaks every installed app (400 errors). Version it (`/v1/`) or only make backward-compatible changes: add optional fields, never rename or remove. Old app versions live on phones for months.
- **Pagination:** a `GET /teas` returning 10,000 items hurts latency and memory, so use `?limit=20&cursor=...`.
- **Validate on the server:** never trust the client's input.
- **Money gotcha:** floats like `18.5` can cause rounding errors. Production APIs often send integer minor units (`1850`) or strings.
- **Trade-off of JSON/REST:** simple and universal, but it can over-fetch or under-fetch (extra round trips; GraphQL addresses this), and JSON is bulkier than binary formats like gRPC.

## Summary: tapping "Buy"

```
 1. DNS lookup (if not cached)        -> 1 RTT
 2. TCP connect                       -> 1 RTT
 3. TLS handshake                     -> 1 RTT
 4. POST /orders + JSON + token       -> 1 RTT + server work
 5. 201 Created {orderId: 981}        -> app shows "Order placed"
```

- Latency ≈ round trips × distance. Cut the count or the distance.
- GET never changes data. Retry POST only with an idempotency key.
- 4xx = client fixes it. 5xx = the server's problem (503 can be retried with backoff).
- An API is a contract: changing it breaks installed clients.
- **Next:** 00.02 Where data lives (step 4, the database).