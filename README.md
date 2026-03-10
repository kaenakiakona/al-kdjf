# StratForge Backend Test Summary

## 1. Scope, Frameworks, and Coverage Strategy

This document summarizes the backend test design for the current repository state. It covers:

- `services/gateway-node` automated unit and integration tests (`Vitest` + `Supertest`)
- `services/ingestor-node` automated unit and integration tests (`Vitest`)
- `services/worker-python` automated unit and integration tests (`unittest.TestCase`)
- backend-facing shell and Python smoke tests, including `scripts/e2e_docker_test.sh`

### 1.1 Test inventory

| Backend area | Primary framework | Suite files | Explicit test cases/methods | Primary focus |
| --- | --- | ---: | ---: | --- |
| `gateway-node` | Vitest | 45 | 808 `it(...)` cases | API validation, controller/service logic, Redis orchestration, virtualization, AI flows |
| `ingestor-node` | Vitest | 8 | 88 `it(...)` cases | Redis control plane, Alpaca feed consumers, publish pipeline, startup/shutdown wiring |
| `worker-python` | `unittest` | 13 | 78 `test_*` methods | Docker spawning, Redis/pubsub handling, strategy runtime, worker lifecycle |
| Shell/Python smoke scripts | Bash / Python | 7 | scenario-driven steps | Full-stack and operator-facing smoke verification |

The counts above come from the current source tree inventory, not from a fresh execution during this edit.

### 1.2 Common test design patterns

- `gateway-node` and `ingestor-node` both use global `setup.ts` files to silence `console.log` / `console.error`, keeping assertions deterministic and test output readable.
- `gateway-node` integration tests build ephemeral Express apps in memory, mount the real route modules, and drive them through `Supertest`. JWTs are generated with the test secret, while Prisma, Redis, Alpaca, Gemini, MCP, and OAuth boundaries are mocked.
- `gateway-node` unit tests isolate controllers, services, middleware, and route wiring. `beforeEach` almost always clears spies; timer-driven suites (`aiRateLimit`, `MarketChartService`, `TradeReconciliationService`) additionally enable fake timers and restore real timers in teardown.
- `ingestor-node` tests use mocked Redis clients, mocked WebSocket transports, and synthetic Alpaca messages. The unit suites validate normalization and error handling in isolation; the integration suites wire real service objects together while keeping Redis and WebSocket boundaries fake.
- `worker-python` tests use `patch.dict(sys.modules, ...)` to inject fake Redis, Docker, PostgreSQL, and model modules. `setUp` / `tearDown` methods construct repeatable in-memory brokers, fake pubsub queues, and fake Docker managers so no real infrastructure is required.
- The shell scripts are intentionally more operational than the automated suites. They exercise real HTTP endpoints, Docker containers, Redis, Postgres, and sometimes live or synthetic market data. Their oracles are exit codes, presence of IDs/tokens, JSON shape checks, container state, and log inspection.

### 1.3 Audit and completeness check

- This revision was reconciled directly against the current repository contents in `services/gateway-node/src/__tests__`, `services/ingestor-node/src/__tests__`, and `services/worker-python/tests`.
- The per-suite case counts below match the current source exactly: `gateway-node` has 45 suite files / 808 named `it(...)` cases, `ingestor-node` has 8 suite files / 88 named `it(...)` cases, and `worker-python` has 13 `unittest.TestCase` suites / 78 `test_*` methods.
- Every currently named `it(...)` / `test_*` case in those three service trees is represented in the per-suite inventories below. The shell/Python smoke scripts in Section 5 are supplemental verification and are not included in those totals.
- The suite writeups below intentionally call out not only happy paths, but also the edge conditions, validation boundaries, timeout/retry behavior, malformed payload handling, and degraded-dependency paths that make the implementation robust.
- Positive-path coverage is substantial throughout the repo: successful registration/login/OAuth, successful AI generation/research, successful asset/ticker/chart retrieval, successful session start/stop with Redis init-ack handling, successful strategy CRUD/version deployment, successful trade placement/cancellation/portfolio aggregation, successful ingestor auth-and-subscribe flows, and successful worker runtime signal generation are all covered below alongside the failure cases.

## 2. `gateway-node` Test Suites

Coverage emphasis: the `gateway-node` suites cover both the happy path and the failure envelope. The positive-path assertions validate successful auth flows, CRUD/list/detail endpoints, AI generation/modification/research responses, Redis-backed session startup, chart/ticker/portfolio payload shaping, and trade execution wiring. The same suites also pin request-shape validation, auth/authz failures, trust-tier gating, rate-limit boundaries, cache hit/miss behavior, Redis ack timing, fallback price resolution, and controller/service behavior when upstream dependencies throw malformed, timeout, or non-`Error` failures.

### 2.1 Unit suites

#### `src/__tests__/unit/AiController.test.ts` (31 cases)

Setup/teardown: direct controller-unit tests with mocked `GeminiService` functions and mocked Prisma user lookups; each case builds synthetic `req`/`res` objects and clears mocks in `beforeEach`.

`AiController.generate`:
Inputs: missing `prompt`, valid `{ prompt, symbol, name }`, non-string `symbol` / `name`, unavailable AI service, invalid strategy output, invalid JSON, timeout, and unknown exceptions.
Oracles: both the success path and the failure envelope: the controller must return `200` and delegate to `generateStrategy` correctly on valid input, while also mapping invalid/malformed upstream outcomes to `400`, `502`, `503`, and `504`.
Cases:
- `passes prompt, symbol, and name to GeminiService and returns the result`;
- `omits non-string symbol and name values`;
- `returns 400 when prompt is missing`;
- `returns 503 when AI service is unavailable`;
- `returns 502 when Gemini reports invalid strategy output`;
- `returns 502 when Gemini reports invalid JSON output`;
- `returns 504 when generation times out`;
- `returns 502 with detail for unknown generate errors`;

`AiController.modify`:
Inputs: missing `instruction`, missing `currentCode`, unauthenticated requests, low-trust users, valid trusted-user modifications with `currentParams`, unavailable AI service, malformed modified code/JSON, timeout, and unknown errors.
Oracles: successful trusted-user modification requests return `200` with correct `modifyStrategy` delegation, while invalid auth, trust-tier, validation, and upstream AI failures map to `400`, `401`, `403`, `502`, `503`, and `504`.
Cases:
- `passes currentParams through and defaults missing params to an empty object`;
- `returns 400 when instruction is missing`;
- `returns 400 when currentCode is missing`;
- `returns 401 when the request is unauthenticated`;
- `returns 403 when the user trust tier is below 2`;
- `returns 503 when modify is unavailable`;
- `returns 502 for invalid modify output`;
- `returns 504 when modify times out`;
- `returns 502 with detail for unknown modify errors`;

`AiController.research`:
Inputs: missing and non-string query bodies, successful research calls, disconnected MCP, `[MCP]`-prefixed errors, tool execution failures, timeout variants, and non-`Error` throws.
Oracles: successful research requests return `200` JSON responses with answer text and correct delegation to `GeminiService.research`, while validation, dependency, tool-execution, and timeout failures map to `400`, `502`, `503`, and `504`.
Cases:
- `returns 200 with answer on success`;
- `passes the query string through to GeminiService`;
- `returns 400 when query is missing`;
- `returns 400 when query is not a string`;
- `returns 400 when body is undefined`;
- `returns 503 when AI service is unavailable (no API key)`;
- `returns 400 when GeminiService rejects with "Query is required"`;
- `returns 503 when MCP client is not connected`;
- `returns 503 for MCP-prefixed errors`;
- `returns 502 when tool execution fails`;
- `returns 504 on AbortError (timeout)`;
- `returns 504 on timeout message`;
- `returns 502 with detail for unknown errors`;
- `handles non-Error thrown values`;

#### `src/__tests__/unit/AlpacaService.test.ts` (53 cases)

Setup/teardown: the Alpaca SDK constructor is fully mocked, `fetch` is stubbed for REST fallback paths, and `RedisService` is mocked for quote caching. `beforeEach` reloads the service with a clean mock state.

`AlpacaService.getAccount`, `getClock`, and `getPositions`:
Inputs: direct wrapper calls to account, clock, and position retrieval methods.
Oracles: the wrapper forwards each call without mutating the SDK response.
Cases:
- `delegates to alpaca.getAccount`;
- `delegates to alpaca.getClock`;
- `delegates to alpaca.getPositions`;

`AlpacaService.placeOrder`:
Inputs: standard market buys, whitespace/lowercase symbols, limit orders with `limit_price`, SDK errors with and without structured `response` payloads, nested Alpaca errors, and detail-free thrown objects.
Oracles: normalized order payloads and stable user-facing error messages.
Cases:
- `creates a market order successfully`;
- `uppercases and trims the symbol`;
- `includes limit_price when provided`;
- `throws formatted error when createOrder fails`;
- `throws generic message when error has no response`;
- `uses nested Alpaca error payloads when response metadata is absent`;
- `falls back to Unknown Alpaca error when the thrown object has no details`;

`AlpacaService.getOrders` and `cancelOrder`:
Inputs: default order-list requests, custom order-list requests, and cancellation requests.
Oracles: default option filling, custom option pass-through, and direct cancellation delegation.
Cases:
- `passes default options`;
- `passes custom options`;
- `delegates to alpaca.cancelOrder`;

`AlpacaService.getLatestPrices`:
Inputs: empty symbol arrays, stock snapshots, crypto snapshots, Redis cache hits, Redis cache failures, malformed cached JSON, Map-shaped and Array-shaped SDK responses, missing latest-trade prices, lower-case SDK field names, previous-daily-bar fallbacks, minute-bar crypto fallbacks, symbols with no data, Redis write-back failures, and full Alpaca failures.
Oracles: quote-retrieval robustness suite; cache short-circuit when valid, deterministic fallback order, `0` for unresolved symbols, and best-effort quote return even when cache writes fail.
Cases:
- `returns empty object for empty symbols array`;
- `fetches stock prices from snapshots`;
- `returns cached prices from Redis without calling Alpaca`;
- `falls back to Alpaca when Redis cache lookup fails`;
- `treats malformed cached JSON as a miss and refetches from Alpaca`;
- `falls back to getLatestTrade when snapshot has no price`;
- `handles crypto symbols via getCryptoSnapshots`;
- `falls back to latest crypto trades when crypto snapshots have no price`;
- `returns 0 for symbols with no data`;
- `handles Array-shaped snapshot response`;
- `reads lowercase trade fields from stock snapshots`;
- `handles Map-shaped snapshot response`;
- `falls back to prev daily bar prices when no latest trade is present`;
- `reads minute bar fallback fields from crypto snapshots`;
- `uses lowercase p when the stock latest trade fallback returns SDK-style fields`;
- `recovers when crypto snapshot lookup fails and uses lowercase crypto trade fields`;
- `writes fetched prices back to Redis`;
- `returns prices even when Redis write-back fails`;
- `returns empty object when Alpaca throws`;

`AlpacaService.getHistoricalBars`:
Inputs: stock REST retrieval, crypto SDK retrieval, uppercase/trim normalization, upstream HTTP non-OK responses, total upstream failure, fallback wider lookback windows, explicit start/end parameters on crypto fallback, oversized responses requiring truncation, symbol filtering for crypto bars, and missing `bars` collections.
Oracles: normalized OHLCV output arrays and empty-array safety on failure.
Cases:
- `fetches stock bars via REST`;
- `uppercases and trims the symbol`;
- `fetches crypto bars via SDK`;
- `returns empty array when API throws`;
- `returns empty array when the REST bars endpoint returns a non-OK response`;
- `falls back to wider window when primary returns empty`;
- `passes both start and end when a crypto request falls back to the historical window search`;
- `trims bars to requested limit`;
- `filters crypto bars by matching symbol`;
- `returns an empty array when the REST payload omits the bars collection`;

`AlpacaService.searchAssets`:
Inputs: stock vs crypto searches, symbol-prefix and name-substring matches, null asset names, no matches, and SDK failures.
Oracles: ranking quality (exact match first, prefix above name-only), 10-item limits, and empty-array safety.
Cases:
- `calls getAssets with us_equity for stock type`;
- `calls getAssets with crypto for crypto type`;
- `filters assets by symbol prefix (case-insensitive)`;
- `filters assets by name substring (case-insensitive)`;
- `returns exact symbol match first`;
- `ranks prefix matches above name-only matches`;
- `limits results to 10`;
- `returns empty array when no assets match`;
- `returns empty array when getAssets throws`;
- `handles assets with null name gracefully`;
- `matches name substring even when symbol does not match`;

#### `src/__tests__/unit/AssetController.test.ts` (12 cases)

Setup/teardown: controller-unit tests with a mocked Alpaca asset search and the controller’s in-memory cache exercised via repeated requests.

`AssetController.search`:
Inputs: missing/empty/whitespace query strings, successful stock and crypto searches, cached repeated requests, separate stock-vs-crypto cache keys, empty service results, thrown `Error`, and non-`Error` throws.
Oracles: both successful search behavior and defensive short-circuiting: valid requests should return simplified `200` results with correct type handling and cache reuse, while malformed or failed service calls should degrade to empty results or `500` responses as appropriate.
Cases:
- `returns 200 with simplified results on success`;
- `strips extra fields from asset results`;
- `defaults to stock type when type is not provided`;
- `passes crypto type through to service`;
- `returns empty array when service returns empty`;
- `returns cached results on second call with same query`;
- `uses different cache keys for stock vs crypto`;
- `returns empty results when q is missing`;
- `returns empty results when q is empty string`;
- `returns empty results when q is only whitespace`;
- `returns 500 when alpacaService.searchAssets throws`;
- `returns 500 on non-Error thrown values`;

#### `src/__tests__/unit/AuthController.test.ts` (25 cases)

Setup/teardown: mocked Prisma, bcrypt, JWT signing, and OAuth service boundaries; every group clears mocks in `beforeEach`.

`AuthController.register`:
Inputs: missing `email`, missing `password`, both missing, password length below minimum, existing-user collisions, valid registration bodies, optional `username`, and Prisma failure.
Oracles: successful user creation as well as validation/conflict handling: valid requests return `201`, hash the password, and normalize missing usernames to `null`, while invalid or duplicate requests map to `400`/`409` and persistence failures map to `500`.
Cases:
- `returns 201 on successful registration`;
- `hashes the password before storing`;
- `sets username to null when not provided`;
- `returns 400 when email is missing`;
- `returns 400 when password is missing`;
- `returns 400 when both email and password are missing`;
- `returns 400 when password is too short`;
- `returns 409 when user already exists`;
- `returns 500 when prisma throws`;

