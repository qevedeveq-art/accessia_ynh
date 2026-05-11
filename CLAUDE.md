# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

YunoHost packaging for **ACCESSIA Pro CRM** — a proprietary CRM (Next.js frontend + FastAPI backend) run as Docker containers behind nginx. The upstream app code lives at a separate repo; this repo only handles deployment lifecycle.

## Key Files

| File | Purpose |
|------|---------|
| `manifest.toml` | App metadata, version, resource declarations (ports, dirs, permissions) |
| `scripts/install` | Clone upstream repo, generate SECRET_KEY, build Docker images, health-check backend |
| `scripts/upgrade` | WAL-safe SQLite backup → git pull → rebuild containers → reload nginx |
| `scripts/backup` / `restore` | SQLite + docker-compose.override.yml + nginx config |
| `scripts/remove` | `docker-compose down -v`, remove images, remove nginx config |
| `conf/nginx.conf` | Reverse proxy: `/` → port 3001 (Next.js), `/api/` → port 8001 (FastAPI), 120s timeout for AI calls |
| `conf/docker-compose.override.yml` | Template with `__VAR__` placeholders; substituted by `ynh_replace_vars` during install/upgrade |

## Architecture

- **Single-instance** (`multi_instance = false`) — only one install per YunoHost host
- **Auth**: YunoHost SSOwat injects `X-Remote-User` header; the app trusts it with no additional auth layer. Public routes (`/share`, `/sign`, `/nps` + their `/api/*` equivalents) are declared as `visitors`-allowed in `manifest.toml` with `auth_header = false`
- **Database**: SQLite at `$data_dir/_ACCESSIA_APP/sensia.db` with WAL mode — always use `PRAGMA wal_checkpoint(TRUNCATE)` before backup
- **Secrets**: `docker-compose.override.yml` is chmod 600, owned by root; SECRET_KEY is 64-char hex generated with `openssl rand`
- **Memory caps**: 512MB hard limit per container defined in the override template

## YunoHost Helpers Cheatsheet

```bash
# Variable substitution in conf/ templates — replaces __VARNAME__ tokens
ynh_replace_vars --file="$install_dir/docker-compose.override.yml"

# Expose a variable for substitution (set before ynh_replace_vars)
declare -A YNH_VARS; YNH_VARS[SECRET_KEY]="$secret_key"

# Standard helper for nginx config
ynh_config_add_nginx

# App user/dir helpers
$install_dir   # where source code lives (cloned from upstream)
$data_dir      # persistent data (SQLite DB, uploads)
```

## Common Operations

**Test nginx config after editing `conf/nginx.conf`:**
```bash
nginx -t
```

**Check backend health (as used in install script):**
```bash
curl -s http://localhost:8001/api/health
```

**Inspect running containers after install:**
```bash
cd $install_dir && docker-compose ps
cd $install_dir && docker-compose logs --tail=50 backend
```

**Validate manifest syntax:**
```bash
yunohost app manifest-validate manifest.toml
```

## Packaging Conventions

- Template variables use `__UPPER_SNAKE_CASE__` format (e.g. `__SECRET_KEY__`, `__PORT_MAIN__`)
- `ynh_replace_vars` replaces these from the YunoHost resource-declared values — do **not** use `sed` manually
- Port variables auto-declared by `[resources.ports]` in manifest are available as `$port_main`, `$port_backend`
- `docker-compose` (v1 CLI syntax, Debian Bookworm package) — not `docker compose` (plugin syntax)
- YunoHost helpers version: 2.1 (`helpers_version = "2.1"` in manifest)
