 WebSockets, Predis (Redis) y Meilisearch — Guía detallada

  ---
  1. WebSockets

  ¿Qué son?

  HTTP normal es unidireccional: el cliente pide, el servidor responde, y la conexión se cierra. Para saber si algo cambió, el cliente tendría que preguntar cada X segundos (polling).

  WebSocket es un protocolo de conexión persistente y bidireccional. Una vez abierta, la conexión queda viva y el servidor puede empujar datos al cliente en cualquier momento sin que el cliente los pida.

  HTTP (polling):
    Cliente: ¿Hay algo nuevo? → Servidor: No.
    Cliente: ¿Hay algo nuevo? → Servidor: No.
    Cliente: ¿Hay algo nuevo? → Servidor: Sí, aquí está.
    [3 requests innecesarios]

  WebSocket:
    Cliente ←→ Servidor: [conexión abierta permanente]
    Servidor → Cliente: ¡Llegó algo nuevo! (cuando ocurre)
    [0 requests innecesarios]

  ---
  En este proyecto hay DOS sistemas WebSocket independientes

  A) Pusher + Laravel Echo — Para eventos del sistema

  ¿Qué hace? Notifica al frontend cuando algo cambia en el backend sin que el usuario tenga que recargar la página.

  Cómo está implementado:

  Backend (Laravel)                    Frontend (Vue)
  ─────────────────                    ──────────────
  Evento ocurre                        Echo escucha canal
  (ej: CalendarEventCreated)
          │
          ▼
  Event::dispatch()
          │
          ▼
  Laravel Broadcasting               echo.ts → getEcho()
  (ShouldBroadcast interface)   →→→  echoInstance.channel('school.5')
          │                          .listen('CalendarEventCreated', fn)
          ▼
  Pusher/Soketi Server              Frontend actualiza UI
  (WebSocket Server en :6001)    ←← automáticamente

  Configuración en el frontend (src/services/echo.ts):
  echoInstance = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,   // '4adeebb29bd54b75e1fe'
    wsHost: import.meta.env.VITE_PUSHER_SOCKET_URL,  // socket.schoolaid.app
    wsPort: 6001,
    forceTLS: false,       // usa ws:// no wss://
    enabledTransports: ['ws', 'wss']
  })

  ¿Dónde se usa en el proyecto?
  - Cola de la puerta (Gate Queue): cuando un bus llega, el frontend se actualiza en vivo mostrando qué familias están esperando
  - Bus tracking: posiciones de buses en tiempo real
  - Eventos de calendario: cuando admin crea un evento, se refleja sin recargar
  - Notificaciones en tiempo real al panel admin

  Importante: El proyecto usa Soketi como servidor WebSocket self-hosted (compatible con el protocolo Pusher), no el servicio cloud de Pusher. Esto se ve porque VITE_PUSHER_SOCKET_URL=socket.schoolaid.app
   apunta a su propio servidor.

  ---
  B) XMPP (Ejabberd) — Para el chat

  ¿Qué hace? Protocolo estándar de mensajería instantánea. Ejabberd es el servidor de chat open source que usa el proyecto.

  Frontend Vue          Ejabberd Server          Otro usuario
       │                     │                      │
       ├──WebSocket──────────►│                      │
       │  (XMPP/WS)          │◄─────WebSocket───────┤
       │                     │      (XMPP/WS)       │
       │◄────mensaje─────────┤                      │
       │                     │──────mensaje─────────►│

  Configuración:
  VITE_XMPP_WEBSOCKET_URL=wss://ejabberd.schoolaid.app:5443/ws/
  VITE_XMPP_DOMAIN=localhost
  VITE_XMPP_MUC_DOMAIN=conference.localhost  # salas de chat grupales

  ¿Por qué XMPP separado de Pusher? Porque XMPP está diseñado específicamente para mensajería: soporta presencia (online/offline), salas de chat (MUC), historial de mensajes, etc. Pusher/Echo es para
  eventos de aplicación, no para chat.

  ---
  ¿Por qué son importantes los WebSockets en este proyecto?

  Sin WebSockets, el Gate Queue (cola de entrada) requeriría que el staff recargara la página manualmente o hiciera polling cada 2 segundos. En una escuela con 500 familias llegando en 20 minutos, eso es
  crítico.

  ---
  Alternativas a Pusher/Echo + XMPP

  ┌────────────────────────────┬────────────────────────────────────────┬───────────────────────────────────┐
  │         Tecnología         │             Para qué sirve             │            Trade-offs             │
  ├────────────────────────────┼────────────────────────────────────────┼───────────────────────────────────┤
  │ Reverb (Laravel oficial)   │ Reemplaza Soketi/Pusher self-hosted    │ Oficial Laravel, pero más nuevo   │
  ├────────────────────────────┼────────────────────────────────────────┼───────────────────────────────────┤
  │ Ably                       │ Reemplaza Pusher cloud                 │ Más robusto para escala global    │
  ├────────────────────────────┼────────────────────────────────────────┼───────────────────────────────────┤
  │ SSE (Server-Sent Events)   │ Alternativa ligera, solo server→client │ No es bidireccional               │
  ├────────────────────────────┼────────────────────────────────────────┼───────────────────────────────────┤
  │ Polling con TanStack Query │ La más simple                          │ Añade latencia, carga el servidor │
  ├────────────────────────────┼────────────────────────────────────────┼───────────────────────────────────┤
  │ Socket.io                  │ WebSocket general                      │ No integrado con Echo             │
  ├────────────────────────────┼────────────────────────────────────────┼───────────────────────────────────┤
  │ Matrix                     │ Alternativa a XMPP para chat           │ Más moderno, pero más complejo    │
  └────────────────────────────┴────────────────────────────────────────┴───────────────────────────────────┘

  ---
  2. Predis (Redis)

  ¿Qué es Redis?

  Redis es una base de datos en memoria (RAM). Es extremadamente rápido porque no escribe a disco en cada operación — los datos viven en RAM y se persisten en disco de forma asíncrona.

  La analogía: MySQL es como un archivero físico (rápido pero requiere abrir cajones). Redis es como tu memoria RAM — acceso instantáneo, pero con capacidad limitada y precio mayor.

  Predis es simplemente el cliente PHP que Laravel usa para hablar con Redis. Es la librería que traduce los comandos PHP a comandos que Redis entiende.

  ---
  Cómo está configurado en este proyecto

  // config/database.php
  'redis' => [
      'client' => 'predis',     // ← el cliente PHP

      'default' => [            // Para Queues y sesiones
          'host' => '127.0.0.1',
          'port' => '6379',
          'database' => '0',   // DB 0 = Queues
      ],

      'cache' => [              // Para Cache
          'host' => '127.0.0.1',
          'port' => '6379',
          'database' => '1',   // DB 1 = Cache (separada)
      ],
  ]

  Se usan dos bases de datos Redis separadas (db:0 y db:1) en el mismo servidor para aislar los datos. Redis soporta hasta 16 bases de datos (0-15) en una instancia.

  ---
  Para qué se usa Redis en el proyecto

  Uso 1: Queue System (db:0) — El más crítico

  Cuando un padre aprueba un permiso de un estudiante, el sistema necesita:
  3. Enviar email de confirmación
  4. Enviar push notification al app móvil
  5. Posiblemente enviar SMS/WhatsApp
  6. Actualizar el índice de Meilisearch

  Si todo eso ocurriera sincrónicamente en el request HTTP, el usuario esperaría 3-5 segundos. Con Redis + Queues:

  Request HTTP: "Aprueba permiso" → guarda en DB → responde en <100ms → ✓
                                         ↓
                                Despacha Job a Redis (db:0)
                                         ↓
                      Horizon Worker (proceso PHP en background)
                      lee de Redis y ejecuta:
                        ├── SendPermissionEmailJob
                        ├── SendPermissionPushJob
                        └── UpdateMeilisearchIndex

  El usuario ya recibió su respuesta. Los workers hacen el trabajo pesado en paralelo.

  Queues configuradas:
  "emails"        → jobs de envío de correo
  "notifications" → push notifications, SMS, WhatsApp
  "default"       → todo lo demás (indexación Meilisearch, QR generation, etc.)

  Horizon es el panel de control (dashboard web) de los workers Redis. Permite ver qué jobs están corriendo, cuáles fallaron, throughput, etc.

  Uso 2: Cache (db:1)

  Datos que son costosos de calcular y no cambian frecuentemente se guardan en Redis:

  // Sin cache: consulta MySQL cada vez (lento)
  $config = School::find($id)->with('parameters')->get();

  // Con cache Redis: consulta MySQL una vez, guarda en RAM
  $config = Cache::remember("school:config:{$id}", 3600, function() {
      return School::find($id)->with('parameters')->get();
  });
  // Próximas 3600 segundos → Redis responde en <1ms en vez de ~50ms MySQL

  En el proyecto esto aplica para: configuración de escuelas, parámetros del sistema, listas de opciones que no cambian frecuentemente.

  ---
  ¿Por qué es importante Redis aquí?

  Sin Redis/Queues, con 50 permisos aprobados simultáneamente (fin de día escolar), el servidor estaría intentando enviar 150+ emails/push notifications sincrónicamente. El sistema colapsaría o daría
  timeouts.

  Con Redis:
  - Los jobs se encolan en microsegundos
  - Los workers los procesan en paralelo y a su ritmo
  - Si un job falla, Horizon lo reintenta automáticamente
  - Si el sistema de email cae, los jobs esperan en la cola hasta que vuelva

  ---
  Alternativas a Redis/Predis

  ┌────────────────────────┬──────────────┬─────────────────────────────────────────────────────────────┐
  │       Tecnología       │     Tipo     │                         Trade-offs                          │
  ├────────────────────────┼──────────────┼─────────────────────────────────────────────────────────────┤
  │ Memcached              │ Solo cache   │ No tiene queues, no persiste datos                          │
  ├────────────────────────┼──────────────┼─────────────────────────────────────────────────────────────┤
  │ Database queue (MySQL) │ Queues       │ Más lento, sobrecarga la DB principal                       │
  ├────────────────────────┼──────────────┼─────────────────────────────────────────────────────────────┤
  │ SQS (Amazon)           │ Queues cloud │ Robusto y managed, pero costo y vendor lock-in              │
  ├────────────────────────┼──────────────┼─────────────────────────────────────────────────────────────┤
  │ RabbitMQ               │ Queues       │ Más potente para mensajería compleja, más difícil de operar │
  ├────────────────────────┼──────────────┼─────────────────────────────────────────────────────────────┤
  │ Beanstalkd             │ Queues       │ Simple, pero sin cache ni ecosistema                        │
  ├────────────────────────┼──────────────┼─────────────────────────────────────────────────────────────┤
  │ Valkey                 │ Cache+Queue  │ Fork open source de Redis (Redis cambió licencia en 2024)   │
  ├────────────────────────┼──────────────┼─────────────────────────────────────────────────────────────┤
  │ DragonflyDB            │ Cache+Queue  │ Compatible con Redis pero más rápido                        │
  └────────────────────────┴──────────────┴─────────────────────────────────────────────────────────────┘

  Para este proyecto: Redis es la elección correcta porque sirve tanto de cache como de queue con una sola tecnología, tiene soporte nativo en Laravel, y Horizon le da visibilidad operacional.

  ---
  3. Meilisearch

  ¿Qué es?

  Meilisearch es un motor de búsqueda especializado — una aplicación separada que corre en el puerto 7700 y guarda sus propios índices optimizados para búsqueda de texto.

  ¿Por qué no simplemente usar WHERE name LIKE '%garcia%' en MySQL?

  -- MySQL LIKE:
  SELECT * FROM user WHERE last_name LIKE '%garcia%'
  -- Problemas:
  -- 1. No encuentra "García" si buscas "garcia" (acentos)
  -- 2. No encuentra "garcia rodriguez" si buscas "rodriguez garcia"
  -- 3. Lento sin índices, LIKE '%...%' no usa índices B-tree
  -- 4. No rankea por relevancia (¿cuál resultado es más relevante?)
  -- 5. No busca en múltiples tablas relacionadas fácilmente

  -- Meilisearch:
  Busca "garcia" → encuentra: García, GARCIA, garcia rodriguez, juan garcia
  -- Ventajas:
  -- 1. Typo tolerance: "gracias" → sugiere "garcia"
  -- 2. Búsqueda en campos relacionados: busca en estudiantes DE la familia
  -- 3. Filtros facetados: school_id=5 AND has_students=true
  -- 4. Ranking de relevancia automático
  -- 5. Respuesta en <50ms aunque haya 100,000 registros

  ---
  Cómo está implementado en el proyecto

  Paso 1: Los modelos se "marcan" como buscables

  // app/Models/User.php
  use Laravel\Scout\Searchable;

  class User extends Model {
      use Searchable;  // ← esto es todo lo que necesita el modelo

      // Define qué datos van al índice de Meilisearch
      public function toSearchableArray(): array {
          $this->loadMissing(['students', 'userEmails', 'school']);

          return [
              'id'             => $this->id,
              'first_name'     => $this->first_name,
              'last_name'      => $this->last_name,
              'full_name'      => $this->full_name,       // "Juan García"
              'school_id'      => $this->school_id,       // para filtrar
              'has_students'   => $this->students->isNotEmpty(),  // filtro boolean
              'student_names'  => $this->students->map(fn($s) => $s->full_name)->toArray(),
              // ↑ buscar una familia por el nombre de su hijo
          ];
      }

      // Solo indexar usuarios activos y no eliminados
      public function shouldBeSearchable(): bool {
          return $this->active && !$this->trashed();
      }
  }

  Paso 2: Meilisearch sabe qué atributos permiten filtros

  // config/scout.php — configuración del índice 'user'
  'user' => [
      'filterableAttributes' => [
          'school_id',      // WHERE school_id = 5
          'has_students',   // WHERE has_students = true
          'active',
      ],
      'searchableAttributes' => [
          'first_name', 'last_name', 'full_name',
          'student_names',  // búsqueda por hijo
          'user_email_addresses',
      ],
      'sortableAttributes' => [
          'last_name', 'family_code', 'students_count',
      ],
  ]

  Paso 3: Indexación automática (vía Redis Queue)

  Usuario se guarda/actualiza en MySQL
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

  Paso 4: Búsqueda en tiempo real

  // FamilyResource::optionsSearch()
  User::search('garcia')           // texto a buscar
      ->take(50)                   // límite
      ->options([
          'filter' => 'school_id = 5 AND has_students = true'
      ])
      ->get();   // retorna modelos Eloquent hidratados

  [Frontend escribe "garcia" en el input]
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

  ---
  Los 6 índices del proyecto

  ┌──────────────┬──────────────────────────────────────────────────────────────────────┐
  │    Índice    │                             Casos de uso                             │
  ├──────────────┼──────────────────────────────────────────────────────────────────────┤
  │ user         │ Buscar familias por apellido, email, nombre de hijo, código familiar │
  ├──────────────┼──────────────────────────────────────────────────────────────────────┤
  │ student      │ Buscar estudiantes por nombre, carnet, grado, bus asignado           │
  ├──────────────┼──────────────────────────────────────────────────────────────────────┤
  │ stop         │ Buscar paradas por nombre, ruta, zona, ciudad                        │
  ├──────────────┼──────────────────────────────────────────────────────────────────────┤
  │ route        │ Buscar rutas por nombre, bus, horario                                │
  ├──────────────┼──────────────────────────────────────────────────────────────────────┤
  │ bus          │ Buscar buses por nombre, placa, conductor                            │
  ├──────────────┼──────────────────────────────────────────────────────────────────────┤
  │ bus_schedule │ Buscar horarios de bus                                               │
  └──────────────┴──────────────────────────────────────────────────────────────────────┘

  ---
  ¿Por qué es importante Meilisearch aquí?

  Una escuela puede tener 2,000+ familias. Cuando un admin busca "garcía" en un select para asignar un permiso, necesita resultados en menos de 200ms. Con MySQL LIKE sería lento y no encontraría
  variaciones de acentos. Con Meilisearch, la búsqueda es instantánea, tolerante a typos, y puede buscar simultáneamente en el nombre de la familia Y en los nombres de sus hijos.

  ---
  Alternativas a Meilisearch

  ┌─────────────────────────────────┬───────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────┐
  │           Tecnología            │               Tipo                │                                  Trade-offs                                  │
  ├─────────────────────────────────┼───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
  │ Typesense                       │ Open source self-hosted           │ Muy similar a Meilisearch, ya está configurado en scout.php como alternativa │
  ├─────────────────────────────────┼───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
  │ Algolia                         │ Cloud managed                     │ El más maduro, también en scout.php, pero de pago ($$$)                      │
  ├─────────────────────────────────┼───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
  │ Elasticsearch                   │ Open source self-hosted           │ El más potente, pero muy complejo de operar                                  │
  ├─────────────────────────────────┼───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
  │ OpenSearch                      │ Fork open source de Elasticsearch │ Similar a Elastic, más amigable                                              │
  ├─────────────────────────────────┼───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
  │ MySQL FULLTEXT                  │ Dentro de MySQL                   │ Sin infraestructura extra, pero sin typo tolerance ni filtros facetados      │
  ├─────────────────────────────────┼───────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
  │ Laravel Scout + Database driver │ Solo PHP/MySQL                    │ Zero infraestructura, pero solo LIKE básico                                  │
  └─────────────────────────────────┴───────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────┘

  Nota importante: El proyecto ya tiene Typesense configurado en config/scout.php (aunque comentado). Si quisieras migrar de Meilisearch a Typesense, solo cambiarías SCOUT_DRIVER=typesense en el .env y
  ajustarías la configuración de índices.

  ---
  Resumen: ¿Qué facilita cada uno?

  ┌──────────────────────────┬───────────────────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────┐
  │        Tecnología        │                                 Sin ella                                  │                              Con ella                               │
  ├──────────────────────────┼───────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
  │ WebSockets (Echo/Pusher) │ Staff recarga página para ver si llegó nueva familia a la puerta          │ Pantalla se actualiza sola en tiempo real                           │
  ├──────────────────────────┼───────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
  │ WebSockets (XMPP)        │ Chat no existe o requiere polling cada segundo                            │ Mensajería instantánea real                                         │
  ├──────────────────────────┼───────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
  │ Redis (Queues)           │ Aprobar permiso tarda 5 segundos (email + push + indexación)              │ Respuesta en 80ms, el resto ocurre en background                    │
  ├──────────────────────────┼───────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
  │ Redis (Cache)            │ Cada request consulta MySQL para obtener config de la escuela             │ Configuración en RAM, respuesta en <1ms                             │
  ├──────────────────────────┼───────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
  │ Meilisearch              │ Buscar "garcia" en 2000 familias tarda 2 segundos y no encuentra "García" │ Resultados en <50ms con tolerancia a typos y búsqueda en relaciones │
  └──────────────────────────┴───────────────────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────┘

✻ Baked for 2m 55s

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
❯
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  ? for shortcuts                                                                                                                                          Update available! Run: brew upgrade claude-code

