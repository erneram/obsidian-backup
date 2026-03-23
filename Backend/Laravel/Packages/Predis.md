---
title: Redis + Predis en Laravel
tags: [laravel, redis, predis, queues, cache]
---

# ⚡ Redis + Predis en Laravel

Redis es una **base de datos en memoria (RAM)** ultrarrápida. En Laravel se usa principalmente para dos cosas: **Queues** y **Cache**. **Predis** es el cliente PHP que traduce comandos Laravel a comandos Redis.

> [!example] Analogía
> MySQL es como un archivero físico — rápido pero requiere abrir cajones.
> Redis es como tu memoria de trabajo — acceso instantáneo, capacidad limitada, precio mayor por GB.

---

## Instalación

```bash
composer require predis/predis
```

```env
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=null

QUEUE_CONNECTION=redis   # activar queues con Redis
CACHE_STORE=redis        # activar cache con Redis
```

```php
// config/database.php
'redis' => [
    'client' => 'predis',

    'default' => [          // DB 0 — Queues y sesiones
        'host'     => env('REDIS_HOST', '127.0.0.1'),
        'port'     => env('REDIS_PORT', 6379),
        'database' => '0',
    ],

    'cache' => [            // DB 1 — Cache (separada de queues)
        'host'     => env('REDIS_HOST', '127.0.0.1'),
        'port'     => env('REDIS_PORT', 6379),
        'database' => '1',
    ],
],
```

> [!info] Dos bases de datos separadas
> Redis soporta 16 DBs (0–15) en una sola instancia. Separar queues y cache en distintas DBs permite flushear la cache sin afectar los jobs pendientes.

---

## Uso 1: Queue System (db:0)

El caso de uso más crítico. Sin queues, acciones que disparan emails, push notifications o indexaciones bloquean el request HTTP.

**Sin queues:**
```
Request HTTP → Guarda en DB → Envía email → Envía push → Indexa → Responde
                                                                  ↑ 3-5 segundos
```

**Con Redis + Queues:**
```
Request HTTP → Guarda en DB → Responde en <100ms ✓
                     ↓
              Despacha Jobs a Redis (db:0)
                     ↓
        Queue Worker (proceso en background):
          ├── SendWelcomeEmailJob
          ├── SendPushNotificationJob
          └── UpdateSearchIndexJob
```

### Crear y despachar un Job

```bash
php artisan make:job SendWelcomeEmail
```

```php
class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public User $user) {}

    public function handle(): void
    {
        Mail::to($this->user)->send(new WelcomeMail($this->user));
    }
}

// Despachar desde un Controller / Service
SendWelcomeEmail::dispatch($user);

// En una queue específica
SendWelcomeEmail::dispatch($user)->onQueue('emails');
```

### Queues recomendadas

```php
// config/queue.php — definir múltiples queues
'connections' => [
    'redis' => [
        'driver' => 'redis',
        'queue'  => ['emails', 'notifications', 'default'],
    ],
],
```

| Queue | Propósito |
|-------|-----------|
| `emails` | Envío de correos |
| `notifications` | Push notifications, SMS, WhatsApp |
| `default` | Todo lo demás (indexación, generación de archivos, etc.) |

### Correr los workers

```bash
# Procesar todas las queues
php artisan queue:work redis

# Con prioridad (emails primero)
php artisan queue:work redis --queue=emails,notifications,default

# En producción: usar Supervisor para mantener los workers vivos
```

### Horizon (dashboard de workers)

```bash
composer require laravel/horizon
php artisan horizon:install
php artisan horizon
```

> [!tip] Laravel Horizon
> Panel web en `/horizon` para ver jobs corriendo, fallidos, throughput por queue y métricas de workers en tiempo real. Esencial en producción.

---

## Uso 2: Cache (db:1)

Guardar en RAM resultados costosos de calcular.

```php
// Sin cache: consulta MySQL en cada request (~50ms)
$config = School::find($id)->with('parameters')->get();

// Con cache: consulta MySQL una vez, sirve desde RAM las siguientes 3600s (<1ms)
$config = Cache::remember("school:config:{$id}", 3600, function () {
    return School::find($id)->with('parameters')->get();
});

// Invalidar cache cuando cambia el dato
Cache::forget("school:config:{$id}");

// O usar tags para invalidar grupos
Cache::tags(['schools'])->remember("config:{$id}", 3600, fn() => ...);
Cache::tags(['schools'])->flush(); // borra todo lo de schools
```

**Aplica para:** configuración de la app, listas de opciones estáticas, resultados de queries pesadas, respuestas de APIs externas.

---

## Uso 3: Rate Limiting

```php
// Limitar intentos de login
RateLimiter::for('login', function (Request $request) {
    return Limit::perMinute(5)->by($request->input('email'));
});
```

---

## Comandos útiles

```bash
# Ver qué hay en Redis (requiere redis-cli)
redis-cli
> KEYS *          # listar todas las keys
> DBSIZE          # cuántas keys hay
> FLUSHDB         # vaciar la DB activa (¡cuidado en producción!)

# Desde Laravel
php artisan queue:work              # procesar jobs
php artisan queue:failed            # ver jobs fallidos
php artisan queue:retry all         # reintentar todos los fallidos
php artisan queue:flush             # limpiar jobs fallidos

php artisan cache:clear             # limpiar cache
```

---

## Alternativas

| Tecnología | Tipo | Trade-offs |
|-----------|------|------------|
| **Memcached** | Solo cache | No tiene queues, no persiste datos |
| **Database queue** | Queues en MySQL | Más lento, sobrecarga la DB principal |
| **SQS (Amazon)** | Queues cloud | Robusto y managed, pero costo y vendor lock-in |
| **RabbitMQ** | Queues | Más potente para mensajería compleja, más difícil de operar |
| **Valkey** | Cache + Queue | Fork open source de Redis (cambió licencia en 2024) |
| **DragonflyDB** | Cache + Queue | Compatible con Redis, más rápido en benchmarks |

---

## ↩️ Navegación
- [[Backend/Laravel/Queues/Jobs + Predis|🔗 Jobs + Redis]] · [[Backend/Laravel/Packages/Meilisearch + Predis|🔗 Meilisearch + Redis]] · [[Backend/Laravel/Laravel|Laravel]] → [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
