# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Each SSE event contains ticker, price, previous price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads recent conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is included in the chat response so the LLM can inform the user.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Canvas-based charting library preferred (Lightweight Charts or Recharts) for performance
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- SSE resilience: disconnect and verify reconnection

---

## 13. Document Review — Questions, Clarifications & Simplification Opportunities

*Review conducted 2026-03-09. Based on current project state: market data backend complete (8 modules, 73 tests passing); all other components not yet started.*

### Questions & Clarifications Needed

**Architecture & Data Flow**

1. **SSE stream scope vs. watchlist**: Section 6 says the SSE stream pushes prices "for all tickers known to the system" and notes this is "equivalent to the user's watchlist" in the single-user model. But what happens when a user adds a ticker via the watchlist API — does the simulator/Massive poller dynamically pick up the new ticker, or does it only stream the original 10? The simulator already starts with fixed seed prices in `seed_prices.py`. The plan should clarify whether the market data source's ticker set is static (the 10 defaults) or dynamically follows the watchlist.

2. **Portfolio snapshot timing**: Section 7 says snapshots are recorded "every 30 seconds by a background task, and immediately after each trade." But the P&L chart on the frontend will look sparse with only 2 data points per minute. Is 30 seconds intentional, or should this be more frequent (e.g., every 5-10 seconds) to make the chart feel alive? Alternatively, could the frontend interpolate between snapshots using live SSE prices?

3. **Chat history depth**: Section 9 says "recent conversation history" is loaded. How many messages? The plan doesn't specify a limit. Unbounded history will eventually exceed LLM context windows and inflate costs. Suggest specifying a cap (e.g., last 20 messages or last N tokens).

4. **Fractional shares**: The schema says `quantity REAL` with "fractional shares supported," but section 2 describes buying/selling "shares" with no mention of fractional UI. Should the trade bar allow decimal quantities (e.g., buy 0.5 shares of AMZN)? Or is this just future-proofing the schema?

5. **"Daily change %"** in watchlist: Section 10 says the watchlist shows "daily change %." But with a simulator, there's no concept of a daily open/close — prices start fresh each time the container launches. What is the baseline for calculating change %? First price since page load? First price since container start? This needs a clear definition.

**API Design**

6. **DELETE /api/watchlist/{ticker}**: Uses the ticker symbol in the URL path. What if a ticker contains special characters (e.g., BRK.B)? Consider using query parameters or URL encoding conventions. Also, should removing a ticker that the user holds a position in be allowed? The plan is silent on this edge case.

7. **POST /api/portfolio/trade response shape**: The plan specifies the request body (`{ticker, quantity, side}`) but not the response. Should it return the updated portfolio state, just the trade confirmation, or both? The frontend needs to know what to refresh after a trade.

8. **GET /api/portfolio response**: Section 8 says it returns "current positions, cash balance, total value, unrealized P&L" but doesn't define the JSON shape. Consider specifying this to align frontend and backend agents.

9. **Error response format**: No standardized error response shape is defined. Should errors return `{error: string, detail?: string}` with appropriate HTTP status codes? This matters for both the frontend and the LLM's trade-failure feedback.

**LLM Integration**

10. **Model availability**: The plan specifies `openrouter/openai/gpt-oss-120b` with Cerebras inference. Model availability on OpenRouter can change. Should there be a fallback model specified? What happens if the model is unavailable?

11. **Structured output reliability**: The plan assumes the LLM always returns valid JSON matching the schema. In practice, even with structured output mode, edge cases occur. The plan mentions "graceful handling of malformed responses" in the testing section but doesn't describe the actual fallback behavior. Suggest: if JSON parsing fails, return the raw text as the message with no actions.

12. **Trade validation in chat flow**: If the LLM requests 3 trades and the 2nd one fails (e.g., insufficient cash after the 1st trade spent most of it), should trades 1 and 3 still execute? Or should it be all-or-nothing? The plan doesn't specify atomicity semantics.

### Simplification Opportunities

**High-value simplifications (recommended)**

- **Drop `user_id` from all tables**: The plan acknowledges this is single-user and says user_id is for "future multi-user support." But adding multi-user later would require auth, sessions, and security — a much bigger change than a schema migration. The `user_id` column adds cognitive overhead to every query for zero current benefit. Suggest removing it and keeping the schema minimal. If multi-user is ever needed, a migration is trivial compared to the auth work required.

