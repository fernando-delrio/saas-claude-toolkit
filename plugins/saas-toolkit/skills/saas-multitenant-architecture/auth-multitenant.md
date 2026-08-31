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