`AuthController.login`:
Inputs: missing fields, unknown users, OAuth-only users without password hashes, incorrect passwords, valid credentials, and Prisma failure.
Oracles: successful token issuance plus defensive rejection of invalid credential states.
Cases:
- `returns token and user on successful login`;
- `signs JWT with correct payload and expiry`;
- `returns 400 when email is missing`;
- `returns 400 when password is missing`;
- `returns 401 when user does not exist`;
- `returns 400 when user has no password (OAuth-only)`;
- `returns 401 when password is incorrect`;
- `returns 500 when prisma throws`;

`AuthController.me`:
Inputs: missing `req.user`, valid token for missing DB user, successful lookup with `trustTier`, and Prisma failure.
Oracles: both authenticated profile fetches and the `401/404/500` rejection paths.
Cases:
- `returns user with trustTier on success`;
- `returns 401 when req.user is missing`;
- `returns 404 when user is not found in DB`;
- `returns 500 when prisma throws`;

`AuthController.googleCallback`:
Inputs: missing `code`, successful OAuth exchange, argument pass-through, and OAuth failure.
Oracles: successful OAuth token exchange plus `400`/`401` handling for invalid callback states.
Cases:
- `returns token and user on success`;
- `passes the code to OAuthService.exchangeGoogleCode`;
- `returns 400 when code is missing`;
- `returns 401 when OAuthService throws`;

#### `src/__tests__/unit/DatabaseService.test.ts` (15 cases)

Setup/teardown: mocked Prisma session and strategy tables with clean spies per describe block.

`databaseService.createSession`:
Inputs: missing strategy records, successful session creation with structured output, and omitted `parameterOverrides`.
Oracles: thrown not-found errors or normalized return objects with default `{}` overrides.
Cases:
- `throws when strategy is not found`;
- `creates session and returns structured result`;
- `defaults parameterOverrides to empty object`;

`databaseService.stopSession`:
Inputs: successful stop-session updates and failed update attempts.
Oracles: update-and-return behavior on success plus graceful null return when the update path fails.
Cases:
- `updates session and returns result`;
- `returns null on error`;

`databaseService.hasActiveSessionsForSymbol` and `stopStrategySessions`:
Inputs: active vs inactive session lookups, running sessions vs none running, missing strategy metadata, and DB failures.
Oracles: boolean answers, stop counts, returned symbol/type metadata, empty-result handling, and propagated errors where expected.
Cases:
- `returns true when active sessions exist`;
- `returns false when no active sessions`;
- `stops all running sessions and returns result`;
- `returns empty result when no sessions running`;
- `returns null symbol/type when strategy not found`;
- `throws on database error`;
- `createSession propagates when prisma.sessions.create throws`;
- `hasActiveSessionsForSymbol propagates DB error`;
- `stopSession returns null when update throws (not found)`;
- `stopStrategySessions throws when updateMany fails`;

#### `src/__tests__/unit/GeminiService.test.ts` (27 cases)

Setup/teardown: mocked Gemini client, mocked MCP client/adapter, and deterministic fake chat/content responses; each describe block resets the model/chat mocks.

`generateStrategy()`:
Inputs: whitespace-only prompts, valid strategy-generation prompts with `symbol` and preferred `name`, invalid `parameters_json`, invalid JSON, missing `python_code`, and strategies not inheriting from `BaseStrategy`.
Oracles: sanitized prompt handling, appended context, safe empty-parameter fallback, and strict structural validation.
Cases:
- `throws when the prompt is empty after trimming`;
- `generates a strategy and appends symbol and preferred name context`;
- `returns empty parameters when parameters_json is invalid`;
- `throws when Gemini returns invalid JSON`;
- `throws when python_code is missing on_bar`;
- `throws when python_code does not inherit from BaseStrategy`;

`modifyStrategy()`:
Inputs: valid modification instructions, invalid returned `parameters_json`, invalid JSON, and missing `python_code`.
Oracles: preserved current params when needed, returned change summaries, and strict structural validation.
Cases:
- `modifies a strategy and preserves parsed params and change summary`;
- `falls back to currentParams when modified parameters_json is invalid`;
- `throws when modify returns invalid JSON`;
- `throws when modified code is missing python_code content`;

`research()`:
Inputs: empty/whitespace queries, missing API key, disconnected MCP, connected MCP with tool declarations, single-round tool calls, parallel tool calls, multi-round tool calls, null args, MAX_TOOL_ROUNDS enforcement, and several failure points (`executeFunctionCall`, `sendMessage`, `response.text`).
Oracles: correct tool-loop choreography, correct first user message, correctly formatted tool responses, and propagated errors on unrecoverable failures.
Cases:
- `rejects an empty string query`;
- `rejects a whitespace-only query`;
- `throws when GEMINI_API_KEY is missing`;
- `returns Gemini text when MCP is disconnected`;
- `does not pass tools to Gemini when MCP is disconnected`;
- `passes tool declarations when MCP is connected`;
- `executes a single tool-calling round`;
- `handles parallel tool calls in a single round`;
- `handles multiple sequential tool-calling rounds`;
- `respects the MAX_TOOL_ROUNDS (5) limit`;
- `handles tool call with null args gracefully`;
- `sends the user query as the first message in the chat`;
- `sends tool responses in the correct format`;
- `propagates error when executeFunctionCall throws during tool round`;
- `propagates error when Gemini sendMessage rejects`;
- `propagates error when response.text() throws`;
- `propagates error when sendMessage fails on a tool-response follow-up`;

#### `src/__tests__/unit/MarketChartController.test.ts` (11 cases)

Setup/teardown: mocked `MarketChartService` with synthetic Express `req`/`res`.

