# Deep Dive: Building a Version-Aware Code RAG Server over MCP

*How I turned a decade-old, multi-platform native codebase into something an AI agent can query by version, resolve log lines against, and trace call paths through — and the design decisions that made it cheap enough to keep running.*

---

## The problem

A large client-side product accumulates a specific kind of debugging pain. A customer sends a diagnostic bundle. It contains log lines from a build that shipped eight months ago. The engineer's checkout is three releases ahead. The log line says `connecting to a secret failed after so many retries` and there are four places in the tree that could have emitted it, two of which did not exist in the customer's version.

Grep-in-an-IDE does not scale to that. Neither does pointing an LLM at the repository and hoping: the codebase is hundreds of thousands of lines across C++, Objective-C, Objective-C++, C#, Java and Swift, and the answer depends on *which* version and *platform* you are asking about.

What I wanted was a service an agent could call with three questions:

1. **"Where does this log line come from, in version X?"** — exact file and line, not a guess.
2. **"Show me the code that does Y, in version X."** — semantic search scoped to a release.
3. **"What calls this function, and what does it call?"** — enough of a call graph to follow a path.

This article is about how I built that as a Model Context Protocol (MCP) server backed by Postgres, how the index stays cheap when every release is indexed, and what I got wrong on the way.

---

## 1. The shape of the solution

```
+-------------------+        +--------------------+        +--------------------+
|  indexer          | writes |  Postgres+pgvector | reads  |  MCP server        |
|  walk / chunk /   |------->|  chunks, log sites,|<-------|  5 tools over      |
|  summarize / embed|        |  symbol edges      |        |  streamable HTTP   |
|  extract / store  |        |                    |        |  or stdio          |
+-------------------+        +--------------------+        +--------------------+
   runs where the source is     writer + reader roles         read-only role
```

Two processes and a database. The **indexer** is pointed at a checkout and a version label; it walks, chunks, summarises, embeds, extracts, and pushes rows. The **MCP server** reads those rows and exposes tools to an agent. Neither knows about the other; the database schema is the contract.

### The privilege boundary

The database has three roles and each process gets exactly one:

| Role | Used by | Can do |
|---|---|---|
| superuser | bootstrap, backups | anything; never handed to an application |
| writer | indexer, migrations | owns every table; `CREATE` on the schema |
| reader | MCP server | `SELECT` only |

The writer *owns* the tables because a full reindex with a changed embedding dimension has to drop and recreate them, and only the owner can do that. The reader keeps `SELECT` after such a recreate because `ALTER DEFAULT PRIVILEGES FOR ROLE <writer>` grants it on anything the writer creates later. `CONNECT` and `CREATE` are revoked from `PUBLIC`. The MCP server, which is the piece facing an LLM, physically cannot modify the index even if it is convinced it should.

---

## 2. The schema: three tables, one trick

```
code_chunks   one row per function/method/class chunk, with embedding + tsvector
log_sites     one row per logging call site, with a regex derived from its format string
symbol_edges  one row per (caller, callee) pair, name-matched
```

The trick is in how versions are stored. The naive design gives every indexed version its own copy of every row. Index twenty releases of a codebase and you have twenty copies of the 95% of functions that did not change, plus twenty LLM summaries and twenty embeddings for each of them.

Instead, **a row is content-addressed and shared by every version it appears in.** Each table carries two parallel arrays, `refs[]` and `commit_shas[]`, with a `CHECK (cardinality(refs) = cardinality(commit_shas))` constraint. `refs[i]` is a version label like `mac_4.8.0`; `commit_shas[i]` is the commit that was checked out when that label was indexed. A chunk that is byte-identical at the same path and line range in six releases is one row with six entries in each array. Queries filter with `refs @> ARRAY[:ref]` against a GIN index.

### Two hashes, two jobs

The chunk table has two hash columns because two different questions need two different notions of "the same":

