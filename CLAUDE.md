# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Ponto Eletrônico** is a Brazilian employee time-tracking (punch clock) web application built with Laravel 7.0 / PHP 7.4, running on Docker with MySQL 5.7. All routes, models, and table names are in Portuguese.

## Development Commands

All commands run inside the Docker container via `docker compose exec app`:

```bash
# Start environment
docker compose up -d --build

# Install PHP dependencies
docker compose exec app composer install

# Generate app key (first-time setup)
docker compose exec app php artisan key:generate

# Run database migrations (creates tables and seeds default data)
docker compose exec app php artisan migrate

# Clear config cache after editing .env
docker compose exec app php artisan config:clear

# Run all tests (uses SQLite in-memory, no DB needed)
docker compose exec app ./vendor/bin/phpunit

# Run a single test file
docker compose exec app ./vendor/bin/phpunit tests/Feature/ExampleTest.php

# Run a specific test method
docker compose exec app ./vendor/bin/phpunit --filter testBasicTest

# Open Laravel REPL (useful for updating user credentials)
docker compose exec app php artisan tinker

# View application logs
docker compose logs -f app
```

App is served on `http://localhost:8000`. The `.env` key `APP_URL` must match the actual host/port — all controller redirects use `getenv('APP_URL')` directly.

## Authentication Architecture

The app has **two completely separate authentication surfaces**, each with its own session namespace and middleware. Laravel's built-in Auth system is **not used**.

| Surface | URL prefix | Session key | Middleware alias |
|---|---|---|---|
| Employee portal | `/` | `login.ponto.usuario_id` | `authMiddleware` |
| Admin panel | `/painel` | `login.ponto.painel.usuario_id` | `authPainelMiddleware` |

Passwords are stored and compared using **SHA1**: `hash('sha1', $password)`.

`admin` flag on `usuario`: `0` = employee, `1` = admin/supervisor. Employees log in via `LoginController`, admins via `LoginPainelController`. Admin credentials are rejected at the employee login (query includes `'admin' => 0`).

Default seeded credentials (CPF `57761749370`, password `password123` → SHA1 `89e495e7941cf9e40e6980d14a16bf023ccd4c91`). To change credentials use Tinker:
```php
DB::table('usuario')->where('id', 1)->update(['cpf' => 'NEW_CPF', 'senha' => hash('sha1', 'NEW_PASSWORD')]);
```

## Code Architecture

### Controller Structure

All controllers live under `App\Http\Controllers\PontoEletronico\` and extend the abstract `PontoEletronicoController` (not Laravel's base `Controller` directly). The abstract base provides an `upload()` helper for file attachments.

```
app/Http/Controllers/PontoEletronico/
├── PontoEletronicoController.php   # Abstract base with upload() helper
├── IndexController.php / IndexPainelController.php   # Login page renders
├── LoginController.php / LoginPainelController.php   # Auth logic
├── DashboardController.php         # Employee today's punches view
├── DashboardPainelController.php   # Admin dashboard
├── PontoController.php             # Employee punch registration (entry/exit)
├── PontoPainelController.php       # Admin punch adjustment from acompanhamento
├── PontoAjusteController.php       # Adjustment requests (CRUD + certification)
├── AcompanhamentoController.php    # Time report + Excel download
└── UsuarioController.php           # Employee CRUD (admin only)
```

### Models and Database Schema

Models are in the `App\` root namespace (not `App\Models\`). Table names are singular Portuguese words.

| Model | Table | Key fields |
|---|---|---|
| `Usuario` | `usuario` | `cpf`, `senha` (SHA1), `admin` (0/1), `ativo` (0/1) |
| `Ponto` | `ponto` | `usuario_id`, `data` (date), `entrada`/`saida` (time), `entrada_status`/`saida_status`, `status` |
| `PontoAjuste` | `ponto_ajuste` | `usuario_id`, `ponto_id`, `ponto_ajuste_id`, `tipo` (entrada/saida/periodo), `data`, `hora`, `ponto_razao_id`, `status`, `anexo` |
| `PontoRazao` | `ponto_razao` | `descricao`, `ativo`; no timestamps |

**Status codes on `ponto`:**
- `entrada_status` / `saida_status`: `0` = self-registered, `1` = adjustment pending supervisor approval, `2` = adjustment approved
- `status`: reserved (currently always `0`)

**Status codes on `ponto_ajuste`:**
- `0` = pending review, `1` = approved, `2` = rejected

**`ponto_ajuste` pairing**: Period adjustments create two linked records. The first record (`tipo=entrada`) has `ponto_ajuste_id = 0`; the second (`tipo=saida`) has `ponto_ajuste_id` = the first record's id. Deleting one cascades to the other via the pairing logic in `PontoAjusteController::delete()`.

### Date Format Convention

Views and form inputs use **DD/MM/YYYY** format. Controllers manually convert to **YYYY-MM-DD** for database queries:
```php
$data_arr = explode("/", $data);
$data_db = $data_arr[2].'-'.$data_arr[1].'-'.$data_arr[0];
```

### Flash Messages / Status Pattern

Status feedback is passed between requests via session keys — not Laravel's `with()` flash shorthand:

```php
Session::put('status.msg', 'Success message');         // Toast/alert text
Session::put('status.error_redirect', $url);           // Optional redirect URL shown in UI
Session::put('status.msg_confirm', 'Confirm text?');   // Confirmation dialog text
Session::put('status.redir_confirm', $url);            // URL to POST to on confirm
```

### Views

Blade templates live under `resources/views/pontoeletronico/`. Layout files (`painel.blade.php`, `admin.blade.php`) are used as base templates by including them in sub-views. There is no `@extends`/`@section` hierarchy — views use `@include` or direct HTML composition.

### File Uploads

Attachments (justification documents for adjustments) are uploaded to `public/upload/razao/` with md5-randomized filenames, handled by `PontoEletronicoController::upload()`. The directory must exist and be writable.

## Docker Setup

Two services defined in `docker-compose.yml`:
- **app**: PHP 7.4-apache on port 8000, mounts the entire repo into `/var/www/html`, document root set to `/var/www/html/public`
- **db**: MySQL 5.7 container named `ponto_db`; DB host inside the app container is `db` (service name)

The database volume is persisted to `./dbdata/` (excluded from git).