`MarketChartController.getChartData`:
Inputs: missing auth, missing or empty `symbol`, default params, custom `timeframe` and `limit`, valid `strategyParams` JSON, filtered non-string strategy params, invalid JSON, array-valued params, and service throws.
Oracles: `401/400/200/500`, symbol normalization, pass-through of valid query options, and safe rejection of malformed JSON.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 400 when symbol is missing`;
- `returns 400 when symbol is empty string`;
- `returns chart data on success with default params`;
- `uppercases and trims the symbol`;
- `passes custom timeframe and limit`;
- `parses valid strategyParams JSON`;
- `filters out non-string values from strategyParams`;
- `returns 400 for invalid strategyParams JSON`;
- `ignores strategyParams when value is an array`;
- `returns 500 when service throws`;

#### `src/__tests__/unit/MarketChartService.test.ts` (42 cases)

Setup/teardown: mocked `AlpacaService.getHistoricalBars`; fake timers are enabled for cache TTL and concurrency tests, then restored in `afterEach`.

`getMarketChartData`:
Inputs: default strategy rendering, symbol/timeframe normalization, unknown timeframe fallback, missing inputs, limit clamping (`min=10`, `max=500`), shorthand OHLCV property names, invalid timestamps, sorted timestamps, no bars, 2-minute aggregation including odd counts, cache hits, cache expiry, in-flight deduplication, distinct cache keys, array-valued extra params, CSV strategy params, `Date` timestamps, and oversized upstream arrays.
Oracles: normalized bars, deterministic cache behavior, stable aggregation, and safe defaults.
Cases:
- `returns bars and all 7 default strategies`;
- `uppercases and trims the symbol`;
- `normalizes timeframe aliases`;
- `defaults to 1min for unknown timeframes`;
- `defaults missing symbol, timeframe, and limit inputs to safe values`;
- `clamps limit to MAX_BARS (500)`;
- `clamps limit to minimum 10`;
- `normalizes bars with shorthand property names`;
- `filters out bars with invalid timestamps`;
- `sorts bars by timestamp ascending`;
- `returns empty bars when AlpacaService returns nothing`;
- `fetches double bars for 2min timeframe and aggregates`;
- `keeps the trailing bar when 2min aggregation receives an odd bar count`;
- `aggregated 2min bars combine OHLCV correctly`;
- `truncates aggregated 2min bars to MAX_BARS when the upstream response is oversized`;
- `caches response and returns cached on second call`;
- `cache expires after TTL`;
- `reuses the in-flight fetch for concurrent identical requests`;
- `uses different cache keys for different params`;
- `builds stable cache keys even when extra strategy params contain arrays`;
- `resolves custom strategy params from CSV strings`;
- `handles bars with Date objects for timestamp`;
- `truncates bars to MAX_BARS (500) when API returns more`;

`strategy signals`:
Inputs: synthetic bar series that force indicator crossings, flat-price regimes, widening ranges, and insufficient history.
Oracles: valid BUY/SELL labels, valid timestamps, alternating signals, no premature signals, correct strategy labels, and sensible no-signal behavior under neutral/flat conditions.
Cases:
- `all signals have BUY or SELL action`;
- `all signals have valid timestamps`;
- `strategies alternate BUY / SELL (no consecutive same-direction signals)`;
- `first signal is always BUY (position starts at 0)`;
- `produces no signals with insufficient data`;
- `strategy labels match expected names`;
- `emits both BUY and SELL for the A/D strategy when accumulation reverses`;
- `emits both BUY and SELL for the Aroon strategy when the oscillator crosses zero twice`;
- `emits both BUY and SELL for the MACD strategy with custom parameters`;
- `emits both BUY and SELL for the ADX strategy with custom parameters`;
- `emits both BUY and SELL for the RSI strategy with custom parameters`;
- `emits both BUY and SELL for the Stochastic strategy with custom parameters`;
- `handles flat bars without generating stochastic or ADX signals`;
- `treats symmetric range expansion as neutral for ADX`;
- `handles a steadily rising RSI series without creating reversal signals`;

`error handling`:
Inputs: upstream throw, missing OHLCV fields, NaN fields, and missing timestamps.
Oracles: propagated fatal upstream errors and robust defaulting/filtering for malformed bars.
Cases:
- `propagates error when AlpacaService.getHistoricalBars throws`;
- `handles bars with missing OHLCV fields (defaults to 0)`;
- `handles bars with NaN values by defaulting to 0`;
- `filters bars with missing timestamps entirely`;

#### `src/__tests__/unit/McpAdapter.test.ts` (13 cases)

Setup/teardown: pure adapter-unit tests with mocked MCP tools and a mocked MCP client.

`mcpToolsToGeminiFunctions`:
Inputs: simple tools, no required fields, empty tool lists, arrays, nested objects, enums, every JSON-schema primitive type, unknown schema types, missing descriptions, and multi-tool conversion.
Oracles: correct Gemini `FunctionDeclaration` shape and safe defaults.
Cases:
- `converts a simple MCP tool to a Gemini FunctionDeclaration`;
- `handles tools with no required fields`;
- `returns empty array when no tools are registered`;
- `converts array-type properties with items`;
- `converts nested object properties recursively`;
- `converts enum properties`;
- `maps all JSON Schema types to Gemini SchemaTypes`;
- `defaults unknown JSON Schema types to STRING`;
- `falls back to empty description when tool has none`;
- `converts multiple tools at once`;

`executeFunctionCall`:
Inputs: a successful MCP tool invocation for `get_quote` with `{ symbol: 'AAPL' }`, a rejected MCP invocation for the same quote lookup, and an arbitrary-argument historical-data invocation for `get_historical` with `{ symbol: 'TSLA', period1: '2024-01-01', interval: '1wk' }`.
Oracles: transparent delegation to `yahooFinanceMcpClient.callTool`, exact preservation of the tool name and argument object, return of the unmodified string payload from the MCP client, and propagation of MCP errors without wrapping or swallowing them.
Cases:
- `delegates to the MCP client and returns the result`;
- `propagates errors from MCP client`;
- `passes arbitrary args through to the MCP client`;

#### `src/__tests__/unit/McpClient.test.ts` (20 cases)

Setup/teardown: mocked MCP transport/client lifecycle with cached-tool state assertions.

`connect` / `isConnected` / `getTools`:
Inputs: enabled vs disabled MCP config, successful tool caching, pre-connect and post-disconnect states.
Oracles: correct lifecycle transitions and cached schema mapping.
Cases:
- `connects to the MCP server and caches tools`;
- `skips connection when config disables MCP`;
- `maps tool schemas to McpTool interface`;
- `returns false before connecting`;
- `returns true after successful connection with tools`;
- `returns false after disconnect`;
- `returns empty array when not connected`;
- `returns cached tools after connection`;

`callTool`:
Inputs: not-connected client state, single text result, multiple text parts, non-text parts, empty content arrays, and no-text content.
Oracles: correct concatenation and safe empty-string output.
Cases:
- `throws when not connected`;
- `calls the tool and returns concatenated text result`;
- `concatenates multiple text content parts with newlines`;
- `filters out non-text content parts`;
- `returns empty string when callTool result has no text content`;
- `returns empty string when callTool result has empty content array`;

`disconnect` / error handling:
Inputs: normal disconnect, double-disconnect, failed `connect()`, failed `listTools()`, and failed SDK `callTool()`.
Oracles: full state reset, `client.close()` invocation, idempotency, and transparent error propagation.
Cases:
- `resets all internal state`;
- `calls client.close()`;
- `is safe to call when already disconnected`;
- `propagates error when connect() fails`;
- `propagates error when listTools() fails after connect`;
- `propagates error when callTool SDK rejects while connected`;

#### `src/__tests__/unit/OAuthService.test.ts` (16 cases)

Setup/teardown: mocked `fetch` for Google token/profile exchanges and mocked Prisma user lookup/update/create paths.

`exchangeGoogleCode`:
Inputs: successful code exchange, token endpoint failure, profile fetch failure, nameless profiles, Bearer-token usage, and network exceptions.
Oracles: returned OAuth profile shape and propagated failures.
Cases:
- `returns OAuthProfile on success`;
- `sends the code to Google token endpoint`;
- `throws when Google token exchange fails`;
- `throws when Google profile fetch fails`;
- `sets name to null when profile has no name`;
- `uses Bearer token when fetching the profile`;
- `propagates error when fetch itself throws (network failure)`;
- `propagates error when profile fetch throws (network failure)`;

`findOrCreateUser`:
Inputs: pre-linked OAuth users, email-linking to existing password accounts, brand-new users, passwordless new-user creation, and failures from `findFirst`, `findUnique`, `update`, and `create`.
Oracles: correct linking/creation behavior with full error propagation.
Cases:
- `returns existing OAuth user when found`;
- `links OAuth to existing email account`;
- `creates a new user when no match found`;
- `creates user without password_hash`;
- `propagates error when prisma.findFirst throws`;
- `propagates error when prisma.findUnique throws`;
- `propagates error when prisma.update throws during link`;
- `propagates error when prisma.create throws for new user`;

#### `src/__tests__/unit/RedisService.test.ts` (12 cases)

Setup/teardown: mocked `redis.createClient` instances exercise publisher lifecycle, streams, cache helpers, and pipelines.

Positive path coverage:
Inputs: client construction, guarded connect behavior, publish plus set membership, stream operations, single-key cache helpers, batched cache helpers, TTL writes, and clean disconnect.
Oracles: successful RedisService behavior across connection, publish, stream, cache, and disconnect helpers.
Cases:
- `creates a Redis client with the configured URL and registers listeners`;
- `connects only when the publisher is closed`;
- `publishes messages and updates set membership`;
- `delegates Redis stream operations on the success path`;
- `supports cache reads, ttl-backed writes, and deletes`;
- `supports batched cache reads and writes, with and without ttl, and disconnects cleanly`;

Negative/edge coverage:
Inputs: already-open publishers, `BUSYGROUP` as success, unexpected stream-group errors, no-TTL writes, empty `mGet`, and empty `mSet`.
Oracles: no-op optimization or correct rethrowing.
Cases:
- `does not reconnect when the publisher is already open`;
- `treats BUSYGROUP as a successful xGroupCreate call`;
- `rethrows unexpected xGroupCreate errors`;
- `writes cache values without ttl metadata when none is provided`;
- `returns an empty list and skips Redis for empty mGet calls`;
- `skips pipeline creation for empty mSet calls`;

#### `src/__tests__/unit/SessionController.test.ts` (9 cases)

Setup/teardown: mocked `sessionService`, validator helpers, and fake Express `req`/`res`.

`SessionController.create`:
Inputs: invalid request bodies, successful session creation, explicit `parameterOverrides`, missing strategies, thrown `Error`, and non-`Error` throws.
Oracles: successful `201` startup responses plus correct `400/404/500` mapping for invalid input and service failures.
Cases:
- `returns 201 on successful session creation`;
- `passes parameterOverrides to startSession`;
- `returns 400 when validation fails`;
- `returns 404 when strategy is not found`;
- `returns 500 on unexpected error`;
- `returns 500 on non-Error throw`;

`SessionController.stop`:
Inputs: successful stop, missing session, and unexpected failure.
Oracles: both normal `200` shutdown and `404/500` error translation.
Cases:
- `returns 200 on successful stop`;
- `returns 404 when session is not found`;
- `returns 500 on unexpected error`;

#### `src/__tests__/unit/SessionService.test.ts` (28 cases)

Setup/teardown: mocked `DatabaseService`, `RedisService`, `AlpacaService`, `prisma.strategies`, `prisma.positions`, and the Redis ack client used by init-ACK waiting. Mocks are cleared per describe block.

`startSession`:
Inputs: missing strategies, happy-path session creation, merged default/base/override params, subscription-set updates, subscribe-event publishing, historical bar retrieval, returned `{ session, event }`, `warmup_bars` and `timeframe` overrides, existing-position lookup, invalid serialized strategy parameters, malformed init-ACK payloads, position-lookup failures, and repeated init-ACK timeout retries.
Oracles: correct Redis/database orchestration, resilient warm-start payload creation, and no crash on malformed acknowledgements.
Cases:
- `throws when strategy is not found`;
- `creates a session in the database`;
- `merges default, strategy base, and override params`;
- `adds symbol to Redis active subscriptions set`;
- `publishes subscribe event to Redis`;
- `fetches historical bars from AlpacaService`;
- `returns session and event`;
- `uses warmup_bars from effective params when present`;
- `uses timeframe from effective params when present`;
- `looks up existing position for the user/symbol`;
- `falls back to empty params when strategy.parameters is invalid`;
- `accepts malformed init ACK payloads without retrying`;
- `continues with a zeroed initial position when position lookup fails`;
- `continues after init ACK timeout and retries all configured attempts`;

`stopSession`:
Inputs: nonexistent sessions, last-session cleanup, symbol-still-active cases, and normal stop flow.
Oracles: unsubscribe publication, conditional active-subscription cleanup, and return of the stopped session.
Cases:
- `returns null when session does not exist`;
- `publishes unsubscribe event`;
- `removes symbol from Redis set when no other sessions remain`;
- `does not remove symbol from Redis when other sessions still active`;
- `returns the stopped session`;

`stopStrategySessions`:
Inputs: no running sessions, multiple stopped sessions, Redis cleanup, and missing-symbol defensive failures.
Oracles: stop counts plus returned symbol/type/session IDs.
Cases:
- `returns count 0 when no running sessions exist`;
- `publishes unsubscribe for each session`;
- `cleans up Redis when no remaining sessions for symbol`;
- `throws when symbol is missing from stopped strategy`;
- `returns count, symbol, type, and sessionIds on success`;

Error-handling coverage:
Inputs: database and Redis failures during session start/stop orchestration.
Oracles: DB and Redis exceptions propagate rather than being silently swallowed.
Cases:
- `startSession propagates when databaseService.createSession throws`;
- `startSession propagates when Redis publish throws`;
- `stopSession propagates when Redis publish throws`;
- `stopStrategySessions propagates when databaseService.stopStrategySessions throws`;

#### `src/__tests__/unit/SnapshotController.test.ts` (8 cases)

Setup/teardown: mocked `SnapshotService` with synthetic Express request/response objects.

`capture`:
Inputs: missing auth, successful capture, and service failure.
Oracles: successful `200` snapshot capture with the expected `{ rowsCreated }` shape, plus `401/500` handling for auth and service failures.
Cases:
- `calls SnapshotService.captureForUser and returns rowsCreated`;
- `returns 401 when user is not authenticated`;
- `returns 500 when service throws`;

`getSnapshots`:
Inputs: missing auth, successful snapshot lists, missing `days` defaulting, optional `strategyId`, and service failure.
Oracles: normalized numeric snapshot payloads on success plus `401/500` handling on failure.
Cases:
- `returns snapshots with numeric balance and pnl`;
- `defaults days to 30 when not provided`;
- `passes strategyId when provided`;
- `returns 401 when user is not authenticated`;
- `returns 500 when service throws`;

#### `src/__tests__/unit/SnapshotService.test.ts` (17 cases)

Setup/teardown: mocked Prisma snapshot/user/position tables and mocked `VirtualizationProxy.getVirtualPortfolio`.

`captureForUser`:
Inputs: users with no positions, multiple strategies, repeated positions for the same strategy, positions missing `strategy_id`, and prior snapshots already present.
Oracles: correct row counts, account-vs-strategy snapshot separation, strategy aggregation by notional balance, and pre-insert cleanup.
Cases:
- `creates an account-level snapshot and returns 1 when no positions`;
- `creates per-strategy snapshots based on positions`;
- `aggregates multiple positions for the same strategy`;
- `skips positions with no strategy_id`;
- `deletes existing snapshots before creating new ones`;

`captureAll`:
Inputs: multi-user iteration and one-user-fails scenarios.
Oracles: best-effort continuation across users while preserving total counts.
Cases:
- `captures snapshots for all users`;
- `continues capturing for other users when one fails`;

`getSnapshots`:
Inputs: account-level fetches, `strategyId` filtering, rolling `days` windows, and result-shape guarantees.
Oracles: correct Prisma `findMany` filters and field selection.
Cases:
- `fetches account-level snapshots when no strategyId`;
- `filters by strategyId when provided`;
- `applies date filter based on days parameter`;
- `returns results from prisma`;
- `selects only date, balance, and pnl fields`;
- `propagates error when prisma.findMany throws`;

Error handling:
Inputs: portfolio lookup failures plus snapshot write and delete failures.
Oracles: propagation of failures from portfolio lookup and snapshot persistence operations.
Cases:
- `captureForUser propagates when VirtualizationProxy throws`;
- `captureForUser propagates when prisma.create throws`;
- `captureForUser propagates when deleteMany throws`;
- `captureAll propagates when users.findMany throws`;

#### `src/__tests__/unit/StaticAnalysisService.test.ts` (26 cases)

Setup/teardown: pure functional tests with no external dependencies.

`StaticAnalysisService.analyzeCode`:
Inputs: safe code, blocked imports (`os`, `subprocess`, `sys`, `socket`, `requests`, `urllib`, `shutil`, `pathlib`, `http`, `ftplib`), comments and string-literal false positives, partial matches like `osutils`, blocked builtins (`eval`, `exec`, `__import__`, `compile`, `open`, `input`), multiple violations, empty input, and builtins reused as variable/method names.
Oracles: accurate violation detection without false positives from comments, strings, or identifier substrings.
Cases:
- `returns safe for clean code`;
- `flags import os`;
- `flags from subprocess import`;
- `flags import sys`;
- `flags import socket`;
- `flags import requests`;
- `flags from urllib import`;
- `flags import shutil`;
- `flags import pathlib`;
- `flags import http`;
- `flags import ftplib`;
- `does not flag imports inside comments`;
- `does not flag partial matches (e.g. "osutils")`;
- `flags eval()`;
- `flags exec()`;
- `flags __import__()`;
- `flags compile()`;
- `flags open()`;
- `flags input()`;
- `does not flag builtins in comments`;
- `reports multiple violations at once`;
- `returns safe for empty string input`;
- `does not flag blocked imports inside string literals`;
- `flags "from os import path" variant`;
- `flags indented imports`;
- `does not flag builtins used as variable/method names`;

#### `src/__tests__/unit/StrategyController.test.ts` (34 cases)

Setup/teardown: mocked Prisma strategy/template tables and transaction flow; synthetic Express `req`/`res`; mocks cleared per describe block.

`listTemplates`:
Inputs: normal template rows, `defaultParameters = null`, and DB failure.
Oracles: mapped template responses with `{}` defaults and `500` on persistence failure.
Cases:
- `returns mapped templates on success`;
- `defaults defaultParameters to {} when null`;
- `returns 500 on database error`;

`create`:
Inputs: unauthenticated requests, missing `name`, missing `symbol`, missing `pythonCode`, `pythonCode` lacking `def on_bar`, crypto-symbol inference via `/`, default stock inference when `assetType` missing, successful transaction-backed creation, and DB failure.
Oracles: `401/400/201/500`, symbol normalization, and correct inferred `assetType`.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 400 when name is missing`;
- `returns 400 when symbol is missing`;
- `returns 400 when pythonCode is missing`;
- `returns 400 when pythonCode lacks def on_bar`;
- `infers crypto asset type from symbol containing /`;
- `defaults to stock when assetType is missing and symbol has no /`;
- `creates strategy via transaction and returns 201`;
- `returns 500 on database error`;

`list`:
Inputs: unauthenticated users, mapped strategy rows, strategies without draft versions, and DB failure.
Oracles: `401/200/500` and correct `hasDraft` derivation.
Cases:
- `returns 401 when user is not authenticated`;
- `returns mapped strategies`;
- `hasDraft is false when no draft versions exist`;
- `returns 500 on database error`;

`getById`:
Inputs: unauthenticated access, missing strategy ownership, successful fetch, and DB failure.
Oracles: `401/404/200/500`.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 404 when strategy is not found`;
- `returns strategy on success`;
- `returns 500 on database error`;

`update`:
Inputs: unauthenticated access, invalid `pythonCode`, nonexistent strategies, successful update, explicit symbol uppercasing, and DB failure.
Oracles: `401/400/404/200/500`.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 400 when pythonCode lacks def on_bar`;
- `returns 404 when strategy does not exist`;
- `updates strategy and returns result`;
- `uppercases symbol when provided`;
- `returns 500 on database error`;

`stopSessions`:
Inputs: unauthenticated access, missing strategies, successful stop counts, and service failure.
Oracles: `401/404/200/500`.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 404 when strategy is not found`;
- `stops sessions and returns count`;
- `returns 500 on service error`;

`delete`:
Inputs: unauthenticated access, missing strategies, successful stop-then-delete flows, and DB failure.
Oracles: `401/404/200/500` with stop-before-delete behavior.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 404 when strategy is not found`;
- `stops sessions, deletes strategy, and returns result`;
- `returns 500 on database error`;

#### `src/__tests__/unit/StrategyDashboardController.test.ts` (14 cases)

Setup/teardown: mocked Prisma strategies/sessions/snapshots; controller computes summary metrics entirely in-process.

`StrategyDashboardController.getDashboardSummary`:
Inputs: unauthenticated access, no strategies, strategies with running sessions, sessions in error, stopped-only histories, no-session idle strategies, positions that imply capital allocation/exposure, session `final_pnl` values, mixed win/loss trade histories, no-trade histories, sufficient snapshot histories for Sharpe, insufficient snapshots, zero-standard-deviation snapshots, and DB failure.
Oracles: correct per-strategy status (`RUNNING`, `ERROR`, `STOPPED`, `IDLE`), exposure and capital calculations, total PnL, win/loss counts, null-or-numeric `winRate`, null-or-numeric Sharpe, and `500` on persistence failure.
Cases:
- `returns 401 when user is not authenticated`;
- `returns empty summary when user has no strategies`;
- `computes RUNNING status when strategy has running sessions`;
- `computes ERROR status when sessions have errors`;
- `computes STOPPED status when sessions exist but none running`;
- `computes IDLE status when no sessions exist`;
- `calculates capitalAllocated and exposure from positions`;
- `calculates totalPnl from session final_pnl values`;
- `calculates win/loss stats and winRate`;
- `returns null winRate when no sessions have trades`;
- `computes Sharpe ratio from portfolio snapshots`;
- `returns null Sharpe when fewer than 3 snapshots`;
- `returns null Sharpe when all balances are identical (zero std dev)`;
- `returns 500 on database error`;

#### `src/__tests__/unit/StrategyDetailController.test.ts` (8 cases)

Setup/teardown: mocked Prisma strategy detail queries with sessions, trades, and positions; no external services.