- **`content_hash`** = sha256 over `path | start_line | end_line | text`. This is *row identity*. If a function moves to a different file or its line range shifts, that is a new row, because the MCP server must return accurate line numbers for the version being asked about.
- **`body_hash`** = sha256 over the chunk text with its context header stripped. This is the *summary reuse key*. It is stable across path renames, import changes, and line shifts, so an unchanged function is never sent to the LLM twice, even when it becomes a new row.

The first time I designed this I had one hash and had to choose between wrong line numbers and paying for duplicate summaries. Two hashes cost one extra column and one extra B-tree index.

---

## 3. Indexing: what goes in and how it is cut

### Index what the compiler compiles

For the platform built with Xcode, the indexer does not glob the source tree. It parses the project file and indexes exactly the files in the compile phases. A large native repo is full of code that is checked in but not built for a given platform — old experiments, other platforms' sources, vendored SDKs. Indexing it wastes money and, worse, produces confidently wrong search hits for code that is not in the shipped binary. For other layouts, a heuristic skip list (vendored, generated, build output, binaries) plus `git ls-files` does the job.

### Chunk on the AST, never on character counts

Chunks are produced with tree-sitter, one per function, method, class, struct, or protocol, with per-language node type tables. Three refinements matter more than they look:

**Every chunk gets a context header.** The first line of the stored text is:

```
// repo:<name> | file:<relative path> | symbol:<Class.method> | imports:<first 20 import lines>
```

The bare body of a 12-line method carries almost no signal about what subsystem it belongs to. The header does, and it is embedded along with the body. It is also what `body_hash` strips off, so the header can change (a file moves) without invalidating the summary.

**Header files are one chunk each.** A C/ObjC header is declarations, enums, constants, protocols. Splitting it into AST fragments loses the cohesion that makes it useful. Whole-file, symbol = file stem.

**Oversized functions split at statement boundaries.** Beyond a configured line cap, the node is split at its child statement starts, each piece labelled `symbol[partN]`. This can cut across a variable's scope; the chunks are semantically approximate and that is accepted. What is *not* accepted is fixed-size character windows, which produce chunks that begin mid-expression and embed as noise.

There is a line-based fallback with overlap for languages without a grammar and for parse failures. It is logged as a fallback so its share is visible.

Chunks with an empty or whitespace-only body are dropped before storage. This was not a design principle; it was a bug fix. The embedding gateway rejects empty input with an HTTP 400, and one empty chunk failed an entire batch.

### Summaries, cached ruthlessly

Every chunk gets a one-sentence "purpose" summary from an LLM, prepended to the text that is embedded as `Purpose: <summary>\n\n<code>`. A short natural-language statement of intent moves the embedding measurably closer to the natural-language questions people ask.

It is also the expensive part. The initial index of a large codebase is well over a hundred thousand LLM calls. So:

- Summaries are looked up by `body_hash` before any call is made. Second and later versions of a codebase summarise only what changed.
- Calls run concurrently behind a semaphore, in dispatch batches that each commit to the database as soon as they finish. A crash loses at most one batch.
- There is a kill switch to index without summaries first and add them later.
- The body sent to the model is truncated at a fixed character cap.

### Embeddings and the dimension problem

The embedding model produces 3072-dimensional vectors. pgvector's HNSW index on the plain `vector` type caps at 2000 dimensions. The workaround is to index a half-precision cast:

```sql
CREATE INDEX ... ON code_chunks USING hnsw ((embedding::halfvec(3072)) halfvec_cosine_ops)
```

Two consequences you have to carry through the whole system: the query's `ORDER BY` must use the identical cast or the planner will not use the index, and the index must be built *after* data is loaded (the indexer creates it at the end of a run if it does not exist). The vector index is deliberately not in the migrations for this reason.

Embeddings are also reused. Before embedding, the indexer looks up existing vectors by `content_hash`; only rows without one are sent to the gateway, in committed batches.

### Full-text as a second opinion

Every chunk also gets a `tsvector` over its text, with a GIN index. Vector search is good at "code that handles reconnection after sleep" and bad at "the function called `onNetworkConnectedV2`". Full-text search is the reverse. The MCP server runs both and fuses them (§5).

