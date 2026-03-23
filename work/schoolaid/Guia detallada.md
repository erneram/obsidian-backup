---
title: SchoolAid – WebSockets, Redis y Meilisearch
tags: [schoolaid, websockets, redis, meilisearch, project]
---

# 🔧 WebSockets, Redis y Meilisearch — Guía Detallada

---

## 1. WebSockets

### ¿Qué son?

HTTP normal es **unidireccional**: el cliente pide, el servidor responde, y la conexión se cierra. Para saber si algo cambió, el cliente tendría que preguntar cada X segundos (polling).

**WebSocket** es un protocolo de conexión **persistente y bidireccional**. Una vez abierta, la conexión queda viva y el servidor puede empujar datos al cliente en cualquier momento.

| Modelo | Flujo | Resultado |
|--------|-------|-----------|
| **HTTP (polling)** | Cliente pregunta → No → Cliente pregunta → No → Sí | 3 requests innecesarios |
| **WebSocket** | Conexión abierta permanente → Servidor empuja cuando ocurre | 0 requests innecesarios |

---

### Los DOS sistemas WebSocket del proyecto

> [!info] Dos WebSockets independientes
> El proyecto usa **Pusher/Echo** para eventos del sistema y **XMPP/Ejabberd** para chat. Son tecnologías distintas con propósitos distintos.

---

#### A) Pusher + Laravel Echo — Para eventos del sistema

**¿Qué hace?** Notifica al frontend cuando algo cambia en el backend sin que el usuario tenga que recargar la página.

**Flujo de implementación:**

```
Backend (Laravel)                         Frontend (Vue)
─────────────────                         ──────────────
Evento ocurre
(ej: CalendarEventCreated)
        │
        ▼
Event::dispatch()
        │
        ▼
Laravel Broadcasting              echo.ts → getEcho()
(ShouldBroadcast interface)  →→→  echoInstance
        │                           .channel('school.5')
        ▼                           .listen('CalendarEventCreated', fn)
Pusher/Soketi Server
(WebSocket Server :6001)      ←←←  Frontend actualiza UI automáticamente
```

**Configuración en el frontend** (`src/services/echo.ts`):

```ts
echoInstance = new Echo({
  broadcaster: 'pusher',
  key: import.meta.env.VITE_PUSHER_APP_KEY,      // '4adeebb29bd54b75e1fe'
  wsHost: import.meta.env.VITE_PUSHER_SOCKET_URL, // socket.schoolaid.app
  wsPort: 6001,
  forceTLS: false,         // usa ws:// no wss://
  enabledTransports: ['ws', 'wss']
})
```

> [!tip] Soketi, no Pusher Cloud
> El proyecto usa **Soketi** como servidor WebSocket self-hosted (compatible con protocolo Pusher), NO el servicio cloud de Pusher. Se nota porque `VITE_PUSHER_SOCKET_URL=socket.schoolaid.app` apunta al propio servidor.

**¿Dónde se usa?**
- **Gate Queue** — cuando un bus llega, el frontend muestra en vivo qué familias están esperando.
- **Bus tracking** — posiciones de buses en tiempo real.
- **Eventos de calendario** — cuando admin crea un evento, se refleja sin recargar.
- **Notificaciones** — alertas en tiempo real al panel admin.

---

#### B) XMPP (Ejabberd) — Para el chat

**¿Qué hace?** Protocolo estándar de mensajería instantánea. Ejabberd es el servidor de chat open source que usa el proyecto.

```
Frontend Vue       Ejabberd Server        Otro usuario
     │                   │                     │
     ├── WebSocket ──────►│                     │
     │   (XMPP/WS)        │◄──── WebSocket ─────┤
     │                    │      (XMPP/WS)      │
     │◄── mensaje ────────┤                     │
     │                    │───── mensaje ───────►│
```

**Variables de entorno:**

```env
VITE_XMPP_WEBSOCKET_URL=wss://ejabberd.schoolaid.app:5443/ws/
VITE_XMPP_DOMAIN=localhost
VITE_XMPP_MUC_DOMAIN=conference.localhost   # salas de chat grupales
```

