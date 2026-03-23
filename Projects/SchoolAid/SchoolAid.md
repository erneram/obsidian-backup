---
title: SchoolAid
tags: [project, schoolaid, laravel, vue]
---

# 🚌 SchoolAid – Hub

Sistema de gestión de transporte escolar. Tres proyectos coordinados con múltiples servicios externos.

---

## 📐 Los 3 Proyectos

| Proyecto | Rol | Stack |
|---------|-----|-------|
| **schoolaid-admin** | API + lógica de negocio + multi-tenancy | Laravel 12, Scout, Horizon, Sanctum |
| **schoolaid-frontend** | SPA consumidora | Vue 3, TanStack Query, Pinia |
| **nadota** | Framework de admin genérico (como Nova) | Laravel Package, Pipeline |

---

## 🗂️ Documentación

- [[work/schoolaid/Schoolaid Architecture.|Architecture]] — arquitectura completa del sistema: capas, servicios externos, diagrama de servicios.
- [[work/schoolaid/Guia detallada|Detailed Guide]] — guía técnica profunda: WebSockets, Redis, Meilisearch.

---

## 🛠️ Stack de Servicios

| Servicio | Propósito |
|---------|-----------|
| MySQL | Datos principales + asistencia + chat |
| MongoDB | GPS tracking + log de notificaciones |
| Redis | Queues (Horizon) + Cache |
| Meilisearch | Búsqueda full-text (6 índices) |
| Pusher/Echo | Real-time events (WebSocket) |
| Ejabberd | Chat XMPP |
| S3 | Archivos, imágenes, PDFs, QRs |
| Kafka | Notificaciones masivas (opcional) |

---

## 🔗 Conceptos Relacionados en el Vault

**Backend:**
- [[Backend/Laravel/Models|Eloquent Models]] — 200+ modelos con multi-tenancy.
- [[Backend/Laravel/Migrations|Migrations]] — esquema de BD.
- [[Backend/Laravel/Factories|Factories]] & [[Backend/Laravel/Seeder|Seeders]] — datos de prueba.
- [[Backend/Laravel/Controllers|Controllers]] — 82+ endpoints API REST.

**Frontend:**
- [[Frontend/Arquitectura de Modulos/Resumen|Módulos Vue]] — arquitectura de la SPA.
- [[Frontend/Arquitectura de Modulos/Composables|Composables]] — TanStack Query + lógica.
- [[Frontend/Arquitectura de Modulos/Infrastructure (Repositories)|Repositories]] — capa de acceso a API.

**Infraestructura:**
- [[CS/DATABASE/Database|Databases]] — SQL, NoSQL concepts.
- [[CS/NoSQL/NoSQL|NoSQL]] — MongoDB y Redis.
- [[Server/AWS/AWS|AWS]] — infraestructura en la nube.

---

## ↩️ Navegación
- [[Projects/Projects|🚀 Projects]] → [[HOME|🏠 HOME]]
