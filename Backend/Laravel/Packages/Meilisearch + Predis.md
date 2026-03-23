---
title: Meilisearch + Predis — Cómo se complementan
tags: [laravel, meilisearch, redis, predis, scout, integration]
---

# 🔗 Meilisearch + Predis — Cómo se complementan

Meilisearch y Redis no compiten — se necesitan mutuamente para que la búsqueda funcione bien en producción. Redis maneja **cuándo y cómo** se indexa; Meilisearch maneja **cómo se busca**.

---

## El problema que resuelven juntos

Cuando un modelo se guarda en MySQL, su índice de búsqueda en Meilisearch necesita actualizarse. Hay dos formas de hacerlo:

| Modo | Qué pasa | Problema |
|------|----------|---------|
| **Síncrono** | El request HTTP espera a que Meilisearch indexe | Agrega 100–500ms a cada `save()` |
| **Async con Redis** | El request responde de inmediato, Redis indexa en background | ✅ Sin impacto en el usuario |

---

## El flujo completo

```
Usuario guarda un registro (POST /api/users)
        │
        ▼
Controller → Service → Model::save()     ← MySQL se actualiza
        │
        ▼
Scout Observer detecta el cambio
        │
        ▼ (SCOUT_QUEUE=true)
Job despachado a Redis db:0 (queue: default)
        │
        ▼
Queue Worker (Horizon) lo recoge
        │
        ▼
User::toSearchableArray() se ejecuta
        │
        ▼
Datos enviados a Meilisearch HTTP API (:7700)
        │
        ▼
Meilisearch actualiza su índice interno
        │
        ▼
Próxima búsqueda devuelve el registro actualizado ✅
```

---

## Variables de entorno necesarias

```env
# Redis (Predis)
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
QUEUE_CONNECTION=redis
CACHE_STORE=redis

# Meilisearch
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
SCOUT_QUEUE=true          ← el puente entre Redis y Meilisearch
```

---

## Redis también acelera las búsquedas

No solo la indexación — Redis puede cachear los resultados de búsquedas frecuentes para que ni siquiera lleguen a Meilisearch:

```php
// Sin cache: cada búsqueda va a Meilisearch (~30ms)
$results = User::search($query)->get();

// Con cache Redis: segunda búsqueda idéntica desde RAM (<1ms)
$results = Cache::remember("search:users:{$query}", 60, function () use ($query) {
    return User::search($query)->get();
});
```

> [!tip] ¿Cuándo cachear búsquedas?
> Solo para búsquedas frecuentes con datos que no cambian cada segundo (ej: catálogos, listas de opciones). Para búsquedas en tiempo real tipo "mientras escribo", ve directo a Meilisearch.

---

## Resumen de responsabilidades

```
┌─────────────────────────────────────────────────┐
│                   Tu Aplicación                  │
│          Controllers / Services / Jobs           │
└──────────────┬──────────────────────┬────────────┘
               │                      │
               ▼                      ▼
  ┌────────────────────┐   ┌─────────────────────┐
  │    Redis (Predis)  │   │     Meilisearch      │
  │                    │   │                      │
  │  • Queue de jobs   │──►│  • Índices de texto  │
  │  • Cache de datos  │   │  • Búsqueda full-text│
  │  • Rate limiting   │   │  • Typo tolerance    │
  │  • Sesiones        │   │  • Filtros facetados │
  └────────────────────┘   └─────────────────────┘
         │                           │
         └──────────┬────────────────┘
                    │
                    ▼
            MySQL (fuente de verdad)
```

| Herramienta | Se encarga de |
|-------------|---------------|
| **MySQL** | Fuente de verdad — todos los datos viven aquí |
| **Redis** | Velocidad — queues async, cache de resultados, sesiones |
| **Meilisearch** | Búsqueda — índices optimizados para texto, typos y filtros |

---

## Casos de uso combinados reales

### Caso 1: Búsqueda de usuarios con select dinámico

```
Admin escribe "garcia" en un input de búsqueda
        ↓
GET /api/users/options?search=garcia
        ↓
Cache::remember('search:garcia', 30s, fn() =>
    User::search('garcia')
        ->options(['filter' => 'active = true'])
        ->get()
)
        ↓
Si no está en cache → Meilisearch responde en <50ms
Si está en cache   → Redis responde en <1ms
```

### Caso 2: Actualización de perfil que refleja en búsqueda

```
Usuario actualiza su nombre
        ↓
User::update(['name' => 'Juan García'])   ← MySQL actualizado
        ↓
Scout despacha job a Redis db:0
        ↓
Cache::forget('search:garcia')            ← invalida cache vieja
        ↓
Horizon worker re-indexa en Meilisearch
        ↓
Próximas búsquedas encuentran el nombre nuevo ✅
```

---

## Ver también

- [[Backend/Laravel/Packages/Meilisearch|Meilisearch]] — instalación, Scout, `toSearchableArray()`, comandos.
- [[Backend/Laravel/Packages/Predis|Predis (Redis)]] — queues, cache, Horizon, workers.
- [[Projects/SchoolAid/SchoolAid|SchoolAid]] — ejemplo real con 6 índices Meilisearch + Redis Horizon en producción.

---

## ↩️ Navegación
- [[Backend/Laravel/Laravel|Laravel]] → [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