> [!question] ¿Por qué XMPP separado de Pusher?
> XMPP está diseñado específicamente para mensajería: soporta **presencia** (online/offline), **salas de chat** (MUC), historial de mensajes, etc. Pusher/Echo es para eventos de aplicación, no para chat.

---

### ¿Por qué son críticos los WebSockets en este proyecto?

> [!warning] Caso crítico: Gate Queue
> Sin WebSockets, el Gate Queue requeriría que el staff recargara la página manualmente o hiciera polling cada 2 segundos. En una escuela con **500 familias llegando en 20 minutos**, eso es inaceptable.

---

### Alternativas a Pusher/Echo + XMPP

| Tecnología | Para qué sirve | Trade-offs |
|-----------|----------------|------------|
| **Reverb** (Laravel oficial) | Reemplaza Soketi/Pusher self-hosted | Oficial Laravel, pero más nuevo |
| **Ably** | Reemplaza Pusher cloud | Más robusto para escala global |
| **SSE** (Server-Sent Events) | Alternativa ligera, solo server→client | No es bidireccional |
| **Polling con TanStack Query** | La más simple | Añade latencia, carga el servidor |
| **Socket.io** | WebSocket general | No integrado con Echo |
| **Matrix** | Alternativa a XMPP para chat | Más moderno, pero más complejo |

---

## 2. Redis (Predis)

### ¿Qué es Redis?

Redis es una **base de datos en memoria (RAM)**. Es extremadamente rápido porque no escribe a disco en cada operación.

> [!example] Analogía
> MySQL es como un archivero físico — rápido pero requiere abrir cajones.
> Redis es como tu memoria de trabajo — acceso instantáneo, capacidad limitada.

**Predis** es el cliente PHP que Laravel usa para hablar con Redis — traduce comandos PHP a comandos Redis.

---

### Configuración en el proyecto

```php
// config/database.php
'redis' => [
    'client' => 'predis',

    'default' => [            // Para Queues y sesiones
        'host'     => '127.0.0.1',
        'port'     => '6379',
        'database' => '0',   // DB 0 = Queues
    ],

    'cache' => [              // Para Cache
        'host'     => '127.0.0.1',
        'port'     => '6379',
        'database' => '1',   // DB 1 = Cache (separada)
    ],
]
```

> [!info] Dos bases de datos Redis
> Se usan `db:0` y `db:1` en la misma instancia para aislar datos. Redis soporta hasta 16 bases de datos (0–15) por instancia.

---

### Uso 1: Queue System (db:0) — El más crítico

**Escenario:** Un padre aprueba el permiso de un estudiante. El sistema necesita:
1. Enviar email de confirmación
2. Enviar push notification
3. Posiblemente SMS/WhatsApp
4. Actualizar índice de Meilisearch

Sin queues, el usuario esperaría **3–5 segundos**. Con Redis + Queues:

```
Request HTTP: "Aprueba permiso"
  → Guarda en DB
  → Responde en <100ms  ✓
  → Despacha Job a Redis (db:0)
        ↓
  Horizon Worker (proceso PHP en background):
    ├── SendPermissionEmailJob
    ├── SendPermissionPushJob
    └── UpdateMeilisearchIndex
```

**Queues configuradas:**

| Queue | Propósito |
|-------|-----------|
| `emails` | Envío de correos |
| `notifications` | Push notifications, SMS, WhatsApp |
| `default` | Indexación Meilisearch, generación QR, etc. |

> [!tip] Horizon
> Horizon es el **dashboard web** de los workers Redis. Permite ver qué jobs corren, cuáles fallaron, throughput en tiempo real, etc.

---

### Uso 2: Cache (db:1)

```php
// Sin cache: consulta MySQL en cada request (~50ms)
$config = School::find($id)->with('parameters')->get();

// Con cache Redis: consulta MySQL una sola vez, luego RAM (<1ms)
$config = Cache::remember("school:config:{$id}", 3600, function() {
    return School::find($id)->with('parameters')->get();
});
// Las próximas 3600 segundos → Redis responde en <1ms
```

**Aplica para:** configuración de escuelas, parámetros del sistema, listas de opciones estáticas.

---

### ¿Por qué es crítico Redis aquí?

