---
title: Laravel Hub
tags: [hub, laravel, backend]
---

# 🧱 Laravel – Hub

Índice de toda la documentación de **Laravel** organizada por categoría.

---

## ⚙️ Setup
*Instalación, configuración inicial y arranque del proyecto.*

- [[Backend/Laravel/Setup/Basics|Basics]] — estructura del framework, CLI, config `.env`, artisan básico.
- [[Backend/Laravel/Setup/Creating Laravel Project|Creating Laravel Project]] — requisitos, instalación, npm, composer.
- [[Backend/Laravel/Setup/Launching|Launching]] — levantar proyecto con Valet, MySQL, `.env`, dominio `.test`.
- [[Backend/Laravel/Setup/Php Snippet|Php Snippet]] — switch de versiones PHP, funciones útiles.

---

## 🗄️ ORM (Eloquent & Database)
*Modelos, esquema, datos de prueba y REPL.*

- [[Backend/Laravel/ORM/Models|Models]] — modelos, relaciones (hasMany, belongsTo, morphTo…), scopes, casts.
- [[Backend/Laravel/ORM/Migrations|Migrations]] — crear y correr migraciones, rollback, tips seguros.
- [[Backend/Laravel/ORM/Factories|Factories]] — generar datos falsos con Faker para tests.
- [[Backend/Laravel/ORM/Seeder|Seeder]] — estrategias de seed y orden de carga.
- [[Backend/Laravel/ORM/Tinker|Tinker]] — REPL interactivo: queries, agregados, relaciones en vivo.

---

## 🌐 HTTP (Controllers & Routing)
*Capa de entrada de peticiones HTTP.*

- [[Backend/Laravel/HTTP/Controllers|Controllers]] — controladores normales, invocables, resource controllers y rutas.

---

## 🎨 Frontend Integration
*Puentes entre el backend Laravel y el frontend.*

- [[Backend/Laravel/Frontend/Inertia.js|Inertia.js]] — SPA sin API tradicional: share props, layouts, page components.
- [[Backend/Laravel/Frontend/Alpine.js|Alpine.js]] — "sprinkles" de interactividad en vistas Blade sin build step.

---

## ⏳ Queues & Async
*Procesamiento asíncrono en background.*

- [[Backend/Laravel/Queues/Jobs|Jobs]] — qué es un Job, cuándo usarlos, dispatch, reintentos, chains, batches.
- [[Backend/Laravel/Queues/Jobs + Predis|Jobs + Predis]] — cómo Redis almacena, distribuye y monitorea los Jobs.
- [[Backend/Laravel/Queues/Horizon|Horizon]] — dashboard de queues: workers, métricas, failed jobs y auto-scaling.

---

## 📦 Paquetes Externos
*Integraciones con servicios y librerías de terceros.*

- [[Backend/Laravel/Packages/Meilisearch|Meilisearch + Scout]] — búsqueda full-text: typo tolerance, filtros facetados, indexación async.
- [[Backend/Laravel/Packages/Predis|Predis (Redis)]] — queues, cache, Horizon workers.
- [[Backend/Laravel/Packages/Meilisearch + Predis|Meilisearch + Predis]] — cómo se complementan: el puente `SCOUT_QUEUE=true`.

---

## ↩️ Navegación
- [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
