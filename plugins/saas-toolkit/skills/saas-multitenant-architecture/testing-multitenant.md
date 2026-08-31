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