> [!warning] Escenario sin Redis
> Con 50 permisos aprobados simultáneamente (fin del día escolar), el servidor estaría intentando enviar **150+ emails/push notifications sincrónicamente**. El sistema colapsaría.

**Con Redis:**
- Los jobs se encolan en microsegundos.
- Los workers los procesan en paralelo a su ritmo.
- Si un job falla, Horizon lo reintenta automáticamente.
- Si el servicio de email cae, los jobs esperan en cola hasta que vuelva.

---

### Alternativas a Redis/Predis

| Tecnología | Tipo | Trade-offs |
|-----------|------|------------|
| **Memcached** | Solo cache | No tiene queues, no persiste datos |
| **Database queue (MySQL)** | Queues | Más lento, sobrecarga la DB principal |
| **SQS (Amazon)** | Queues cloud | Robusto y managed, pero costo y vendor lock-in |
| **RabbitMQ** | Queues | Más potente para mensajería compleja, más difícil de operar |
| **Beanstalkd** | Queues | Simple, pero sin cache ni ecosistema |
| **Valkey** | Cache + Queue | Fork open source de Redis (cambió licencia en 2024) |
| **DragonflyDB** | Cache + Queue | Compatible con Redis pero más rápido |

> [!success] ¿Por qué Redis es la elección correcta aquí?
> Sirve tanto de **cache** como de **queue** con una sola tecnología, tiene soporte nativo en Laravel, y Horizon le da visibilidad operacional completa.

---

## 3. Meilisearch

### ¿Qué es?

Meilisearch es un **motor de búsqueda especializado** — una aplicación separada que corre en el puerto `7700` con sus propios índices optimizados para búsqueda de texto.

### ¿Por qué no usar `WHERE name LIKE '%garcia%'` en MySQL?

```sql
-- MySQL LIKE — problemas:
SELECT * FROM user WHERE last_name LIKE '%garcia%'
-- ❌ No encuentra "García" si buscas "garcia" (acentos)
-- ❌ No encuentra "garcia rodriguez" si buscas "rodriguez garcia"
-- ❌ Lento: LIKE '%...%' no usa índices B-tree
-- ❌ No rankea por relevancia
-- ❌ No busca en múltiples tablas relacionadas fácilmente
```

```
-- Meilisearch:
Busca "garcia" → encuentra: García, GARCIA, garcia rodriguez, juan garcia
✅ Typo tolerance: "gracias" → sugiere "garcia"
✅ Búsqueda en relaciones: busca en estudiantes DE la familia
✅ Filtros facetados: school_id=5 AND has_students=true
✅ Ranking de relevancia automático
✅ Respuesta en <50ms con 100,000 registros
```

---

### Implementación paso a paso

#### Paso 1: Marcar modelos como buscables

```php
// app/Models/User.php
use Laravel\Scout\Searchable;

class User extends Model {
    use Searchable;  // ← esto es todo lo que necesita el modelo

    public function toSearchableArray(): array {
        $this->loadMissing(['students', 'userEmails', 'school']);

        return [
            'id'            => $this->id,
            'first_name'    => $this->first_name,
            'last_name'     => $this->last_name,
            'full_name'     => $this->full_name,       // "Juan García"
            'school_id'     => $this->school_id,       // para filtrar
            'has_students'  => $this->students->isNotEmpty(),
            'student_names' => $this->students->map(fn($s) => $s->full_name)->toArray(),
            //                  ↑ buscar una familia por el nombre de su hijo
        ];
    }

    public function shouldBeSearchable(): bool {
        return $this->active && !$this->trashed();
    }
}
```

#### Paso 2: Definir atributos filtrables

```php
// config/scout.php — configuración del índice 'user'
'user' => [
    'filterableAttributes' => [
        'school_id',      // WHERE school_id = 5
        'has_students',   // WHERE has_students = true
        'active',
    ],
    'searchableAttributes' => [
        'first_name', 'last_name', 'full_name',
        'student_names',        // búsqueda por nombre de hijo
        'user_email_addresses',
    ],
    'sortableAttributes' => [
        'last_name', 'family_code', 'students_count',
    ],
]
```

#### Paso 3: Indexación automática (vía Redis Queue)

