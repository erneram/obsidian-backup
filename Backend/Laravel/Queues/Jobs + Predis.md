---
title: Jobs + Predis — Cómo se relacionan
tags: [laravel, jobs, redis, predis, queues, async]
---

# 🔗 Jobs + Predis — Cómo se relacionan

Un **Job** define *qué* hay que hacer. **Redis (Predis)** define *dónde* se guarda y *cómo* se distribuye el trabajo. Son la misma moneda, dos caras.

Sin Redis, los Jobs solo pueden correr de forma síncrona o guardar en una tabla MySQL (lento). Con Redis, los Jobs se procesan en paralelo, a escala, con reintentos automáticos y visibilidad en tiempo real.

---

## El flujo completo

```
Tu código llama dispatch()
        │
        ▼
Laravel serializa el Job (clase PHP → JSON)
        │
        ▼
Predis lo escribe en Redis db:0
  KEY: queues:emails  →  [job1, job2, job3...]
        │
        ▼
Worker (queue:work / Horizon) hace BLPOP en Redis
  Redis entrega el siguiente job de la lista
        │
        ▼
Laravel deserializa el JSON → reconstruye la clase PHP
        │
        ├── handle() exitoso → Redis elimina el job
        │
        └── handle() lanza excepción
                │
                ├── quedan reintentos → Redis lo reencola con delay (backoff)
                │
                └── se agotaron reintentos → tabla failed_jobs en MySQL
```

---

## Configuración que los une

```env
# .env — los dos settings críticos
QUEUE_CONNECTION=redis     # Jobs van a Redis, no a MySQL ni sync
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

```php
// config/database.php — Redis db:0 es exclusivo para queues
'redis' => [
    'client' => 'predis',

    'default' => [
        'database' => '0',   // ← aquí viven los Jobs en cola
    ],
    'cache' => [
        'database' => '1',   // ← separado, solo para Cache::remember()
    ],
]
```

```php
// config/queue.php — cómo Laravel habla con Redis para queues
'connections' => [
    'redis' => [
        'driver'     => 'redis',
        'connection' => 'default',    // usa redis.default (db:0)
        'queue'      => ['emails', 'notifications', 'default'],
        'retry_after' => 90,          // segundos antes de considerar el job muerto
        'block_for'   => null,
    ],
]
```

---

## Qué guarda Redis exactamente

Cuando haces `SendWelcomeEmail::dispatch($user)`, Redis almacena algo así:

```json
{
  "uuid": "abc-123",
  "displayName": "App\\Jobs\\SendWelcomeEmail",
  "job": "Illuminate\\Queue\\CallQueuedHandler@call",
  "data": {
    "commandName": "App\\Jobs\\SendWelcomeEmail",
    "command": "O:30:\"App\\Jobs\\SendWelcomeEmail\":1:{s:4:\"user\";...}"
  },
  "attempts": 0,
  "maxTries": 3,
  "backoff": 60,
  "timeout": 30,
  "pushedAt": 1700000000.123
}
```

El Job se serializa completo — incluyendo los modelos Eloquent que le pasaste. El worker lo deserializa y reconstruye el objeto para ejecutarlo.

---

## El worker: el proceso que une todo

```bash
# El worker escucha Redis y ejecuta jobs continuamente
php artisan queue:work redis --queue=emails,notifications,default
```

Internamente hace un loop:

```
loop:
  job = Redis.BLPOP('queues:emails', timeout=5)   ← espera hasta 5s por un job
  if job:
    ejecuta job.handle()
    if éxito:  Redis.DEL(job)
    if falla:  job.attempts++
               if attempts < maxTries: Redis.ZADD('queues:emails:delayed', ...)
               else: INSERT INTO failed_jobs ...
  goto loop
```

> [!warning] El worker no se reinicia solo
> Si el worker muere (por OOM, deploy, etc.), los jobs se quedan en Redis sin procesar.
> En producción siempre usar **Supervisor** o **Horizon** para que el worker se reinicie automáticamente.

---

## Horizon: el panel que hace todo visible

Horizon es el puente de observabilidad entre Jobs y Redis. Reemplaza a `queue:work` en producción.

```bash
composer require laravel/horizon
php artisan horizon:install
php artisan horizon          # arranca workers + dashboard
```

```php
// config/horizon.php — define cuántos workers por queue
'environments' => [
    'production' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue'      => ['emails', 'notifications', 'default'],
            'balance'    => 'auto',       // auto-escala workers según carga
            'processes'  => 5,
            'tries'      => 3,
        ],
    ],
],
```

**Qué muestra Horizon en `/horizon`:**

| Métrica | Qué significa |
|---------|---------------|
| Jobs procesados/min | Throughput actual de los workers |
| Jobs fallidos | Cuántos agotaron sus reintentos |
| Tiempo promedio | Cuánto tarda cada Job en ejecutarse |
| Jobs pendientes | Cuántos esperan en Redis ahora mismo |
| Workers activos | Cuántos procesos PHP consumen la cola |

---

## Visibilidad directa en Redis

```bash
redis-cli -n 0   # conectar a db:0 (queues)

# Ver cuántos jobs hay en cada queue
LLEN queues:emails
LLEN queues:notifications
LLEN queues:default

# Ver el primer job sin sacarlo
LINDEX queues:emails 0

# Ver jobs con delay (reintentando)
ZRANGE queues:emails:delayed 0 -1 WITHSCORES
```

---

## Jobs + Redis en la práctica real

### Escenario: aprobación de permiso con 4 efectos secundarios

```php
// PermissionService.php
public function approve(Permission $permission): void
{
    $permission->update(['status' => 'approved']);   // MySQL — síncrono

    // Todo lo demás: asíncrono a Redis
    SendPermissionApprovedEmail::dispatch($permission)->onQueue('emails');
    SendPermissionPushNotification::dispatch($permission)->onQueue('notifications');
    UpdateMeilisearchIndex::dispatch($permission->student)->onQueue('default');
    NotifyDriverJob::dispatch($permission)->onQueue('notifications');
}
```

**Lo que ocurre en Redis:**

```
queues:emails        → [SendPermissionApprovedEmail]
queues:notifications → [SendPermissionPushNotification, NotifyDriverJob]
queues:default       → [UpdateMeilisearchIndex]
```

**El usuario recibe respuesta en `<100ms`.**
Los 4 workers procesan en paralelo en background.

---

## Resumen de responsabilidades

| Componente | Rol |
|-----------|-----|
| **Job** (`ShouldQueue`) | Define qué trabajo hay que hacer |
| **`dispatch()`** | Lo serializa y lo manda a Redis |
| **Redis db:0** | Lo guarda en lista FIFO por queue |
| **Worker / Horizon** | Consume Redis y ejecuta el Job |
| **`failed_jobs`** | Tabla MySQL de Jobs que fallaron definitivamente |

---

## Ver también

- [[Backend/Laravel/Queues/Jobs|Jobs]] — crear, despachar, chains, batches.
- [[Backend/Laravel/Packages/Predis|Predis (Redis)]] — configuración completa de Redis en Laravel.
- [[Backend/Laravel/Packages/Meilisearch + Predis|Meilisearch + Predis]] — otro ejemplo de Redis como capa async.

---

## ↩️ Navegación
- [[Backend/Laravel/Laravel|Laravel]] → [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
