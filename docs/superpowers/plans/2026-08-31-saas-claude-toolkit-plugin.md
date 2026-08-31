# saas-claude-toolkit Plugin — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Package Fernando's 4 existing review agents plus one new `saas-multitenant-architecture` skill into an installable Claude Code plugin, published as a public repo with MIT license.

**Architecture:** A single-plugin marketplace (`.claude-plugin/marketplace.json` → `plugins/saas-toolkit/`). The 4 agents are copied byte-for-byte from `~/.claude/agents/`. The new skill is 4 markdown files (`SKILL.md` + 3 reference docs) derived from a real multi-tenant security audit, with content fully anonymized (no "Weldix", "taller", "soldadura").

**Tech Stack:** Plain Markdown + JSON. No build step, no test runner — this is a content/config repo. "Tests" are JSON validity checks, an anonymization grep, and a manual `claude plugin marketplace add` / `install` smoke test.

## Global Constraints

- Zero content changes to the 4 agent files copied from `~/.claude/agents/*.md` — copy verbatim, only the location changes.
- Every skill/reference file must be free of "weldix", "taller", "soldadura" (case-insensitive) — verified by grep before the repo is considered done.
- Repo is public, MIT licensed.
- Single plugin bundle (not split into multiple plugins) — this is already decided, do not revisit.
- Skill uses separate reference files (`auth-multitenant.md`, `testing-multitenant.md`, `billing-stripe.md`), not one giant `SKILL.md`, and no executable code templates/scaffolding.
- Target directory: `c:\Users\FeR\OneDrive\Escritorio\full stack\saas-claude-toolkit` (already a git repo, currently only contains `docs/`).

---

## File Structure

```
saas-claude-toolkit/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   └── saas-toolkit/
│       ├── plugin.json
│       ├── agents/
│       │   ├── backend-reviewer.md
│       │   ├── craft-reviewer.md
│       │   ├── conventions-reviewer.md
│       │   └── test-writer.md
│       └── skills/
│           └── saas-multitenant-architecture/
│               ├── SKILL.md
│               ├── auth-multitenant.md
│               ├── testing-multitenant.md
│               └── billing-stripe.md
├── README.md
└── LICENSE
```

---

### Task 1: Repo scaffold — marketplace manifest, plugin manifest, LICENSE

**Files:**
- Create: `.claude-plugin/marketplace.json`
- Create: `plugins/saas-toolkit/plugin.json`
- Create: `LICENSE`

**Interfaces:**
- Produces: the plugin id `saas-toolkit` and marketplace id `saas-claude-toolkit`, referenced by the README install instructions in Task 7.

- [ ] **Step 1: Create `.claude-plugin/marketplace.json`**

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "saas-claude-toolkit",
  "description": "Agentes de revisión de código y una skill de arquitectura multi-tenant para SaaS, empaquetados como plugin de Claude Code.",
  "owner": {
    "name": "Fernando"
  },
  "plugins": [
    {
      "name": "saas-toolkit",
      "source": "./plugins/saas-toolkit",
      "description": "4 agentes de revisión (backend, craft, conventions, tests) + skill de arquitectura multi-tenant derivada de una auditoría de seguridad real.",
      "category": "development"
    }
  ]
}
```

- [ ] **Step 2: Create `plugins/saas-toolkit/plugin.json`**

```json
{
  "name": "saas-toolkit",
  "version": "1.0.0",
  "description": "4 agentes de revisión de código (backend, craft, conventions, tests) + skill de arquitectura multi-tenant para SaaS derivada de una auditoría de seguridad real.",
  "author": {
    "name": "Fernando"
  }
}
```

- [ ] **Step 3: Create `LICENSE`** (MIT, copyright Fernando, 2026)

```
MIT License

Copyright (c) 2026 Fernando

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 4: Validate both JSON files parse**

Run (from repo root):
```bash
python -m json.tool .claude-plugin/marketplace.json > /dev/null && echo OK_MARKETPLACE
python -m json.tool plugins/saas-toolkit/plugin.json > /dev/null && echo OK_PLUGIN
```
Expected: `OK_MARKETPLACE` and `OK_PLUGIN` printed, no traceback.

- [ ] **Step 5: Commit**

```bash
git add .claude-plugin/marketplace.json plugins/saas-toolkit/plugin.json LICENSE
git commit -m "chore: scaffold marketplace and plugin manifests"
```