`StrategyDetailController.getDetail`:
Inputs: unauthenticated access, missing strategy ownership, rich strategy histories with trades/positions, no-trade histories, multiple active sessions, and DB failure.
Oracles: correct detail response shape, performance metrics, win/loss math, exposure aggregation, active-session counting, null `winRate` when appropriate, and `500` on DB error.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 404 when strategy is not found`;
- `returns strategy details with performance metrics`;
- `calculates win/loss and winRate correctly`;
- `returns null winRate when no sessions have trades`;
- `counts active sessions correctly`;
- `aggregates exposure across multiple sessions`;
- `returns 500 on database error`;

#### `src/__tests__/unit/StrategyVersionController.test.ts` (19 cases)

Setup/teardown: mocked Prisma strategy/version/user tables and transaction wrapper; mocked `StaticAnalysisService`.

`list`:
Inputs: unauthenticated access, missing strategies, successful version lookup, and DB failure.
Oracles: `401/404/200/500` and descending version ordering.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 404 when strategy is not found`;
- `returns versions for a valid strategy`;
- `returns 500 on database error`;

`createDraft`:
Inputs: unauthenticated access, low-trust users with submitted `pythonCode`, users missing trust-tier rows, missing parent strategies, static-analysis failure, happy-path creation with incremented version numbers, draft creation without trust-tier checks when code is omitted, using stored strategy code when body code is omitted, and DB failure.
Oracles: `401/403/404/400/201/500`, correct trust-tier enforcement, static-analysis gating, and next-version calculation.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 403 when user trust_tier < 2 and submitting pythonCode`;
- `returns 403 when user has no trust_tier record`;
- `returns 404 when strategy is not found`;
- `returns 400 when static analysis fails`;
- `creates draft with next version number and returns 201`;
- `skips trust tier check when no pythonCode submitted`;
- `uses strategy code when no pythonCode in body`;
- `returns 500 on database error`;

`deploy`:
Inputs: unauthenticated access, missing strategy ownership, missing version IDs, cross-strategy version misuse, successful transactional deployment, and DB failure.
Oracles: `401/404/200/500`, undeploy/deploy transactional behavior, and parent strategy update.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 404 when strategy is not found`;
- `returns 404 when version is not found`;
- `returns 404 when version belongs to different strategy`;
- `deploys version via transaction and returns success`;
- `returns 500 on database error`;

#### `src/__tests__/unit/TickerController.test.ts` (8 cases)

Setup/teardown: mocked `tickerService`; synthetic Express request/response objects.

`TickerController.getTicker`:
Inputs: successful quote retrieval, empty result arrays, pass-through of `req.user.id`, missing `req.user`, missing or empty `req.user.id`, thrown `Error`, and non-`Error` throws.
Oracles: `200/401/500` plus correct user-ID delegation.
Cases:
- `returns 200 with quotes on success`;
- `returns empty quotes array when service returns none`;
- `passes the correct userId from req.user`;
- `returns 401 when req.user is undefined`;
- `returns 401 when req.user exists but id is missing`;
- `returns 401 when req.user.id is empty string`;
- `returns 500 when tickerService throws an Error`;
- `returns 500 on non-Error throw`;

#### `src/__tests__/unit/TickerService.test.ts` (18 cases)

Setup/teardown: mocked Redis cache helpers, Alpaca quote/snapshot lookups, and Prisma position queries.

`TickerService.getTickerData`:
Inputs: all-benchmark cache hits, all-cache misses, user positions added to the benchmark watch list, overlapping benchmark/position symbols, repeated position symbols, previous-close edge cases (`prevClose = 0`), snapshot payloads as arrays or plain objects, Redis `mGet` failure, Redis `mSet` failure, Alpaca latest-price failure, previous-close failure, zero-price quotes, invalid cached JSON, falsy/empty position symbols, users with no positions, and mixed hit/miss caches.
Oracles: deduplicated symbol lists, correct `change` / `changePct` math, best-effort degraded operation under cache/API failures, exclusion of zero-price symbols, and proper cache write-back for misses.
Cases:
- `returns benchmark quotes from cache when all cached`;
- `fetches cache misses from Alpaca and caches them`;
- `includes user position symbols alongside benchmarks`;
- `deduplicates position symbols that overlap with benchmarks`;
- `deduplicates repeated position symbols`;
- `calculates change and changePct correctly`;
- `handles changePct=0 when prevClose is 0`;
- `handles getPrevCloses with Array snapshot format`;
- `handles getPrevCloses with plain object snapshot format`;
- `continues when Redis mGet fails`;
- `continues when Redis mSet fails`;
- `continues when Alpaca getLatestPrices fails`;
- `continues when getPrevCloses fails`;
- `excludes symbols with price=0`;
- `handles invalid JSON in cache gracefully (treats as miss)`;
- `filters out empty/falsy position symbols from DB`;
- `handles user with no positions (empty DB result)`;
- `mixed cache hits and misses are handled correctly`;

#### `src/__tests__/unit/TradeController.test.ts` (36 cases)

Setup/teardown: mocked `VirtualizationProxy`, `alpacaService`, `prisma`, and synthetic Express `req`/`res` objects; each describe block clears all mocks.

`placeOrder`:
Inputs: valid buy/sell payloads, omitted `type` / `time_in_force`, numeric string `qty` / `limit_price`, missing `symbol`, missing `qty`, missing `side`, and virtualization failures.
Oracles: normalized inputs to `VirtualizationProxy.executeOrder` and thrown validation/proxy errors.
Cases:
- `delegates to VirtualizationProxy.executeOrder with correct params`;
- `defaults type to market and time_in_force to day`;
- `converts qty and limit_price to numbers`;
- `throws when symbol is missing`;
- `throws when qty is missing (0)`;
- `throws when side is missing`;
- `propagates VirtualizationProxy errors`;

`listOrders`:
Inputs: unauthenticated access, absent `status`, upper/mixed-case `status`, invalid `status`, Alpaca orders with fallback field names, and Alpaca failures.
Oracles: `401/400/200/500` and stable order-field mapping.
Cases:
- `returns 401 when user is not authenticated`;
- `defaults status to all when not provided`;
- `lowercases the status query parameter`;
- `returns 400 for invalid status`;
- `maps order fields correctly including fallback properties`;
- `returns 500 when Alpaca throws`;

