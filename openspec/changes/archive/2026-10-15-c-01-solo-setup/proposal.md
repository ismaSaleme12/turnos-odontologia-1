# Proposal

## Why

El directorio `src/` quedó plano-mezclado (backend + frontend en la misma raíz más un `src/src/` anidado), lo que diverge de `knowledge-base/08_arquitectura_propuesta.md` §Estructura y del layout mudado que documenta `CHANGES.md` C-01. Sin esta reestructura no se pueden cerrar las verificaciones pendientes del ciclo fundacional (tasks 1.4/4.2/4.3: `compose up --build` + health 200, CI verde, verificación integral).

## What Changes

- Reestructura SOLO de layout con `git mv` (sin tocar modelos ni datos):
  - Backend → `src/backend/`: `app/` → `src/backend/app/`, `alembic/` → `src/backend/alembic/`, `alembic.ini` + `pyproject.toml` + `Dockerfile` → `src/backend/`.
  - Frontend → `src/frontend/`: `package.json`, `package-lock.json`, `vite.config.ts`, `tsconfig*.json`, `tailwind.config.js`, `postcss.config.js`, `index.html`, `nginx.conf`, `public/`, `src/` anidado, `Dockerfile.frontend` → `src/frontend/` (el anidado queda en `src/frontend/src/`).
- Rewiring de paths sin cambiar comportamiento:
  - `docker-compose.yml`: 3 contexts `./src` → `./src/backend` (api, worker, migrar) + servicio frontend si corresponde.
  - `src/backend/Dockerfile` + `src/frontend/Dockerfile.frontend`: WORKDIRs y COPYs ajustados a su nuevo contexto.
  - `pytest.ini` (`pythonpath`), `tests/conftest.py` (`sys.path`), `pyproject.toml` (pytest config), `alembic.ini` (`prepend_sys_path` / `script_location`).
  - `.github/workflows/ci.yml`: 6 paths (install backend `pip install -e`, `working-directory` alembic, `ruff`/`mypy` config+paths, `cache-dependency-path`, `working-directory` npm).
  - `vite.config.ts` / `tsconfig*.json` / `tailwind` / `index.html`: includes y contents relativos al nuevo root frontend.
- Criterio de done: `docker compose up --build` + `GET /api/health` → 200, `pytest` verde, `tsc --noEmit` + `vite build` verdes, `ruff` + `mypy` verdes (cierra tasks 1.4/4.2/4.3 pendientes del archive).
- PROHIBIDO en este change: tocar modelos, migraciones, seeds, GRANTs de auditoría o reglas de negocio (task 2.4 queda pendiente para un change CRITICO separado; governance de este change: BAJO).

## Capabilities

### New Capabilities

- Ninguna (refactor puro de layout; no cambia comportamiento observable).

### Modified Capabilities

- Ninguna (los requirements de `foundation-setup` y `core-models-multitenant` no cambian; solo cambia dónde vive el código que los implementa).

## Impact

- Afecta: `src/**`, `docker-compose.yml`, `pytest.ini`, `tests/conftest.py`, `.github/workflows/ci.yml`, `alembic.ini` (movido), Dockerfiles, configs Vite/TS/Tailwind.
- **BREAKING** solo para paths locales (imports `from app...` se preservan vía `pythonpath` ajustado; ningún endpoint, schema o contrato API cambia).
- Desbloquea: cierre verificable de C-01 fundacional y base de paths estable para C-03 en adelante.
- No toca: modelos SQLAlchemy, migración 001, seed, permisos DB, secretos (R10), datos reales (R13: solo fixtures sintéticos en verificaciones).