---

### Task 2: Migrate the 4 existing agents

**Files:**
- Create: `plugins/saas-toolkit/agents/backend-reviewer.md` (copy of `~/.claude/agents/backend-reviewer.md`)
- Create: `plugins/saas-toolkit/agents/craft-reviewer.md` (copy of `~/.claude/agents/craft-reviewer.md`)
- Create: `plugins/saas-toolkit/agents/conventions-reviewer.md` (copy of `~/.claude/agents/conventions-reviewer.md`)
- Create: `plugins/saas-toolkit/agents/test-writer.md` (copy of `~/.claude/agents/test-writer.md`)

**Interfaces:**
- Consumes: `~/.claude/agents/*.md` (source files), read-only, do not modify the originals.
- Produces: 4 agent files whose frontmatter `name:` fields (`backend-reviewer`, `craft-reviewer`, `conventions-reviewer`, `test-writer`) are what the README in Task 7 references.

- [ ] **Step 1: Copy the 4 files verbatim**

```bash
mkdir -p "plugins/saas-toolkit/agents"
cp "$HOME/.claude/agents/backend-reviewer.md" "plugins/saas-toolkit/agents/backend-reviewer.md"
cp "$HOME/.claude/agents/craft-reviewer.md" "plugins/saas-toolkit/agents/craft-reviewer.md"
cp "$HOME/.claude/agents/conventions-reviewer.md" "plugins/saas-toolkit/agents/conventions-reviewer.md"
cp "$HOME/.claude/agents/test-writer.md" "plugins/saas-toolkit/agents/test-writer.md"
```

On this machine, `$HOME` resolves to `C:/Users/FeR` under Git Bash — if it doesn't, use the absolute path `/c/Users/FeR/.claude/agents/`.

- [ ] **Step 2: Verify byte-for-byte match against the source**

```bash
diff "$HOME/.claude/agents/backend-reviewer.md" "plugins/saas-toolkit/agents/backend-reviewer.md" && echo MATCH_backend
diff "$HOME/.claude/agents/craft-reviewer.md" "plugins/saas-toolkit/agents/craft-reviewer.md" && echo MATCH_craft
diff "$HOME/.claude/agents/conventions-reviewer.md" "plugins/saas-toolkit/agents/conventions-reviewer.md" && echo MATCH_conventions
diff "$HOME/.claude/agents/test-writer.md" "plugins/saas-toolkit/agents/test-writer.md" && echo MATCH_test_writer
```
Expected: no diff output, and all four `MATCH_*` lines printed.

- [ ] **Step 3: Commit**

```bash
git add plugins/saas-toolkit/agents/
git commit -m "feat: migrate 4 existing review agents into the plugin bundle"
```

---

### Task 3: Write `saas-multitenant-architecture/SKILL.md`

**Files:**
- Create: `plugins/saas-toolkit/skills/saas-multitenant-architecture/SKILL.md`

**Interfaces:**
- Produces: links to `auth-multitenant.md`, `testing-multitenant.md`, `billing-stripe.md` (Tasks 4–6) — the filenames referenced here must match exactly what those tasks create.

- [ ] **Step 1: Write the file**