---

## 4. The log-site index: from format string to regex

This is the part that pays for the rest. A logging call in source looks like:

```cpp
printf("connecting to a secret failed after so many retries");
```

A line in a customer's log looks like:

```
2032-03-14 10:22:01.334 connecting to a secret failed after so many retries
```

Matching one to the other, in the right version, is a static-analysis problem:

**1. Know your logging macros.** A pattern file lists every logging call shape in the codebase — one line per macro or function, pipe-delimited: name, a regex that matches the call prefix on a source line, the level it implies, the languages it applies to, and whether it is deprecated. This file is maintained by hand. The first draft was generated by handing an LLM a prompt and a sample of the codebase, then corrected; adding a new macro is appending one line. Objective-C++ files get both the C++ and Objective-C patterns, because they freely mix both.

**2. Extract the format string.** After a pattern matches, take the first quoted string literal at or after the match end. Patterns that consume a leading constant argument land their match end right before the format string, so no special-casing is needed. Multi-line calls where the literal starts on the next line are a known miss; the site is still recorded with file, line, level and enclosing symbol, just without a template.

**3. Convert the format string to a regex.** Every printf/ObjC conversion (`%d`, `%s`, `%@`, `%llx`, `%.2f`, `%%`, and the rest) maps to a capture group; literal text is escaped. `%s` and `%@` become non-greedy `(.+?)`.

**4. Store the literal segments for prefiltering.** The non-specifier text of the template — `connecting to  :  failed after  retries` — goes into a `trigrams` column with a `pg_trgm` GIN index.

**5. Attribute the site to its enclosing function.** One tree-sitter walk per file builds a list of `(start, end, symbol)` intervals; each site takes the innermost interval containing its line. The same walk feeds the symbol graph.

At query time the MCP server does the cheap thing first and the expensive thing second: a trigram prefilter with `word_similarity(trigrams, :log_line) > threshold`, ordered by similarity, capped at a couple of hundred candidates — then a Python `re.search` of each candidate's regex against the raw line. `word_similarity`, not `similarity`, because the stored literal segments are short and the log line has a timestamp and level prefix in front of the message; `word_similarity` asks whether the short string matches *any substring* of the long one. `re.search`, not `re.match`, for the same reason.

The result is a list of `(path, line, level, template, symbol)` for the requested version. Usually one. When there are several, the agent gets all of them and the enclosing symbols disambiguate.

---

## 5. Retrieval: hybrid search fused with RRF

`query_code` runs two searches and merges them:

1. Embed the query.
2. **Vector**: top 4k by cosine distance using the halfvec HNSW index, filtered by repo and ref.
3. **Full-text**: top 4k by `ts_rank` against `websearch_to_tsquery`, same filters.
4. **Reciprocal Rank Fusion**: each result's score is the sum of `1 / (rank + 60)` over the lists it appears in. Items that show up in both lists rise; items that dominate one list still surface.
5. Return top k with path, line range, symbol, language, summary, full text, and score.

Over-fetching (`4k` from each) before fusion matters; fusing two lists of exactly k throws away the reordering that fusion exists to do. The constant 60 is the one from the original RRF paper and I did not find a reason to tune it.

### The symbol graph

`symbol_edges` is deliberately modest: for each call expression (or Objective-C message expression), the caller is the enclosing symbol from the interval map and the callee is the rightmost identifier of the callee expression — which handles `foo()`, `obj.method()`, `ns::func()`, `obj->fn()`, and `[obj selector:]` in one rule. It misses virtual dispatch, function pointers, macro-wrapped calls, and anything across repositories. It is name-matched, with no type resolution.

That is fine. Its job is not to be a compiler's call graph. Its job is to let an agent that has just resolved a log line to `ZSomething::reconnect` ask "what calls this?" and get a list of names to `find_symbol` next. Approximate and instant beats exact and unavailable.

---

## 6. Storage lifecycle: making reindexing safe

Every indexing run for a version does the same three steps against each table, in this order:

1. **Detach the ref.** Remove this version's entry from `refs[]` and the matching entry from `commit_shas[]` on every row that lists it, `RETURNING id`. Remember those ids.
2. **Upsert.** Insert the produced rows with `refs = [this ref]`; on conflict with the `(repo, content_hash)` unique constraint, `array_append` the ref and commit instead. The `RETURNING xmax <> 0` idiom tells you which rows took the update branch, i.e. how many were shared with another version.
3. **Delete orphans.** Delete rows with `cardinality(refs) = 0` — but *only among the ids detached in step 1*.

The order and the id scoping both exist because of bugs. If you delete empty-refs rows *before* the upsert, then re-indexing a version that is the only one in the database detaches every row, sees them all empty, and deletes the entire index before re-inserting it — paying for every summary and embedding again. If you delete *all* empty-refs rows rather than only the ones you detached, a crashed run for a different version that left rows mid-detach gets swept away by yours.

Every batch is 500 rows, because asyncpg has a 32,767 bind-parameter limit and a chunk row has a lot of columns.

### Stages commit independently and verify themselves

Upsert, summaries, embeddings, full-text vectors, vector index, log sites, symbol edges — each stage commits as it goes. After summaries and after embeddings, a verification query asserts that every chunk in this ref has one. If not, the run raises and exits non-zero with a message that says: re-run the same command. Because every stage checks for existing work before doing it, a re-run is idempotent and only does what is missing. This turned a "the gateway rate-limited me at chunk 80,000" incident from a restart-from-scratch into a retry.

### The backup prompt

Every run begins with a blocking prompt that prints the target database host and exactly what the run will modify — detach-and-reattach one ref; delete every row for a product; or drop and recreate all tables — and asks the operator to type `yes` to confirm a backup has been taken. Anything else aborts before any write. `--yes` exists for unattended runs and logs a warning when used.

This is the least sophisticated thing in the system and it has prevented the most damage. The index costs real money and real hours to rebuild, and the destructive flags are one typo away from the normal ones.

---

## 7. The MCP server

The server is small on purpose. It uses a FastMCP-style framework, initialises the embedding client at startup (no model is loaded; it is a gateway client), and exposes five tools:

| Tool | Answers |
|---|---|
| `describe_index` | How many chunks, log sites, and edges exist for this repo/platform/version |
| `query_code` | Hybrid semantic + lexical search, top k |
| `match_log_line` | Which source call site(s) emitted this raw log line |
| `find_symbol` | Chunks whose symbol matches (exact matches ranked first, then substring) |
| `trace_symbol` | Callers and/or callees of a symbol |

Three design rules that came out of watching agents use it:

**Scope is mandatory on every tool.** Every call takes `repo`, `platform`, and `version`, validated (non-empty, no whitespace, bounded length), and the database ref is resolved as `{platform}_{version}`. There is no "search everything" mode. An agent that is allowed to omit the version will omit it, get hits from the wrong release, and reason confidently from them. `describe_index` exists so the agent can check that the version it was told about is actually indexed before it starts.

**Results carry what the agent needs to cite.** Every hit returns path, start and end line, symbol, the one-line summary, and the full chunk text. The agent does not need a second call to read the code, and it can quote `file:line` in its answer. The summary lets it skim twenty results and decide which three to read.

**Limits are bounded at the schema.** `limit` is `ge=1, le=25` for search, `le=200` for graph traversal; query strings have a maximum length. The reader role means a runaway agent can only ever waste CPU on the database, but bounding the parameters means it cannot even do much of that.

The transport is stateless streamable HTTP with JSON responses when deployed, or stdio for local use. Stateless matters: every request carries its full scope, so the server can be restarted or scaled without session bookkeeping.

### How it is actually used

The consumer is an agent skill for log-bundle analysis. Its loop, roughly:

1. `describe_index` for the bundle's reported version. If empty, stop and say so.
2. For each interesting log line, `match_log_line` → file, line, enclosing symbol.
3. `find_symbol` on that symbol → the code that emitted it, with context.
4. `trace_symbol` → who calls it; `find_symbol` on the interesting callers.
5. `query_code` for the natural-language question the evidence raises ("what happens when the tunnel reconnects after sleep").
6. Rank hypotheses, cite `file:line` for each, propose a confirming test.

