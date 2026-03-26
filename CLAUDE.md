# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI Manager is a cross-platform desktop app for managing AI assistant services (including the OpenClaw CLI stack under `~/.openclaw/`), built with Tauri 2.0 + React 18 + TypeScript + Rust. It provides a unified interface for configuring AI providers, messaging channels, and modules like ClaudeCode and Codex. Primary focus is Tuzi API integration.

## Build & Development Commands

```bash
npm install                  # Install frontend dependencies
npm run tauri:dev            # Full dev mode with hot reload (Vite + Tauri)
npm run tauri:build          # Production build (creates platform installers)
npm run dev                  # Frontend-only dev server (port 1420)
npm run build                # Frontend-only build (tsc + vite)
cd src-tauri && cargo check  # Check Rust code
cd src-tauri && cargo test   # Run Rust tests
```

No test framework (vitest/jest), ESLint, or Prettier is configured. TypeScript strict mode is the primary code quality gate.

## Architecture

### Frontend (src/)
- **Routing**: Manual state-based routing in `App.tsx` — no React Router. Three top-level modules (openclaw, claudecode, codex) with sub-pages managed via state.
- **State**: Zustand store (`src/stores/appStore.ts`) for global state (service status, in-app toast notifications). Local state via useState. Toasts are rendered by `ToastStack` (`src/components/ToastStack.tsx`); `src/lib/terminalToast.ts` pushes “open a new terminal” hints after CLI install/route actions.
- **Tauri IPC**: All backend calls go through `src/lib/tauri.ts` which wraps `invoke()` with logging (`invokeWithLog`). Type definitions for all backend responses live here.
- **Logging**: Module-based logger (`src/lib/logger.ts`) with color-coded console output, in-memory store (max 500 entries). Debug via `window.setLogLevel()`.
- **Styling**: TailwindCSS with custom dark theme. Colors defined in `tailwind.config.js` — primary is `claw-*` (lobster red #f94d3a), dark backgrounds `dark-900` to `dark-400`.
- **Animations**: Framer Motion for page transitions.
- **Path alias**: `@/` maps to `src/` (configured in vite.config.ts and tsconfig.json).

### Backend (src-tauri/)
- **Entry**: `src/main.rs` registers 50+ Tauri commands and initializes plugins (shell, fs, process, notification).
- **Commands**: Organized by domain in `src/commands/` — service.rs, config.rs, diagnostics.rs, installer.rs, claudecode.rs, codex.rs, skills.rs, process.rs.
- **Models**: `src/models/config.rs` mirrors the `~/.openclaw/openclaw.json` schema. `src/models/status.rs` for status types.
- **Utils**: `src/utils/platform.rs` (OS detection, config paths), `src/utils/shell.rs` (command execution with extended PATH for nvm/fnm/Volta), `src/utils/file.rs`.
- **Error pattern**: `Result<T, String>` throughout all commands.
- **Cross-platform**: `#[cfg(target_os = "...")]` for platform-specific code. Windows gets `CREATE_NO_WINDOW` for shell commands.

### Data Flow
```
React Component → invokeWithLog() → Tauri Command (Rust) → File System / Shell
                                                                    ↓
React Component ← State Update ← Response ← Result<T, String>
```

### Key Paths
- Config directory: `~/.openclaw/` (Unix) or `%USERPROFILE%\.openclaw\` (Windows)
- Config file: `~/.openclaw/openclaw.json`
- Environment file: `~/.openclaw/env` (shell-format key=value pairs)

## Conventions

- Components live in `src/components/{Feature}/index.tsx`
- Shared UI components: `src/components/InstallUI/`, `src/components/FaqUI/`
- Module registry: `src/modules/registry.ts`
- Type definitions: `src/types/modules.ts`
- Frontend uses PascalCase components, camelCase functions, `@/` import paths
- Rust uses snake_case, `#[command]` macro for Tauri handlers
- Service status polling runs every 3 seconds in App.tsx
- Vite dev server is fixed to port 1420 (required by Tauri)
- Release profile: LTO enabled, panic=abort, opt-level=s, strip=true
- `src-tauri/Cargo.lock` is listed in `.gitignore`; do not expect it in git. Bump workspace version in `Cargo.toml` only.

## Lessons learned (Claude Code, Codex, UI, release)

### Route file vs shell environment (Claude Code)

- `~/.config/tuzi/claude_route_status.txt` records `current_route` (e.g. `改版`), but **the CLI still reads `ANTHROPIC_BASE_URL` / `ANTHROPIC_API_KEY` from the shell** (often exported in `~/.zshrc` / `~/.bashrc`).
- After installing or switching to **改版**, call `clear_env_in_rc()` so stale `tu-zi` (or other) exports are removed; otherwise the UI shows 改版 while traffic still hits the old base URL.
- Switching **to** 改版 must clear rc env, not only skip `apply_env_to_rc`.

### Codex: multi-route `config.toml`

- Prefer **`write_codex_config_merged`** so all `[model_providers.*]` / `[profiles.*]` blocks for known routes stay in one file; do not rewrite only a single route and drop custom entries.
- **`filter_codex_config_strips`**: strip sections by route name set before rewriting.
- **Custom routes**: need a persisted `base_url` (via `override_base_url` in `configure_openai_route` or `add_codex_route`). Built-ins `gac` / `tuzi` can fall back to `route_base_url()`.
- **Install / reinstall**: `install_codex` and `reinstall_codex` accept optional **`route_base_url`** when the route name is a custom slug.
- **Shell compatibility**: official `setup_codex.sh` uses **`CODEX_KEY`**; the app writes **both** `CODEX_API_KEY` and `CODEX_KEY` in rc files and strips both on cleanup. `get_codex_status` treats either env var as the effective key for masking.
- **Route management UI**: enable editing when `install_type !== 'gac'`, not only `=== 'openai'`. CLI-only installs often yield `install_type: unknown` without `install_state`; treating that as non-editable disables all inputs by mistake.

### Install cards and perceived “no click response”

- **`InstallActionCard`**: when **`children`** (inputs) exist, put the primary **action button below** the form. A single full-width button that only wraps the title makes users click inputs and think nothing works.
- Use **`type="button"`** on segmented controls inside cards to avoid accidental form submit.

### Post-install UX: terminal reminder

- After **successful** install, upgrade, route switch, add route, or key update (see `src/lib/terminalToast.ts` for allowlists), call **`showNewTerminalToastIfNeeded`** so users open a **new terminal** after rc or config changes. Also honor backend **`restart_required`** on the action result.

### Versioning and release

- Keep **`package.json`**, **`src-tauri/Cargo.toml`**, and **`src-tauri/tauri.conf.json`** versions **in sync** for each release.
- Pushing a tag matching **`v*`** runs `.github/workflows/build.yml`, which creates a **draft** GitHub Release (`softprops/action-gh-release`). Publish the draft after verifying artifacts.
- If `git push` returns **403**, the local commit/tag are still valid; push with credentials that have write access to the org repo (or fix `remote` URL / PAT / SSH).
