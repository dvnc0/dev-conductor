# Secure Defaults by Stack

Build-mode and fix reference for the `security-and-hardening` skill: the safe way to do common security-sensitive tasks in each ecosystem, and the unsafe pattern to look for. Prefer what the project already uses (its ORM, its auth library, its HTTP client) over introducing something new from this table. Library and API names drift between major versions, so check the project's installed version before recommending a specific call.

Stacks covered: **JS/TS** (Node and browser), **Python**, **Go**, **Java/Kotlin**, **C#/.NET**, **Ruby**, **PHP**, **Rust**.

---

## SQL queries

Rule: values go in as bound parameters, never by string building. Dynamic identifiers (table, column, sort direction) come from a hard-coded allow-list map.

| Stack | Safe | Unsafe |
|---|---|---|
| JS/TS | `pg`: `query('... WHERE id = $1', [id])`; `mysql2`: `execute('... = ?', [id])`; ORM query builders (Prisma, Knex, Drizzle, TypeORM) with bound values; Prisma `$queryRaw` tagged template | template literals or `+` into `query()`; Prisma `$queryRawUnsafe` with input |
| Python | DB-API `cursor.execute("... = %s", (id,))` (or `?` for sqlite3); SQLAlchemy `text("... = :id")` with params; Django ORM | f-strings or `%` formatting into `execute()`; Django `.raw()` / `.extra()` with formatted strings |
| Go | `db.QueryContext(ctx, "... = $1", id)` (`?` for MySQL); sqlx, GORM with placeholders | `fmt.Sprintf` into a query string |
| Java/Kotlin | `PreparedStatement` with `setX`; JPA named or positional parameters; jOOQ; Spring `JdbcTemplate` with args | `Statement.execute` with concatenation; JPQL built with `+` |
| C#/.NET | `SqlCommand` with `Parameters.Add`; Dapper with an anonymous param object; EF Core LINQ or `FromSql` / `FromSqlInterpolated` | `FromSqlRaw` or `SqlCommand` with interpolated or concatenated strings |
| Ruby | ActiveRecord hash conditions `where(id: id)` or placeholders `where("id = ?", id)`; Sequel datasets | `where("id = #{id}")`, `find_by_sql` with interpolation, `order(params[:sort])` |
| PHP | PDO `prepare` + `execute([$id])`; Laravel query builder and Eloquent bindings | `query()` with concatenation; `DB::raw` / `whereRaw` with input |
| Rust | sqlx `query!` macros or `.bind()`; Diesel query DSL | `format!` into a query string |

NoSQL: never pass a request object directly as a filter. Build the filter from typed fields so `{"$ne": null}` can't arrive as a "string".

---

## Running processes

Rule: call the program directly with an argument list. No shell, so no shell parsing of input. Validate any argument that could be read as a flag (starts with `-`), or put `--` before user-supplied positional arguments.

| Stack | Safe | Unsafe |
|---|---|---|
| JS/TS | `execFile(cmd, args)`, `spawn(cmd, args)` without `shell: true` | `exec(string)`, `execSync(string)`, `spawn(..., { shell: true })` |
| Python | `subprocess.run([cmd, arg1, arg2])` | `shell=True` with input; `os.system`; `os.popen` |
| Go | `exec.CommandContext(ctx, name, args...)` | `exec.Command("sh", "-c", userString)` |
| Java/Kotlin | `ProcessBuilder(List.of(cmd, arg))` | `Runtime.exec(String)` with concatenation; `sh -c` |
| C#/.NET | `ProcessStartInfo` with `ArgumentList` and `UseShellExecute = false` | concatenated `Arguments` string; `cmd /c` with input |
| Ruby | `system(cmd, arg1, arg2)` (multi-arg form); `Open3.capture2(cmd, arg)` | backticks, `%x()`, single-string `system` with interpolation |
| PHP | `proc_open([$cmd, $arg], ...)` (array form, PHP 7.4+); otherwise `escapeshellarg` on every argument | `exec`, `shell_exec`, `system`, `passthru` with unescaped input |
| Rust | `Command::new(cmd).arg(a).arg(b)` | `Command::new("sh").arg("-c").arg(user_string)` |

