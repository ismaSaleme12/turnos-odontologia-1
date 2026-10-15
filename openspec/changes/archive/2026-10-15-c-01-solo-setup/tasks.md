# Tasks

## 1. Moves backend con `git mv`

- [x] 1.1 Mover `src/app→src/backend/app`, `src/alembic→src/backend/alembic`, `src/alembic.ini` + `src/pyproject.toml` + `src/Dockerfile` + `src/.dockerignore→src/backend/` con `git mv` y verificar con `git status` que son renombres (R) y que `src/backend/{app,alembic,alembic.ini,pyproject.toml,Dockerfile}` existen
- [x] 1.2 Verificar historia preservada del backend con `git log --follow --oneline -- src/backend/app/main.py` (o equivalente) mostrando commits previos al move y confirmar que ningún modelo/migración cambió de contenido con `git diff --stat` vacío en `src/backend/app/domain/`

## 2. Moves frontend con `git mv`

- [x] 2.1 Mover `package.json`, `package-lock.json`, `vite.config.ts`, `tsconfig.json`, `tsconfig.node.json`, `tailwind.config.js`, `postcss.config.js`, `index.html`, `nginx.conf`, `public/`, `src/` anidado, `Dockerfile.frontend→src/frontend/` con `git mv` y verificar que `src/frontend/src/` existe y que la raíz `src/` ya no contiene archivos frontend sueltos
- [x] 2.2 Verificar que ningún contenido frontend cambió en el move con `git diff --stat -- src/frontend/` mostrando solo renombres y que `grep -rn "from app" src/frontend/ || true` no devuelve imports de backend filtrados

## 3. Rewiring backend (paths + Docker + pytest unificado)

- [x] 3.1 Actualizar `pytest.ini` (`pythonpath = src/backend`) + `tests/conftest.py` (`sys.path` a `src/backend`) + `src/backend/pyproject.toml` (`testpaths = ["../../tests"]`) y verificar que `python -m pytest tests/test_cobertura_verify.py -q` pasa desde la raíz Y que `python -m pytest ../../tests -q` pasa con cwd en `src/backend/`
- [x] 3.2 Rewiring `docker-compose.yml` (3 × `context: ./src` → `./src/backend`) + revisar `src/backend/.dockerignore` (que no excluya `alembic.ini`/`alembic/`) y verificar con `docker compose config` que los 3 contexts resuelven y con `grep -rn "\./src[^/]" docker-compose.yml || true` sin contexts viejos residuales
- [ ] 3.3 Verificar Alembic solo desde `src/backend/` con `alembic upgrade head` + `alembic downgrade -1` + `alembic upgrade head` en contenedor/CI fresco sin errores y confirmar que `alembic.ini` no necesitó cambios de contenido
- [x] 3.4 Pasar `ruff check/format` + `mypy src/backend/app src/backend/alembic` en limpio y verificar ambos comandos en verde sin nuevos `Any` en schemas públicos

## 4. Rewiring frontend + CI (6 paths)

- [x] 4.1 Confirmar cero cambios de contenido frontend (`vite.config.ts`, `tsconfig*.json`, `tailwind`, `index.html`) y verificar con `npm ci` + `npm run build` (incluye `tsc --noEmit`) en verde con cwd en `src/frontend/`
- [x] 4.2 Actualizar los 6 paths de `.github/workflows/ci.yml` (install `src/backend`, migrate wd `src/backend`, ruff config `src/backend/pyproject.toml`, mypy `src/backend/...`, npm cache `src/frontend/package-lock.json`, npm wd `src/frontend`) y verificar con `grep -n "working-directory\|cache-dependency-path\|src/backend\|src/frontend" .github/workflows/ci.yml` que no queda ningún path `src/` plano y con `act -j backend --dry-run` o `yamllint` que el workflow sigue válido
- [x] 4.3 PROHIBIDO tocar modelos/datos: verificar con `git diff --name-only` que ningún archivo bajo `src/backend/app/domain/`, `src/backend/alembic/versions/` ni seeds cambió, y que no hay secretos en el diff (`grep -ri "password\|secret\|token" --include="*.yml" --include="*.toml"` solo muestra `${VAR}`/placeholders)

## 5. Verificación integral (cierra 1.4/4.2/4.3)

- [ ] 5.1 Levantar stack limpio con `docker compose down -v`, `docker compose up --build`, migrar + seed, y verificar `GET /api/health` → 200 (cierra task 1.4 del archive)
- [x] 5.2 Correr suite completa `pytest` en verde + `tsc --noEmit`/`vite build` en verde + `openspec validate --change "c-01-solo-setup"` sin errores, y verificar `git status` limpio salvo los paths previstos (cierra tasks 4.2/4.3 del archive; 2.4 GRANT queda explícitamente pendiente para change CRITICO separado)
