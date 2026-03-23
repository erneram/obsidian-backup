---
title: Jobs + Predis — Cómo se relacionan
tags: [laravel, jobs, redis, predis, queues, async]
---

# 🔗 Jobs + Predis — Cómo se relacionan

> [!example] La analogía
> Imagina una cocina de restaurante.
> El **mesero** toma el pedido del cliente y lo pone en una **bandeja** en la ventana de cocina — no espera a que la comida esté lista, regresa de inmediato a atender otra mesa.
> El **cocinero** ve la bandeja, prepara el plato y lo entrega cuando está listo.
>
> En Laravel: el **mesero** es tu código, la **bandeja** es Redis, y el **cocinero** es el worker.

---

## El problema que resuelven juntos

Algunas acciones en tu app toman tiempo: enviar un email, mandar una notificación push, actualizar un índice de búsqueda. Si las haces en el mismo momento que el usuario aprieta un botón, ese usuario tiene que *esperar* a que todo termine.

**Sin Jobs + Redis:**
```
Usuario aprueba algo → tu app envía email → envía push → actualiza índice → responde "listo"
                 ↑_________________ el usuario espera 3-5 segundos _________________↑
```

**Con Jobs + Redis:**
```
Usuario aprueba algo → tu app responde "listo" en <100ms ✓
                              ↓
                    Redis guarda las tareas pendientes
                              ↓
                    Workers las hacen en background, solos
```

El usuario ya está en otra pantalla cuando los emails salen. Nunca sintió la espera.

---

## ¿Cómo funciona paso a paso?

### 1. Tu código crea el Job y lo despacha

```php
// Esto es todo lo que tú escribes
SendWelcomeEmail::dispatch($user);
```

En este momento, Laravel **no ejecuta el email**. Solo le dice a Redis: *"oye, guarda esta tarea, alguien la hará después"*.

---

### 2. Redis lo guarda en una lista de espera

Redis es una base de datos en memoria (muy rápida). Internamente tiene listas por tipo de tarea:

```
Redis db:0
  queues:emails         → [tarea1, tarea2, tarea3...]
  queues:notifications  → [tarea4]
  queues:default        → [tarea5, tarea6...]
```

Cada "tarea" es el Job convertido a texto (JSON) con toda la información que necesita para ejecutarse — incluyendo el usuario que le pasaste.

> [!info] ¿Por qué Redis y no MySQL?
> Podrías guardar los jobs en una tabla de MySQL, pero Redis vive en RAM y es mucho más rápido. La diferencia es como comparar buscar algo en tu escritorio vs ir al archivo del sótano.

---

### 3. El worker está mirando Redis todo el tiempo

El worker es un proceso PHP que corre en el servidor, **siempre activo**, esperando que lleguen tareas:

```bash
php artisan queue:work redis --queue=emails,notifications,default
```

Cuando Redis tiene un job en la lista, el worker lo agarra, lo ejecuta y lo borra de Redis. Si no hay nada, espera.

```
Worker: "¿Hay algo en emails?"   Redis: "Sí, aquí tienes"  → Worker ejecuta → borra de Redis
Worker: "¿Hay algo en emails?"   Redis: "No"               → Worker espera...
Worker: "¿Hay algo en emails?"   Redis: "No"               → Worker espera...
Worker: "¿Hay algo en emails?"   Redis: "Sí, aquí tienes"  → Worker ejecuta → borra de Redis
```

---

### 4. ¿Qué pasa si el Job falla?

Cosas pasan — el servidor de email puede estar caído, hay un error en el código, se cortó la conexión. Redis lo maneja:

```
Job falla
  ↓
¿Tiene reintentos disponibles?
  ├── SÍ → Redis espera unos segundos y vuelve a intentarlo
  └── NO → el Job va a la tabla "failed_jobs" en MySQL para que puedas revisarlo
```

Tú defines cuántos reintentos quieres en el Job:

```php
class SendWelcomeEmail implements ShouldQueue
{
    public int $tries = 3;      // intenta 3 veces antes de darse por vencido
    public int $backoff = 60;   // espera 60 segundos entre cada intento
}
```

---

## Horizon: ver todo lo que pasa

Correr workers con `queue:work` es como manejar a ciegas — no sabes si están vivos, cuántas tareas hay pendientes o cuántas fallaron.

**Horizon** es el tablero de control. Lo instalas una vez y te da un panel en `/horizon` donde puedes ver en tiempo real:

- Cuántos jobs se están procesando por minuto
- Cuántos fallaron y por qué
- Cuánto tiempo tarda cada tipo de job
- Cuántos jobs están esperando en Redis ahora mismo

> [!tip] Horizon también reinicia los workers solos
> Si un worker se muere (por un deploy, falta de memoria, etc.), Horizon lo detecta y levanta uno nuevo. Sin Horizon, los jobs se quedan parados en Redis sin que nadie los procese.
> Ver [[Backend/Laravel/Queues/Horizon|Horizon]] para la guía completa.

---

## Un ejemplo real completo

Un padre aprueba el permiso de su hijo en la app. En ese momento necesitan pasar 4 cosas:

```php
public function approve(Permission $permission): void
{
    // Esto sí es inmediato — actualizar en MySQL
    $permission->update(['status' => 'approved']);

    // Esto va a Redis — no bloquea al usuario
    SendApprovalEmail::dispatch($permission)->onQueue('emails');
    SendPushNotification::dispatch($permission)->onQueue('notifications');
    NotifyDriverJob::dispatch($permission)->onQueue('notifications');
    UpdateSearchIndex::dispatch($permission->student)->onQueue('default');
}
```

**Lo que el usuario vive:** aprieta aprobar, la pantalla responde en menos de un segundo.

**Lo que pasa en Redis mientras tanto:**

```
queues:emails         → [SendApprovalEmail]
queues:notifications  → [SendPushNotification, NotifyDriverJob]
queues:default        → [UpdateSearchIndex]
```

Tres workers distintos agarran cada lista y procesan en paralelo. En 2-3 segundos todo está hecho, sin que el usuario haya esperado nada.

---

## En resumen

| Quién | Qué hace |
|-------|----------|
| **Tu código** | Crea el Job y llama `dispatch()` — su trabajo termina ahí |
| **Redis** | Guarda la tarea en una lista, la entrega al worker cuando esté listo |
| **Worker** | Está siempre corriendo, agarra tareas de Redis y las ejecuta |
| **Horizon** | Vigila a los workers, los reinicia si mueren, muestra métricas |
| **`failed_jobs`** | Donde terminan los jobs que fallaron demasiadas veces |

---

## Ver también

- [[Backend/Laravel/Queues/Jobs|Jobs]] — cómo crear y configurar Jobs.
- [[Backend/Laravel/Packages/Predis|Predis (Redis)]] — configuración completa de Redis en Laravel.
- [[Backend/Laravel/Packages/Meilisearch + Predis|Meilisearch + Predis]] — otro caso donde Redis actúa de capa async.

---

## ↩️ Navegación
- [[Backend/Laravel/Laravel|Laravel]] → [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