```
Usuario guardado/actualizado en MySQL
        ↓
Scout detecta el cambio (Observer interno)
        ↓
SCOUT_QUEUE=true → despacha job a Redis
        ↓
Worker ejecuta: User::toSearchableArray()
        ↓
Envía datos a Meilisearch HTTP API (localhost:7700)
        ↓
Meilisearch actualiza su índice interno
```

#### Paso 4: Búsqueda en tiempo real

```php
// FamilyResource::optionsSearch()
User::search('garcia')
    ->take(50)
    ->options(['filter' => 'school_id = 5 AND has_students = true'])
    ->get();   // retorna modelos Eloquent hidratados
```

**Flujo completo frontend → resultado:**

```
[Usuario escribe "garcia" en el input]
        ↓
GET /nadota-api/family/resource/options?search=garcia&school_id=5
        ↓
FamilyResource::optionsSearch() → User::search('garcia')
        ↓
Laravel Scout → Meilisearch en :7700
        ↓
Meilisearch busca en su índice optimizado
        ↓
Retorna IDs relevantes: [45, 123, 67]
        ↓
Scout hidrata: User::whereIn('id', [45, 123, 67])->get()
        ↓
[Frontend muestra resultados en <100ms]
```

---

### Los 6 índices del proyecto

| Índice | Casos de uso |
|--------|-------------|
| `user` | Buscar familias por apellido, email, nombre de hijo, código familiar |
| `student` | Buscar estudiantes por nombre, carnet, grado, bus asignado |
| `stop` | Buscar paradas por nombre, ruta, zona, ciudad |
| `route` | Buscar rutas por nombre, bus, horario |
| `bus` | Buscar buses por nombre, placa, conductor |
| `bus_schedule` | Buscar horarios de bus |

---

### ¿Por qué es crítico Meilisearch aquí?

> [!warning] Escenario real
> Una escuela tiene **2,000+ familias**. Cuando un admin busca "garcía" para asignar un permiso, necesita resultados en menos de 200ms. Con MySQL LIKE sería lento y no encontraría variaciones de acentos.

Con Meilisearch: búsqueda instantánea, tolerante a typos, busca simultáneamente en el nombre de la familia **y** en los nombres de sus hijos.

---

### Alternativas a Meilisearch

| Tecnología | Tipo | Trade-offs |
|-----------|------|------------|
| **Typesense** | Open source self-hosted | Muy similar a Meilisearch — ya configurado en `scout.php` como alternativa |
| **Algolia** | Cloud managed | El más maduro, también en `scout.php`, pero de pago ($$$) |
| **Elasticsearch** | Open source self-hosted | El más potente, pero muy complejo de operar |
| **OpenSearch** | Fork de Elasticsearch | Similar a Elastic, más amigable |
| **MySQL FULLTEXT** | Dentro de MySQL | Sin infraestructura extra, pero sin typo tolerance ni filtros facetados |
| **Scout + Database driver** | Solo PHP/MySQL | Zero infraestructura, pero solo LIKE básico |

> [!tip] Migrar a Typesense
> El proyecto ya tiene Typesense configurado en `config/scout.php` (comentado). Para migrar: cambia `SCOUT_DRIVER=typesense` en el `.env` y ajusta la configuración de índices.

---

## 4. Resumen: ¿Qué facilita cada tecnología?

| Tecnología | Sin ella | Con ella |
|-----------|----------|----------|
| **WebSockets (Echo/Pusher)** | Staff recarga página para ver si llegó una familia | Pantalla se actualiza sola en tiempo real |
| **WebSockets (XMPP)** | Chat requiere polling cada segundo | Mensajería instantánea real |
| **Redis (Queues)** | Aprobar permiso tarda 5 segundos | Respuesta en 80ms, el resto en background |
| **Redis (Cache)** | Cada request consulta MySQL para config de escuela | Configuración en RAM, respuesta en <1ms |
| **Meilisearch** | Buscar "garcia" en 2,000 familias tarda 2 seg y no encuentra "García" | Resultados en <50ms con typo tolerance y búsqueda en relaciones |

---

## ↩️ Navegación
- [[Projects/SchoolAid/SchoolAid|🚌 SchoolAid]] → [[Projects/Projects|🚀 Projects]] → [[HOME|🏠 HOME]]