---

## HTML output (XSS)

Rule: let the template engine or UI framework escape output for the context. Treat every "raw HTML" escape hatch as a review point. If user HTML must be rendered, sanitize it with a maintained allow-list sanitizer (e.g. DOMPurify in browsers; bleach-successor `nh3` in Python; `sanitize-html` in Node; HtmlSanitizer in .NET; `rails-html-sanitizer`; HTML Purifier in PHP; `ammonia` in Rust) and add a CSP.

| Stack | Auto-escaping | Escape hatches to review |
|---|---|---|
| React / Preact | JSX expressions | `dangerouslySetInnerHTML`, `href={userUrl}` (`javascript:` URLs) |
| Vue | `{{ }}` | `v-html`, dynamic `:href` |
| Angular | interpolation and bindings | `bypassSecurityTrust*`, `[innerHTML]` (sanitized, but check) |
| Svelte | `{ }` | `{@html}` |
| Plain DOM | `textContent`, `setAttribute` on safe attributes | `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write` |
| Jinja2 / Django | autoescape on (check Jinja2 `Environment(autoescape=...)`) | `|safe`, `Markup()`, `mark_safe`, `{% autoescape off %}` |
| Go | `html/template` | `text/template` for HTML; `template.HTML(userInput)` |
| Java | Thymeleaf `th:text`; JSP with JSTL `<c:out>` | `th:utext`; raw `<%= %>` |
| .NET | Razor `@value` | `Html.Raw`, `MarkupString` |
| Rails | ERB `<%= %>` | `raw`, `html_safe`, `<%== %>` |
| PHP | Twig, Blade `{{ }}`; `htmlspecialchars($s, ENT_QUOTES, 'UTF-8')` | Blade `{!! !!}`, Twig `|raw`, bare `echo` |

Also: user-controlled URLs in `href`/`src` need a scheme allow-list (`http`, `https`, `mailto`). JSON embedded in a `<script>` tag needs the framework's safe serializer, not string concatenation.

---

## Password hashing

Rule: a slow, salted, memory-hard or adaptive hash. Prefer Argon2id; scrypt or bcrypt are acceptable. Never a fast hash (MD5, SHA-*), with or without salt. If the framework has a built-in password hasher, use it.

| Stack | Use |
|---|---|
| JS/TS | `argon2` or `bcrypt` packages; Node `crypto.scrypt` |
| Python | `argon2-cffi`, `bcrypt`; Django's `make_password` / configured hashers; `passlib` in older code |
| Go | `golang.org/x/crypto/argon2` or `golang.org/x/crypto/bcrypt` |
| Java/Kotlin | Spring Security `Argon2PasswordEncoder` / `BCryptPasswordEncoder` / `DelegatingPasswordEncoder` |
| C#/.NET | ASP.NET Core Identity `PasswordHasher<T>`; or a vetted Argon2/bcrypt library |
| Ruby | `bcrypt` gem; Rails `has_secure_password` |
| PHP | `password_hash($p, PASSWORD_ARGON2ID)` or `PASSWORD_DEFAULT`, `password_verify` |
| Rust | `argon2` crate (RustCrypto) with `PasswordHasher` |

---

## Randomness for tokens, IDs, and keys

Rule: security-relevant values (session IDs, reset tokens, API keys, nonces, CSRF tokens) come from a CSPRNG, with at least 128 bits of entropy.

| Stack | Safe | Not for security |
|---|---|---|
| JS/TS | `crypto.randomBytes`, `crypto.randomUUID`, `crypto.getRandomValues` | `Math.random` |
| Python | `secrets.token_urlsafe`, `secrets.token_bytes` | `random` module |
| Go | `crypto/rand` | `math/rand` |
| Java/Kotlin | `SecureRandom` | `java.util.Random`, `ThreadLocalRandom` |
| C#/.NET | `RandomNumberGenerator` | `System.Random` |
| Ruby | `SecureRandom` | `rand`, `Random` |
| PHP | `random_bytes`, `random_int` | `rand`, `mt_rand`, `uniqid` |
| Rust | `OsRng` / `rand::rngs::OsRng`, `getrandom` | seeded non-crypto RNGs |

