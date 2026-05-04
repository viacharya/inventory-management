# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System — full-stack demo with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
- **vue-expert**: **MANDATORY** for creating or significantly modifying any `.vue` file
- **code-reviewer**: Use after writing significant code
- **Explore**: Use for codebase exploration and pattern searches
- **general-purpose**: Use for complex multi-step tasks

### Skills
- **backend-api-test**: Use when writing or modifying tests in `tests/backend/`

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches only — use `git checkout -b`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Frontend: `http://localhost:3000` | API: `http://localhost:8001`

## Stack
- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI + Uvicorn (port 8001) | API docs: `http://localhost:8001/docs`
- **Data**: JSON files in `server/data/` loaded at startup into memory via `server/mock_data.py`

## Commands

### Start servers
```bash
# macOS/Linux one-command
./scripts/start.sh

# Manual (run in separate terminals)
cd server && uv run python main.py
cd client && npm run dev
```

### First-time backend setup
```bash
cd server
uv venv && uv sync
```

### Tests
```bash
# All backend tests
cd tests && uv run pytest -v

# Single file
cd tests && uv run pytest backend/test_orders.py -v

# With coverage
cd tests && uv run pytest --cov=../server

# Frontend build check (no test suite)
cd client && npm run build
```

### Frontend build
```bash
cd client && npm run build  # Output: client/dist/
```

## Architecture

**Filter System**: 4 global filters (Time Period, Warehouse, Category, Order Status) flow as query params from the `useFilters()` composable → `client/src/api.js` → FastAPI → in-memory filtering → Pydantic-validated JSON → Vue computed properties.

**Reactivity pattern**: Raw API data in `ref()` (`allOrders`, `inventoryItems`), derived/filtered data in `computed()`. Watch filter changes to re-fetch.

**Composables** (`client/src/composables/`):
- `useFilters()` — shared filter state across all views
- `useAuth()` — user authentication state
- `useI18n()` — translations (English/Japanese, locales in `client/src/locales/`)

**Routes** (`client/src/main.js`):
```
/ → Dashboard   /inventory → Inventory   /orders → Orders
/spending → Spending   /demand → Demand   /reports → Reports
```

## API Endpoints
- `GET /api/inventory` — Filters: warehouse, category (no month)
- `GET /api/orders` — Filters: warehouse, category, status, month
- `GET /api/dashboard/summary` — All filters
- `GET /api/demand`, `/api/backlog` — No filters
- `GET /api/spending/summary|monthly|categories|transactions`
- `GET /api/reports/quarterly`, `/api/reports/monthly-trends`

## Common Issues
1. Use unique keys in `v-for` — use `sku`, `month`, etc., never `index`
2. Validate dates before calling `.getMonth()`
3. Update Pydantic models in `server/main.py` when changing JSON data structure
4. Inventory endpoint does not support `month` filter (no time dimension on inventory)
5. Revenue goals: $800K/month (single warehouse), $9.6M YTD (all warehouses)

## File Locations
- Views: `client/src/views/*.vue`
- Components: `client/src/components/*.vue`
- API client: `client/src/api.js`
- Composables: `client/src/composables/`
- Backend: `server/main.py`, `server/mock_data.py`
- Data: `server/data/*.json`
- Tests: `tests/backend/test_*.py`
- Global styles: `client/src/App.vue`

## Code Style
- Always document non-obvious logic changes with comments

## Design System
- Colors: Slate/gray (`#0f172a`, `#64748b`, `#e2e8f0`)
- Status colors: green / blue / yellow / red
- Charts: Custom SVG; layouts use CSS Grid
- No emojis in UI
