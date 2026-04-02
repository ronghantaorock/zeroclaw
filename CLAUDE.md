# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ZeroClaw is a Rust-first autonomous AI agent runtime (<5MB RAM target). It connects to 30+ messaging channels, 20+ LLM providers, 80+ tools, and optional hardware peripherals. The binary embeds a React web dashboard.

## Common Commands

```bash
# Build
cargo build                          # debug
cargo build --release --locked       # release (size-optimized, slow compile)
cargo build --profile release-fast   # release (faster compile, parallel codegen)

# Test
cargo test --locked                  # all tests
cargo test --lib                     # unit tests only (fastest)
cargo test --test component          # component tests
cargo test --test integration        # integration tests
cargo test --test system             # system tests
cargo test --test live -- --ignored  # live tests (needs credentials)
cargo test <test_name>               # single test by name

# Lint & Format
cargo fmt --all -- --check           # format check
cargo fmt --all                      # format apply
cargo clippy --all-targets -- -D warnings  # lint (warnings = errors in CI)

# Full pre-PR validation (runs in Docker)
./dev/ci.sh all

# Quality gate (used by pre-push hook)
./scripts/ci/rust_quality_gate.sh
./scripts/ci/rust_quality_gate.sh --strict   # all clippy warnings

# Web frontend (auto-built by build.rs, manual if needed)
cd web && npm ci && npm run build

# Just task runner shortcuts
just ci       # fmt-check + lint + test
just test     # all tests
just test-lib # unit tests
just lint     # clippy
just fmt      # format
just doc      # cargo doc --open
```

## Architecture

Trait-driven, modular design. Extend by implementing a trait and registering in the corresponding factory module.

**Key extension traits:**
- `src/providers/traits.rs` — `Provider` (LLM providers: OpenAI, Anthropic, Gemini, Ollama, etc.)
- `src/channels/traits.rs` — `Channel` (messaging: Telegram, Discord, Slack, WhatsApp, etc.)
- `src/tools/traits.rs` — `Tool` (shell, file, browser, git, Jira, etc.)
- `src/memory/traits.rs` — `Memory` (SQLite, Markdown, PostgreSQL, Qdrant vector)
- `src/observability/traits.rs` — `Observer`
- `src/runtime/traits.rs` — `RuntimeAdapter` (native, Docker)
- `src/peripherals/traits.rs` — `Peripheral` (STM32, RPi GPIO)

**Core flow:** `main.rs` (CLI/Clap) -> `agent/` (orchestration loop: prompt construction -> provider call -> tool dispatch -> loop) -> `gateway/` (Axum HTTP/WS/SSE server, web dashboard, webhooks)

**Key directories:**
- `src/config/` — layered TOML config schema and loading (`~/.zeroclaw/config.toml`)
- `src/security/` — DM pairing, workspace sandboxing, secret store, autonomy levels
- `src/hands/` — multi-agent orchestration (agent swarms)
- `src/hooks/` — lifecycle hooks for LLM calls and tool executions
- `src/sop/` — event-driven Standard Operating Procedures
- `src/plugins/` — WASM plugin system (feature-gated: `plugins-wasm`)
- `src/tunnel/` — tunnel support (Cloudflare, Tailscale, ngrok)
- `crates/robot-kit/` — robot control toolkit crate
- `crates/aardvark-sys/` — low-level hardware adapter bindings (`unsafe` only permitted here)
- `apps/tauri/` — desktop Tauri 2.0 app
- `web/` — React 19 + Vite 6 + Tailwind 4 + TypeScript dashboard (embedded via `rust-embed`)
- `firmware/` — ESP32, STM32, Arduino, Pico firmware targets

**Feature gating:** Many subsystems are behind Cargo features (e.g., `channel-matrix`, `hardware`, `plugins-wasm`, `whatsapp-web`, `voice-wake`).

## Test Organization

| Tier | Location | Command |
|------|----------|---------|
| Unit | `src/**` (inline `#[cfg(test)]`) | `cargo test --lib` |
| Component | `tests/component/` | `cargo test --test component` |
| Integration | `tests/integration/` | `cargo test --test integration` |
| System | `tests/system/` | `cargo test --test system` |
| Live | `tests/live/` | `cargo test --test live -- --ignored` |
| Benchmarks | `benches/` | `cargo bench` |
| Fuzz | `fuzz/` | `cargo fuzz` |

Test fixtures in `tests/fixtures/`, shared support code in `tests/support/`. CI uses `cargo-nextest` for parallel execution.

## Code Style & Constraints

- **Max line width:** 100 chars (`rustfmt.toml`)
- **Clippy:** pedantic, warnings are errors. Thresholds: cognitive-complexity=30, too-many-args=10, too-many-lines=200
- **TOML formatting:** enforced via `taplo`
- **Rust edition:** 2021, MSRV 1.87
- **Release profile:** opt-level "z" (size), fat LTO, strip, panic=abort
- **License:** Dual MIT OR Apache-2.0

## Risk Tiers

- **Low:** docs/chore/tests-only changes
- **Medium:** most `src/**` behavior changes without boundary/security impact
- **High:** `src/security/**`, `src/runtime/**`, `src/gateway/**`, `src/tools/**`, `.github/workflows/**`, access-control boundaries

When uncertain, classify as higher risk.

## Workflow Rules

- Read before write — inspect existing module, factory wiring, and adjacent tests before editing.
- One concern per PR — no mixed feature+refactor+infra patches.
- Minimal patch — no speculative abstractions or config keys without a concrete use case.
- Work from a non-`master` branch, PR to `master`. Conventional commit titles. Small PRs preferred.
- Follow `.github/pull_request_template.md`. Never commit secrets or personal data.
- Pre-push hook runs quality gate + tests. Enable: `git config core.hooksPath .githooks`

## Anti-Patterns

- No heavy dependencies for minor convenience.
- No silently weakening security policy or access constraints.
- No speculative config/feature flags "just in case".
- No massive formatting-only changes mixed with functional changes.
- No modifying unrelated modules "while here".
- No bypassing failing checks without explicit explanation.
- No hiding behavior-changing side effects in refactor commits.

## Key References

- `AGENTS.md` — cross-tool AI agent instructions (shared with other AI tools)
- `docs/contributing/change-playbooks.md` — adding providers, channels, tools, peripherals
- `docs/contributing/pr-discipline.md` — privacy rules, superseded-PR templates
- `docs/contributing/docs-contract.md` — docs system contract, i18n rules