---

## Constant-time comparison

Rule: compare secrets (tokens, HMACs, signatures, API keys) with a constant-time function, after checking lengths if the function requires equal lengths.

| Stack | Use |
|---|---|
| JS/TS | `crypto.timingSafeEqual` (equal-length Buffers) |
| Python | `hmac.compare_digest` / `secrets.compare_digest` |
| Go | `crypto/subtle.ConstantTimeCompare`; `hmac.Equal` |
| Java/Kotlin | `MessageDigest.isEqual` |
| C#/.NET | `CryptographicOperations.FixedTimeEquals` |
| Ruby | `ActiveSupport::SecurityUtils.secure_compare`, `Rack::Utils.secure_compare` |
| PHP | `hash_equals` |
| Rust | `subtle::ConstantTimeEq` |

---

## Encryption and signing

- Use a high-level, authenticated construction from a maintained library: AES-GCM or ChaCha20-Poly1305 for encryption, HMAC-SHA-256 or Ed25519 for signing. Higher-level libraries (libsodium bindings, Tink, `cryptography`'s Fernet in Python) are better still.
- Never ECB mode; never reuse a nonce with the same key; never hard-code keys or IVs.
- Keys come from a KMS or secret manager and are rotatable.
- JWTs: verify the signature with a pinned algorithm list, and validate `exp`, `nbf`, `iss`, and `aud`. Never accept `alg: none`. Never use a "decode" function that skips verification for authorization decisions. Check the library's verify API for the installed version.

---

## Deserialization and parsing

Rule: only data formats (JSON, protobuf, safe YAML) into explicit types. Never native object deserialization of untrusted bytes.

| Stack | Safe | Unsafe with untrusted data |
|---|---|---|
| JS/TS | `JSON.parse` + schema validation (zod, valibot, ajv) | `eval`, `node-serialize`, `vm` |
| Python | `json`, `yaml.safe_load`, pydantic | `pickle`, `dill`, `shelve`, `yaml.load` without `SafeLoader`, `jsonpickle`, `torch.load` without `weights_only=True` |
| Go | `encoding/json` into structs | (rarely unsafe; watch for `reflect`-based plugin loading) |
| Java/Kotlin | Jackson without default typing; Gson into concrete classes | `ObjectInputStream`, `XMLDecoder`, XStream without allow-list, Jackson `enableDefaultTyping` / class-based `@JsonTypeInfo` |
| C#/.NET | `System.Text.Json`; Json.NET with `TypeNameHandling.None` | `BinaryFormatter`, `NetDataContractSerializer`, `LosFormatter`, Json.NET `TypeNameHandling` other than `None` |
| Ruby | `JSON.parse`, `YAML.safe_load` | `Marshal.load`, `YAML.load` / `YAML.unsafe_load` on older Psych, `Oj.load` in object mode |
| PHP | `json_decode` | `unserialize` (and `phar://` paths reaching file functions) |
| Rust | `serde` into concrete types | (rarely unsafe) |

**XML:** disable DTDs and external entities. Java: `factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)` on `DocumentBuilderFactory` / `SAXParserFactory`, and equivalent settings on `XMLInputFactory`. .NET: `DtdProcessing.Prohibit` (default in modern .NET, but check custom `XmlReaderSettings` and `XmlDocument.XmlResolver`). Python: `defusedxml`. PHP 8+: entity loading disabled by default; avoid `LIBXML_NOENT`. Go `encoding/xml` and Rust parsers don't resolve external entities.

---

## File paths

Rule: never let input choose a path directly. Generate server-side names, or resolve the final path and check it's inside the base directory (after resolving `..` and symlinks).

| Stack | Pattern |
|---|---|
| JS/TS | `const p = path.resolve(base, name); if (!p.startsWith(path.resolve(base) + path.sep)) reject` |
| Python | `p = (base / name).resolve(); p.is_relative_to(base.resolve())` (3.9+) |
| Go | `os.Root` / `OpenInRoot` (Go 1.24+); otherwise `filepath.Clean` + `filepath.Rel` and reject `..` |
| Java/Kotlin | `base.resolve(name).normalize()` then `startsWith(base)`; use `toRealPath` for symlinks |
| C#/.NET | `Path.GetFullPath(Path.Combine(base, name))` then `StartsWith(fullBase + Path.DirectorySeparatorChar)` |
| Ruby | `File.expand_path(name, base)` then `start_with?(base + File::SEPARATOR)` |
| PHP | `realpath()` then `str_starts_with($p, $base . DIRECTORY_SEPARATOR)` |
| Rust | `canonicalize()` then `starts_with(base)` |

Archives: apply the same check to every entry before extracting (zip-slip), and cap total extracted size and file count (zip bombs).

---

## Outbound HTTP

- **Timeouts on every client.** No default timeout: Python `requests`, Go `http.DefaultClient`, `fetch` (use `AbortSignal.timeout(ms)`). Others have long defaults (.NET `HttpClient` 100 s) that may need tightening.
- **TLS verification stays on.** Flags that turn it off: `rejectUnauthorized: false`, `NODE_TLS_REJECT_UNAUTHORIZED=0`, `verify=False`, `InsecureSkipVerify: true`, trust-all `X509TrustManager` / `HostnameVerifier`, `ServerCertificateCustomValidationCallback` returning `true`, `OpenSSL::SSL::VERIFY_NONE`, `CURLOPT_SSL_VERIFYPEER => false`, `danger_accept_invalid_certs(true)`.
- **User-supplied URLs:** allow-list hosts, check the resolved IP against private ranges at connect time, and disable or re-validate redirects (see SSRF in the catalog).
- **Reuse clients** (.NET `IHttpClientFactory`, a shared Go `http.Client`) rather than creating one per request.

---

## Sessions and cookies

- Prefer the framework's session management.
- Cookies holding auth state: `HttpOnly`, `Secure`, `SameSite=Lax` or `Strict`, scoped `Path` and `Domain`, a sensible expiry.
- Rotate the session ID on login and privilege change; invalidate it on logout and password change.
- Don't store auth tokens in `localStorage` or `sessionStorage` when an `HttpOnly` cookie will do: any XSS can read web storage.
- Keep CSRF protection on for cookie-authenticated, state-changing requests.

---

## Mass assignment

Bind requests to explicit input types, not directly to persistence models.

| Stack | Safe | Unsafe |
|---|---|---|
| JS/TS | pick fields after schema validation | `Object.assign(model, req.body)`, `{...req.body}` into an update |
| Django / DRF | explicit `fields = [...]` | `fields = '__all__'` on writable serializers or forms |
| Spring | request DTOs | binding entities directly via `@ModelAttribute` / `@RequestBody` |
| ASP.NET | input models or `[Bind]` allow-lists | binding EF entities directly |
| Rails | strong params `.permit(:a, :b)` | `.permit!` |
| Laravel | `$fillable` | `$guarded = []` |

---

## Secrets

- Load from a secret manager or the environment, validated at startup (fail fast if missing).
- `.env` files are git-ignored; commit a `.env.example` with placeholders.
- Don't put secrets in build args, container images, client bundles, logs, error messages, URLs, or prompts.
- If one was committed, rotate it. Deleting it from the repo doesn't remove it from history or from anyone who cloned it.

---

## Dependency audit commands

Run only when allowed and when the command doesn't modify the repo. Most of these need network access.

| Ecosystem | Command |
|---|---|
| npm / pnpm / yarn | `npm audit`, `pnpm audit`, `yarn npm audit` (Berry) or `yarn audit` (v1) |
| Python | `pip-audit` |
| Go | `govulncheck ./...` (reports only vulnerabilities in code you actually call) |
| Java/Kotlin | OWASP Dependency-Check (Maven or Gradle plugin) |
| .NET | `dotnet list package --vulnerable --include-transitive` |
| Ruby | `bundle audit check --update` (bundler-audit) |
| PHP | `composer audit` |
| Rust | `cargo audit` |
| Containers / multi-ecosystem | `osv-scanner`, `trivy fs .` |