`handlePlaceOrder`:
Inputs: unauthenticated access, successful order placement, and thrown validation errors.
Oracles: `401/201/500`.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 201 on success`;
- `returns 500 when placeOrder throws (missing fields)`;

`cancelOrder`:
Inputs: unauthenticated access, trade ownership miss, successful cancel, and Alpaca cancel failure.
Oracles: `401/404/200/500`.
Cases:
- `returns 401 when user is not authenticated`;
- `returns 404 when trade is not found for user`;
- `calls alpacaService.cancelOrder and returns 200 on success`;
- `returns 500 when Alpaca cancelOrder fails`;

`listUserTrades`:
Inputs: unauthenticated access, trades with linked sessions, trades with `sessions = null`, and DB failure.
Oracles: mapped strategy names with `Unknown` fallback and `401/200/500`.
Cases:
- `returns 401 when user is not authenticated`;
- `returns mapped trades with strategy name`;
- `defaults strategyName to Unknown when sessions is null`;
- `returns 500 on DB error`;

`getPortfolio` and `getUserPositions`:
Inputs: unauthenticated access, exposure calculations, absent snapshot defaults, yesterday-snapshot PnL, grouped trade stats, current-price enrichment, `avgEntryPrice` fallback, empty positions, zero cost basis, and DB failure.
Oracles: computed portfolio totals, daily PnL, trade stats, enriched positions, zero-safe PnL percentages, and `500` handling.
Cases:
- `returns 401 when user is not authenticated`;
- `computes exposure metrics from positions`;
- `uses yesterdayBalance=100000 when no snapshot exists`;
- `computes dailyPnl from yesterday snapshot when available`;
- `aggregates trade stats correctly`;
- `returns 500 on error`;
- `returns 401 when user is not authenticated`;
- `enriches positions with current prices and computes P&L`;
- `falls back to avgEntryPrice when current price unavailable`;
- `skips getLatestPrices call when no positions`;
- `handles unrealizedPlPercent=0 when costBasis is 0`;
- `returns 500 on DB error`;

#### `src/__tests__/unit/TradeExecutionService.test.ts` (16 cases)

Setup/teardown: mocked Redis subscriber/publisher clients, mocked `VirtualizationProxy.executeOrder`, mocked Prisma session lookup; `beforeEach` rebuilds the service and captures the subscribe callback.

`lifecycle`:
Inputs: first `start()`, duplicate `start()`, `stop()`, and Redis error-listener registration.
Oracles: idempotent startup, two-client connection behavior, subscribed channel registration, and clean shutdown.
Cases:
- `connects to Redis and subscribes on start`;
- `does nothing if start is called twice`;
- `quits Redis connections on stop`;
- `registers error handlers on both clients`;

`processMessage`:
Inputs: valid BUY and SELL signals, crypto vs stock symbols affecting `time_in_force`, strategies without `asset_type`, missing sessions, execution failure, invalid JSON, valid `signalPrice`, `signalPrice = 0`, non-numeric `signalPrice`, and preservation of `client_signal_id`.
Oracles: correct order dispatch to `VirtualizationProxy`, failure publication when execution cannot proceed, non-throwing malformed-message handling, and correct optional `signalPrice` inclusion.
Cases:
- `executes a BUY signal via VirtualizationProxy`;
- `executes a SELL signal`;
- `uses gtc time_in_force for crypto assets`;
- `infers crypto from symbol containing "/" when strategy has no asset_type`;
- `uses day time_in_force for stock symbols`;
- `publishes failure when session is not found`;
- `publishes failure when VirtualizationProxy.executeOrder throws`;
- `does not throw on invalid JSON`;
- `passes signalPrice when price is valid`;
- `omits signalPrice when price is 0`;
- `omits signalPrice when price is not a number`;
- `includes client_signal_id in success result`;

#### `src/__tests__/unit/TradeReconciliationService.test.ts` (17 cases)

Setup/teardown: mocked Alpaca order polling and mocked `VirtualizationProxy.reconcileExecution`; fake timers are used to drive periodic polling and reset in teardown.

`lifecycle`:
Inputs: initial start calls, repeated start calls, and stop lifecycle calls.
Oracles: running-state setup, double-start idempotency, and stop semantics.
Cases:
- `start sets the service to running state`;
- `start is idempotent (calling twice does not error)`;
- `stop prevents further polling`;

`checkTrades`:
Inputs: `FILLED`, `CANCELED`, `REJECTED`, and `EXPIRED` orders; partially-filled/unmapped statuses; multi-order polls; empty polls; Alpaca polling failure; one-order reconciliation failure while others continue; `filled_at` present vs absent; price parsing from `filled_avg_price` vs `price`; `after` cursor use from `lastCheckTime`; and repeated poll intervals.
Oracles: correct status mapping, resilience to one bad order or one bad poll, correct fill timestamp selection, correct parsed price/qty values, and continued timer-driven polling.
Cases:
- `reconciles a FILLED order`;
- `reconciles a CANCELED order`;
- `reconciles a REJECTED order`;
- `reconciles an EXPIRED order`;
- `skips orders with unmapped statuses (e.g. partially_filled)`;
- `processes multiple orders in a single poll`;
- `does not call reconcileExecution when no orders returned`;
- `continues polling even when getOrders throws`;
- `continues processing remaining orders when one reconciliation fails`;
- `uses filled_at date when available`;
- `falls back to current date when filled_at is missing`;
- `parses filled_avg_price correctly, falls back to price field`;
- `passes after timestamp from lastCheckTime`;
- `polls again after checkIntervalMs`;

#### `src/__tests__/unit/VirtualizationProxy.test.ts` (33 cases)

Setup/teardown: mocked Alpaca quotes/order placement and mocked Prisma transaction, trade, user, and position tables.

`executeOrder`:
Inputs: valid BUY orders, valid SELL orders, pending-order conflicts, insufficient cash, insufficient holdings, missing live price without `signalPrice`, `signalPrice` fallback, and limit-order balance reservation using `limit_price`.
Oracles: trade creation, balance/position validation, correct price source selection, and correct error messages.
Cases:
- `validates balance and places a BUY order successfully`;
- `validates position and places a SELL order`;
- `throws when a pending order already exists for symbol`;
- `throws when balance is insufficient for BUY`;
- `throws when position is insufficient for SELL`;
- `throws when price cannot be fetched for BUY and no signalPrice`;
- `falls back to signalPrice when getLatestPrices returns 0`;
- `uses limit_price for balance validation on limit orders`;

`reconcileExecution`:
Inputs: filled buys, weighted-average buy updates, partial sells, full-position sells, canceled buys, canceled sells, missing trade records, already-final trades, trades with `userId = null`, transaction errors, and rejected orders treated as cancels.
Oracles: correct virtual-cash refunds/credits, position upsert/update/delete behavior, safe skipping of irreconcilable trades, and propagated transaction errors.
Cases:
- `processes a FILLED BUY order — refunds difference and upserts position`;
- `processes a FILLED BUY order — updates existing position with weighted avg`;
- `processes a FILLED SELL order — credits proceeds and updates position`;
- `deletes position when SELL fills entire holding`;
- `processes CANCELED BUY — refunds reserved cash`;
- `does NOT refund on CANCELED SELL order`;
- `skips processing when trade is not found`;
- `skips processing when trade is already in a final state`;
- `skips when userId is null`;
- `re-throws transaction errors`;
- `processes REJECTED orders same as CANCELED`;

`validateBalance`, `validatePosition`, `getVirtualBalance`, and `getVirtualPortfolio`:
Inputs: sufficient vs insufficient cash, sufficient vs insufficient holdings, missing users, missing positions, unavailable current prices, empty position sets, multiple positions, and alias behavior.
Oracles: thrown validation errors when appropriate, default `0`/`100000` values for missing users, price fallback to `avgEntryPrice`, correct aggregate portfolio math, and `calculatePortfolio` alias parity.
Cases:
- `resolves when balance is sufficient`;
- `throws when balance is insufficient`;
- `treats missing user as 0 balance`;
- `resolves when position is sufficient`;
- `throws when position is insufficient`;
- `treats missing position as 0 quantity`;
- `returns balance as number`;
- `returns 0 when user not found`;
- `returns portfolio with positions enriched with current prices`;
- `falls back to avgEntryPrice when current price is unavailable`;
- `returns empty positions when user has none`;
- `defaults cashBalance to 100000 when user not found`;
- `aggregates multiple positions correctly`;
- `calculatePortfolio is an alias for getVirtualPortfolio`;

#### `src/__tests__/unit/aiRateLimit.test.ts` (9 cases)

Setup/teardown: fake timers and an in-memory user counter map are reset between cases.

`aiRateLimit middleware`:
Inputs: unauthenticated requests, first requests for new users, exactly 20 requests in-window, the 21st request, post-window reset, independent user counters, accurate mid-window counts, exact-boundary allowance, and large-map pruning of expired users.
Oracles: `next()` vs `429`, correct per-user accounting, and state cleanup under load.
Cases:
- `allows unauthenticated requests through`;
- `allows the first request from a new user`;
- `allows up to 20 requests within the window`;
- `blocks the 21st request with 429`;
- `resets the counter after the window expires`;
- `tracks users independently`;
- `counts requests accurately across the window`;
- `allows exactly 20 requests at the boundary`;
- `prunes expired users when the in-memory store grows large`;

#### `src/__tests__/unit/assetRoutes.test.ts` (6 cases)

Setup/teardown: ephemeral Express app with the real route module and mocked controller/service behavior.

`assetRoutes`:
Inputs: route wiring, missing query strings, successful stock and crypto searches, service throws, and unknown subroutes.
Oracles: correct endpoint mounting, response bodies, and `404/500` behavior.
Cases:
- `GET /assets/search is wired correctly`;
- `returns results from the controller`;
- `returns empty results for missing query`;
- `passes type=crypto query parameter through`;
- `returns 500 when service throws`;
- `returns 404 for unknown routes under /assets`;

#### `src/__tests__/unit/authMiddleware.test.ts` (12 cases)

Setup/teardown: direct middleware-unit tests with generated JWTs and mocked Express `req`/`res`/`next`.

`authMiddleware`:
Inputs: missing auth headers, non-Bearer headers, invalid tokens, valid tokens, expired tokens, and tokens signed with the wrong secret.
Oracles: `401` rejection or successful `req.user` population.
Cases:
- `returns 401 when no authorization header`;
- `returns 401 when header does not start with Bearer`;
- `returns 401 when token is invalid`;
- `sets req.user and calls next() with valid token`;
- `returns 401 when token is expired`;
- `returns 401 when token is signed with wrong secret`;

`optionalAuth`:
Inputs: the strict middleware but should never reject the request when auth is absent/invalid.
Oracles: either `req.user` population or silent pass-through.
Cases:
- `calls next() without setting user when no header`;
- `calls next() without setting user when header is not Bearer`;
- `sets req.user and calls next() with valid token`;
- `calls next() without user when token is invalid`;
- `calls next() without user when token is expired`;

`JWT_SECRET`:
Inputs: module import of the exported JWT secret constant.
Oracles: the exported secret is a non-empty string.
Cases:
- `exports a non-empty string`;

#### `src/__tests__/unit/authRoutes.test.ts` (10 cases)

Setup/teardown: ephemeral Express app mounting the real auth routes with mocked controller/service dependencies.

`Route groups`:
Inputs: missing fields, successful registration/login/OAuth, missing auth headers, invalid tokens, valid tokens, and unknown subroutes.
Oracles: correct route mounting, auth middleware interaction, and `404` handling across registration, login, OAuth callback, and `/me`.
Cases:
- `is wired and returns 400 for missing fields`;
- `returns 201 on successful registration`;
- `is wired and returns 400 for missing fields`;
- `returns token on successful login`;
- `is wired and returns 400 when code is missing`;
- `returns token on successful OAuth`;
- `returns 401 without auth header`;
- `returns 401 with invalid token`;
- `returns user info with valid token`;
- `returns 404 for unknown routes under /auth`;

#### `src/__tests__/unit/sessionRoutes.test.ts` (10 cases)

Setup/teardown: ephemeral Express app with real `sessionRoutes`, signed JWTs, and mocked session/database services.

`auth guard`:
Inputs: missing and invalid JWTs on `POST /` and `POST /:id/stop`.
Oracles: both protected session routes reject unauthenticated requests before controller logic runs.
Cases:
- `POST / returns 401 without token`;
- `POST /:id/stop returns 401 without token`;
- `returns 401 with invalid token`;

`POST / (create)`:
Inputs: invalid bodies, successful startup, missing strategies, and internal errors.
Oracles: `400/201/404/500`.
Cases:
- `returns 400 when validation fails`;
- `returns 201 on success`;
- `returns 404 when strategy not found`;
- `returns 500 on internal error`;

`POST /:
Inputs: success, missing session IDs, and internal errors.
Oracles: id/stop`; `200/404/500`.
Cases:
- `returns 200 on successful stop`;
- `returns 404 when session not found`;
- `returns 500 on internal error`;

#### `src/__tests__/unit/sessionValidator.test.ts` (28 cases)

Setup/teardown: pure validation tests with no external dependencies.

`parseStrategyParams`:
Inputs: empty objects, scalar values (`string`, `number`, `boolean`), `null`, arrays, strings, blank keys, whitespace-only keys, nested objects, arrays as values, `null`/`undefined`, `NaN`, and `Infinity`.
Oracles: acceptance of only flat scalar dictionaries.
Cases:
- `accepts an empty object`;
- `accepts string, number, and boolean values`;
- `rejects non-object input (null)`;
- `rejects non-object input (array)`;
- `rejects non-object input (string)`;
- `rejects empty key`;
- `rejects whitespace-only key`;
- `rejects nested object values`;
- `rejects array values`;
- `rejects null values`;
- `rejects undefined values`;
- `rejects NaN`;
- `rejects Infinity`;

`validateStartSessionRequest`:
Inputs: valid stock/crypto requests, valid `parameterOverrides`, non-object bodies, missing/blank/non-string `symbol`, missing/blank `strategyId`, invalid `type`, invalid `parameterOverrides`, and omitted `parameterOverrides`.
Oracles: either a normalized valid payload or structured validation failure.
Cases:
- `accepts a valid request with symbol and strategyId`;
- `accepts optional type "stock"`;
- `accepts optional type "crypto"`;
- `accepts valid parameterOverrides`;
- `rejects non-object body (null)`;
- `rejects non-object body (string)`;
- `rejects missing symbol`;
- `rejects empty string symbol`;
- `rejects whitespace-only symbol`;
- `rejects non-string symbol`;
- `rejects missing strategyId`;
- `rejects empty string strategyId`;
- `rejects invalid type`;
- `rejects invalid parameterOverrides`;
- `includes no parameterOverrides when not provided`;

#### `src/__tests__/unit/snapshotRoutes.test.ts` (9 cases)

Setup/teardown: ephemeral Express app mounting real snapshot routes with mocked snapshot services.

`auth guard`:
Inputs: missing and invalid JWTs on `POST /capture` and `GET /`.
Oracles: both protected snapshot routes reject unauthenticated requests before controller logic runs.
Cases:
- `POST /capture returns 401 without token`;
- `GET / returns 401 without token`;
- `returns 401 with invalid token`;

`POST /capture`:
Inputs: successful snapshot capture and service failure.
Oracles: correct `{ rowsCreated }` response shape and `500` propagation.
Cases:
- `returns success with rowsCreated`;
- `returns 500 when service throws`;

`GET /`:
Inputs: returned snapshots, default `days = 30`, optional `strategyId`, and service failure.
Oracles: numeric response fields and `500` propagation.
Cases:
- `returns snapshots with numeric values`;
- `defaults days to 30`;
- `passes strategyId query param`;
- `returns 500 when service throws`;

#### `src/__tests__/unit/strategyRoutes.test.ts` (22 cases)

Setup/teardown: ephemeral Express app with real strategy routes, signed JWTs, and mocked controllers/services.

`auth guard`:
Inputs: missing and invalid JWTs across all protected strategy routes.
Oracles: all protected strategy routes reject unauthenticated requests before hitting controllers.
Cases:
- `GET / returns 401 without token`;
- `POST / returns 401 without token`;
- `GET /:id returns 401 without token`;
- `PUT /:id returns 401 without token`;
- `DELETE /:id returns 401 without token`;
- `POST /:id/stop returns 401 without token`;
- `returns 401 with invalid token`;

Endpoint wiring:
Inputs: template listing, missing create fields, invalid `pythonCode`, successful create, successful list, missing/successful get-by-ID, missing/successful update, missing/successful session stop, missing/successful delete, missing strategy for version listing, and successful version listing.
Oracles: route mounting, auth behavior, and response passthrough.
Cases:
- `returns templates list`;
- `returns 400 when required fields missing`;
- `returns 400 when pythonCode lacks on_bar`;
- `returns 201 on successful creation`;
- `returns strategies for authenticated user`;
- `returns 404 when strategy not found`;
- `returns strategy on success`;
- `returns 404 when strategy not found`;
- `returns 200 on successful update`;
- `returns 404 when strategy not found`;
- `stops sessions and returns count`;
- `returns 404 when strategy not found`;
- `deletes strategy and returns result`;
- `returns 404 when strategy not found`;
- `returns versions list`;

#### `src/__tests__/unit/tradeRoutes.test.ts` (22 cases)

Setup/teardown: ephemeral Express app with real trade routes, signed JWTs, and mocked market/portfolio/trading services.

`auth guard`:
Inputs: missing and invalid JWTs across the protected trade routes, including chart, orders, cancellation, history, portfolio, positions, and ticker endpoints.
Oracles: all protected trade routes reject unauthenticated requests before controller logic runs.
Cases:
- `returns 401 with invalid token` plus the parameterized `GET/POST/DELETE` cases generated by `it.each`;

Endpoint coverage:
Inputs: missing chart symbols, successful chart fetches, query pass-through, chart-service failure, mapped order lists, optional/invalid order status, successful order placement, placement failure, successful/unsuccessful cancellation, trade history, portfolio calculation, enriched positions, and ticker lookup.
Oracles: route wiring, auth enforcement, controller/service delegation, and `400/500/404/201/200` mapping.
Cases:
- `returns 400 when symbol is missing`;
- `returns chart data for authenticated request`;
- `passes query parameters through to controller`;
- `returns 500 when service throws`;
- `returns mapped orders on success`;
- `passes status query parameter`;
- `returns 400 for invalid status parameter`;
- `returns 500 when alpaca service throws`;
- `returns 201 on successful order placement`;
- `returns 500 when order execution fails`;
- `returns 200 on successful cancellation`;
- `returns 404 when order not found`;
- `returns 500 when cancellation fails`;
- `returns mapped trade history`;
- `returns 500 on error`;
- `returns portfolio data on success`;
- `returns 500 on error`;
- `returns enriched positions with current prices`;
- `returns 500 on error`;
- `returns ticker quotes on success`;
- `returns 500 when ticker service throws`;

### 2.2 Integration suites

Integration-suite setup pattern: each file constructs an in-memory Express app in `beforeAll`, mounts the real route modules, signs JWTs with the test secret, and resets mocked Prisma/service boundaries in `beforeEach`. No live DB, Redis, Alpaca, or Gemini access is required.

These integration suites are not limited to negative HTTP assertions. They also verify successful end-to-end route wiring for login and `/me`, AI generate/modify/research success responses, asset/ticker/chart reads, session creation and stop flows, strategy creation/update/delete/version deployment, snapshot capture/listing, order placement/cancellation, trade history, and portfolio/positions aggregation.

#### `src/__tests__/integration/aiStrategyEndpoints.integration.test.ts` (15 cases)

`src/__tests__/integration/aiStrategyEndpoints.integration.test.ts`:
Inputs: missing auth on generate, invalid auth on modify, missing/non-string prompts, missing `instruction`, missing `currentCode`, low-trust modify requests, happy-path generate and modify requests, missing `GEMINI_API_KEY`, invalid JSON, invalid modified strategy output, timeout exceptions, timeout messages, and shared rate-limit exhaustion across generate/modify.
Oracles: both successful end-to-end AI responses and defensive middleware/controller behavior: valid requests return `200` with normalized strategy payloads, while invalid auth/input and upstream AI failures map to `401/400/403/502/503/504`, and rate limits are enforced consistently across both endpoints.
Cases:
- `returns 200 with generated strategy data and passes symbol/name context through`;
- `returns 200 with modified strategy data for a trusted user`;
- `returns 401 for generate without an auth token`;
- `returns 401 for modify with an invalid auth token`;
- `returns 400 when generate is called without a prompt`;
- `returns 400 when generate prompt is not a string`;
- `returns 400 when modify is missing instruction`;
- `returns 400 when modify is missing currentCode`;
- `returns 403 when modify is requested by a low-trust user`;
- `returns 503 for generate when GEMINI_API_KEY is empty`;
- `returns 502 for generate when Gemini returns invalid JSON`;
- `returns 502 for modify when Gemini returns an invalid strategy`;
- `returns 504 for generate on AI timeout`;
- `returns 504 for modify when the AI reports a timeout message`;
- `enforces the shared AI rate limit across generate and modify endpoints`;

#### `src/__tests__/integration/authEndpoints.integration.test.ts` (16 cases)

`src/__tests__/integration/authEndpoints.integration.test.ts`:
Inputs: missing registration fields, short passwords, duplicate users, successful registration, missing login fields, bad credentials, OAuth-only users attempting password login, incorrect passwords, successful login, missing Google `code`, OAuth exchange failures, successful OAuth callback, missing auth on `/me`, invalid auth on `/me`, valid token for missing user, and successful `/me` after login.
Oracles: the full auth lifecycle: successful register/login/OAuth/profile fetch paths return correct `201/200` payloads, while invalid input, invalid credentials, bad tokens, and missing users map to `400/401/404/409`.
Cases:
- `returns 201 and creates a user on successful registration`;
- `returns 200 with a signed token on successful login`;
- `returns 200 with a signed token after a successful Google OAuth callback`;
- `returns the authenticated user profile from /me after a successful login`;
- `returns 400 when required registration fields are missing`;
- `returns 400 when registration password is too short`;
- `returns 409 when registration hits an existing user`;
- `returns 400 when login fields are missing`;
- `returns 401 when login credentials do not match a user`;
- `returns 400 when password login is attempted for an OAuth-only user`;
- `returns 401 when the password is incorrect`;
- `returns 400 when the Google OAuth callback code is missing`;
- `returns 401 when Google OAuth exchange fails`;
- `returns 401 when /me is requested without an auth token`;
- `returns 401 when /me is requested with an invalid token`;
- `returns 404 when /me uses a valid token for a missing user`;

#### `src/__tests__/integration/marketDataEndpoints.integration.test.ts` (16 cases)

`GET /api/trade/ticker`:
Inputs: missing/invalid auth, all-cache-hit benchmark + position quotes, Redis lookup failure, and position-lookup failure.
Oracles: successful quote responses plus `401/500` handling when auth or dependency lookups fail.
Cases:
- `returns benchmark and position quotes when cache entries are present`;
- `falls back to Alpaca when Redis lookup fails`;
- `returns 401 without a token`;
- `returns 401 for an invalid token`;
- `returns 500 when position lookup fails`;

`GET /api/trade/market-chart`:
Inputs: missing auth, missing symbol, invalid JSON for `strategyParams`, successful chart queries with symbol/timeframe normalization, 2-minute aggregation from 1-minute upstream bars, and upstream failure.
Oracles: normalized `200` chart payloads on success plus `401/400/500` handling for auth, validation, and upstream failures.
Cases:
- `returns chart data, normalizes the symbol, and applies timeframe aliases`;
- `aggregates 2min requests from 1min upstream bars`;
- `returns 401 without a token`;
- `returns 400 when symbol is missing`;
- `returns 400 when strategyParams is invalid JSON`;
- `returns 500 when chart loading fails upstream`;

`GET /api/assets/search`:
Inputs: missing `q`, successful stock searches, crypto searches, repeated identical searches hitting the in-memory cache, and upstream search failure.
Oracles: empty/successful `200` responses on valid or intentionally short-circuited requests plus `500` on service failure.
Cases:
- `returns simplified stock search results`;
- `passes through crypto asset searches`;
- `reuses the in-memory asset cache on repeated identical searches`;
- `returns empty results when q is missing`;
- `returns 500 when asset search fails`;

#### `src/__tests__/integration/researchEndpoint.test.ts` (11 cases)

`src/__tests__/integration/researchEndpoint.test.ts`:
Inputs: missing auth, invalid auth, missing `query`, numeric `query`, happy-path research without tools, happy-path research with one tool call, multiple tool calls in one round, missing API key, generic Gemini failure, rate-limit exhaustion, and response `content-type`.
Oracles: full-stack success cases for both direct-answer and tool-calling research flows, plus correct `401/400/502/503` handling and rate limiting for failure paths.
Cases:
- `returns 200 with a research answer (no MCP tools)`;
- `returns 200 after a tool-calling round`;
- `returns 200 with multiple tool calls in one round`;
- `returns JSON content-type`;
- `returns 401 without an auth token`;
- `returns 401 with an invalid token`;
- `returns 400 without a query field`;
- `returns 400 when query is a number`;
- `returns 503 when GEMINI_API_KEY is empty`;
- `returns 502 when Gemini throws a generic error`;
- `enforces rate limit after 20 requests`;

#### `src/__tests__/integration/sessionEndpoints.integration.test.ts` (12 cases)

`src/__tests__/integration/sessionEndpoints.integration.test.ts`:
Inputs: missing auth on create, missing `symbol`, invalid nested `parameterOverrides`, successful startup with merged default/stored/override params, init-payload publication with current-position data and Redis ack wait, invalid stored strategy params, missing strategies, post-validation startup failure, successful stop with Redis cleanup, successful stop while keeping active subscriptions for still-running symbols, nonexistent sessions, and mid-stop-flow failure.
Oracles: successful session start/stop flows as well as validation and orchestration failures: the happy path returns `201/200`, publishes the correct Redis payloads, waits for init ack, and maintains active-subscription state correctly; failure cases map to `401/400/404/500`.
Cases:
- `returns 201 and merges defaults, strategy params, and overrides when starting a session`;
- `publishes the init payload with current position data and waits for Redis ack`;
- `still starts a session when stored strategy parameters are invalid`;
- `returns 200 and cleans up Redis when stopping the last active session`;
- `returns 200 and leaves Redis subscriptions intact when the symbol is still active`;
- `returns 401 when creating a session without a token`;
- `returns 400 when symbol is missing from the session request`;
- `returns 400 when parameterOverrides contains a nested object`;
- `returns 404 when the referenced strategy does not exist`;
- `returns 500 when session startup fails after validation`;
- `returns 404 when stopping a session that does not exist`;
- `returns 500 when the stop flow fails mid-request`;

#### `src/__tests__/integration/snapshotEndpoints.integration.test.ts` (9 cases)

`POST /api/snapshots/capture`:
Inputs: missing auth, invalid auth, successful account + strategy snapshot capture, positions that do not map to a strategy, and capture failure.
Oracles: successful `200` capture counts plus `401/500` handling when auth or snapshot creation fails.
Cases:
- `captures account and per-strategy snapshots`;
- `captures only the account snapshot when positions do not map to a strategy`;
- `returns 401 without a token`;
- `returns 401 for an invalid token`;
- `returns 500 when capture fails`;

`GET /api/snapshots`:
Inputs: missing auth, default `days = 30`, `strategyId` + custom day filters, normalized numeric snapshot payloads, and retrieval failure.
Oracles: successful filtered `200` snapshot retrieval plus `401/500` handling for failures.
Cases:
- `returns normalized account snapshots and defaults days to 30`;
- `filters snapshots by strategyId and custom day window`;
- `returns 401 without a token`;
- `returns 500 when snapshot retrieval fails`;

#### `src/__tests__/integration/strategyEndpoints.integration.test.ts` (11 cases)

`src/__tests__/integration/strategyEndpoints.integration.test.ts`:
Inputs: protected-route auth failures, template listing, missing required create fields, missing `on_bar`, successful strategy creation with inferred crypto asset type, mapped list responses with draft-state flags, missing ownership on fetch, successful update with symbol uppercasing, stop-all-sessions flows, successful delete-after-stop, and delete failure after session shutdown.
Oracles: full CRUD/list/stop wiring: success cases return correctly mapped `200/201` payloads with inferred asset-type and normalized symbols, while ownership, validation, and deletion failures map to `401/400/404/500`.
Cases:
- `returns mapped strategy templates`;
- `returns 201 and infers crypto asset type during strategy creation`;
- `returns a mapped strategy list including draft status`;
- `updates a strategy and uppercases the symbol`;
- `stops all sessions for a strategy and returns the stopped count`;
- `deletes a strategy after stopping its sessions`;
- `returns 401 for protected routes without a token`;
- `returns 400 when creating a strategy without required fields`;
- `returns 400 when creating a strategy without an on_bar method`;
- `returns 404 when fetching a strategy that does not belong to the user`;
- `returns 500 when strategy deletion fails after session shutdown`;

#### `src/__tests__/integration/strategyViews.integration.test.ts` (10 cases)

`src/__tests__/integration/strategyViews.integration.test.ts`:
Inputs: protected-route auth failures, dashboard-summary computation with Sharpe, strategy detail pages with sorted trades and metrics, missing strategies when listing versions, version ordering, low-trust draft creation, static-analysis failure, successful draft creation with incremented version numbers, deploying wrong-strategy versions, and successful transactional deployment.
Oracles: successful dashboard/detail/version-management flows alongside trust-tier and ownership protections.
Cases:
- `returns a computed dashboard summary including sharpe`;
- `returns strategy detail with sorted trades and performance metrics`;
- `returns strategy versions in descending order`;
- `creates a draft strategy version with the next version number`;
- `deploys a strategy version in a transaction and updates the parent strategy`;
- `returns 401 for strategy view routes without a token`;
- `returns 404 when requesting versions for a missing strategy`;
- `returns 403 when creating a draft with code as a low-trust user`;
- `returns 400 when static analysis fails during draft creation`;
- `returns 404 when deploying a version that does not belong to the strategy`;

#### `src/__tests__/integration/tradeEndpoints.integration.test.ts` (12 cases)

`src/__tests__/integration/tradeEndpoints.integration.test.ts`:
Inputs: protected-route auth failures, mapped order lists with fallback fields, invalid order-status filters, successful order placement with default `type`/`time_in_force`, controller-side validation failure during placement, cancellation of foreign trades, successful cancellation, trade-history mapping with strategy attribution, computed portfolio metrics combining positions/trades/sessions/snapshots, enriched current-price positions, and empty-position short-circuiting.
Oracles: successful end-to-end trading reads/writes as well as correct auth, validation, and ownership enforcement.
Cases:
- `returns mapped orders and normalizes fallback order fields`;
- `returns 201 and defaults order type/time_in_force on successful order placement`;
- `returns 200 when canceling a known order`;
- `returns mapped trade history with strategy attribution`;
- `returns computed portfolio metrics from portfolio, trade, session, and snapshot data`;
- `returns enriched user positions with current prices and unrealized P&L`;
- `returns an empty position list without requesting live prices when the user has no holdings`;
- `returns 401 for protected trade routes without a token`;
- `returns 401 for trade routes with an invalid token`;
- `returns 400 for an invalid order status query`;
- `returns 500 when order placement fails validation inside the controller`;
- `returns 404 when canceling an order that is not owned by the user`;

## 3. `ingestor-node` Test Suites

Coverage emphasis: the `ingestor-node` suites cover the normal streaming lifecycle as well as its failure envelope. Positive-path cases verify websocket connection/authentication, startup reconciliation, live subscribe/unsubscribe routing, stock/crypto bar and trade publishing, and quote-handling behavior. Boundary/error-path cases then stress pre-auth subscription queues, reconnect timing, malformed Redis/websocket payloads, empty-subscription reconciliation, unknown control actions, and publish/logging failures that must not take the service down.

### 3.1 Shared setup

- Global setup silences console output.
- Unit suites use mocked Redis clients and mocked WebSockets; the integration suites compose real `ingestor-node` services around those mocks.
- Timer-based reconnect tests enable fake timers and restore them in teardown.

### 3.2 Unit suites

#### `src/__tests__/unit/BaseConsumer.test.ts` (10 cases)

- Setup/teardown: a lightweight `TestConsumer` subclass captures `flushSubscriptions()` calls; mocks and spies are reset before each case and restored after each case.

`BaseConsumer subscriptions`:
Inputs: blank symbols, duplicate symbols, later incremental subscription batches, and empty lists.
Oracles: symbol normalization, deduplication, incremental flush behavior, callback storage, correct uppercasing/trimming, no duplicate flushes, and no-op empty batches.
Cases:
- `stores the bar callback for later use`;
- `stores the trade callback for later use`;
- `normalizes and flushes unique bar subscriptions`;
- `normalizes and flushes unique trade subscriptions`;
- `flushes only incremental bar subscriptions on later calls`;
- `does not flush duplicate bar subscriptions`;
- `does not flush duplicate trade subscriptions`;
- `ignores blank and falsy bar symbols`;
- `ignores blank and falsy trade symbols`;
- `does not flush empty subscription batches`;

#### `src/__tests__/unit/ControlSubscriber.test.ts` (10 cases)

- Setup/teardown: mocked Redis client plus mocked stock/crypto consumers; each case resets the mocked reconciliation and message-routing environment.

`src/__tests__/unit/ControlSubscriber.test.ts`:
Inputs: startup reconciliation with and without active subscriptions, stock-vs-crypto subscribe commands, unsubscribe commands, unknown actions, malformed control JSON, reconciliation failure during connect, and explicit disconnect.
Oracles: correct routing to the appropriate consumer, resilient connect behavior, logged-but-nonfatal malformed inputs, and clean Redis shutdown.
Cases:
- `creates a Redis client with the configured URL and registers listeners`;
- `reconciles active stock and crypto subscriptions and starts listening on connect`;
- `routes stock subscribe commands to the stock consumer by default`;
- `routes crypto subscribe commands to the crypto consumer`;
- `routes unsubscribe commands to the correct consumer`;
- `skips reconciliation work when there are no active subscriptions`;
- `continues connecting when reconciliation fails`;
- `ignores unknown control actions`;
- `logs malformed control messages without throwing`;
- `disconnects cleanly by quitting the Redis client`;

#### `src/__tests__/unit/CryptoConsumer.test.ts` (18 cases)

- Setup/teardown: mocked WebSocket transport, mocked logger, and fake timers for reconnect behavior; teardown restores timers and clears socket state.

`src/__tests__/unit/CryptoConsumer.test.ts`:
Inputs: socket open events, Alpaca welcome/auth flows, array websocket payloads containing subscription confirms, bars and trades, incremental bar-only and trade-only flushes, unsubscribe requests with slash-form and slashless symbols, manual disconnect, invalid helper symbols, invalid symbols during flush, transport errors, malformed JSON, unexpected close/reconnect, unsupported messages, invalid unsubscribe requests, and disconnect-without-socket.
Oracles: correct auth sequencing, normalized `BTC/USD` handling, publish suppression before auth, robust logging without crashes, reconnect retries after unexpected close, and no-op safety on invalid or missing state.
Cases:
- `connects to the crypto stream and resolves on socket open`;
- `authenticates after the welcome event and flushes stored subscriptions`;
- `processes array websocket messages for subscription confirmations, bars, and trades`;
- `flushes incremental bar-only subscriptions`;
- `flushes incremental trade-only subscriptions`;
- `sends unsubscribe payloads for tracked crypto symbols`;
- `accepts slashless unsubscribe inputs for tracked crypto symbols`;
- `disconnects cleanly and suppresses reconnects after manual shutdown`;
- `normalizes slashless USD symbols into Alpaca format`;
- `does not flush subscriptions before authentication`;
- `rejects blank and non-USD helper symbols`;
- `warns and skips invalid symbols during flush`;
- `logs transport and stream-level errors`;
- `logs malformed websocket payloads safely`;
- `reconnects after unexpected closes and logs retry failures`;
- `ignores callback-free and unsupported messages without crashing`;
- `returns early for invalid unsubscribe symbols`;
- `allows disconnect without an active socket`;

#### `src/__tests__/unit/RedisPublisher.test.ts` (8 cases)

- Setup/teardown: mocked Redis client with explicit listener registration, publish, connect, and disconnect stubs.

`src/__tests__/unit/RedisPublisher.test.ts`:
Inputs: normal connect/publish/disconnect paths plus connect, publish, and disconnect failures.
Oracles: direct delegation, log emission for connect/error listeners, and transparent propagation of operation failures.
Cases:
- `creates a Redis client with the configured URL and registers listeners`;
- `logs connect events from the Redis client`;
- `delegates connect to the Redis client`;
- `publishes messages and disconnects cleanly`;
- `logs Redis client errors`;
- `propagates connection failures`;
- `propagates publish failures`;
- `propagates disconnect failures`;

#### `src/__tests__/unit/StockConsumer.test.ts` (20 cases)

- Setup/teardown: mocked WebSocket transport, mocked logger, fake timers for reconnect tests, and restored timer state after each case.

`src/__tests__/unit/StockConsumer.test.ts`:
Inputs: default IEX connection, SIP connection, welcome/auth sequence, message arrays with subscriptions/bars/trades/quotes, incremental bar-only and trade-only flushes, `subscribeQuotes()`, unsubscribe while connected, disconnect, helper payload generation without an active socket, pre-auth subscriptions, malformed JSON, transport errors, unexpected close/reconnect, unsupported messages, untracked-symbol unsubscribe, disconnected unsubscribe, disconnect without socket, empty flushes, and subscriptions made before connect.
Oracles: correct websocket URI selection, quote logging without trade/bar side effects, tracked-symbol persistence, and resilient reconnect/error handling.
Cases:
- `connects to the default IEX stock stream and resolves on socket open`;
- `connects to the SIP stock stream when requested`;
- `authenticates after the welcome event and flushes stored subscriptions`;
- `processes array messages for subscription confirmations, bars, trades, and quotes`;
- `flushes incremental bar-only subscriptions`;
- `flushes incremental trade-only subscriptions`;
- `supports subscribeQuotes by reusing tracked bar subscriptions`;
- `sends unsubscribe payloads for tracked connected symbols`;
- `disconnects cleanly and suppresses reconnects after manual shutdown`;
- `builds auth and subscribe payloads safely even without an active socket`;
- `does not flush subscriptions before authentication`;
- `logs malformed websocket payloads safely`;
- `logs transport and stream-level errors`;
- `reconnects after unexpected closes and logs retry failures`;
- `ignores callback-free and unsupported messages without crashing`;
- `does not send unsubscribe payloads for untracked symbols`;
- `does not send unsubscribe payloads while disconnected`;
- `allows disconnect without an active socket`;
- `returns early when flush has no bars or trades`;
- `keeps tracked symbols when subscribing before connect`;

### 3.3 Integration suites

#### `src/__tests__/integration/bootstrap.integration.test.ts` (6 cases)

- Setup/teardown: real `index.ts` bootstrap logic is exercised with mocked Redis publisher and mocked stock/crypto consumers; teardown restores process signals and timer state.

`src/__tests__/integration/bootstrap.integration.test.ts`:
Inputs: normal startup order, callback wiring for bar/trade publishing, `SIGINT` shutdown, bar publish failures, trade publish failures, and fatal startup failure.
Oracles: correct startup ordering, shared publisher reuse, graceful shutdown of all components, nonfatal callback publish errors, and `process.exit(1)` on unrecoverable bootstrap failure.
Cases:
- `boots services in the correct order and wires startup subscriptions`;
- `wires market-data callbacks through the shared Redis publisher`;
- `shuts down cleanly on SIGINT`;
- `logs publish failures from bar callbacks without aborting`;
- `logs publish failures from trade callbacks without aborting`;
- `exits fatally when bootstrap startup fails`;

#### `src/__tests__/integration/controlPlane.integration.test.ts` (8 cases)

- Setup/teardown: real `ControlSubscriber` plus real stock/crypto consumer instances are run against a mocked Redis bus and mocked socket sends.

`src/__tests__/integration/controlPlane.integration.test.ts`:
Inputs: startup reconciliation with active stock and crypto subscriptions, live stock subscribe messages, live crypto subscribe messages with slashless normalization, unsubscribe messages, empty reconciliation state, reconciliation failure, malformed control JSON, and unknown control actions.
Oracles: correct upstream subscribe/unsubscribe payloads, preserved listener liveness after errors, and zero accidental feed traffic on malformed/unknown messages.
Cases:
- `reconciles active stock and crypto subscriptions into connected feeds`;
- `routes live stock subscribe commands into the stock feed`;
- `routes live crypto subscribe commands into the crypto feed with normalized symbols`;
- `routes unsubscribe commands into the correct connected feeds`;
- `skips reconciliation sends when Redis has no active subscriptions`;
- `continues listening when reconciliation fails`;
- `logs malformed control messages without emitting feed traffic`;
- `ignores unknown control actions without emitting feed traffic`;

#### `src/__tests__/integration/marketDataPipeline.integration.test.ts` (8 cases)

- Setup/teardown: real stock and crypto consumer classes are wired to a mocked WebSocket transport and a mocked Redis publisher.

`src/__tests__/integration/marketDataPipeline.integration.test.ts`:
Inputs: stock startup auth/subscriptions, crypto startup auth/subscriptions, stock bar/trade events, crypto bar/trade events with symbol normalization, quote events, malformed websocket payloads, bar publish failures, and trade publish failures.
Oracles: correct channel selection (`market_data:*`, `market_trades:*`), correct slash-form symbol normalization for crypto, no Redis publishes for quotes, and continued service operation after malformed payloads or publish failures.
Cases:
- `authenticates the stock feed and flushes startup stock subscriptions`;
- `authenticates the crypto feed and flushes startup crypto subscriptions`;
- `publishes stock bars and trades to the correct Redis channels`;
- `publishes crypto bars and trades with normalized slash-form symbols`;
- `logs quote events without publishing Redis messages`;
- `logs malformed WebSocket payloads without publishing`;
- `logs bar publish failures without aborting the service`;
- `logs trade publish failures without aborting the service`;

## 4. `worker-python` Test Suites

Coverage emphasis: the `worker-python` suites cover both steady-state runtime behavior and failure handling. Positive-path cases verify strategy initialization, bar handling, signal emission, session startup/shutdown, container spawn/cleanup, control-plane routing, bar ingestion, trade-result processing, and main-thread startup/cleanup. The boundary/error-focused cases then cover missing config, malformed pubsub traffic, duplicate or missing session identifiers, warm-start initialization edge cases, successful vs failed trade-result handling, Redis/DB/Docker dependency failures, and interrupt-driven cleanup of long-running threads.

### 4.1 Shared setup

- The Python suites use `unittest.TestCase`, explicit fake classes, and `patch.dict(sys.modules, ...)` to replace infrastructure modules before import time.
- Most suites prepend the worker root (and, when needed, the Docker strategy root) to `sys.path` so the runtime modules can be imported exactly as production code imports them.
- `setUp` normally resets global or per-class state; `tearDown` is used in integration suites to stop fake pubsub brokers and join worker threads.

### 4.2 Unit suites

#### `tests/unit/test_base_strategy.py` (`BaseStrategyTests`, 6 methods)

- Setup/teardown: `setUp` suppresses deprecation warnings; helper loaders inject a fake `redis` module, fixed timestamps, and deterministic environment variables (`SYMBOL`, `SESSION_ID`, `STRATEGY_ID`) before importing `base_strategy`.

`tests/unit/test_base_strategy.py`:
Inputs: `BTCUSD` buy orders using the BTC sentinel quantity, oversize sells against a `0.75`-share position, 1-minute bar payloads, Redis publish failure during `_send_signal`, init payloads containing parameters/current position/historical bars, and 5-minute resampling after five 1-minute bars.
Oracles: correct default BTC buy quantity, position clamping on sell, bar-history updates, graceful `_send_signal` failure, correct warm-start initialization from payloads, and correct OHLCV aggregation for 5-minute bars.
Cases:
- `test_buy_uses_btc_sentinel_default_qty_and_tracks_pending_signal`;
- `test_sell_clamps_to_available_position_and_tracks_pending_signal`;
- `test_handle_bar_updates_history_and_invokes_user_callback_for_1min`;
- `test_send_signal_returns_false_when_redis_publish_fails`;
- `test_initialize_from_payload_seeds_position_and_historical_bars`;
- `test_handle_bar_aggregates_and_emits_when_resample_buffer_reaches_five_minute_target`;

#### `tests/unit/test_control_listener.py` (`ControlListenerTests`, 8 methods)

- Setup/teardown: fake Redis pubsub messages are injected before importing `services.control_listener`; mocked managers capture `start_session`, `stop_session`, and `stop_by_symbol` calls.

`tests/unit/test_control_listener.py`:
Inputs: valid `subscribe` JSON, non-message pubsub events, `unsubscribe` by `sessionId`, `unsubscribe` by `symbol`, subscribe without `sessionId`, unsubscribe without `sessionId` or `symbol`, malformed JSON followed by valid JSON, and handler exceptions followed by later valid messages.
Oracles: correct routing to the strategy manager, logged validation errors for incomplete control messages, and continued listening after malformed JSON or handler exceptions.
Cases:
- `test_start_routes_valid_subscribe_message`;
- `test_start_ignores_non_message_pubsub_events`;
- `test_handle_event_unsubscribe_with_session_id_stops_specific_session`;
- `test_handle_event_unsubscribe_by_symbol_stops_all_sessions_for_symbol`;
- `test_handle_event_subscribe_without_session_id_logs_error`;
- `test_handle_event_unsubscribe_without_session_id_or_symbol_logs_error`;
- `test_start_logs_invalid_json_and_continues_processing_messages`;
- `test_start_logs_handler_exception_and_continues_processing_messages`;

#### `tests/unit/test_data_consumer.py` (`DataConsumerTests`, 6 methods)

- Setup/teardown: fake Redis pubsub patterns and fake validated `Bar`/`Trade` models are injected before import; `start_consumer()` suppresses print noise while driving the listener loop.

`tests/unit/test_data_consumer.py`:
Inputs: startup with no messages, valid bar messages, non-`pmessage` events, trade messages, malformed JSON followed by valid JSON, and invalid bar payloads followed by valid payloads.
Oracles: correct pattern subscriptions, forwarding of only bar messages to `manager.on_bar`, no forwarding of trades in the current design, and continued processing after malformed JSON or invalid bar payloads.
Cases:
- `test_start_subscribes_to_bar_and_trade_patterns`;
- `test_start_routes_bar_messages_to_manager`;
- `test_start_ignores_non_pmessage_events`;
- `test_start_skips_trade_messages_without_calling_manager`;
- `test_start_swallows_invalid_json_and_continues_processing`;
- `test_start_swallows_invalid_bar_payload_and_continues_processing`;

#### `tests/unit/test_db_client.py` (`DBClientTests`, 6 methods)

- Setup/teardown: the suite injects a fake `psycopg2` module, fake `RealDictCursor`, fake config constants, and a mocked logger before importing `services.db_client`.

`tests/unit/test_db_client.py`:
Inputs: successful connection construction, successful strategy fetch, missing strategies, connection failure, query execution failure, and cursor-close failure.
Oracles: correct connection arguments, correct SQL and cursor factory, resource cleanup on the happy path, `None` on failures, and logger activity on error paths.
Cases:
- `test_get_connection_uses_configured_postgres_settings`;
- `test_fetch_strategy_returns_row_and_closes_resources`;
- `test_fetch_strategy_returns_none_for_missing_row_and_closes_resources`;
- `test_fetch_strategy_returns_none_when_connection_fails`;
- `test_fetch_strategy_returns_none_when_query_execution_fails`;
- `test_fetch_strategy_returns_none_when_cursor_close_fails`;

#### `tests/unit/test_docker_manager.py` (`DockerManagerTests`, 8 methods)

- Setup/teardown: a fake Docker client, fake Docker exception classes, and a mocked logger are injected before import.

`tests/unit/test_docker_manager.py`:
Inputs: missing strategy images, successful container spawn, successful stop/remove, byte-string log output, duplicate session spawns, container-run failure, stopping an already-missing container, and missing Dockerfile during image build.
Oracles: correct image build arguments, environment-variable injection into containers, session/container tracking, decoded logs, duplicate-spawn suppression, cleanup when containers are already gone, and hard failure when the Dockerfile is absent.
Cases:
- `test_ensure_strategy_image_builds_when_image_missing`;
- `test_spawn_container_runs_container_and_tracks_session`;
- `test_stop_container_stops_removes_and_untracks_session`;
- `test_get_container_logs_decodes_bytes`;
- `test_spawn_container_returns_existing_container_id_when_session_already_tracked`;
- `test_spawn_container_returns_none_when_container_run_fails`;
- `test_stop_container_cleans_tracking_when_container_is_already_gone`;
- `test_ensure_strategy_image_raises_when_dockerfile_is_missing`;

#### `tests/unit/test_main.py` (`MainTests`, 4 methods)

- Setup/teardown: fake Redis client, strategy manager, control listener, and data consumer classes are injected before importing `main`; thread creation and `time.sleep()` are patched per test.

`tests/unit/test_main.py`:
Inputs: healthy Redis startup, repeated sleep iterations followed by `KeyboardInterrupt`, failed Redis health checks, and unexpected runtime errors from the sleep loop.
Oracles: correct creation of two daemon threads, clean `cleanup_all()` on shutdown, immediate `exit(1)` on failed Redis health checks, and propagation of unexpected errors without accidental cleanup.
Cases:
- `test_main_starts_both_worker_threads_as_daemons_when_redis_is_available`;
- `test_main_keeps_sleeping_until_keyboard_interrupt_then_shuts_down_cleanly`;
- `test_main_exits_immediately_when_redis_connection_check_fails`;
- `test_main_propagates_unexpected_sleep_errors_without_running_cleanup`;

#### `tests/unit/test_redis_client.py` (`RedisClientTests`, 6 methods)

- Setup/teardown: the suite injects a fake `redis` module plus fake `ConnectionError` and config constants before importing `services.redis_client`.

`tests/unit/test_redis_client.py`:
Inputs: initial client creation, `pubsub()` lookup, successful `PING`, connection-error `PING`, unexpected `PING` failure, and unexpected `pubsub()` failure.
Oracles: correct host/port configuration, singleton client reuse, boolean return on connection errors, propagation of unexpected errors, and pass-through of pubsub failures.
Cases:
- `test_get_client_initializes_redis_with_configured_host_and_port`;
- `test_get_pubsub_returns_pubsub_from_cached_client`;
- `test_test_connection_returns_ping_result_when_ping_succeeds`;
- `test_test_connection_returns_false_on_connection_error`;
- `test_test_connection_propagates_unexpected_ping_errors`;
- `test_get_pubsub_propagates_client_pubsub_errors`;

#### `tests/unit/test_run_strategy.py` (`RunStrategyTests`, 4 methods)

- Setup/teardown: fake `BaseStrategy`, fake `Bar`/`Trade`, fake Redis client/pubsub, and fake environment variables are injected before importing `run_strategy`; `setUp` clears the static list of fake strategy instances.

`tests/unit/test_run_strategy.py`:
Inputs: the legacy import-compatibility shim, a full init-ack + bar + trade + trade-result message flow, missing `STRATEGY_CODE`, and strategy code that raises during `exec`.
Oracles: registration of `strategies.base_strategy`, correct pubsub subscribe/unsubscribe sequence, publication of `strategy_init_ack:*`, correct delivery of bar/trade/trade-result messages into the strategy instance, and `SystemExit(1)` for missing/bad strategy code.
Cases:
- `test_install_import_compat_shims_registers_legacy_module_path`;
- `test_main_processes_init_ack_and_live_messages`;
- `test_main_exits_when_strategy_code_is_missing`;
- `test_main_exits_when_strategy_code_fails_during_exec`;

#### `tests/unit/test_strategy_manager.py` (`StrategyManagerTests`, 11 methods)

- Setup/teardown: fake DB client, fake Docker manager, fake market-data models, and a mocked logger are injected before importing `services.strategy_manager`.

`tests/unit/test_strategy_manager.py`:
Inputs: DB symbol overrides on startup, normal stop flow, stop-by-symbol filtering, cleanup-all behavior, bar routing to active strategies, list-active copy safety, missing `session_id`, missing strategies, container-spawn failure, bars for unknown symbols, and trade routing to active strategies.
Oracles: correct DB lookup before spawn, correct Docker manager arguments, accurate active-session tracking, correct stop counts, copy-on-read behavior for `list_active()`, and no work on invalid startup inputs.
Cases:
- `test_start_session_tracks_session_and_uses_db_symbol_override`;
- `test_stop_session_removes_active_session_when_container_stops`;
- `test_stop_by_symbol_stops_only_matching_sessions_and_returns_count`;
- `test_cleanup_all_stops_tracked_sessions_and_calls_docker_cleanup`;
- `test_on_bar_routes_to_registered_strategies_when_mapping_exists`;
- `test_list_active_returns_independent_copy_of_session_mapping`;
- `test_start_session_returns_none_without_session_id`;
- `test_start_session_returns_none_when_strategy_lookup_fails`;
- `test_start_session_returns_none_when_container_spawn_fails`;
- `test_on_bar_ignores_symbols_without_registered_strategies`;
- `test_on_trade_routes_to_registered_strategies_when_mapping_exists`;

### 4.3 Integration suites

#### `tests/integration/test_strategy_runtime.py` (`StrategyRuntimeIntegrationTests`, 7 methods)

- Setup/teardown: each test creates a new in-memory `FakeRedisBroker`, starts `run_strategy` in a daemon thread, and `tearDown` sets `shutdown_requested`, stops all pubsubs, and joins the thread.

`tests/integration/test_strategy_runtime.py`:
Inputs: normal init payloads, live bar-triggered buys, legacy `from strategies.base_strategy import ...` imports, missing or bad strategy code, successful trade-result messages that enable later sell signals, failed trade-result messages that should block later sells, and init payloads carrying historical bars.
Oracles: publication of `strategy_init_ack:*`, emission of correct `trade:signals`, legacy import compatibility, `SystemExit(1)` for invalid code, correct handling of successful vs failed trade execution results, and proper warm-start behavior from historical bars.
Cases:
- `test_runtime_publishes_init_ack_after_init_payload`;
- `test_runtime_emits_trade_signal_after_live_bar`;
- `test_runtime_supports_legacy_base_strategy_import_path`;
- `test_runtime_exits_cleanly_on_missing_or_bad_strategy_code`;
- `test_runtime_processes_trade_result_success_message`;
- `test_runtime_processes_trade_result_failure_message`;
- `test_runtime_historical_bars_init_contract`;

#### `tests/integration/test_worker_control_plane.py` (`WorkerControlPlaneIntegrationTests`, 5 methods)

- Setup/teardown: `setUp` constructs a queued pubsub, fake DB with three strategies, fake Docker manager, real `StrategyManager`, real `ControlListener`, and a listener thread; `tearDown` stops the pubsub and joins the thread.

`tests/integration/test_worker_control_plane.py`:
Inputs: subscribe events for a single session, unsubscribe by `sessionId`, unsubscribe by `symbol`, duplicate subscribe messages, and malformed JSON followed by valid JSON.
Oracles: DB fetch before container spawn, correct active-session entries, stopping only the intended sessions, duplicate-subscribe suppression, and listener survival after malformed messages.
Cases:
- `test_subscribe_event_fetches_strategy_spawns_container_and_tracks_session`;
- `test_unsubscribe_by_session_stops_only_target_session`;
- `test_unsubscribe_by_symbol_stops_all_matching_sessions`;
- `test_duplicate_subscribe_does_not_spawn_second_container`;
- `test_malformed_control_message_does_not_kill_listener`;

#### `tests/integration/test_worker_data_ingress.py` (`WorkerDataIngressIntegrationTests`, 3 methods)

- Setup/teardown: `setUp` constructs a queued pubsub, loads the real `DataConsumer`, starts it in a thread, and waits for pattern subscription; `tearDown` stops the pubsub and joins the thread.

`tests/integration/test_worker_data_ingress.py`:
Inputs: valid bar messages, valid trade messages, and malformed JSON followed by valid bar payloads.
Oracles: forwarding of bar messages to `manager.on_bar`, intentional non-forwarding of trade messages in the current design, and consumer liveness after invalid payloads.
Cases:
- `test_bar_message_is_parsed_and_forwarded_to_manager`;
- `test_trade_message_is_consumed_but_not_forwarded_in_current_design`;
- `test_invalid_payload_does_not_stop_consumer`;

#### `tests/integration/test_worker_main_lifecycle.py` (`WorkerMainLifecycleIntegrationTests`, 4 methods)

- Setup/teardown: `setUp` creates separate queued pubsubs for control and data channels; `tearDown` stops both queues. Tests patch thread creation and `time.sleep()` to deterministically trigger shutdown.

`tests/integration/test_worker_main_lifecycle.py`:
Inputs: healthy Redis startup, `KeyboardInterrupt` cleanup, failed Redis health checks, and malformed market-data payloads delivered while the main loop is still running.
Oracles: creation of daemon control/data threads sharing the same manager, correct startup subscriptions (`system:subscription_updates`, `market_data:*`, `market_trades:*`), `cleanup_all()` on interrupt, immediate `exit(1)` on failed Redis health checks, and survival of the data thread after parse errors.
Cases:
- `test_main_starts_control_and_data_workers_when_redis_is_healthy`;
- `test_main_calls_cleanup_all_on_keyboard_interrupt`;
- `test_main_exits_when_redis_health_check_fails`;
- `test_main_survives_data_consumer_parse_errors_without_crashing`;

## 5. Shell and Python Smoke Tests

These files are not framework-driven unit/integration tests, but they are part of the backend verification story because they exercise the deployed stack, Docker runtime, or strategy logic in an operator-friendly way.

### 5.1 `scripts/e2e_docker_test.sh`

Framework equivalent: manual end-to-end smoke test with fail-fast Bash (`set -e`).

Setup:

- Assumes Docker Desktop, Redis, Postgres, the gateway, the worker, and the ingestor are already running.
- Uses `curl` against `http://localhost:3000`.
- Creates or reuses `e2etest@test.com` / `testpass123`.

