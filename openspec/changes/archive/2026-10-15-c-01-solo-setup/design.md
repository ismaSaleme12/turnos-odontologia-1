# Design

## Context

Ver `proposal.md` (Why) para la motivación. Estado actual verificado en disco: `src/` plano-mezclado contiene a la vez backend (`app/`, `alembic/`, `alembic.ini`, `pyproject.toml`, `Dockerfile`) y frontend (`package.json`, `package-lock.json`, `vite.config.ts`, `tsconfig*.json`, `tailwind.config.js`, `postcss.config.js`, `index.html`, `nginx.conf`, `public/`, `src/` anidado, `Dockerfile.frontend`). El archive `2026-10-03-c-01-foundation-core-models` es historia inmutable (NO modificarlo); sus tasks 1.4/4.2/4.3 quedaron sin tildar y este change las cierra por vía de paths. Constraints: imports `from app...` en todo el backend, `git mv` para conservar historia, R10 (sin secretos) y R13 (solo fixtures sintéticos en verificaciones), governance BAJO.

## Goals / Non-Goals

**Goals:**

- Layout final `src/backend/` + `src/frontend/` con historia git preservada y cero cambios de comportamiento (mismos endpoints, schemas, migraciones).
- Un solo criterio de verdad por entrypoint: `pytest` corre desde raíz y desde `src/backend/`; `alembic` corre desde `src/backend/`; `npm` corre desde `src/frontend/`.
- Cerrar 1.4 (`compose up --build` + health 200), 4.2 (CI paralela verde) y 4.3 (verificación integral) del archive sin reabrirlo.

**Non-Goals:**

- Task 2.4 (GRANT INSERT+SELECT sobre `auditoria`) queda FUERA: es governance CRITICO y va en un change propio, no mezclado con este BAJO.
- No se agrega servicio frontend a `docker-compose.yml` (solo rewiring de los 3 contexts existentes); no se toca RLS, seeds, migración 001 ni reglas de negocio.
- No se renombran imports (`from app...` se preserva vía `pythonpath`; renombrar a `from backend.app...` sería churn innecesario y rompería el archive como referencia).

## Decisions

### D1. Layout destino: `src/backend/` + `src/frontend/`, `tests/` y `db/` quedan en raíz

**Qué:** backend completo bajo `src/backend/`; frontend completo bajo `src/frontend/` (el `src/` anidado queda en `src/frontend/src/`); `tests/`, `db/init/`, `pytest.ini` raíz y `docker-compose.yml` raíz NO se mueven.
**Por qué:** `tests/` valida desde raíz en CI (`python -m pytest tests`) y `db/init/` lo monta compose por path relativo a raíz; moverlos agranda el blast radius sin beneficio.
**Alternativa considerada:** `backend/` + `frontend/` en raíz según KB §08 literal — descartada porque CHANGES.md C-01 ya fijó el layout mudado `src/` como decisión documentada y el costo de mover compose/tests/db supera al beneficio nominal.

### D2. Moves con `git mv` en dos tandas (backend, luego frontend), sin editar contenido en el mismo commit

**Qué:** tanda 1: `src/app→src/backend/app`, `src/alembic→src/backend/alembic`, `src/alembic.ini` + `src/pyproject.toml` + `src/Dockerfile` + `src/.dockerignore→src/backend/`; tanda 2: `package.json`, `package-lock.json`, `vite.config.ts`, `tsconfig*.json`, `tailwind/postcss`, `index.html`, `nginx.conf`, `public/`, `src/` anidado, `Dockerfile.frontend→src/frontend/`. Commits separados por tanda + uno de rewiring + uno de verificación.
**Por qué:** `git mv` preserva historia por archivo y los commits separados permiten revertir el rewiring sin perder los moves.
**Alternativa:** mover y reescribir paths en un solo commit — descartada por hacer el review imposible de auditar.

### D3. `pythonpath` unificado, imports intactos

**Qué:** `pytest.ini` raíz: `pythonpath = src/backend` (antes `src`); `tests/conftest.py`: `parents[1] / "src"` → `parents[1] / "src/backend"`; `pyproject.toml` (ya en `src/backend/`): `ini_options.pythonpath = ["."]`, `testpaths = ["../../tests"]` (antes `["../tests"]`, válido solo cuando `pyproject` vivía en `src/`).
**Por qué:** los imports `from app...` siguen resolviendo igual desde cualquier entrypoint (raíz vía `pytest.ini`, `src/backend/` vía pyproject); fuente canónica para CI (`python -m pytest tests` desde raíz) es `pytest.ini`.
**Alternativa:** renombrar imports a `from backend.app...` — descartada (churn masivo, rompe referencia al archive, cero valor funcional).

### D4. `alembic.ini` se muda sin cambios de contenido

**Qué:** `script_location = alembic` y `prepend_sys_path = .` quedan tal cual; la disciplina es de cwd: `alembic` SIEMPRE se invoca con cwd `src/backend/` (compose `migrar`, CI `working-directory`, dev local).
**Por qué:** ambas claves son relativas al cwd, que tras la mudanza coincide (`src/backend/` contiene `alembic/` igual que `src/` lo contenía); tocarlas sería churn sin efecto.
**Alternativa:** absolutizar `script_location` — descartada (rompe portabilidad Windows/Linux de CI).

### D5. Docker: cambian los contexts, NO los WORKDIRs internos

