# core-python-base · estándar Python para servicios CIS

**Propósito**: template canónico de configuración Python para todo servicio nuevo del ecosistema CIS, y target de migración para servicios existentes (P0.4 del plan maestro 2026-04-29).

Define **un solo lugar** donde vive la verdad de:
- `[tool.ruff]` (lint + format)
- `[tool.black]` (formateo, fallback)
- `[tool.pytest]` (test runner)
- `[tool.coverage]` (cobertura mínima 60%)
- `[tool.mypy]` (types, modo gradual)
- `.pre-commit-config.yaml` (hooks: ruff, trailing-whitespace, yaml-validate, large-file)
- `requirements` discipline (uv lockfile obligatorio en producción)
- **`.github/workflows/ci-python.yml`** — reusable CI workflow callable desde cualquier repo (W7)

## Reusable CI workflow

`.github/workflows/ci-python.yml` es un workflow `on: workflow_call:` que cualquier repo del org puede invocar:

```yaml
# en el repo consumidor: .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: innovacionsantiago/core-python-base/.github/workflows/ci-python.yml@main
    with:
      coverage_min: 60
      python_version: "3.12"
```

Inputs disponibles:

| Input | Default | Descripción |
|---|---|---|
| `coverage_min` | `60` | pytest-cov fail-under threshold |
| `validators_layers` | `data,api,security,structure,contracts` | cis-validators layers (TODO V2) |
| `python_version` | `"3.12"` | setup-python version |
| `working_directory` | `"."` | Subdir del repo |
| `skip_coverage_gate` | `false` | warning-only si true |

Steps: ruff (lint+format) → mypy (gradual, warning-only) → pytest+coverage → bandit (report) → gitleaks (report) → upload artifacts.

Severity gates per ADR-034: CRITICAL bloquea merge; HIGH/MED/LOW report-only V1.

ADR de referencia: **ADR-018 · Python project standard** en `cds/cis/cis-plan/DECISIONS.md`.

---

## Cómo se adopta en un proyecto existente

Asumiendo que el proyecto ya tiene `pyproject.toml` y `requirements.txt`:

```bash
cd /srv/projects/<path-al-servicio>

# 1. copiar pre-commit config canónico
cp /srv/projects/core/core-python-base/.pre-commit-config.yaml ./

# 2. mergear secciones [tool.ruff], [tool.black], etc. en tu pyproject.toml
#    desde /srv/projects/core/core-python-base/pyproject.toml.template
#    (mantenés tu [project], [project.dependencies], etc. propios)

# 3. instalar pre-commit en el repo
pre-commit install

# 4. lockfile reproducible (uv obligatorio en prod)
uv pip compile requirements.txt -o requirements.lock

# 5. validar
pre-commit run --all-files
```

## Cómo se usa en un proyecto nuevo

`@cis/service-template` (cookiecutter, ADR-016) genera el scaffold ya con esto adoptado. Pero si lo creás manual:

```bash
mkdir my-service && cd my-service
cp /srv/projects/core/core-python-base/{.pre-commit-config.yaml,pyproject.toml.template} .
mv pyproject.toml.template pyproject.toml
# editá [project] name/description
git init && pre-commit install
```

## Convenciones que esto impone

- **Python 3.10+** (target). Servicios nuevos: 3.12 explícito.
- **Line length 88** (compatible black/ruff).
- **isort dentro de ruff** — no instalar isort por separado.
- **black como fallback de format** — ruff format es default; black queda para casos donde el equipo lo prefiera.
- **pytest + pytest-asyncio + pytest-cov** — coverage mínimo 60% (configurable por proyecto si arranca <60%, pero target es 60%).
- **mypy modo gradual** — `strict = false` para servicios existentes; `strict = true` esperado en nuevos.
- **structlog JSON** para logs (ADR-019).
- **slowapi prometheus** para métricas en endpoints HTTP.
- **uv lockfile** — `requirements.lock` commiteado; `requirements.txt` solo declara, no instala en prod.

## No-objetivos

- No es un paquete pypi-instalable. Es un repositorio de templates y archivos canónicos.
- No prescribe estructura de directorios del código (`src/`, `app/`, etc. quedan a criterio del proyecto).
- No incluye dependencias compartidas (ese es el rol de `core/py-common` cuando lo construyamos — ADR-016).

## Estado de adopción (P0.4)

Los 8 servicios live target del Bloque P0.4:

- [ ] cis-admin
- [ ] cis-core
- [ ] cis-platform
- [ ] cis-inbox
- [ ] cis-mailer
- [ ] cis-pagos
- [ ] cis-monitoreo
- [ ] cis-sii-gateway

Marcar al adoptar. Track final en `STATUS.md` cuando los 8 estén verde.

## Referencias

- Proyecto-modelo de origen: `cds/cis/periodismo2-www/` (único proyecto del workspace que tenía pre-commit a 2026-04-29).
- ADR-018 (Python project standard) — formaliza este template.
- ADR-019 (Error taxonomy + logging spec) — extiende con structlog convention.
- ADR-025 (Service topology standard) — qué proyecto Python debería ser monolítico vs split.
