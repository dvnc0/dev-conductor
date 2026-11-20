# Performance and Resources

Performance checks for the `code-review-and-quality` skill. Flag an issue only when you can name **the growth driver** (what gets bigger: rows, users, items per request, request rate) or **the hot path** (what runs often or on a latency-critical path). A slow line that runs once at startup is not a finding. Where you can, put a number on it: "one query per order line; a 200-line order makes 201 round trips".

Severity guide: an unbounded query or N+1 on a table or list that grows with users is usually **major**. Something that can take a service down (unbounded memory, missing timeout on a dependency that hangs, blocking the event loop) can be a **blocker**. Micro-optimizations are **minor** at most, and usually not worth mentioning.

---

## Data access

### N+1 queries
A query per item in a loop, often hidden behind lazy-loaded relations.

| Stack | What it looks like | Usual fix |
|---|---|---|
| Django | loop over a queryset touching `obj.related` / `obj.related_set.all()` | `select_related`, `prefetch_related` |
| SQLAlchemy | lazy relationship access in a loop | `selectinload`, `joinedload` |
| Rails / ActiveRecord | `each` touching an association | `includes`, `preload`, `eager_load` |
| Hibernate / JPA | lazy `@OneToMany` touched in a loop | `JOIN FETCH`, entity graphs, batch fetching |
| Entity Framework | navigation property in a loop, lazy loading proxies | `Include` / `ThenInclude`, projection |
| Prisma / TypeORM / Sequelize | `await` a query inside `for`/`map` | `include` / relations, one `IN` query |
| GORM / raw SQL in Go | `db.Find` or `QueryContext` inside a range loop | `Preload`, a single `IN` / join |
| GraphQL resolvers | per-field resolver hitting the DB | DataLoader-style batching |

Grep: an ORM call or `await` on a DB or HTTP client inside `for`, `forEach`, `map`, `each`, `range`, list comprehensions.

### Unbounded reads
- `SELECT` without `LIMIT` on a table that grows; `findAll()`, `.all()`, `Model.objects.all()`, `ToList()` on an unfiltered query.
- List endpoints with no pagination, or with a client-controlled page size and no maximum.
- Reading a whole file, response body, or upload into memory when it could be streamed.
- `COUNT(*)` on a large table on every request.

### Query shape
- Filtering, sorting, or joining on columns with no index (check the migration or schema for the index).
- Functions on indexed columns in `WHERE` (`LOWER(email) = ...`) defeating the index.
- `SELECT *` pulling large columns (blobs, JSON) that aren't used.
- Many small writes in a loop instead of one batch or bulk insert.
- Long transactions holding locks across network calls.

### Migrations
- Adding an index, a column with a default, or a constraint on a large table can lock it. Does the database and migration tool do it online? Is there a backfill plan?

---

## Async, blocking, and concurrency

- **Blocking the event loop or UI thread:** sync file or network APIs in Node (`readFileSync`, `execSync` on a request path), sync I/O or CPU-heavy work inside `async def` in Python, `.Result` / `.Wait()` in .NET async code (also risks deadlock), heavy work on the main thread in mobile or desktop UI.
- **Serial awaits that could run concurrently:** independent calls awaited one after another in a loop when they could be gathered (`Promise.all`, `asyncio.gather`, `errgroup`, `Task.WhenAll`), and the opposite: unbounded fan-out that fires 10,000 concurrent requests at a dependency. Concurrency needs a limit.
- **Thread or goroutine leaks:** goroutines blocked forever on a channel nobody closes; threads started per request with no pool; tasks never cancelled when the caller gives up.
- **Lock contention:** a global lock around slow work; a lock held across I/O.

---

## Network and external calls

- **Missing timeouts.** Several common clients have no timeout by default: Python `requests`, Go's `http.DefaultClient`, and `fetch` in Node and browsers. A dependency that hangs then hangs your workers. Check for an explicit timeout or a configured client.
- **Retries** without backoff and jitter, without a cap, or stacked at several layers (client retries × SDK retries × queue redelivery).
- **Chatty calls:** one call per item where a batch endpoint exists.
- **No caching** of something expensive, stable, and read often; or caching with no TTL or invalidation, which turns into a correctness bug.

---

## Memory and resources

- Collections, maps, caches, or queues that are appended to but never trimmed or evicted, especially module-level or static ones.
- Event listeners, subscriptions, observers, or intervals registered without being removed (front-end components and long-lived services alike).
- Connections, file handles, cursors, and streams opened without a guaranteed close: look for the language's scoped-cleanup construct (`with`, `using`, `defer`, `try-with-resources`, `finally`, RAII). A missing close on an error path is the usual leak.
- Large objects created inside hot loops; string concatenation in a loop building a big string in languages where that's quadratic.
- Loading an entire dataset to compute something the database could aggregate.

---

## CPU and algorithms

- Nested loops over two growing collections (O(n·m)) where a set or map lookup would make it linear.
- Repeated work inside a loop that could be hoisted (recompiling a regex, re-parsing config, re-sorting).
- Sorting or deduplicating repeatedly instead of once.
- **Catastrophic regex backtracking** on untrusted input: nested quantifiers like `(a+)+`, `(.*)*`, or alternation with overlap, in backtracking engines (JavaScript, Python `re`, Java, .NET without a timeout, Ruby, PHP PCRE). Go's `regexp` and Rust's `regex` are linear-time and immune. This is also a security issue (ReDoS).

---

## Front end

- Components re-rendering on every parent render because of new object, array, or function props created inline; missing memoization where the render is expensive and measured to matter.
- State stored too high in the tree so every keystroke re-renders the whole page.
- Long lists rendered in full instead of virtualized.
- Large dependencies added to the main bundle for a small feature; imports that defeat tree-shaking (`import _ from 'lodash'`).
- Images without dimensions (layout shift), unoptimized or oversized images, render-blocking scripts.
- Effects that fetch on every render because of a missing or wrong dependency array.

---

## What not to flag

- Micro-optimizations with no measured or obvious hot path.
- "Use a faster library" with no evidence the current one is the bottleneck.
- Premature caching. A cache is a consistency problem you're choosing to take on, so it needs a reason.
