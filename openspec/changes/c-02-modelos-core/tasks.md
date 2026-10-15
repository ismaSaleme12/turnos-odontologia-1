# Tasks

## 1. Aislamiento multi-tenant + constraints (base contra Postgres real)

- [ ] 1.1 Correr `test_core_models.py` + `test_repository_isolation.py` con `DATABASE_URL_TEST` contra Postgres real y verificar verde incluyendo: alta feliz, email duplicado mismo tenant rechazado, mismo email en otro tenant permitido, odontólogo sin matrícula rechazado, tenant A no ve filas de B, repositorio sin `tenant_id` levanta error en vez de devolver datos sin filtro
- [ ] 1.2 Verificar `BaseRepository`+`UnitOfWork` con ≥2 casos borde extra si faltan (agregar vía `agregar` con `tenant_id` ajeno rechazado; `obtener` por PK de otro tenant devuelve `None`) y verificar que la suite del grupo pasa en verde contra Postgres real

## 2. Cierre 2.4 CRITICO: rol app INSERT+SELECT sobre `auditoria` (análisis primero, D5)

- [ ] 2.1 Diagnosticar el estado actual sin modificar SQL: levantar Postgres en contenedor, correr migración 001 como owner, confirmar rol efectivo con `SELECT current_user` en la sesión `APP_ROLE_URL_TEST` y correr `test_auditoria_appendonly.py`, verificando que los casos UPDATE/DELETE corren (sin skip) y pasan, o registrando el defecto exacto (GRANT, rol `01-roles.sh`, env, o test) si fallan
- [ ] 2.2 Si 2.1 revela GRANTs insuficientes: presentar el diff SQL propuesto + impacto (rol, tabla, privilegio, motivo) y esperar revisión humana antes de escribir cualquier SQL de permisos (PROHIBIDO commitear GRANTs/REVOKEs sin aprobación explícita), luego aplicar lo aprobado con su test TDD primero y verificar verde contra Postgres real
- [ ] 2.3 Verificar reversibilidad de permisos: `alembic downgrade -1` revoca GRANTs y elimina tablas + `upgrade head` los restaura en contenedor fresco, con INSERT+SELECT ok y UPDATE/DELETE rechazados después del re-upgrade, todo verificado con comandos en verde y ≥2 casos de rechazo

## 3. Migración 001 + seed piloto idempotente

- [ ] 3.1 Verificar migración 001 reversible en contenedor Postgres fresco con `alembic upgrade head` + `downgrade -1` + `upgrade head` (cwd según layout vigente) sin errores y con tablas `tenant`, `usuario`, `auditoria` + índices (`ix_tenant_slug`, `ix_usuario_tenant_email`, `ix_auditoria_tenant_entidad`) + constraints (`uq_usuario_tenant_email`, `ck_usuario_matricula_odontologo`) presentes vía inspección de schema
- [ ] 3.2 Verificar seed idempotente con doble ejecución sin duplicados y `SELECT` mostrando exactamente 1 tenant piloto + 1 usuario Dueño activo + matriz de permisos base, con datos 100% sintéticos (R13) y verificado en `test_seed.py` en verde

## 4. Gates de calidad R1/R2/R9 (sin PHI, sin `Any`, forbid)

- [ ] 4.1 Pasar `ruff check` + `ruff format --check` + `mypy` sobre backend en limpio y verificar ambos en verde, con `extra='forbid'` en schemas Pydantic de input y sin `Any` en schemas públicos
- [ ] 4.2 Verificar R9 con `grep -ri` sobre backend/tests: ningún DNI, teléfono, dato clínico ni secreto en logs, mensajes de error visibles ni ejemplos, y `grep -r` sin secretos hardcodeados (solo `${VAR}`/placeholders, R10)

## 5. Paridad post-move + verificación integral (BLOQUEADA por c-01 hasta su apply)

- [ ] 5.1 [BLOQUEADA por `c-01-solo-setup`: requiere `src/backend/app/` existente; si c-01 no está aplicado, reportar bloqueada sin marcar] Verificar regression post-move: imports `from app...` intactos, `tests/conftest.py` con `sys.path` a `src/backend`, `pytest` en verde desde raíz Y desde `src/backend/`, y `DATABASE_URL_TEST` resolviendo a Postgres real (R11)
- [ ] 5.2 [BLOQUEADA por `c-01-solo-setup`] Verificación integral del change: suite completa `pytest` en verde (sin skips salvo los solo-Postgres documentados cuando corresponda), `alembic upgrade/downgrade/upgrade` en verde, `openspec validate --change "c-02-modelos-core"` sin errores, y `git status`/`git diff --name-only` mostrando que ningún archivo fuera de `tests/`, `db/init/`, `alembic/versions/` o permisos fue tocado salvo lo aprobado en 2.2
