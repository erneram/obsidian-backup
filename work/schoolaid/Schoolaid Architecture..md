 Arquitectura del Sistema SchoolAid

  Visión General — Los 3 Proyectos

  ┌─────────────────────────────────────────────────────────────────────┐
  │  schoolaid-frontend (Vue 3 + TS)                                    │
  │  SPA que consume API REST y endpoints dinámicos de Nadota           │
  └─────────────────────────┬───────────────────────────────────────────┘
                            │ HTTP (Axios) + WebSocket (Pusher/Echo)
  ┌─────────────────────────▼───────────────────────────────────────────┐
  │  schoolaid-admin (Laravel 12)                                       │
  │  API principal + lógica de negocio + multi-tenancy por escuela      │
  │         ├── /api/* → Endpoints específicos por dominio              │
  │         └── /nadota-api/* → CRUD genérico via paquete Nadota        │
  └─────────────────────────┬───────────────────────────────────────────┘
                            │ usa como dependencia local
  ┌─────────────────────────▼───────────────────────────────────────────┐
  │  nadota (Paquete Laravel)                                           │
  │  Framework de admin basado en Resources — similar a Laravel Nova    │
  └─────────────────────────────────────────────────────────────────────┘

  ---
  1. schoolaid-admin — Backend Laravel 12

  Arquitectura en capas

  Models → Repositories → Services → Controllers (API) → API Resources
                                          ↑
                                Form Requests (validación)

  ┌──────────────┬──────────────────────────────────────────┬──────────┐
  │     Capa     │                Propósito                 │ Cantidad │
  ├──────────────┼──────────────────────────────────────────┼──────────┤
  │ Models       │ Eloquent + multi-tenancy via SchoolScope │ 200+     │
  ├──────────────┼──────────────────────────────────────────┼──────────┤
  │ Repositories │ Abstracción de acceso a datos            │ 50+      │
  ├──────────────┼──────────────────────────────────────────┼──────────┤
  │ Services     │ Lógica de negocio por dominio            │ 25+      │
  ├──────────────┼──────────────────────────────────────────┼──────────┤
  │ Controllers  │ Endpoints API REST                       │ 82+      │
  ├──────────────┼──────────────────────────────────────────┼──────────┤
  │ Policies     │ Autorización granular                    │ 137      │
  ├──────────────┼──────────────────────────────────────────┼──────────┤
  │ Jobs         │ Procesamiento asíncrono                  │ 26       │
  ├──────────────┼──────────────────────────────────────────┼──────────┤
  │ Events       │ Eventos de dominio                       │ 23       │
  ├──────────────┼──────────────────────────────────────────┼──────────┤
  │ Listeners    │ Reacción a eventos (queued)              │ 17       │
  ├──────────────┼──────────────────────────────────────────┼──────────┤
  │ Enums        │ Type-safe enums                          │ 33       │
  └──────────────┴──────────────────────────────────────────┴──────────┘

  Patrones usados: Repository, Service Layer, Event-Driven, CQRS parcial, Multi-tenancy con Global Scopes, Policy-based Authorization.

  ---
  2. nadota — Paquete de Admin Genérico

  Es el Laravel Nova interno del proyecto. Define recursos una vez y expone automáticamente CRUD + filtros + acciones + búsqueda + exportación.

  Pipeline de Index (el corazón del paquete)

  Request
    → BuildQueryPipe       (query base + eager loading)
    → ApplySearchPipe      (búsqueda en atributos y relaciones)
    → ApplyFiltersPipe     (filtros declarativos)
    → ApplySortingPipe     (ordenamiento)
    → PaginateAndTransformPipe  (paginación + transformación)
    → IndexResource (respuesta JSON)

  Endpoints generados automáticamente por cada Resource

  GET    /nadota-api/{resource}/resource           # listar
  GET    /nadota-api/{resource}/resource/{id}      # show
  POST   /nadota-api/{resource}/resource           # crear
  PUT    /nadota-api/{resource}/resource/{id}      # actualizar
  DELETE /nadota-api/{resource}/resource/{id}      # eliminar
  GET    /nadota-api/{resource}/resource/options   # opciones para selects/relaciones
  GET    /nadota-api/{resource}/resource/field/{field}/options  # opciones de campo
  POST   /nadota-api/{resource}/resource/actions/{action}      # ejecutar acción
  GET    /nadota-api/{resource}/resource/export    # exportar Excel/CSV

  El {resource} se deriva automáticamente del nombre de clase: FamilyResource → family, StudentResource → student.

  ---
  3. schoolaid-frontend — Vue 3 + TypeScript

  Arquitectura en capas

  Router (Vue Router)
    → Stores (Pinia): auth, menu, school, lang, pagination
    → TanStack Query: fetching + caching + invalidación
    → Repositories (infrastructure): ApiAuthRepository, ApiStudentRepository…
    → HTTP Client (Axios): withCredentials + CSRF + interceptor 401

  El módulo Nadota en el frontend

  Es el adaptador que convierte la API dinámica del backend en vistas renderizadas automáticamente:

  Backend envía → { fields: [{ type: 'email', name: 'email' }], ... }
  Frontend renderiza → <component :is="loadField('email')" />
                          → FieldEmail.vue (cargado dinámicamente)

  Tiene 3 niveles de uso según necesidad:
  4. Composables — useIndex(), useShow(), useStore() (máxima flexibilidad)
  5. Wrappers — gestión de estado + slots para UI custom
  6. Views — solución lista: <ShowView />, <IndexView />

  ---
  7. Servicios Externos — Meilisearch, Redis y más

  Meilisearch

  Propósito: Búsqueda full-text con filtros y geo-búsqueda.

  Modelos indexados:

  ┌─────────────┬──────────────┬─────────────────────────────────┐
  │   Modelo    │    Índice    │          Filtros clave          │
  ├─────────────┼──────────────┼─────────────────────────────────┤
  │ User        │ user         │ school_id, has_students, active │
  ├─────────────┼──────────────┼─────────────────────────────────┤
  │ Student     │ student      │ automático                      │
  ├─────────────┼──────────────┼─────────────────────────────────┤
  │ Stop        │ stop         │ school_id, geo (_geo)           │
  ├─────────────┼──────────────┼─────────────────────────────────┤
  │ Route       │ route        │ school_id, bus_id, weekday      │
  ├─────────────┼──────────────┼─────────────────────────────────┤
  │ Bus         │ bus          │ school_id, active               │
  ├─────────────┼──────────────┼─────────────────────────────────┤
  │ BusSchedule │ bus_schedule │ school_id, route_type_id        │
  └─────────────┴──────────────┴─────────────────────────────────┘

  Cómo se implementa (ciclo completo):

  8. INDEXACIÓN (cuando se guarda un modelo)
     Model::save() → Scout detecta cambio
                   → SCOUT_QUEUE=true → despacha job async
                   → Job llama toSearchableArray()
                   → Envía datos a Meilisearch

  9. BÚSQUEDA (cuando se llama al endpoint)
     GET /nadota-api/family/resource/options?search=García
     → FamilyResource::optionsSearch()
     → User::search('García')->options(['filter' => 'school_id = 5'])
     → Meilisearch retorna IDs
     → Scout hidrata los modelos Eloquent
     → ResourceOptionsService formatea la respuesta

  10. FRONTEND consume
     El frontend llama al endpoint de options para
     poblar selects, búsquedas y relaciones BelongsTo.

  ⚠️ Para crear índices: Si un índice no existe (error "Index not found"), se crea con:
  php artisan scout:import "App\Models\User"
  php artisan scout:import "App\Models\Student"
  # etc.

  En el frontend: Meilisearch también se consume directamente para búsqueda global via Vue InstantSearch (búsqueda en tiempo real mientras el usuario escribe, sin pasar por el backend de Laravel).

  ---
  Redis

  Propósito: Cache + Queues. Usa predis como cliente PHP.

  ┌───────────────┬─────────────────────┬────────────────────────┐
  │      Uso      │ Base de datos Redis │         Config         │
  ├───────────────┼─────────────────────┼────────────────────────┤
  │ Queues (jobs) │ db 0                │ QUEUE_CONNECTION=redis │
  ├───────────────┼─────────────────────┼────────────────────────┤
  │ Cache         │ db 1                │ CACHE_STORE=redis      │
  └───────────────┴─────────────────────┴────────────────────────┘

  Queues configurados:
  - emails — envío de correos
  - notifications — push notifications
  - default — jobs genéricos

  Flujo típico con Redis:
  Evento de dominio (ej: StudentPermissionApproved)
    → Listener queued (ShouldQueue)
    → Job en Redis db:0
    → Horizon worker lo procesa
    → Job ejecuta: enviar email + push + Kafka (si habilitado)

  ---
  MongoDB

  Propósito: Datos de tracking/posiciones en tiempo real y notificaciones.

  ┌──────────┬──────────────────────────┬─────────────────────────┐
  │ Conexión │      Base de datos       │           Uso           │
  ├──────────┼──────────────────────────┼─────────────────────────┤
  │ mongodb  │ schoolbuzz_mongo         │ Posiciones GPS de buses │
  ├──────────┼──────────────────────────┼─────────────────────────┤
  │ mongodb2 │ schoolbuzz_notifications │ Log de notificaciones   │
  └──────────┴──────────────────────────┴─────────────────────────┘

  ---
  Amazon S3

  Propósito: Almacenamiento de archivos (adjuntos, imágenes, documentos, QRs, PDFs).

  ┌────────────────┬─────────────────────────────────┐
  │     Bucket     │               Uso               │
  ├────────────────┼─────────────────────────────────┤
  │ schoolaidapp   │ Principal (docs, imágenes, QRs) │
  ├────────────────┼─────────────────────────────────┤
  │ sa-masterclass │ Contenido educativo             │
  └────────────────┴─────────────────────────────────┘

  ---
  Pusher + Laravel Echo

  Propósito: Comunicación en tiempo real frontend ↔ backend.

  El frontend usa Laravel Echo con Pusher para recibir eventos en vivo (por ejemplo: actualización de cola en puerta, tracking de bus).

  ---
  Ejabberd (XMPP)

  Propósito: Servidor de chat entre familias y escuela.

  El frontend usa un cliente XMPP directo via WebSocket. El backend tiene una DB separada (ejabberd en MySQL) y un servicio de integración.

  ---
  Kafka (opcional)

  Propósito: Streaming de eventos a escala para notificaciones masivas.

  8 topics: note_created, email_batch, push_batch, sms_batch, whatsapp_batch, etc. Deshabilitado por defecto, se activa cuando el volumen lo requiere.

  ---
  Sentry

  Propósito: Tracking de errores en producción (tanto backend Laravel como frontend Vue).

  ---
  5. Diagrama de Servicios Completo

  ┌──────────────────────────────────────────────────────────────────┐
  │                     schoolaid-frontend                           │
  │  Vue 3 + Pinia + TanStack Query + Vue InstantSearch              │
  └──┬──────────────┬──────────────┬────────────┬────────────────────┘
     │ HTTP/REST    │ WebSocket    │ XMPP/WS    │ Meilisearch directo
     ▼              ▼              ▼            ▼
  ┌──────────┐  ┌────────┐  ┌──────────┐  ┌────────────┐
  │ Laravel  │  │ Pusher │  │ Ejabberd │  │Meilisearch │
  │  Admin   │  │(Echo)  │  │ (XMPP)   │  │  :7700     │
  └──┬───┬───┘  └────────┘  └──────────┘  └────────────┘
     │   │                                      ▲
     │   └──── Scout indexa ────────────────────┘
     │
     ├──► MySQL (schoolbuzz) — datos principales
     ├──► MySQL (schoolbuzz_attendance) — asistencia
     ├──► MySQL (ejabberd) — chat
     ├──► MongoDB (posiciones GPS, notificaciones)
     ├──► Redis db:0 — Queues (Horizon workers)
     ├──► Redis db:1 — Cache
     ├──► S3 — archivos/imágenes/PDFs
     └──► Kafka — notificaciones masivas (opcional)

  ---
  Resumen ejecutivo

  ┌────────────────────┬─────────────────────────────────────────┬─────────────────────────────────────┐
  │      Proyecto      │                   Rol                   │          Tecnología clave           │
  ├────────────────────┼─────────────────────────────────────────┼─────────────────────────────────────┤
  │ nadota             │ Framework de admin genérico (como Nova) │ Laravel Package, Pipeline, Strategy │
  ├────────────────────┼─────────────────────────────────────────┼─────────────────────────────────────┤
  │ schoolaid-admin    │ API + lógica de negocio                 │ Laravel 12, Scout, Horizon, Sanctum │
  ├────────────────────┼─────────────────────────────────────────┼─────────────────────────────────────┤
  │ schoolaid-frontend │ SPA consumidora                         │ Vue 3, TanStack Query, Pinia        │
  ├────────────────────┼─────────────────────────────────────────┼─────────────────────────────────────┤
  │ Meilisearch        │ Búsqueda full-text + filtros            │ 6 índices, Scout queue              │
  ├────────────────────┼─────────────────────────────────────────┼─────────────────────────────────────┤
  │ Redis              │ Queues + Cache                          │ Predis, 2 DBs separadas             │
  ├────────────────────┼─────────────────────────────────────────┼─────────────────────────────────────┤
  │ MongoDB            │ GPS tracking + notificaciones           │ 2 conexiones                        │
  ├────────────────────┼─────────────────────────────────────────┼─────────────────────────────────────┤
  │ Pusher/Echo        │ Real-time events                        │ WebSocket                           │
  ├────────────────────┼─────────────────────────────────────────┼─────────────────────────────────────┤
  │ Ejabberd           │ Chat XMPP                               │ WebSocket directo                   │
  ├────────────────────┼─────────────────────────────────────────┼─────────────────────────────────────┤
  │ S3                 │ Archivos                                │ AWS, 2 buckets                      │
  └────────────────────┴─────────────────────────────────────────┴─────────────────────────────────────┘
