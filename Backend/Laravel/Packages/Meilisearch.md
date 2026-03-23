---
title: Meilisearch + Laravel Scout
tags: [laravel, meilisearch, search, scout]
---

# 🔍 Meilisearch + Laravel Scout

Motor de búsqueda especializado que corre como servicio separado en el puerto `7700`. Se integra con Laravel a través del paquete **Laravel Scout**.

---

## ¿Por qué no usar `LIKE '%...%'` en MySQL?

```sql
-- MySQL LIKE — problemas:
SELECT * FROM users WHERE last_name LIKE '%garcia%'
-- ❌ No encuentra "García" si buscas "garcia" (acentos)
-- ❌ No encuentra "garcia rodriguez" si buscas "rodriguez garcia"
-- ❌ Lento: LIKE '%...%' no puede usar índices B-tree
-- ❌ No rankea resultados por relevancia
-- ❌ No busca fácilmente en múltiples tablas relacionadas
```

**Meilisearch resuelve todo esto:**
- ✅ Typo tolerance — `"gracias"` → encuentra `"garcia"`
- ✅ Búsqueda multi-campo e incluso en relaciones
- ✅ Filtros facetados — `school_id=5 AND active=true`
- ✅ Ranking de relevancia automático
- ✅ Respuesta en `<50ms` con 100,000+ registros

---

## Instalación

```bash
composer require laravel/scout
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"

# Driver de Meilisearch
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle
```

```env
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=your_master_key   # opcional en local
SCOUT_QUEUE=true                  # indexación async via Redis
```

> [!tip] `SCOUT_QUEUE=true`
> Con esto activado, la indexación se despacha como un Job a la cola de Redis en vez de hacerse sincrónicamente. Esencial en producción para no bloquear el request HTTP. Ver [[Backend/Laravel/Packages/Predis|Predis]] para la configuración de queues.

---

## Implementación

### 1. Marcar el modelo como buscable

```php
use Laravel\Scout\Searchable;

class User extends Model
{
    use Searchable;

    // Define qué campos van al índice de Meilisearch
    public function toSearchableArray(): array
    {
        $this->loadMissing(['roles', 'profile']);

        return [
            'id'         => $this->id,
            'name'       => $this->name,
            'email'      => $this->email,
            'role'       => $this->roles->pluck('name')->toArray(),
            'created_at' => $this->created_at->timestamp,
        ];
    }

    // Condición para indexar (opcional)
    public function shouldBeSearchable(): bool
    {
        return $this->active && !$this->trashed();
    }
}
```

### 2. Configurar atributos del índice

```php
// config/scout.php
'meilisearch' => [
    'host' => env('MEILISEARCH_HOST', 'http://localhost:7700'),
    'key'  => env('MEILISEARCH_KEY'),
    'index-settings' => [
        'users' => [
            'filterableAttributes'  => ['role', 'active', 'created_at'],
            'searchableAttributes'  => ['name', 'email'],
            'sortableAttributes'    => ['name', 'created_at'],
        ],
    ],
],
```

### 3. Buscar

```php
// Búsqueda simple
User::search('garcia')->get();

// Con filtros y límite
User::search('garcia')
    ->take(20)
    ->options(['filter' => 'active = true'])
    ->get();   // retorna colección de Eloquent Models

// Solo IDs (más rápido)
User::search('garcia')->keys();
```

---

## Ciclo de indexación

```
Model::save() / Model::update()
        ↓
Scout detecta el cambio (Observer interno)
        ↓
SCOUT_QUEUE=true → despacha Job a Redis (async)
        ↓
Queue Worker ejecuta toSearchableArray()
        ↓
Envía datos a Meilisearch HTTP API (:7700)
        ↓
Meilisearch actualiza su índice interno
```

---

## Comandos útiles de Artisan

```bash
# Indexar todos los registros de un modelo
php artisan scout:import "App\Models\User"

# Limpiar el índice
php artisan scout:flush "App\Models\User"

# Ver índices en Meilisearch (requiere meilisearch-php)
php artisan scout:index users
```

> [!warning] Índice no encontrado
> Si aparece el error `"Index users not found"`, ejecuta `php artisan scout:import` para crear e indexar desde cero.

---

## Alternativas

| Tecnología | Tipo | Trade-offs |
|-----------|------|------------|
| **Typesense** | Open source self-hosted | Muy similar, más rápido en algunos casos |
| **Algolia** | Cloud managed | El más maduro, pero de pago ($$$) |
| **Elasticsearch** | Open source self-hosted | El más potente, muy complejo de operar |
| **OpenSearch** | Fork de Elasticsearch | Similar a Elastic, más amigable |
| **MySQL FULLTEXT** | Dentro de MySQL | Sin infraestructura extra, pero sin typo tolerance |
| **Scout + Database driver** | Solo PHP | Zero infraestructura, solo LIKE básico |

---

## ↩️ Navegación
- [[Backend/Laravel/Packages/Meilisearch + Predis|🔗 Cómo se complementan]] · [[Backend/Laravel/Laravel|Laravel]] → [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