Each of those tools does one thing. The composition is the agent's job.

---

## 8. Evaluation

Retrieval quality is easy to feel and hard to measure. I keep a small recall@K harness: a JSON file of `{query, expected}` pairs where `expected` is a symbol name or a file-name fragment, and a script that runs `search_code` for each and counts a hit if any top-K result matches the expected symbol exactly, contains it as a substring, or has it in the path. The matching is deliberately permissive; the question is whether the right *area* of code surfaces, not the exact line. Misses are printed in a table.

This is what told me that the context header and the summary prefix were each worth their cost, and that fusing with FTS fixed a class of exact-identifier queries that vector search alone missed. Without a number, every one of those would have been an argument.

---

## 9. Operations, briefly

The database is deployed as four containers in enforced order: Postgres with pgvector (health-checked, tuned for `shared_buffers`, `effective_cache_size`, and `maintenance_work_mem` — the last one matters for HNSW builds); a one-shot roles job that idempotently creates the writer and reader and their grants; a one-shot migrations job that runs as the *writer* so the writer owns what it creates; and a backup sidecar that rotates `pg_dump` output on last/daily/weekly/monthly schedules. Re-running the deployment is safe: roles re-sync, applied migrations are skipped, data is untouched.

Things that cost me time:

- The superuser password is read from the environment only when the data volume is *first initialised*. Changing it in `.env` later does nothing; you have to `ALTER ROLE` inside the container, and the symptom is the roles job failing to authenticate.
- Docker-published ports bypass the host firewall's zones. If you ever restrict the database port by source, the rules go in the `DOCKER-USER` chain, and a firewall reload flushes Docker's chains until Docker is restarted.
- A VPN client that accepts a TCP connection locally and then silently drops it produces "server closed the connection unexpectedly" with no auth error on the server side. `tcpdump` on the server showing nothing is the tell. An SSH tunnel is the workaround while the network team adds the port.

---

## 10. The checklist

If you are building a code RAG index for an agent:

1. **Version is a first-class dimension.** Scope every row and every query by it. Make it impossible to search "everything".
2. **Content-address rows and share them across versions.** Parallel `refs[]`/`commit_shas[]` arrays with a cardinality check. Most code does not change between releases; do not pay to store or summarise it twice.
3. **Use two hashes.** One for row identity (includes location, so line numbers stay right), one for expensive-derivation reuse (excludes location, so a moved function is not re-summarised).
4. **Index what the compiler compiles**, not what is in the tree.
5. **Chunk on the AST.** Prepend a context header. Whole-file for headers. Statement-boundary splits for giants. Never fixed-size character windows.
6. **Summarise and cache.** A one-line purpose statement prepended to the embedded text is worth the LLM cost — once.
7. **Check your vector index's dimension cap** before you pick an embedding model, and carry the cast into every query.
8. **Hybrid search, fused.** Vector for concepts, full-text for identifiers, RRF to merge, over-fetch before fusing.
9. **Log lines are a static-analysis problem.** Format string → regex; literal segments → trigram prefilter; `word_similarity` and `re.search` because real log lines have prefixes.
10. **An approximate call graph you have beats an exact one you do not.** Name-match, document the limitations, ship it.
11. **Detach, upsert, then delete orphans — scoped to what you detached.** In that order.
12. **Commit per stage, verify per stage, make re-runs idempotent.**
13. **One writer role that owns the tables, one reader role for anything an LLM can reach.** Default privileges so a recreate does not lock the reader out.
14. **Make the operator confirm a backup before every destructive run.** It is the least clever safeguard and it will save you the most.
15. **Keep the tool surface small and the results self-sufficient.** Path, lines, symbol, summary, text. The agent composes; the tools do one thing each.
16. **Measure recall.** Every retrieval improvement was an opinion until the harness said otherwise.