- **Drop `portfolio_snapshots` background task**: Instead of a background task writing snapshots every 30 seconds, consider computing portfolio value on-demand from positions + live prices. The P&L chart could use trade timestamps as data points (portfolio value at time of each trade), plus the current live value. This eliminates a background task, a database table, and the timing question from item #2 above.

- **Simplify chat_messages.actions column**: Storing executed actions as a JSON string in a TEXT column works but makes querying difficult. Since this data is only read back for display (showing "Bought 10 AAPL" inline in chat), consider just including action summaries in the `content` field itself, eliminating the separate `actions` column.

- **Drop Windows PowerShell scripts**: The plan calls for both bash and PowerShell start/stop scripts. Docker Desktop on Windows supports bash via WSL2 or Git Bash. Maintaining two sets of scripts doubles the work for a narrow audience. Suggest providing bash scripts only with a note that Windows users should use WSL2 or Git Bash.

**Lower-priority simplifications (consider)**

- **Static export vs. SPA routing**: The plan specifies Next.js with `output: 'export'` for a single-page app. For a true SPA with no server-side routing needs, a lighter build tool (Vite + React) would eliminate Next.js complexity (file-based routing, app directory conventions) that goes unused. However, if Next.js is a deliberate teaching choice for the course, keep it.

- **Treemap/heatmap complexity**: The portfolio heatmap (treemap) is visually impressive but complex to implement well — sizing, labeling, color scaling, and responsiveness are all non-trivial. With only 10 possible tickers and likely fewer positions, a simple colored bar chart or card grid would convey the same information with much less implementation effort. Consider whether the treemap is worth the complexity for the capstone scope.

### Missing from the Plan

- **CORS / static file serving configuration**: The plan says "no CORS needed" because the frontend is served from the same origin. But during local development (frontend on port 3000, backend on port 8000), CORS will be needed. The plan should mention a development-mode CORS configuration or a proxy setup.

- **Development workflow**: There's no section on how to run the backend and frontend locally for development (outside Docker). Developers will need: `uv run uvicorn` for the backend, `npm run dev` for the frontend, and some way to proxy API calls. This is important for the agents building the system.

- **Database reset / fresh start**: How does a user reset their portfolio to $10k and clear all trades? The plan should specify whether there's an API endpoint or if it requires deleting the SQLite file and restarting.

- **Rate limiting on trade execution**: Nothing prevents a user (or the LLM) from executing hundreds of trades per second. While this is a demo, a runaway LLM loop could fill the trades table quickly. Consider a simple rate limit or sanity check.

- **SSE reconnection behavior**: The plan says "EventSource has built-in retry" but doesn't specify what happens to the sparkline data or price cache on reconnection. Does the frontend restart sparklines from scratch, or does it preserve accumulated data? This affects the user experience during brief network interruptions.

---

## 14. Independent Document Review — Questions, Gaps & Simplifications

*Review conducted 2026-03-09 by Claude Opus 4.6 (codex-reviewer). Based on reading the full PLAN.md and inspecting the implemented codebase: 8 market data modules under `backend/app/market/`, 73 tests passing, `pyproject.toml`, `backend/CLAUDE.md`. No frontend, database, Docker, or LLM integration code exists yet.*

### Executive Summary

PLAN.md is a high-quality specification — well-structured, opinionated, and detailed enough for most components. However, five categories of issues need attention before the next implementation phase: (1) the plan does not specify the FastAPI application entrypoint or how components are wired together, which is the most immediate blocker; (2) the SSE event format described in the plan contradicts the implemented code; (3) no API response schemas are defined, which will cause frontend/backend misalignment; (4) the LLM dependency (`litellm`) is missing from `pyproject.toml`; and (5) there is no development workflow section, so agents won't know how to run the app locally.

Section 13's self-review is strong and catches many real issues. This review confirms those findings and adds several not previously identified.

### Questions Requiring Resolution

**Q1. Where does the FastAPI `app` object live and how is the lifecycle wired?**
The plan describes routes, SSE, database init, and background tasks, but never specifies the application entrypoint. There is no `backend/app/main.py` (or equivalent). The next agent needs to know:
- Where the `FastAPI()` instance is created
- How `lifespan` or `on_event("startup")`/`on_event("shutdown")` starts and stops the `MarketDataSource`
- How the `PriceCache` singleton is shared across the SSE router, trade execution, and portfolio valuation
- How `StaticFiles` mounts the frontend build (must come after `/api/*` routes)

