---
title: Laravel Horizon
tags: [laravel, horizon, queues, redis, workers]
---

# 🌅 Laravel Horizon

## ¿Qué es?

Horizon es el **panel de control oficial de Laravel para tus queues**. Es un paquete que installas una vez y te da una interfaz web donde puedes ver en tiempo real todo lo que está pasando con tus Jobs y workers.

Sin Horizon, los workers corren en silencio — no sabes si están vivos, cuántas tareas hay acumuladas, o por qué un email nunca llegó. Con Horizon, todo eso se vuelve visible.

> [!example] Analogía
> Es como el tablero de un aeropuerto: tú no controlas los aviones (workers), pero puedes ver cuáles están en pista, cuáles se retrasaron, cuáles tuvieron un problema y dónde está tu vuelo (job) en la cola.

---

## El problema sin Horizon

Cuando corres workers con el comando básico:

```bash
php artisan queue:work
```

Solo sabes que "está corriendo". No sabes:
- ¿Cuántos jobs están esperando en Redis ahora mismo?
- ¿Ese email que debía salir hace 10 minutos, salió?
- ¿El worker se cayó durante el deploy y nadie lo notó?
- ¿Cuánto tarda en promedio cada tipo de job?

En local esto no importa mucho. En producción, con usuarios reales esperando notificaciones, importa muchísimo.

---

## Qué puedes ver en Horizon

Horizon vive en `/horizon` de tu app. Desde ahí tienes:

| Sección | Qué muestra |
|---------|-------------|
| **Dashboard** | Jobs por minuto, tiempo de espera promedio, jobs fallidos recientes |
| **Monitoring** | Queues activas y cuántos jobs tienen pendientes |
| **Recent Jobs** | Historial de los últimos jobs ejecutados y su estado |
| **Failed Jobs** | Jobs que fallaron con el error completo y stack trace |
| **Workers** | Cuántos procesos están corriendo y en qué queue |

---

## Instalación

```bash
composer require laravel/horizon
php artisan horizon:install
```

Esto crea el archivo `config/horizon.php` y los assets del dashboard. Solo necesitas hacerlo una vez.

---

## Configuración básica

En `config/horizon.php` defines cuántos workers quieres y en qué queues:

```php
'environments' => [

    'production' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue'      => ['emails', 'notifications', 'default'],
            'balance'    => 'auto',    // Horizon decide cuántos workers según la carga
            'processes'  => 5,         // máximo 5 workers en paralelo
            'tries'      => 3,
        ],
    ],

    'local' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue'      => ['default'],
            'balance'    => 'simple',
            'processes'  => 1,         // en local con 1 worker alcanza
            'tries'      => 1,
        ],
    ],

],
```

> [!info] `balance: auto`
> Con esto, si la queue `emails` tiene 100 jobs acumulados y `notifications` tiene 2, Horizon manda más workers a `emails` automáticamente. Se adapta a la carga real.

---

## Correrlo

```bash
# Arranca Horizon (workers + dashboard)
php artisan horizon

# En producción, con Supervisor para que no se muera
# (Supervisor lo reinicia automáticamente si falla)
```

Una vez corriendo, entra a `tuapp.com/horizon`.

---

## La parte más útil: Failed Jobs

Cuando un job falla (agota sus reintentos), Horizon lo guarda con:
- El error exacto que lanzó
- El stack trace completo
- Cuántas veces lo intentó
- En qué momento falló

Desde el panel puedes **reintentarlo con un clic** sin tener que tocar código ni la terminal.

```bash
# También desde terminal si prefieres
php artisan horizon:list-failed
php artisan queue:retry [id-del-job]
php artisan queue:retry all
```

---

## Proteger el dashboard en producción

Por defecto Horizon solo es accesible en `local`. En producción necesitas definir quién puede verlo:

```php
// app/Providers/HorizonServiceProvider.php
protected function gate(): void
{
    Gate::define('viewHorizon', function ($user) {
        return in_array($user->email, [
            'tu@email.com',   // solo estos emails pueden entrar
        ]);
    });
}
```

---

## Horizon en pocas palabras

```
Sin Horizon:  php artisan queue:work  →  corre en silencio, a ciegas
Con Horizon:  php artisan horizon     →  workers + dashboard + métricas + failed jobs
```

Horizon no cambia cómo funcionan los Jobs ni Redis — solo los hace **observables**. Todo sigue igual por dentro, pero ahora puedes ver qué está pasando.

---

## Ver también

- [[Backend/Laravel/Queues/Jobs|Jobs]] — cómo crear y despachar Jobs.
- [[Backend/Laravel/Queues/Jobs + Predis|Jobs + Predis]] — cómo Redis almacena y distribuye los Jobs.
- [[Backend/Laravel/Packages/Predis|Predis (Redis)]] — configuración de Redis en Laravel.

---

## ↩️ Navegación
- [[Backend/Laravel/Laravel|Laravel]] → [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