**Qué:** `docker-compose.yml`: 3 × `context: ./src` → `context: ./src/backend` (api, worker, migrar; `dockerfile: Dockerfile` ya resuelve al estar en la raíz del nuevo contexto). WORKDIRs de contenedor (`/srv`, `/srv/src`) y COPYs relativos del backend se conservan. `Dockerfile.frontend` no cambia COPYs (su contexto pasa a `./src/frontend/` y sus rutas relativas ya coinciden). Compact rules aplicadas (skill `multi-stage-dockerfile`): builder separado de runtime, tag exacto, `.dockerignore` siempre (el backend conserva el mudado; verificar en apply que no excluya `alembic.ini`), USER no-root, HEALTHCHECK por tipo de app, sin secretos en imagen (env solo por `${VAR}`).
**Por qué:** los paths de contenedor son independientes del host; minimizar el diff a contexts reduce el riesgo de imágenes rotas.
**Alternativa:** reescribir WORKDIRs a `/srv/app` — descartada (invalida HEALTHCHECK y CMD probados sin motivo).

### D6. CI: 6 paths, misma topología de jobs

**Qué:** en `.github/workflows/ci.yml`: (1) `pip install -e "./src[dev]"` → `"./src/backend[dev]"`; (2) `working-directory: src` del paso migrar → `src/backend`; (3) `ruff --config src/pyproject.toml` → `src/backend/pyproject.toml`; (4) `mypy src/app src/alembic` → `mypy src/backend/app src/backend/alembic`; (5) `cache-dependency-path: src/package-lock.json` → `src/frontend/package-lock.json`; (6) `working-directory: src` de los 2 pasos npm → `src/frontend`. Jobs siguen paralelos, sin agregar jobs nuevos.
**Por qué:** es el mínimo que restaura la señal verde de task 4.2; cambiar topología de jobs sería scope creep.
**Alternativa:** unificar todo en un job — descartada (el archive D6 fijó CI paralela como decisión y 4.2 la exige).

### D7. Frontend: cero cambios de contenido por relatividad + guards R5/R6 intactos

**Qué:** `vite.config.ts`, `tsconfig*.json` (`include: ["src"]`), `tailwind content`, `index.html → /src/main.tsx` NO cambian contenido: al moverse juntos a `src/frontend/`, sus relativos siguen resolviendo. Se verifica `tsc --noEmit` + `vite build` en verde y `grep -r "any"` limpio (R5), y que ningún dato clínico toque `localStorage` (R6, precedente aunque este change no maneja clínica). Compact rules aplicadas (skill `vercel-react-best-practices`, adaptada a Vite SPA sin RSC): sin barrel imports, sin `useMemo`/`useCallback` especulativos, `startTransition` para updates no urgentes.
**Por qué:** mover el árbol completo preserva la invariancia de relativos; cualquier edición de contenido sería riesgo sin retorno.
**Alternativa:** reescribir includes a `src/frontend/src` absolutos — descartada (acopla el frontend a la posición del monorepo).

## Risks / Trade-offs

- [Risk] Algún path cableado a `src/` que la auditoría no vio (ej. `tailwind content`, `nginx.conf`, `public/` refs, `Dockerfile.frontend` COPYs) rompe el build → Mitigación: tasks ordenan backend → frontend → verificaciones con `grep -rn` de `src/` residual y builds en verde por capa antes del integral.
- [Risk] `.dockerignore` mudado excluye algo que el nuevo contexto necesita (o el frontend queda sin ignores y la imagen engorda) → Mitigación: task explícita de revisión de `.dockerignore` por contexto antes del `compose up --build`.
- [Risk] Doble config pytest (`pytest.ini` raíz vs `ini_options` en pyproject) diverge a futuro → Mitigación: D3 deja ambas coherentes y tasks verifican `pytest` desde AMBOS cwds; documentar en tasks que la canónica para CI es la raíz.
- [Risk] `alembic` invocado desde cwd equivocado deja de encontrar `alembic/` → Mitigación: D4 + tasks que verifican `upgrade head`/`downgrade -1`/`upgrade` solo desde `src/backend/`.
- [Trade-off] Se conserva `from app...` en vez de namespacing `backend.app`: acoplamiento leve al `pythonpath` a cambio de cero churn y archive intacto como referencia → aceptado y reversible en cualquier change futuro si se desea.
- [Trade-off] Task 2.4 (GRANT auditoría) sigue pendiente tras este change: se acepta conscientemente por no mezclar governance CRITICO en un change BAJO; queda como next action explícito.

## Migration Plan

1. `git mv` tanda backend → commit; `git mv` tanda frontend → commit (verificable con `git status` + historia por archivo).
2. Rewiring de paths (D3–D7) → commit.
3. Verificación por capas: `pytest` (raíz y `src/backend/`), `tsc --noEmit` + `vite build` (desde `src/frontend/`), `ruff` + `mypy` en verde.
4. Verificación integral: `docker compose down -v`, `up --build`, migrar, seed, `GET /api/health` → 200 (cierra 1.4/4.3).
5. Rollback: revert de commits en orden inverso (rewiring → frontend → backend); sin datos reales en juego no hay backfill (R13).

## Open Questions

- Ninguna que cambie specs, enfoque o tasks. Las PO del KB (hosting, retención legal, pricing) siguen abiertas para changes futuros.