```markdown
---
name: saas-multitenant-architecture
description: Arquitectura de aislamiento multi-tenant para un SaaS B2B desde cero — elige entre tenant_id compartido, schema por tenant o BD por tenant, aplica el checklist IDOR a cada query, y enlaza con auth, testing y billing multi-tenant. Úsalo al arrancar un SaaS nuevo o al auditar el aislamiento de uno existente.
---

# Arquitectura multi-tenant — decisión de aislamiento

Esta skill nace de una auditoría de seguridad real sobre un SaaS multi-tenant en producción: 5 fugas de aislamiento (IDOR) encontradas y corregidas en un solo día. No viene de un libro — viene de bugs reales. Por eso el checklist de la sección 3 pesa tanto como la tabla de estrategias.

## 1. Las 3 estrategias de aislamiento

| Estrategia | Coste | Complejidad operativa | Garantía de aislamiento | Migrar a la siguiente |
|---|---|---|---|---|
| **`tenant_id` compartido** — una tabla, una columna `tenant_id` en cada fila | Bajo — un solo servidor, un solo schema | Baja — un pool de conexiones, una migración para todos | Depende 100% de que CADA query filtre por `tenant_id`. Un solo endpoint que lo olvide es una fuga (IDOR) | Media-alta — hay que particionar datos existentes por tenant |
| **Schema por tenant** — mismo servidor Postgres, un schema por cliente | Medio — mismo servidor, N schemas, N migraciones a aplicar (o un runner que itera) | Media — el aislamiento lo da Postgres, no tu código, pero conectar al schema correcto en cada request es una responsabilidad más | Aislamiento fuerte a nivel de motor; un bug de aplicación no cruza schemas | Baja — mover un tenant a su propia BD es un `pg_dump`/`pg_restore` de un solo schema |
| **BD por tenant** — aislamiento físico total | Alto — N bases de datos, N conexiones, N migraciones, backups por separado | Alta — cada tenant es una unidad operativa propia (monitorización, escalado, incidentes) | Total — un bug de aplicación no puede filtrar entre tenants ni por accidente | N/A — ya es el aislamiento máximo |

**Recomendación por defecto para un SaaS B2B pequeño/mediano: `tenant_id` compartido.** Es el que menos fricción operativa añade mientras el número de tenants y el tamaño de los datos son manejables, y el que más rápido se construye. El coste que paga a cambio es que el aislamiento vive enteramente en la disciplina de código — de ahí el checklist de la sección 3.

## 2. Cuándo NO usar el default

No uses `tenant_id` compartido cuando:
- **Compliance exige aislamiento físico** — sanidad (HIPAA), banca, o cualquier contrato que exija que los datos de un cliente nunca compartan disco/proceso con los de otro. Ahí vas directo a BD por tenant, sin importar el tamaño.
- **Un tenant es mucho más grande que el resto** y necesita rendimiento dedicado (su propio índice, su propio tuning, su propia ventana de mantenimiento) sin que un vecino ruidoso lo degrade. Esquema o BD por tenant para ese cliente concreto; el resto puede seguir en `tenant_id` compartido (modelo híbrido).

## 3. El checklist IDOR (la lección real)

**Regla:** toda query que recibe un `id` externo controlado por el cliente (path param, query param o campo del body) necesita una de estas dos cosas antes de tocar la BD:
1. `tenant_id` en el filtro de la query, o
2. Una validación explícita de que el recurso referenciado pertenece al tenant del usuario autenticado, antes de usarlo.

### Ejemplo anonimizado del bug real

Un endpoint recibe en el body el id de OTRO recurso relacionado (no el recurso principal de la URL) y lo usa sin validar que pertenece al mismo tenant que el usuario autenticado:

```python
# ❌ INCORRECTO — el id del body no se valida contra el tenant
@router.patch("/shift-changes/{change_id}/review")
def review_shift_change(change_id: int, body: ReviewInput, user: User = Depends(get_current_user)):
    change = db.query(ShiftChange).filter(
        ShiftChange.id == change_id,
        ShiftChange.tenant_id == user.tenant_id,   # el recurso PRINCIPAL sí está filtrado
    ).first()
    if not change:
        raise HTTPException(404)

    # body.reviewer_id llega del cliente y NO se valida contra el tenant.
    # Un usuario del Tenant A puede pasar el id de un empleado del Tenant B
    # y el sistema lo acepta como revisor válido.
    change.reviewer_id = body.reviewer_id
    db.commit()
```

```python
# ✅ CORRECTO — todo id que entra por el body se valida contra el tenant también
@router.patch("/shift-changes/{change_id}/review")
def review_shift_change(change_id: int, body: ReviewInput, user: User = Depends(get_current_user)):
    change = db.query(ShiftChange).filter(
        ShiftChange.id == change_id,
        ShiftChange.tenant_id == user.tenant_id,
    ).first()
    if not change:
        raise HTTPException(404)

    reviewer = db.query(Employee).filter(
        Employee.id == body.reviewer_id,
        Employee.tenant_id == user.tenant_id,      # el id del BODY también se valida
    ).first()
    if not reviewer:
        raise HTTPException(400, detail="Revisor no encontrado en este tenant")

    change.reviewer_id = reviewer.id
    db.commit()
