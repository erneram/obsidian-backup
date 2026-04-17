---
title: Tenancy – Architecture
tags: [laravel, tenancy, architecture, events, middleware, routes]
---

# 🏛️ Tenancy – Architecture

Cómo funciona Tenancy por dentro: eventos, middlewares, rutas centrales vs tenant.

---

## Event-driven architecture

Todo en Tenancy se basa en **eventos**. Cuando un tenant se crea, inicializa o termina, se disparan eventos a los que el `TenancyServiceProvider` reacciona.

### Eventos principales

| Evento | Cuándo se dispara |
|--------|-------------------|
| `TenantCreated` | Al hacer `Tenant::create(...)` |
| `TenancyInitialized` | Al detectar el tenant en el request |
| `TenancyEnded` | Al terminar el ciclo del tenant |
| `DatabaseCreated` | Al crear la DB del tenant |
| `MigrationsRan` | Después de correr las migraciones del tenant |

### TenancyServiceProvider (simplificado)

```php
// app/Providers/TenancyServiceProvider.php

Events::listen(TenantCreated::class, JobPipeline::make([
    CreateDatabase::class,
    MigrateDatabase::class,
    // SeedDatabase::class, // opcional
])->send(fn (TenantCreated $event) => $event->tenant)->toListener());
```

---

## Bootstrappers

Los bootstrappers son los que aíslan cada recurso cuando se inicializa un tenant:

| Bootstrapper | Qué aísla |
|--------------|-----------|
| `DatabaseTenancyBootstrapper` | Conexión a la DB |
| `CacheTenancyBootstrapper` | Prefijo en caché |
| `FilesystemTenancyBootstrapper` | Paths del filesystem |
| `QueueTenancyBootstrapper` | Payload de los jobs |
| `RedisTenancyBootstrapper` | Prefix en Redis |

Configurados en `config/tenancy.php`:

```php
'bootstrappers' => [
    Stancl\Tenancy\Bootstrappers\DatabaseTenancyBootstrapper::class,
    Stancl\Tenancy\Bootstrappers\CacheTenancyBootstrapper::class,
    Stancl\Tenancy\Bootstrappers\FilesystemTenancyBootstrapper::class,
    Stancl\Tenancy\Bootstrappers\QueueTenancyBootstrapper::class,
],
```

---

## Rutas: Central vs Tenant

### Rutas centrales (`routes/web.php`)

Envolver en middleware para que solo sean accesibles desde dominios centrales:

```php
// routes/web.php
Route::middleware(['web'])->group(function () {
    Route::get('/', HomeController::class);
    Route::get('/register', RegisterController::class);
});
```

### Rutas de tenant (`routes/tenant.php`)

```php
// routes/tenant.php
declare(strict_types=1);

use Illuminate\Support\Facades\Route;
use Stancl\Tenancy\Middleware\InitializeTenancyByDomain;
use Stancl\Tenancy\Middleware\PreventAccessFromCentralDomains;

Route::middleware([
    'web',
    InitializeTenancyByDomain::class,
    PreventAccessFromCentralDomains::class,
])->group(function () {
    Route::get('/', fn () => 'Tenant: ' . tenant('id'));
    // Todas las rutas que pertenecen al tenant
});
```

---

## Middlewares clave

| Middleware | Función |
|------------|---------|
| `InitializeTenancyByDomain` | Detecta el tenant por el dominio del request y lo inicializa |
| `InitializeTenancyBySubdomain` | Detecta por subdominio (variante) |
| `PreventAccessFromCentralDomains` | Bloquea acceso a rutas tenant desde dominios centrales |

---

## Database drivers

| Driver | Cuándo usarlo |
|--------|---------------|
| `mysql` | Una DB por tenant en el mismo servidor MySQL |
| `pgsql` | Igual pero PostgreSQL |
| `sqlite` | Dev/testing, una DB por tenant como archivo `.sqlite` |
| Schema-based | Un schema por tenant en la misma DB (solo PostgreSQL) |

Configurar en `config/tenancy.php`:

```php
'database' => [
    'suffix' => '',
    'managers' => [
        'sqlite' => Stancl\Tenancy\Database\DatabaseManager::class,
        'mysql'  => Stancl\Tenancy\Database\DatabaseManager::class,
        'pgsql'  => Stancl\Tenancy\Database\DatabaseManager::class,
    ],
],
```

---

## Acceder al tenant actual

```php
// En cualquier parte del código (dentro del contexto tenant)
tenant();          // instancia del Tenant actual
tenant('id');      // campo específico
tenancy()->tenant; // igual
```

---

## Flujo completo del ciclo de vida

```txt
1. Request entra con host "foo.localhost"
2. InitializeTenancyByDomain middleware corre
3. Tenancy busca `domains` donde `domain = "foo.localhost"`
4. Encuentra → obtiene el Tenant asociado
5. Dispara evento TenancyInitialized
6. Bootstrappers corren:
   - DB connection switch → "tenant_foo" DB
   - Cache prefix → "foo_"
   - Filesystem → storage/app/foo/
7. Request procesa en contexto aislado del tenant "foo"
8. Response generado
9. Evento TenancyEnded → bootstrappers hacen revert
10. Conexión central restaurada
```

---

## ↩️ Navegación

- [[Backend/Laravel/Packages/Tenancy/Tenancy|🏢 Tenancy Hub]]
