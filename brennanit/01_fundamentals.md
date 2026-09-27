# Level 1 — Fundamentals

Goal: be fluent in the basic building blocks the JD assumes you already have, so interviewers can move fast into intermediate/advanced discussion. Each topic below includes **what it is**, **how it works internally**, and what to be ready to explain.

---

## 1. Python Basics

**What it is**: Python's core data model and control flow.

**How it works internally**:
- Every Python value is an object with a type, a reference count, and (for containers) internal storage. `list` is a dynamic array (over-allocates capacity so `append` is usually O(1) amortized); `tuple` is a fixed-size immutable array, which is why it can be hashed and used as a dict key when its contents are hashable.
- `dict`/`set` are hash tables: a key's hash determines a bucket; collisions are resolved with open addressing. This is why average lookup is O(1) but worst case (bad hash distribution) degrades.
- Variables are names bound to objects (not boxes holding values). `a = b` makes `a` point to the same object as `b` — this is why mutating a list through one name affects the other, but reassigning `a = []` does not.
- CPython uses reference counting + a cycle-detecting garbage collector for objects that reference each other in a cycle (e.g., two objects pointing at each other).
- The GIL (Global Interpreter Lock) means only one thread executes Python bytecode at a time in CPython — this is why CPU-bound work uses multiprocessing, not threading, for real parallelism, while I/O-bound work (network calls, disk) benefits from threading or `asyncio` because the GIL is released during I/O waits.

**Be ready to explain**: difference between a shallow and deep copy (shallow copies the outer container but keeps references to the same inner objects); when you'd use a generator instead of a list (large or infinite sequences, streaming data — generator computes one item at a time instead of holding everything in memory).

---

## 2. Relational Databases & SQL

**What it is**: structured, related tables queried with SQL.

**How it works internally**:
- Data is stored in tables on disk in pages/blocks. An index (commonly a **B-tree**) is a separate sorted structure pointing to row locations, so the database can find a row in O(log n) instead of scanning every row (a "full table scan"). This is why indexing a column you filter/join on often turns a slow query fast.
- A `JOIN` is executed by the query planner choosing a strategy — nested loop join, hash join, or merge join — based on table sizes and available indexes. The planner picks this via a cost-based optimizer, which is why the *same* query can run differently depending on data volume.
- Transactions provide ACID guarantees via mechanisms like write-ahead logging (changes are logged before being applied, so a crash mid-write can be recovered) and locking or MVCC (multi-version concurrency control — readers see a consistent snapshot without blocking writers).

**Be ready to explain**: an N+1 query problem (looping over N records and issuing one query per record instead of a single joined/batched query) and how you'd fix it (`select_related`/`prefetch_related` in Django, or a single JOIN/`IN` query).

---

## 3. Git & Version Control

**What it is**: distributed version control tracking snapshots of a codebase.

**How it works internally**:
- Git stores content as objects in `.git/objects`, addressed by the SHA-1 (or SHA-256 in newer repos) hash of their content: **blobs** (file contents), **trees** (directory structure, pointing to blobs/trees), and **commits** (pointing to a tree plus parent commit(s), author, message). A branch is just a movable pointer (a file containing a commit hash) — this is why creating a branch is instant, not a copy of the whole codebase.
- A commit's hash changes if any content in its tree changes, which is why history is tamper-evident — changing an old commit changes its hash and every descendant commit's hash.
- `merge` creates a new commit with two parents, preserving both histories. `rebase` replays your commits on top of another branch's tip, rewriting commit hashes — this is why you don't rebase commits that have already been pushed and shared.

**Be ready to explain**: how you keep a feature branch in sync with a fast-moving main branch (regularly merge/rebase main into your branch); merge vs rebase trade-offs.

---

## 4. REST & HTTP Fundamentals

**What it is**: the request/response protocol the web (and most APIs) run on.

**How it works internally**:
- A client opens a TCP connection (or reuses one via HTTP/1.1 keep-alive / HTTP/2 multiplexing) to the server's IP and port, then sends an HTTP request: a method, path, headers, and optional body. The server parses this, routes it to a handler, and returns a status line, headers, and body.
- TLS (the "S" in HTTPS) wraps this in an encrypted channel via a handshake that negotiates a shared symmetric key, so the plaintext HTTP is encrypted in transit.
- "RESTful" statelessness means the server holds no session context between requests — each request carries everything needed to process it (e.g., a token in the `Authorization` header) rather than relying on server-side session memory.
- A JWT (JSON Web Token) is a base64-encoded header + payload + a signature (HMAC or RSA) computed over the header and payload. The server verifies the signature with a secret/public key to confirm the token wasn't tampered with, without needing to look anything up in a database — this is what makes it "stateless" auth.

**Be ready to explain**: difference between authentication (who you are) and authorization (what you're allowed to do); why idempotency matters for PUT/DELETE (retrying a failed request shouldn't cause a duplicate side effect).

---

## 5. Linux & Bash Basics

**What it is**: the OS environment most backend services run and deploy on.

**How it works internally**:
- Every running program is a **process** with its own memory space, tracked by the kernel via a process table; `ps`/`top` read this table. Processes communicate via pipes, sockets, or signals.
- File permissions (`rwx` for owner/group/other) are enforced by the kernel on every file access — `chmod`/`chown` change metadata the kernel checks before allowing a read/write/execute.
- A pipe (`|`) connects one process's stdout directly to another's stdin as a byte stream, handled by the kernel — no temporary file is written.
- Environment variables are key-value pairs attached to a process, inherited by child processes — this is why exporting a variable in one shell doesn't affect a sibling shell, only children spawned after the export.

**Be ready to explain**: how you'd find which process is using a given port (`lsof -i :PORT` or `ss -tulpn`), then stop it (`kill`, escalate to `kill -9` if needed).

---

## 6. Web Application Basics

**What it is**: the client-server structure of a web app.

**How it works internally**:
- MVT (Django) / MVC: a URL router maps an incoming path to a **view/controller** function, which queries the **model** (database layer) and renders a **template** or serializes data (for an API) back to the client.
- Sessions: the server stores state (often in a DB or cache like Redis) keyed by a session ID; the ID is sent to the client as a cookie on each request so the server can look the state back up. Tokens (JWT) instead embed the state/claims in the token itself, so no server-side lookup is needed.

---

### Self-check before moving on
You should be able to, without notes:
- Write a small Python function with proper error handling, and explain what object identity vs equality means.
- Write a SQL query joining two tables with a filter and aggregation, and say whether an index would help.
- Explain a Git branching workflow you've used, and what a commit hash actually represents.
- Describe what happens between a browser sending a request and a Django view returning a response, at the TCP/HTTP level.
