# Module 1: Backend REST APIs (Node.js/TypeScript & Python/FastAPI)

---

## PART 1: Fundamentals (must know cold, no excuses)

- **HTTP verbs & idempotency**: GET (safe, no side effects), POST (create, not idempotent), PUT (replace, idempotent), PATCH (partial update), DELETE (idempotent). Idempotent = calling it 5 times has the same effect as calling it once.
- **Status codes**: 200/201/204 success; 400 bad request; 401 not authenticated; 403 not authorized; 404 not found; 409 conflict; 422 validation error; 500 server error. Know the difference between 401 and 403 cold — it's asked constantly.
- **Request/response anatomy**: headers, body, query params vs path params vs body params — know when to use each (path = identifies a resource, query = filters/options, body = data payload).
- **Statelessness**: a REST server should not rely on server-side session memory between requests; each request carries what it needs (typically a token).
- **Validation at the boundary**: never trust client input — validate types, required fields, and ranges before business logic runs.
- **Layered architecture**: Router/Controller → Service (business logic) → Repository/DAO (data access) → Database. Know this cold; it's the backbone of every API you've built.
- **Authentication vs authorization**: authentication = who you are (login, token issuance); authorization = what you're allowed to do (role/permission check).
- **Pagination & filtering**: any list endpoint over an unbounded dataset needs pagination (`page`/`limit` or cursor-based) — returning everything doesn't scale.
- **Idempotency keys**: for POST/PUT that create side effects, a client-supplied unique key lets the server recognize and safely ignore a duplicate retry.

---

## PART 2: Interview Questions & Answers

**Q1: What's the difference between PUT and PATCH?**
> PUT replaces the entire resource — you send the full object and whatever isn't included is treated as removed/reset. PATCH applies a partial update — you send only the fields that changed. PUT is idempotent by definition (sending the same full object twice leaves the same end state); PATCH can be idempotent depending on how it's implemented (e.g., "set field X to 5" is idempotent, "increment field X by 5" is not).

**Q2: How do you design pagination for an endpoint returning millions of rows?**
> I'd avoid offset-based pagination (`OFFSET 100000 LIMIT 20`) at that scale, since the database still has to scan and discard all the skipped rows, making later pages progressively slower. Instead I'd use cursor-based pagination — the client sends the last seen ID or timestamp, and the query does `WHERE id > last_id ORDER BY id LIMIT 20`, which uses an index seek regardless of how deep into the dataset you are.

**Q3: How would you version a REST API, and why does it matter?**
> I put the version in the URL path (`/v1/users`) since it's explicit and cache-friendly, though header-based versioning is a valid alternative for cleaner URLs. It matters because once external or internal clients depend on a response shape, changing that shape breaks them silently — versioning lets you introduce a breaking change as `/v2/...` while `/v1/...` keeps working for existing consumers until they migrate.

**Q4: A client complains a POST endpoint sometimes creates duplicate records when their network is flaky. How do you fix it?**
> Their client is likely retrying after a timeout without knowing if the original request actually succeeded server-side. I'd have the client send an idempotency key (a UUID generated once per logical operation) with the request; the server checks if it's already processed that key — if so, it returns the original result instead of creating a new record, making retries safe.

**Q5: How do you keep error responses consistent across a large API surface?**
> I define one error response shape (e.g., `{ "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [...] } }`) and route every thrown exception through a single centralized error-handling middleware that maps exception types to that shape and the right HTTP status — so no individual route handler has to remember to format errors correctly, and clients only need one error-parsing code path.

---

## PART 3: How It Works Internally

**What actually happens between "client sends a request" and "server responds"**:
1. Client resolves the server's domain to an IP (DNS), opens a TCP connection (reused via HTTP keep-alive if possible, or multiplexed over one connection in HTTP/2).
2. TLS handshake negotiates an encrypted channel (for HTTPS) — a symmetric session key is agreed on so the plaintext HTTP exchange that follows is encrypted in transit.
3. The client sends the HTTP request line (method + path + version), headers, and body over that connection.
4. The server's web framework (Express/FastAPI/etc., sitting on top of an HTTP server like Node's `http` module or Uvicorn) parses the raw bytes into a structured request object.
5. **Middleware chain** runs in order — commonly: logging → CORS handling → body parsing → authentication → your route handler. Each middleware can short-circuit (e.g., auth middleware returns 401 immediately without reaching your handler) or pass control forward.
6. The router matches the path+method against registered routes (internally often a tree/trie structure for efficient prefix matching, not a linear scan).
7. Your handler runs, typically calling into the service layer, which calls the repository layer, which executes a SQL query against the database driver's connection pool.
8. The response is serialized (usually to JSON) and written back over the same TCP connection; headers like `Content-Type` and `Content-Length` are set so the client knows how to parse it.

**Why validation-at-the-boundary works the way it does internally**: a library like Zod (TypeScript) or Pydantic (Python) defines a schema once, then parses the incoming raw object against it — on any mismatch (wrong type, missing required field) it throws a structured validation error *before* any of your business logic runs, which is what makes "fail fast with a clear 400" possible without scattering manual `if` checks everywhere.

**Why idempotency keys work**: the server maintains a small store (a DB table or Redis) mapping `idempotency_key → result`. On each request, it checks that store first — a cache-lookup pattern — before doing the actual work, so a network-retried duplicate request is caught before it re-executes the side effect.
