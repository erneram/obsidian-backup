---
title: Jobs en Laravel
tags: [laravel, jobs, queues, async]
---

# ⚙️ Jobs en Laravel

## ¿Qué es un Job?

Un **Job** es una clase PHP que representa una **unidad de trabajo** que puede ejecutarse de forma síncrona o asíncrona (en background). Es la forma que tiene Laravel de decir: *"esto no necesita hacerse ahora mismo, hazlo después y no bloquees al usuario"*.

Cuando en cualquier parte del código ves palabras como `dispatch()`, `ShouldQueue`, `queue:work`, `Horizon`, o `failed_jobs` — se están refiriendo a Jobs.

---

## Síncrono vs Asíncrono

```
SIN Jobs (síncrono):
Request → Controller → guarda en DB → envía email → envía push → responde
                                                                  ↑ 3-5 seg de espera

CON Jobs (asíncrono):
Request → Controller → guarda en DB → responde en <100ms ✓
                              ↓
                     dispatch(new SendEmailJob($user))
                              ↓
                     Job va a la cola (Redis/DB)
                              ↓
                     Worker lo procesa en background
```

---

## Crear un Job

```bash
php artisan make:job SendWelcomeEmail
```

```php
// app/Jobs/SendWelcomeEmail.php
class SendWelcomeEmail implements ShouldQueue   // ← esto lo hace asíncrono
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public User $user   // los datos que necesita el job
    ) {}

    public function handle(): void
    {
        // Todo lo que esté aquí se ejecuta en background
        Mail::to($this->user)->send(new WelcomeMail($this->user));
    }
}
```

> [!info] `ShouldQueue`
> Si el Job implementa `ShouldQueue`, Laravel lo manda a la cola configurada (Redis, DB, SQS…).
> Si NO implementa `ShouldQueue`, se ejecuta sincrónicamente en el mismo request — útil para tests o tareas rápidas.

---

## Despachar un Job

```php
// Desde un Controller, Service, o cualquier lugar
SendWelcomeEmail::dispatch($user);

// Con delay (ejecutar en 10 minutos)
SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(10));

// En una queue específica
SendWelcomeEmail::dispatch($user)->onQueue('emails');

// Síncrono aunque tenga ShouldQueue (útil en tests)
SendWelcomeEmail::dispatchSync($user);

// Condicional
SendWelcomeEmail::dispatchIf($user->wantsEmails, $user);
```

---

## Queues disponibles (configurar en `.env`)

```env
QUEUE_CONNECTION=redis    # redis, database, sync, sqs
```

| Driver | Cuándo usarlo |
|--------|---------------|
| `redis` | Producción — rápido, Horizon, persistente |
| `database` | Sin Redis — guarda jobs en tabla `jobs` de MySQL |
| `sync` | Local/tests — ejecuta el job inmediatamente (sin queue) |
| `sqs` | AWS — cloud managed, para escala masiva |

---

## Reintentos y fallos

```php
class SendWelcomeEmail implements ShouldQueue
{
    public int $tries = 3;                    // reintentos máximos
    public int $backoff = 60;                 // segundos entre reintentos
    public int $timeout = 30;                 // timeout por intento

    public function failed(\Throwable $e): void
    {
        // Se llama cuando el job agota todos sus reintentos
        Log::error("Job falló para user {$this->user->id}: {$e->getMessage()}");
    }
}
```

---

## Correr los workers

```bash
# Procesar todas las queues
php artisan queue:work

# Con prioridad
php artisan queue:work redis --queue=emails,notifications,default

# Ver jobs fallidos
php artisan queue:failed

# Reintentar todos los fallidos
php artisan queue:retry all

# Limpiar jobs fallidos
php artisan queue:flush
```

> [!tip] Laravel Horizon
> En producción, usar **Horizon** en vez de `queue:work` directamente. Horizon gestiona los workers, los reinicia si mueren, y da un dashboard en `/horizon` con métricas en tiempo real.
> Ver [[Backend/Laravel/Packages/Predis|Predis (Redis)]] para la configuración completa.

---

## Patrones comunes

### Job encadenado (chain)

```php
// Se ejecutan en orden — si uno falla, los siguientes no corren
Bus::chain([
    new ProcessPayment($order),
    new SendConfirmationEmail($order),
    new UpdateInventory($order),
])->dispatch();
```

### Batch (lote de jobs en paralelo)

```php
$batch = Bus::batch([
    new ImportRow($rows->chunk(100)[0]),
    new ImportRow($rows->chunk(100)[1]),
    new ImportRow($rows->chunk(100)[2]),
])->then(fn(Batch $batch) => Log::info('Import completado'))
  ->catch(fn(Batch $batch, \Throwable $e) => Log::error($e))
  ->dispatch();
```

### Evento → Listener → Job (el patrón más común)

```php
// 1. El evento ocurre
event(new UserRegistered($user));

// 2. El Listener lo captura (también puede implementar ShouldQueue)
class SendWelcomeEmailListener implements ShouldQueue
{
    public function handle(UserRegistered $event): void
    {
        SendWelcomeEmail::dispatch($event->user);
    }
}
```

---

## ¿Cuándo se mencionan Jobs en el código?

| Término | Significa |
|---------|-----------|
| `dispatch()` | Encolar un Job para ejecución |
| `ShouldQueue` | Interfaz que hace el Job asíncrono |
| `queue:work` | Worker que consume y ejecuta Jobs de la cola |
| `failed_jobs` | Tabla DB donde van los Jobs que fallaron |
| `Horizon` | Dashboard + gestor de workers para Redis |
| `Bus::chain()` | Jobs en secuencia — uno tras otro |
| `Bus::batch()` | Jobs en paralelo — todos a la vez |
| `shouldBeUnique` | Job que no se duplica si ya está en cola |

---

## ↩️ Navegación
- [[Backend/Laravel/Queues/Jobs + Predis|🔗 Cómo Jobs usa Redis]] · [[Backend/Laravel/Laravel|Laravel]] → [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