Scenario phases and oracles:

- Step 0, auth bootstrap: registers then logs in; oracle is extraction of a non-empty JWT token, otherwise the script exits with the raw login response.
- Step 1, strategy creation: posts a full RSI-based `BaseStrategy` implementation for `BTC/USD` with parameters such as `rsi_period=14`, `oversold_threshold=30`, `overbought_threshold=70`, `trade_size=1`, `stop_loss_pct=5`, `take_profit_pct=10`, `timeframe=1Min`, `max_drawdown_pct=5`, and `warmup_bars=50`. Oracle is a non-empty `strategyId`.
- Step 2, session start: posts `{ "symbol": "BTC/USD", "strategyId": "<id>" }` to `/sessions`. Oracle is a non-empty `sessionId`.
- Step 3, container/runtime verification: waits 3 seconds, looks for a `stratforge-session-*` container, tails container logs for up to 300 seconds, and treats a missing container as a warning. The intended oracle is visible strategy startup plus live market-data processing in the container logs.
- Step 4, session shutdown: posts to `/sessions/<sessionId>/stop`, then polls Docker for up to 10 seconds until the session container disappears. Oracle is eventual container termination.
- Step 5, cleanup: deletes the created strategy. Oracle is successful HTTP completion without an early shell exit.

Edge/error coverage:

- `REGISTER_RESPONSE` is allowed to fail without aborting the script (`|| true`) so reruns can still log in.
- Missing token, missing strategy ID, or missing session ID each terminate the script immediately.
- The script is intentionally a smoke test rather than a CI-safe test because it depends on live services and a 5-minute log tail.

### 5.2 `scripts/e2e_new_indicator_templates.sh`

Framework equivalent: multi-strategy end-to-end smoke test for the newly added indicator templates.

Setup:

- Requires gateway, Redis, Docker, and worker services plus access to the Redis container for synthetic-bar publishing.
- Supports environment overrides for `SYMBOL`, `BAR_COUNT`, `PATTERN`, `BASE_PRICE`, `STEP_SIZE`, `DELAY_MS`, `CLEANUP`, and log directory placement.

Scenario design:

- Registers/logs in a fresh user, then creates one strategy each for RSI, Stochastic, Aroon, OBV, and A/D.
- Starts one session per strategy.
- Publishes synthetic bars into Redis so the containers can warm up indicators and emit debug logs.
- Inspects per-container logs for the expected indicator-specific debug tags such as `[RSI DEBUG]`, `[STOCH DEBUG]`, `[AROON DEBUG]`, `[OBV DEBUG]`, and `[A/D DEBUG]`.
- Cleans up sessions and strategies when `CLEANUP=true`.

