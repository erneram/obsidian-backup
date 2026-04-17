---
title: Tenancy – Tenant Creation
tags: [laravel, tenancy, tenant, database, migrations]
---

# 🏗️ Tenant Creation

Cómo crear tenants, asociarles dominios, y estructurar las migraciones.

---

## 1. Estructura de migraciones

Las migraciones se dividen en dos grupos:

| Carpeta | Corre en | Contiene |
|---------|----------|----------|
| `database/migrations/` | DB central | `tenants`, `domains` |
| `database/migrations/tenant/` | DB de cada tenant | `users`, `passwords`, tu lógica de negocio |

Mover las migraciones que pertenecen al tenant:

```bash
mkdir database/migrations/tenant
mv database/migrations/0001_01_01_000000_create_users_table.php database/migrations/tenant/
mv database/migrations/0001_01_01_000001_create_cache_table.php database/migrations/tenant/
mv database/migrations/0001_01_01_000002_create_jobs_table.php database/migrations/tenant/
```

Cuando se crea un tenant, el paquete crea su DB y corre automáticamente las migraciones en `database/migrations/tenant/`.

---

## 2. Crear un tenant (Tinker)

```bash
php artisan tinker
```

```php
// Crear tenant
$tenant = App\Models\Tenant::create(['id' => 'acme']);

// Asociar dominio
$tenant->domains()->create(['domain' => 'acme.localhost']);
```

El evento `TenantCreated` se dispara automáticamente → crea la DB del tenant → corre las migraciones tenant.

---

## 3. Crear tenant programáticamente

```php
use App\Models\Tenant;

$tenant = Tenant::create([
    'id' => 'acme',
    // datos extra opcionales:
    'name' => 'Acme Corp',
    'plan' => 'pro',
]);

$tenant->domains()->create([
    'domain' => 'acme.localhost',
]);
```

---

## 4. Campos personalizados en el tenant

El modelo Tenant usa una columna `data` (JSON) para campos extras:

```php
// config/tenancy.php
'custom_columns' => ['name', 'plan'],
```

O simplemente guardar en el JSON sin config extra:

```php
Tenant::create([
    'id' => 'acme',
    'name' => 'Acme Corp', // se guarda en columna `data`
]);
```

---

## 5. Correr migraciones en un tenant específico

```bash
php artisan tenants:migrate --tenants=acme
```

Correr en todos los tenants:

```bash
php artisan tenants:migrate
```

---

## 6. Ejecutar código en contexto de un tenant

```php
$tenant->run(function () {
    // Aquí la conexión DB ya es la del tenant
    User::create([...]);
});
```

---

## 7. Seeders por tenant

```bash
php artisan tenants:seed --class=TenantDatabaseSeeder
```

---

## ⚠️ Errores comunes

| Error | Causa | Solución |
|-------|-------|----------|
| `No such table: users` en central | Migración de users no movida a `tenant/` | Moverla a `database/migrations/tenant/` |
| DB del tenant no se crea | Service Provider no registrado | Agregar a `bootstrap/providers.php` |
| Dominio no resuelve | `/etc/hosts` no configurado | Ver [[Backend/Laravel/Packages/Tenancy/Valet + Hosts\|Valet + Hosts]] |

---

## ↩️ Navegación

- [[Backend/Laravel/Packages/Tenancy/Tenancy|🏢 Tenancy Hub]]