This is the single most critical gap — without it, the backend agent must invent the application architecture and may make choices that conflict with the existing market data code.

**Q2. What is the actual SSE event format?**
Section 6 says "Each SSE event contains ticker, price, previous price, timestamp, and change direction" — implying one event per ticker. The implemented `stream.py` (line 81-83) sends a single `data:` event containing ALL tickers as a JSON object keyed by symbol:
```
data: {"AAPL": {"ticker": "AAPL", "price": 190.50, ...}, "GOOGL": {...}, ...}
```
This is actually the better design (fewer events, atomic snapshot), but the plan must be updated to match the implementation or the frontend agent will build the wrong parser.

**Q3. What JSON shapes do the API endpoints return?**
Section 8 defines five REST endpoints but specifies zero response schemas. At minimum, the plan needs shapes for:
- `GET /api/portfolio` — How are positions structured? Does `unrealized_pnl` come from `PriceCache` prices? Is `total_value` included at the top level?
- `POST /api/portfolio/trade` — Does it return the executed trade, the updated portfolio, or both?
- `GET /api/watchlist` — Just ticker strings, or ticker objects with current prices from the cache?
- `POST /api/chat` — Is the response the raw LLM structured output or a wrapper with metadata?
- All errors — What HTTP status codes and body shape? FastAPI defaults to `{"detail": "..."}` for `HTTPException`, but this should be stated explicitly.

**Q4. Where does the main chart get its data?**
Section 10 says clicking a ticker shows "a larger detailed chart... with at minimum price over time." But the only data source is the SSE stream, which provides only the latest price. Is the main chart built from client-side accumulated SSE data (starts empty, fills in over time)? Or should there be a REST endpoint for historical prices? The plan must specify this — it changes both frontend and potentially backend work.

**Q5. How does the frontend keep portfolio value live?**
The header shows "portfolio total value (updating live)" and the positions table shows "current price." The only real-time channel is the price SSE stream. Three options exist:
- (a) Poll `GET /api/portfolio` periodically
- (b) Compute portfolio value client-side from SSE prices + locally-cached positions
- (c) Add a second SSE stream for portfolio data
Option (b) is most efficient but requires the frontend to maintain position state and recompute on every tick. The plan should specify the approach.

**Q6. Can a user buy a ticker not on the watchlist?**
The trade bar has a free-form ticker field. If the user types a ticker not on the watchlist, there is no price in the `PriceCache`, so the trade has no execution price. The plan should specify: either auto-add to the watchlist on trade, or require the ticker to be on the watchlist first (with a clear error message).

**Q7. What happens to positions when a ticker is removed from the watchlist?**
Section 13 question #6 raises this but does not resolve it. If a user holds 100 AAPL shares and removes AAPL from the watchlist, the `MarketDataSource.remove_ticker()` stops producing prices and `PriceCache.remove()` deletes the cached price. The portfolio display would then show stale or missing price data. Options:
- (a) Prevent removal of tickers with open positions (simplest, recommended)
- (b) Keep streaming the price even if removed from watchlist (complex — decouples watchlist from data source)
- (c) Allow removal and show last known price (misleading)

**Q8. Is database init on startup or on first request?**
Section 7 says "The backend checks for the SQLite database on startup (or first request)." These are different strategies. Startup initialization is simpler, avoids first-request latency, and is consistent with how the `MarketDataSource` must also start on boot (to begin producing prices). The plan should pick one — recommend startup via FastAPI `lifespan`.

**Q9. How does the backend know the database file path?**
The plan says the SQLite file lives at `db/finally.db` relative to the project root, and at `/app/db/finally.db` inside Docker. But the backend code will need a configurable path. Should there be a `DATABASE_PATH` env var, or should the backend hardcode a relative path and rely on the working directory? This must be specified.

**Q10. Is `OPENROUTER_API_KEY` actually required?**
Section 5 labels it "Required" but the watchlist, portfolio, market data, and trading features should all work without an LLM key — only chat depends on it. The plan should clarify that `OPENROUTER_API_KEY` is required only for the chat feature, and that the app starts and operates normally without it.

### Internal Contradictions

**C1. SSE event format** (Section 6 vs. implemented `stream.py`): As detailed in Q2, the plan describes per-ticker events but the code sends all tickers in a single event. Update the plan to match the code.

**C2. `backend/db/` vs. `db/`**: Section 4 says `backend/db/` contains "schema definitions, seed data, migration logic" and `db/` at the project root is the "runtime volume mount." These are two different directories with confusingly similar purposes. The existing code puts all backend modules under `backend/app/` — database code should follow the same pattern (e.g., `backend/app/db/`). The plan should reconcile these.