Oracle:

- Each indicator strategy should initialize, receive data, and write indicator-specific debug output without crashing.
- The script is designed to reveal template-specific breakage in warm-up handling, indicator calculations, or container startup.

### 5.3 `scripts/test-ai-routes.sh`

Framework equivalent: authenticated API smoke test for the AI endpoints.

Setup:

- Uses a unique email/username per run, registers/logs in, and extracts a bearer token.
- Requires `GEMINI_API_KEY`, running gateway, Redis, and Postgres.

Scenarios:

- `POST /api/ai/generate` with a buy-above-previous-high / sell-below-previous-low prompt for `AAPL`; oracle is a response containing `pythonCode`.
- `POST /api/ai/research` with a conceptual moving-average question; oracle is a response containing `answer`.
- `POST /api/ai/research` with a live-market-data question for `AAPL`; oracle is a response containing `answer`, demonstrating the MCP-backed path.

### 5.4 `test_ai_endpoint.sh`

Framework equivalent: minimal generate-endpoint smoke test.

Scenario:

- Logs in as `test@test` / `test123`, extracts a bearer token, then calls `POST /api/ai/generate` with a Bollinger-Bands momentum prompt for `AAPL` and preferred name `BBMomentum`.

Oracle:

- The script succeeds only if login returns a token and the generate endpoint returns JSON; `jq` is used for human-readable inspection.

### 5.5 `scripts/test_order.sh`

Framework equivalent: authenticated trade-endpoint connectivity smoke test.

Scenario:

- Registers/logs in as `test@test`, extracts a bearer token, then calls `GET /api/trade/orders` and `GET /api/trade/portfolio`.

Oracle:

- Successful responses prove that auth, trade-route mounting, and the corresponding controller stack are reachable from the running gateway.

### 5.6 `scripts/test_strategy_signals.py`

Framework equivalent: offline strategy-logic test harness using in-memory strategies instead of Docker/Redis.

Setup:

- Loads `services/worker-python/docker/base_strategy.py` dynamically.
- Replaces Redis publishing with an in-memory `captured_signals` list.
- Defines inline RSI and MACD strategies and feeds them synthetic bars.

Scenarios and oracles:

- RSI warm-up: 10 flat bars should produce no signals before the indicator is ready.
- RSI buy crossover: downtrend then recovery should produce at least one BUY, with `qty` equal to `trade_size`.
- RSI sell crossover: uptrend then pullback with a pre-seeded position should produce at least one SELL.
- Parameter sensitivity: changing `oversold_threshold` from `30` to `15` should change signal behavior.
- Trade-size sensitivity: `trade_size=1` vs `trade_size=5` should change emitted BUY quantity.
- Default config access: `trade_size`, `max_drawdown_pct`, `stop_loss_pct`, `take_profit_pct`, and `timeframe` should all be visible through `self.config`.
- MACD warm-up and crossover: 80 sinusoidal bars should produce at least one signal and at least one BUY.
- Position guard: overbought reversal with `position=0` should not emit a SELL.

### 5.7 `services/worker-python/test_conn.py`

Framework equivalent: developer connectivity probe rather than an automated test suite.

Scenario:

- Writes and reads a Redis key on `localhost:6379`, then attempts a PostgreSQL connection to `localhost:5433` with database/user/password `stratforge/stratforge/changeme`.

Oracle:

- Successful Redis `set/get` and successful `SELECT NOW()` confirm local dependency reachability.
- The script prints errors rather than asserting, so it is diagnostic rather than CI-grade.

## 6. Overall Testing Assessment

The backend test design is strongest where the codebase has the highest coordination risk:

- `gateway-node` aggressively tests request validation, authz, trust-tier checks, static analysis, virtualization accounting, Redis orchestration, and AI/MCP error translation.
- `ingestor-node` focuses on the hard failure modes for a streaming service: reconnects, malformed websocket/control payloads, normalization of market symbols, and graceful degradation when Redis publishing fails.
- `worker-python` focuses on import-time isolation, container lifecycle safety, warm-start initialization, trade-result handling, and nonfatal Redis/pubsub parse failures.

The main edge-case themes covered repeatedly across the backend are:

- missing auth, missing IDs, malformed JSON, and invalid body/query shapes
- cache hits, cache misses, malformed cache values, and best-effort fallback behavior
- duplicate subscriptions/orders/sessions and idempotent start/stop semantics
- timeouts, retry loops, disconnected Redis/MCP/AI dependencies, and non-`Error` throwables
- boundary values such as rate-limit request `20` vs `21`, bar-limit clamping (`10` to `500`), zero balances/cost basis, null trust-tier rows, and empty historical datasets

That combination gives the TAs a clear testing story: unit tests pin the local logic and edge handling, integration tests verify the service boundaries and route wiring, and the shell scripts provide operational smoke coverage for the real multi-process stack.
