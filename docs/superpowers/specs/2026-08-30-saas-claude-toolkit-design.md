# saas-claude-toolkit — Diseño

## Qué es

Un plugin instalable de Claude Code (`claude plugin install saas-toolkit@...`) que empaqueta:

1. **4 agentes ya existentes** de Fernando (hoy sueltos en `~/.claude/agents/`, sin versionar), copiados tal cual — son genéricos, agnósticos de lenguaje/framework, y ya cubren gran parte de lo que se necesitaba antes de plantear nada nuevo.
2. **1 skill nueva**: arquitectura para arrancar un SaaS multi-tenant desde cero. Nace directamente de la auditoría de seguridad real hecha hoy sobre Weldix (5 fugas de aislamiento multi-tenant encontradas y corregidas), no de un libro.

Público, con atribución explícita de las fuentes (libros) que hay detrás de cada agente.

## Por qué

Fernando ya tenía 4 agentes reutilizables (`backend-reviewer`, `craft-reviewer`, `conventions-reviewer`, `test-writer`) pero:
- Viven sueltos en `~/.claude/agents/`, sin control de versiones, sin backup, sin forma fácil de reinstalar en una máquina nueva o compartirlos.
- No cubrían el caso de **diseñar una arquitectura multi-tenant desde cero** — solo revisan código ya escrito.

## Alcance de esta primera versión

**Dentro:**
- Empaquetar los 4 agentes existentes sin tocar su contenido.
- Escribir la skill nueva de arquitectura multi-tenant con 4 archivos (SKILL.md + 3 de referencia).
- `marketplace.json` + `plugin.json` correctos para que `claude plugin install` funcione.
- README con instrucciones de instalación y tabla de atribución de fuentes.
- Repo público en GitHub, licencia MIT.

**Fuera (YAGNI por ahora):**
- Plantillas de código ejecutables (scaffolding de proyecto) — se decidió no incluirlas porque atarían el repo a un stack concreto (ej. FastAPI) cuando el resto es agnóstico de lenguaje.
- Splitear en varios plugins independientes — se decidió un solo plugin bundle por simplicidad de mantenimiento e instalación.
- CI/CD para el propio repo (lint de los `.md`, etc.) — se puede añadir después si hace falta.

## Estructura de archivos

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

## Los 4 agentes (migración)

Copia literal de `~/.claude/agents/*.md` — ya son genéricos (Python/Java/Node/Go/.NET para backend-reviewer; React/Angular/Vue/Svelte para conventions-reviewer; cualquier lenguaje para craft-reviewer; Vitest+MSW/pytest+httpx para test-writer). Cero cambios de contenido, solo de ubicación.

## Skill nueva: `saas-multitenant-architecture`

### `SKILL.md` — la decisión que más pesa

Contenido:
1. **Las 3 estrategias de aislamiento**, con tabla de trade-offs (coste, complejidad operativa, garantía de aislamiento, dificultad de migrar de una a otra):
   - Tenant_id compartido (una tabla, una columna `tenant_id` en cada fila) — recomendación por defecto para SaaS B2B pequeño/mediano.
   - Esquema por tenant (mismo servidor Postgres, un schema por cliente).
   - Base de datos por tenant (aislamiento físico total).
2. **Cuándo NO usar el default**: compliance que exige aislamiento físico (sanidad, banca), tenants muy grandes con necesidades de rendimiento dedicado.
3. **El checklist IDOR** (la lección real de hoy): toda query que recibe un `id` externo controlado por el cliente necesita `tenant_id` en el filtro, o una validación explícita del recurso referenciado (ej. `operario_id` de un body) antes de usarlo. Ejemplo anonimizado basado en el bug real de `revisar_cambio_turno` de hoy (sin nombrar Weldix ni el dominio de negocio).
4. Puntero a los 3 archivos de referencia de abajo.

### `auth-multitenant.md`
- `tenant_id` en el claim del JWT (stateless, rápido) vs resolverlo de la BD en cada request (revocable al instante, una query más). Trade-off explícito.
- Modelo de roles (admin/usuario estándar, o equivalente) y dónde vive el chequeo de rol.
- Patrón de superadmin cross-tenant: acceso especial, siempre auditado, nunca el camino por defecto.

### `testing-multitenant.md`
- El patrón de fixture "dos tenants reales" (Taller A / Taller B o equivalente genérico) usado hoy en los tests de regresión.
- Disciplina de verificación: revertir el fix con `git stash`, confirmar que el test de regresión FALLA contra el código viejo, antes de dar el test por bueno.

### `billing-stripe.md`
- Verificación de firma de webhook: leer el body crudo antes de parsear, verificar con el secreto de `.env`, cortar con 400 si la firma falla o falta — sin ningún camino que continúe sin verificar.
- Relación tenant ↔ `customer_id` de Stripe.
- Qué NO hacer: confiar en el contenido del body antes de verificar la firma.

## README — atribución de fuentes

Tabla explícita de libros/fuentes por agente, verificada leyendo el contenido real de cada uno (no de memoria):

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

La skill `saas-multitenant-architecture` se documenta explícitamente como derivada de experiencia práctica real (auditoría de Weldix), no de un libro — sin inventarle una fuente que no tiene.

## Testing / verificación

No hay "tests" en el sentido de código ejecutable (son archivos Markdown). La verificación es:
1. `claude plugin marketplace add <ruta-local-o-repo>` + `claude plugin install saas-toolkit@...` en una sesión de prueba, confirmar que los 4 agentes y la skill aparecen listados y son invocables.
2. Revisión manual de que ningún archivo menciona "Weldix" ni detalles del negocio del taller (grep de "weldix", "taller", "soldadura" antes de dar el repo por terminado).

## Fuera de duda / decisiones ya tomadas
- Plugin único con todo dentro (no varios plugins separados).
- Repo público, licencia MIT.
- Skill con archivos de referencia separados (no todo en un único SKILL.md, no plantillas de código).
- Ubicación local: `full stack/saas-claude-toolkit`, hermano de Weldix.