```

**La fuga no está en el recurso principal de la URL** (ese casi siempre se filtra bien porque es el "camino feliz" que se prueba primero). **Está en los ids secundarios que llegan por el body o por query params** — se asume que si el usuario está autenticado, cualquier id que envíe es válido. No lo es: hay que probar que ese id también pertenece a su tenant.

### Checklist para cada endpoint nuevo
- [ ] ¿El recurso principal de la URL filtra por `tenant_id`?
- [ ] ¿Todo id que llega por el body se valida contra el tenant del usuario autenticado, no solo contra su propia tabla?
- [ ] ¿Todo id que llega por query params (filtros, ordenación por FK) se valida igual?
- [ ] ¿El test de regresión usa DOS tenants reales e intenta cruzar datos entre ellos? (ver `testing-multitenant.md`)

## 4. Siguientes pasos

- **`auth-multitenant.md`** — JWT claim vs resolución en BD, modelo de roles, patrón de superadmin cross-tenant.
- **`testing-multitenant.md`** — fixture de dos tenants reales, disciplina de verificación con `git stash`.
- **`billing-stripe.md`** — verificación de firma de webhook, relación tenant ↔ `customer_id`.
```

- [ ] **Step 2: Verify no anonymization leak**

```bash
grep -iE "weldix|taller|soldadura" "plugins/saas-toolkit/skills/saas-multitenant-architecture/SKILL.md"
```
Expected: no output (grep exits non-zero / prints nothing). If it prints a match, rewrite that line before continuing.

- [ ] **Step 3: Commit**

```bash
git add plugins/saas-toolkit/skills/saas-multitenant-architecture/SKILL.md
git commit -m "feat: add saas-multitenant-architecture SKILL.md"
```

---

### Task 4: Write `auth-multitenant.md`

**Files:**
- Create: `plugins/saas-toolkit/skills/saas-multitenant-architecture/auth-multitenant.md`

- [ ] **Step 1: Write the file**

```markdown
# Auth multi-tenant

## `tenant_id` en el JWT vs resolverlo de la BD

| | Claim en el JWT | Resolver de la BD en cada request |
|---|---|---|
| Velocidad | Sin query extra — el `tenant_id` ya está en el token decodificado | Una query más por request (normalmente cacheable) |
| Revocación | El tenant queda fijo hasta que el token expira — si cambias a un usuario de tenant o lo desactivas, sigue funcionando con el token viejo | Revocable al instante — el siguiente request ya ve el cambio |
| Cuándo usarlo | Tokens de vida corta (minutos) donde el desfase es aceptable | Cuando cambios de tenant/rol deben aplicar YA (ej. quitar acceso a un empleado despedido) |

**Trade-off explícito:** si usas el claim del JWT, decide con qué frecuencia expira el access token — cuanto más corto, menos tiempo de desfase, pero más refresh. Si el negocio necesita revocación inmediata (offboarding de empleados, por ejemplo), resuelve de BD aunque cueste una query.

## Modelo de roles

Dos niveles mínimos:
- **Rol dentro del tenant** (admin del tenant / usuario estándar) — vive en la tabla de membresía usuario↔tenant, no en el usuario global. Un mismo usuario puede ser admin en un tenant y estándar en otro.
- **Chequeo de rol**: en el mismo sitio que el chequeo de tenant, nunca separado. Si filtras por `tenant_id` en el service y compruebas el rol en el router (o viceversa), es fácil que un endpoint nuevo se olvide de una de las dos comprobaciones.

```python
def require_tenant_role(role: str):
    def _check(user: User = Depends(get_current_user)):
        if user.tenant_role != role:
            raise HTTPException(403)
        return user
    return _check
```

## Superadmin cross-tenant

Un superadmin que puede ver/tocar datos de cualquier tenant es un patrón legítimo (soporte, operaciones), pero:
- **Nunca es el camino por defecto.** El query de superadmin es una ruta de código explícitamente distinta (`if user.is_superadmin: query_without_tenant_filter()`), no un `tenant_id` opcional en el filtro normal.
- **Siempre auditado.** Cada acceso cross-tenant de un superadmin escribe un registro (quién, cuándo, qué tenant, qué acción) — es el tipo de acceso que se revisa en una auditoría de seguridad.
- **Endpoint separado o flag explícito**, nunca implícito. Si `GET /orders?tenant_id=X` acepta un `tenant_id` de query param "porque el superadmin lo necesita", cualquier usuario normal puede probar a mandarlo también. Sepáralo: `GET /admin/tenants/{id}/orders` con su propio `require_role("superadmin")`.
```

- [ ] **Step 2: Verify no anonymization leak**

