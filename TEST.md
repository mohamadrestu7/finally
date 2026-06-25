# FinAlly — Project Review

## Overview

FinAlly is an AI-powered trading workstation built entirely by coding agents as a capstone project for an agentic AI coding course. The concept is ambitious and well-scoped: a Bloomberg-style terminal with a built-in LLM assistant that can analyze portfolios and execute trades through natural language.

---

## Architecture Assessment

### Strengths

**Single-container simplicity** — The decision to serve everything (static frontend + API + SSE) on one port from one Docker container is excellent. It eliminates CORS concerns entirely, makes deployment trivial, and reduces the cognitive overhead for students running the project. One `docker run` command and you're live.

**SSE over WebSockets** — A well-reasoned choice. The data flow is strictly server→client (price updates), so WebSocket bidirectionality adds complexity with no benefit. SSE has built-in browser reconnection, works through HTTP/2 multiplexing, and is simpler to debug.

**SQLite with lazy initialization** — Perfect for a single-user simulation. No migration tooling, no separate DB service, no startup ordering issues. The `user_id` column defaulting to `"default"` is a pragmatic forward-compatibility nod without over-engineering.

**Abstract market data interface** — Having both the GBM simulator and the Massive API client implement the same interface is the right design. All downstream code (SSE, price cache, portfolio) remains agnostic to the data source, and switching is a single env-var change.

**uv for Python** — Modern, fast, reproducible. Good choice for a course project where students need to reproduce environments reliably.

### Areas to Watch

**No authentication** — Intentional and documented, but worth flagging: the single-user `"default"` model means any network access to port 8000 has full control over the portfolio and chat. Fine for local development; needs a note if ever exposed publicly.

**SQLite under concurrent writes** — If the portfolio snapshot background task and a trade execution happen simultaneously, SQLite's write locking could cause transient failures. WAL mode (`PRAGMA journal_mode=WAL`) should be enabled at initialization to reduce this risk.

**LLM auto-execution without confirmation** — The design doc justifies this as a deliberate demo choice (fake money, impressive UX). Valid for the course context, but it means a malformed or hallucinated LLM response could silently alter portfolio state. The structured output schema and validation logic are the only guardrail — they need to be robust.

---

## Market Data Component (Completed)

The GBM simulator is well-conceived. Using geometric Brownian motion with per-ticker drift/volatility parameters, correlated moves across sectors, and occasional random "events" will produce price action that feels realistic enough to make the demo compelling without requiring any external dependencies. Starting from realistic seed prices (AAPL ~$190, NVDA, etc.) grounds the simulation in familiar territory for users.

---

## Frontend Design

The Bloomberg/terminal aesthetic with the specified color palette (accent yellow `#ecad0a`, blue `#209dd7`, purple `#753991`) is a strong visual direction. The sparkline-from-SSE approach — accumulating price history on the client since page load rather than pre-loading historical data — is a smart simplification that still produces a useful visual.

The price flash animation (brief green/red CSS highlight fading over ~500ms) is a well-understood pattern that makes streaming data feel alive. The implementation detail to watch: applying and removing a CSS class rather than inline styles ensures the browser's transition engine handles the fade cleanly.

---

## Testing Strategy

The split between unit tests (pytest / React Testing Library) and E2E tests (Playwright in a separate docker-compose) is appropriate. Running E2E with `LLM_MOCK=true` is the right call — it makes tests fast, free, and deterministic. The mock must cover enough response variety (trades, watchlist changes, plain messages) to give the E2E suite real coverage.

---

## Overall Assessment

**FinAlly is a well-designed capstone project.** The architectural decisions are consistent, well-justified, and appropriate to the constraint of "one Docker command, no signup, immediately impressive." The project demonstrates real production patterns — SSE streaming, structured LLM outputs, agentic auto-execution, abstract data source interfaces — while keeping operational complexity low.

The market data layer is complete. The remaining work (frontend UI, portfolio API, chat integration, Docker build, E2E tests) is substantial but well-specified. The planning documentation is unusually clear, which should enable agent-driven development to proceed with minimal ambiguity.

**Risk areas to monitor during development:**
1. SQLite WAL mode for concurrent access safety
2. LLM structured output validation robustness
3. SSE client reconnection behavior under flaky network conditions
4. Static Next.js export compatibility with any dynamic routing patterns
