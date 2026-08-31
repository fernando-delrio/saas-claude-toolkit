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
