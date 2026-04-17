---
title: Tenancy – Installation
tags: [laravel, tenancy, install, setup]
---

# ⚙️ Tenancy – Installation

---

## 1. Instalar el paquete

```bash
composer require stancl/tenancy
```

---

## 2. Publicar y ejecutar el installer

```bash
php artisan tenancy:install
```

Esto genera:
- `config/tenancy.php`
- `app/Providers/TenancyServiceProvider.php`
- Migraciones para las tablas `tenants` y `domains`
- Rutas de ejemplo en `routes/tenant.php`

---

## 3. Registrar el Service Provider

En `bootstrap/providers.php`:

```php
return [
    App\Providers\AppServiceProvider::class,
    App\Providers\TenancyServiceProvider::class, // ← agregar
];
```

---

## 4. Correr migraciones centrales

```bash
php artisan migrate
```

Esto crea en la DB central:
- `tenants` — registro de cada tenant
- `domains` — dominios asociados a cada tenant

---

## 5. Crear el modelo Tenant

```bash
php artisan make:model Tenant
```

```php
<?php

namespace App\Models;

use Stancl\Tenancy\Database\Models\Tenant as BaseTenant;
use Stancl\Tenancy\Contracts\TenantWithDatabase;
use Stancl\Tenancy\Database\Concerns\HasDatabase;
use Stancl\Tenancy\Database\Concerns\HasDomains;

class Tenant extends BaseTenant implements TenantWithDatabase
{
    use HasDatabase, HasDomains;
}
```

---

## 6. Actualizar config/tenancy.php

```php
'tenant_model' => App\Models\Tenant::class,
```

---

## ✅ Resultado

- Tablas `tenants` y `domains` en la DB central
- Modelo `Tenant` listo para crear tenants con DB propia
- Service Provider activo con todos los event listeners

---

## ↩️ Navegación

- [[Backend/Laravel/Packages/Tenancy/Tenancy|🏢 Tenancy Hub]]