```bash
grep -iE "weldix|taller|soldadura" "plugins/saas-toolkit/skills/saas-multitenant-architecture/auth-multitenant.md"
```
Expected: no output.

- [ ] **Step 3: Commit**

```bash
git add plugins/saas-toolkit/skills/saas-multitenant-architecture/auth-multitenant.md
git commit -m "feat: add auth-multitenant.md reference doc"
```

---

### Task 5: Write `testing-multitenant.md`

**Files:**
- Create: `plugins/saas-toolkit/skills/saas-multitenant-architecture/testing-multitenant.md`

- [ ] **Step 1: Write the file**

```markdown
# Testing multi-tenant

## El patrón de fixture "dos tenants reales"

No basta con testear un tenant y confiar en que el filtro funciona. El test de aislamiento crea DOS tenants reales con datos reales y prueba explícitamente que uno no puede leer ni escribir los datos del otro.

```python
@pytest.fixture
def two_tenants(db):
    tenant_a = TenantFactory.create(name="Tenant A")
    tenant_b = TenantFactory.create(name="Tenant B")
    user_a = UserFactory.create(tenant_id=tenant_a.id)
    user_b = UserFactory.create(tenant_id=tenant_b.id)
    resource_a = ResourceFactory.create(tenant_id=tenant_a.id)
    return {"tenant_a": tenant_a, "tenant_b": tenant_b, "user_a": user_a, "user_b": user_b, "resource_a": resource_a}

def test_tenant_b_cannot_read_tenant_a_resource(async_client, two_tenants):
    # ARRANGE
    token_b = create_access_token(two_tenants["user_b"].id)
    # ACT
    response = await async_client.get(
        f"/resources/{two_tenants['resource_a'].id}",
        headers={"Authorization": f"Bearer {token_b}"},
    )
    # ASSERT — 404, no 403: no revela que el recurso existe en otro tenant
    assert response.status_code == 404

def test_tenant_b_cannot_reference_tenant_a_resource_by_id(async_client, two_tenants):
    # ARRANGE — el bug real: un id de OTRO tenant colado en un campo del body
    token_b = create_access_token(two_tenants["user_b"].id)
    own_resource = ResourceFactory.create(tenant_id=two_tenants["tenant_b"].id)
    # ACT
    response = await async_client.patch(
        f"/resources/{own_resource.id}/link",
        json={"related_id": two_tenants["resource_a"].id},
        headers={"Authorization": f"Bearer {token_b}"},
    )
    # ASSERT
    assert response.status_code == 400
```

El segundo test es el que importa: reproduce exactamente el patrón del checklist IDOR (ver `SKILL.md`) — un id de otro tenant colado por el body, no por la URL.

## Disciplina de verificación: el test tiene que poder fallar

Un test de regresión que nunca se confirmó que falla contra el código viejo no prueba nada — podría estar mal escrito y pasar siempre. Antes de dar un test de seguridad por bueno:

```bash
git stash                          # vuelve al código con el bug
pytest tests/test_shift_review.py::test_tenant_b_cannot_reference_tenant_a_resource_by_id -v
# Debe FALLAR aquí. Si pasa con el bug presente, el test no vale nada.
git stash pop                      # recupera el fix
pytest tests/test_shift_review.py::test_tenant_b_cannot_reference_tenant_a_resource_by_id -v
# Debe PASAR aquí.
```

Si el test pasa en ambos casos, el test está mal escrito (assert trivial, fixture que no reproduce el bug, o el bug no era donde pensabas). No lo des por bueno hasta que falle contra el código viejo y pase contra el fix.
```

- [ ] **Step 2: Verify no anonymization leak**

```bash
grep -iE "weldix|taller|soldadura" "plugins/saas-toolkit/skills/saas-multitenant-architecture/testing-multitenant.md"
```
Expected: no output.

- [ ] **Step 3: Commit**

```bash
git add plugins/saas-toolkit/skills/saas-multitenant-architecture/testing-multitenant.md
git commit -m "feat: add testing-multitenant.md reference doc"
```

---

### Task 6: Write `billing-stripe.md`

**Files:**
- Create: `plugins/saas-toolkit/skills/saas-multitenant-architecture/billing-stripe.md`

- [ ] **Step 1: Write the file**

