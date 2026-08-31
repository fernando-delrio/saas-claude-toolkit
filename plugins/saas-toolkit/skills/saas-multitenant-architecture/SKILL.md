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
