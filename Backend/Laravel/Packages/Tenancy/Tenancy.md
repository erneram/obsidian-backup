---
title: Tenancy for Laravel – Hub
tags: [hub, laravel, tenancy, multitenancy, saas]
---

# 🏢 Tenancy for Laravel – Hub

Multi-tenancy automático para Laravel. Cada tenant tiene su propia base de datos, caché, filesystem, colas y Redis. Basado en arquitectura de **eventos**.

- Paquete: `stancl/tenancy`
- Docs: https://tenancyforlaravel.com/docs/v3/

---

## 🧠 Arquitectura

```txt
Request → Dominio (foo.localhost)
  → Middleware InitializeTenancyByDomain
    → Busca tenant por dominio
      → Cambia conexión de DB al tenant
        → Aísla caché, filesystem, queues
          → Request procesa en contexto del tenant
```

### Central vs Tenant

| Scope | Descripción |
|-------|-------------|
| **Central** | DB principal, lógica de admin, creación de tenants |
| **Tenant** | DB aislada por tenant, datos del usuario final |

---

## 📂 Páginas

- [[Backend/Laravel/Packages/Tenancy/Installation|Installation]] — composer, artisan install, migrate, service provider.
- [[Backend/Laravel/Packages/Tenancy/Tenant Creation|Tenant Creation]] — modelo Tenant, crear tenant, crear dominio, migraciones tenant.
- [[Backend/Laravel/Packages/Tenancy/Valet + Hosts|Valet + Hosts]] — cómo funciona Valet con tenancy, `/etc/hosts`, subdominios locales.
- [[Backend/Laravel/Packages/Tenancy/Architecture|Architecture]] — eventos, middlewares, rutas centrales vs tenant, configuración avanzada.

---

## ↩️ Navegación

- [[Backend/Laravel/Laravel|🧱 Laravel]] → [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