**C3. Charting library recommendation**: Section 10 says "Canvas-based charting library preferred (Lightweight Charts or Recharts)." Recharts is SVG-based, not canvas-based. These use fundamentally different rendering approaches with different performance characteristics. Recommend: Lightweight Charts (canvas, by TradingView) for the main price chart and sparklines; Recharts (SVG) for the P&L line chart if a simpler API is preferred there.

**C4. Root directory name**: Section 4 shows the tree rooted at `finally/` but the actual repo is `shisuke-financial-ally`. Cosmetic but could confuse agents referencing the tree.

### Gaps That Would Block or Delay Implementation

**G1. No development workflow section.**
Agents building the frontend and backend need to know how to run them locally outside Docker. The plan should specify:
- Backend: `cd backend && uv run uvicorn app.main:app --reload --port 8000`
- Frontend: `cd frontend && npm run dev` (port 3000)
- API proxy: Next.js `rewrites` in `next.config.js` to proxy `/api/*` to `localhost:8000`, OR FastAPI CORS middleware in dev mode
Without this, every agent will spend time figuring out how to run the project, potentially making incompatible choices.

**G2. `litellm` not in dependencies.**
Sections 3 and 9 say "LiteLLM → OpenRouter" for LLM calls. But `litellm` is not in `pyproject.toml`. Either add it, or specify using `openai` client with a custom `base_url` pointing to OpenRouter (which is the simpler approach and avoids LiteLLM's large dependency tree).

**G3. No `.env.example` file.**
Section 4 says ".env.example committed" but it doesn't exist. Should be created with placeholder values as part of project scaffolding.

**G4. No error response format.**
No standardized error response shape is defined. FastAPI defaults to `{"detail": "..."}` for `HTTPException` — the plan should state this explicitly, or define a custom format. Both the frontend and the LLM's trade-failure feedback depend on a predictable error shape.

**G5. SQLite concurrent write safety.**
The portfolio snapshot background task (every 30s), trade execution (on API request), and chat message storage (on API request) can all write to SQLite concurrently. SQLite's default journal mode serializes writes and can cause `SQLITE_BUSY` errors. The plan should specify WAL mode (`PRAGMA journal_mode=WAL`), which is standard practice for SQLite in web applications.

**G6. `GET /api/portfolio/history` has no query parameters.**
The P&L chart endpoint needs some way to control the time range. Without parameters, this returns an unbounded dataset that grows indefinitely. Specify at minimum a `limit` parameter (e.g., last N snapshots) or a `since` timestamp filter.

### Technical Risks

**R1. `threading.Lock` in an async application.**
The `PriceCache` (line 6, `cache.py`) uses `threading.Lock`, which blocks the event loop thread when contended. In practice this is likely fine because the lock is held for microseconds (dict operations), but it is architecturally incorrect for an asyncio application. The `MassiveDataSource` already uses `asyncio.to_thread()` for API calls, meaning the thread pool could contend with the event loop on the cache lock. Consider switching to `asyncio.Lock` or documenting why `threading.Lock` is acceptable here.

**R2. Next.js static export limitations.**
`output: 'export'` disables server-side features: API routes, middleware, image optimization, dynamic routes with `getServerSideProps`. The plan should note this explicitly so the frontend agent doesn't waste time trying to use unavailable Next.js features. All server logic must go through FastAPI.

**R3. Structured output reliability.**
The plan assumes the LLM always returns valid JSON. Even with structured output mode, edge cases occur. Define the fallback: if JSON parsing fails, return the raw text as the `message` with no `trades` or `watchlist_changes`.

**R4. No Docker layer caching guidance.**
The multi-stage Dockerfile installs both Node.js and Python dependencies. Without caching guidance, a clean build will take several minutes every time. The plan should note: copy `package.json`/`package-lock.json` before source code (and `pyproject.toml`/`uv.lock` before backend source) to leverage Docker layer caching.

### Confirming Section 13 Findings

| # | Finding | Verdict |
|---|---------|---------|
| 1 | SSE stream scope vs. watchlist | **Resolved by code**: `MarketDataSource` supports dynamic `add_ticker()`/`remove_ticker()`. The ticker set follows the watchlist. Update the plan to state this. |
| 2 | Portfolio snapshot timing (30s too sparse) | **Agree**. Recommend: snapshot on each trade + frontend interpolation from live SSE prices. Consider dropping the periodic background task entirely. |
| 3 | Chat history depth unspecified | **Agree**. Cap at 20 messages (10 exchanges). |
| 4 | Fractional shares in UI | **Agree**. Schema supports them; trade bar UI should accept whole numbers only. LLM may request fractional — allow it. |
| 5 | "Daily change %" baseline undefined | **Agree**. For simulator: change from first price since container boot. For Massive: actual daily change from market open. State both. |
| 6 | DELETE watchlist special characters | **Minor risk**. URL encoding handles `BRK.B` fine. The bigger issue (removing tickers with positions) is unresolved — see Q7. |
| 7-9 | Response shapes undefined | **Critical**. See Q3 above. |
| 10 | Model availability / fallback | **Agree**. Specify a fallback model. |
| 11 | Structured output reliability | **Agree**. See R3 above. |
| 12 | Trade atomicity for LLM multi-trade | **Agree**. Recommend best-effort: execute sequentially, skip failures, include both successes and failures in the response. |
| Drop `user_id` | **Agree**. Cognitive overhead for zero benefit. Multi-user requires auth, not just a column. |
| Drop `portfolio_snapshots` task | **Partially agree**. Keep snapshots on trade execution for page-refresh persistence. Drop the periodic 30s task. Frontend supplements with live-computed values. |
| Drop Windows PowerShell scripts | **Agree**. WSL2 and Git Bash cover Windows users. |
| Simplify `chat_messages.actions` | **Disagree**. Separate `actions` column lets the frontend render trade confirmations with distinct styling (icons, color, undo). JSON in TEXT is standard SQLite practice. |
| Vite vs. Next.js | **Worth considering**. For a static SPA with no server-side routing, Vite + React eliminates unused Next.js complexity. But if Next.js is a deliberate course teaching choice, keep it. |
| Treemap complexity | **Agree**. With ≤10 positions, a colored card grid or bar chart conveys the same information with far less implementation effort. |

### Additional Simplification Opportunities

**S1. Consolidate `backend/db/` into `backend/app/db/`.**
The existing backend puts all modules under `backend/app/`. Database code should follow the same convention rather than having a separate `backend/db/` directory. This avoids confusion with the runtime `db/` volume mount at the project root.

**S2. Use `openai` client instead of `litellm`.**
OpenRouter is OpenAI-API-compatible. Using the `openai` Python package with `base_url="https://openrouter.ai/api/v1"` is simpler than adding `litellm` (which has a large dependency tree and its own abstraction layer). This eliminates a dependency and reduces complexity.

**S3. Cap sparkline data arrays.**
The plan doesn't specify a limit on client-side price history accumulation for sparklines. Unbounded arrays will grow indefinitely in a long-running session. Recommend capping at 200 data points per ticker (a rolling window).

**S4. Drop the separate `users_profile` table.**
With `user_id` dropped (per Section 13 recommendation), the `users_profile` table holds exactly one row with just `cash_balance` and `created_at`. This could be a simple key-value entry in a `config` table, or even just an in-memory value seeded from a constant and persisted alongside trades. A dedicated table for a single scalar value is over-engineering.

### Prioritized Recommendations

**Must fix before next agent starts:**
1. Add "Backend Application Structure" section (FastAPI entrypoint, lifecycle, DI, static file serving)
2. Define API response schemas for all endpoints
3. Update SSE event format description to match implemented code
4. Add "Development Workflow" section (local dev, API proxy)
5. Resolve LLM dependency: add `litellm` or specify `openai` client with OpenRouter base URL

**Should fix before implementation proceeds:**
6. Specify chat history depth limit (20 messages)
7. Clarify `OPENROUTER_API_KEY` is optional (chat degrades gracefully)
8. Define behavior for buying tickers not on the watchlist
9. Define behavior for removing watched tickers with open positions
10. Specify SQLite WAL mode
11. Reconcile `backend/db/` vs `db/` directory confusion
12. Fix charting library recommendation (Recharts is SVG, not canvas)
13. Define "change %" baseline (first price since container start)

**Nice to have:**
14. Create `.env.example` file
15. Drop `user_id` column from all tables
16. Add Docker layer caching guidance to Dockerfile section
17. Specify fallback LLM model
18. Define `GET /api/portfolio/history` query parameters
19. Cap sparkline data points (200 per ticker)
20. Add `POST /api/reset` endpoint for resetting to initial state