```markdown
# Billing con Stripe en un SaaS multi-tenant

## Verificación de firma de webhook — sin excepciones

El body crudo (raw bytes, sin parsear a JSON) es lo que Stripe firma. Si tu framework parsea el body a JSON antes de que lo veas, la verificación de firma falla siempre porque el string re-serializado no es byte-a-byte idéntico al original.

```python
@router.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    payload = await request.body()                    # BODY CRUDO, antes de cualquier parseo
    sig_header = request.headers.get("stripe-signature")

    if not sig_header:
        raise HTTPException(400, detail="Missing signature")

    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, settings.STRIPE_WEBHOOK_SECRET   # secreto de .env, nunca hardcodeado
        )
    except (ValueError, stripe.error.SignatureVerificationError):
        raise HTTPException(400, detail="Invalid signature")

    # A partir de aquí, y SOLO a partir de aquí, el body es de confianza.
    handle_event(event)
    return {"received": True}
```

**Regla dura:** no existe un camino en el código que llegue a `handle_event()` sin haber pasado por `construct_event()` primero. Ni un modo debug, ni un "si falla la verificación pero el evento parece válido, seguimos". Firma inválida o ausente → 400, fin.

## Relación tenant ↔ `customer_id` de Stripe

Guarda el `customer_id` de Stripe en la tabla del tenant (no al revés) — un tenant tiene como mucho un `customer_id`, pero Stripe no sabe nada de tu concepto de tenant.

```python
class Tenant(Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    stripe_customer_id: Mapped[str | None] = mapped_column(unique=True, nullable=True)
```

Cuando llega un webhook, el evento trae `customer` (el id de Stripe) — resuelve el tenant a partir de ahí, nunca al revés (nunca confíes en un `tenant_id` que venga en el payload del webhook, aunque lo hayas puesto tú mismo en metadata; resuélvelo por `customer_id`, que es el dato que Stripe garantiza).

## Qué NO hacer

- **No confiar en el contenido del body antes de verificar la firma** — ni para logging "solo para depurar", ni para un shortcut de desarrollo. Un log que imprime `payload` antes de `construct_event()` ya es una superficie de ataque si ese log es accesible.
- **No usar el `tenant_id` del metadata del payload sin resolverlo también por `customer_id`** — si algún día un metadata mal formado o un evento de test cruza tenants, quieres que la resolución por `customer_id` sea la fuente de verdad, no un campo que cualquiera con acceso al dashboard de Stripe puede editar.
- **No manejar el evento dos veces** — Stripe reintenta webhooks que no devuelven 200 a tiempo. Guarda el `event.id` procesado (idempotencia) antes de aplicar efectos con dinero real de por medio.
```

- [ ] **Step 2: Verify no anonymization leak**

```bash
grep -iE "weldix|taller|soldadura" "plugins/saas-toolkit/skills/saas-multitenant-architecture/billing-stripe.md"
```
Expected: no output.

- [ ] **Step 3: Commit**

```bash
git add plugins/saas-toolkit/skills/saas-multitenant-architecture/billing-stripe.md
git commit -m "feat: add billing-stripe.md reference doc"
```

---

### Task 7: Write `README.md`

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: the plugin id `saas-toolkit` and marketplace id `saas-claude-toolkit` from Task 1; the agent `name:` values from Task 2; the skill name `saas-multitenant-architecture` from Task 3.

- [ ] **Step 1: Write the file**

```markdown
# saas-claude-toolkit

Plugin de Claude Code que empaqueta 4 agentes de revisión de código y 1 skill de arquitectura multi-tenant para arrancar un SaaS B2B desde cero.

## Instalación

```bash
claude plugin marketplace add <ruta-o-url-de-este-repo>
claude plugin install saas-toolkit@saas-claude-toolkit
```

## Qué incluye

**4 agentes** (read-only, invocables con el tool Agent):
- `backend-reviewer` — seguridad (OWASP) y escalabilidad de backend, cualquier lenguaje.
- `craft-reviewer` — calidad y legibilidad de código, cualquier lenguaje.
- `conventions-reviewer` — convenciones de arquitectura frontend (React/Angular/Vue/Svelte).
- `test-writer` — escribe y ejecuta tests (Vitest+MSW / pytest+httpx).

**1 skill nueva**: `saas-multitenant-architecture` — cómo elegir estrategia de aislamiento de tenants, el checklist IDOR, auth, testing y billing multi-tenant. Nace de una auditoría de seguridad real, no de un libro.

## Atribución de fuentes

| Fuente | Autor | Agente/skill |
|---|---|---|
| *A Philosophy of Software Design* | John Ousterhout | `craft-reviewer` |
| Prólogo de *Código Limpio* | James O. Coplien | `craft-reviewer` |
| *Code Complete* | Steve McConnell | `craft-reviewer` |
| *Dive Into Design Patterns* / *Dive Into Refactoring* | Alexander Shvets | `craft-reviewer` |
| SOLID / *Clean Code* | Robert C. Martin | `craft-reviewer` |
| DRY · KISS · YAGNI · SOC · LOD | (principios generales) | `craft-reviewer` |
| *Test Driven Development* | Kent Beck | `test-writer` |
| OWASP Top 10 | OWASP Foundation | `backend-reviewer` |

`saas-multitenant-architecture` es experiencia práctica propia (auditoría de seguridad real sobre un SaaS multi-tenant en producción), no deriva de un libro.

## Licencia

MIT — ver [LICENSE](LICENSE).
```

- [ ] **Step 2: Verify no anonymization leak**

```bash
grep -iE "weldix|taller|soldadura" README.md
```
Expected: no output.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add README with install instructions and source attribution"
```

---

### Task 8: Full-repo verification and install smoke test

**Files:** none created — this task only verifies Tasks 1–7.

- [ ] **Step 1: Repo-wide anonymization grep**

```bash
grep -rniE "weldix|taller|soldadura" --include="*.md" --include="*.json" .
```
Expected: no output. If anything matches, go back to the file it's in and rewrite that line before proceeding — do not skip this.

- [ ] **Step 2: Re-validate both JSON manifests** (guards against hand-edits after Task 1)

```bash
python -m json.tool .claude-plugin/marketplace.json > /dev/null && echo OK_MARKETPLACE
python -m json.tool plugins/saas-toolkit/plugin.json > /dev/null && echo OK_PLUGIN
```
Expected: both `OK_*` lines printed.

- [ ] **Step 3: Confirm every skill frontmatter has `name:` and `description:`**

```bash
head -n 4 plugins/saas-toolkit/skills/saas-multitenant-architecture/SKILL.md
```
Expected: a `---` fenced block containing `name: saas-multitenant-architecture` and a `description:` line.

- [ ] **Step 4: Local marketplace add + install smoke test**

From a Claude Code session (not this one — a fresh test session, since a plugin install of the currently-running session's own toolkit is unsupported):

```bash
claude plugin marketplace add "c:/Users/FeR/OneDrive/Escritorio/full stack/saas-claude-toolkit"
claude plugin install saas-toolkit@saas-claude-toolkit
```
Expected: install succeeds; `/help` or the agent/skill listing in that session shows `backend-reviewer`, `craft-reviewer`, `conventions-reviewer`, `test-writer`, and `saas-multitenant-architecture`.

- [ ] **Step 5: Commit the plan's completion marker** (only if any fixes were made in Steps 1–4; skip if nothing changed)

```bash
git add -A
git commit -m "chore: fix anonymization leaks found in final verification pass"
```

---

## Self-Review

**Spec coverage:**
- 4 agents packaged verbatim → Task 2. ✓
- `marketplace.json` + `plugin.json` → Task 1. ✓
- `SKILL.md` with 3 isolation strategies table, when-not-to-use-default, IDOR checklist with anonymized example, pointers to reference files → Task 3. ✓
- `auth-multitenant.md` (JWT claim vs DB, roles, superadmin) → Task 4. ✓
- `testing-multitenant.md` (two-tenant fixture, git stash discipline) → Task 5. ✓
- `billing-stripe.md` (webhook signature, tenant↔customer_id, what not to do) → Task 6. ✓
- README with install instructions + attribution table → Task 7. ✓
- LICENSE (MIT) → Task 1. ✓
- Verification: plugin installs and lists agents/skill; no "Weldix"/"taller"/"soldadura" mentions → Task 8. ✓

**Placeholder scan:** no TBD/TODO, no "similar to Task N", every step has literal file content or a runnable command. Clean.

**Type/name consistency:** plugin id `saas-toolkit`, marketplace id `saas-claude-toolkit`, skill id `saas-multitenant-architecture`, and the four agent `name:` frontmatter values are used identically across Tasks 1, 2, 3, and 7.
