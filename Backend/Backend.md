---
title: Backend
tags: [hub, backend]
---

# 🛠️ Backend – Hub

Este es el índice principal del área **Backend**. Resume qué hay en la carpeta y te deja navegar por los temas clave (especialmente **Laravel**). La organización sigue el flujo de desarrollo: **setup → HTTP → datos → integración frontend → utilidades**.

---

## 📦 Resumen rápido

- **Laravel**: apuntes prácticos sobre estructura, arranque de proyectos, controladores, modelos/Eloquent, migraciones, factories/seeders y puentes con **Inertia.js**/**Alpine.js**.
- **Otros (raíz de Backend)**: notas sueltas como **CORS** y utilidades generales.

> Tip: Ancla esta página en **Starred** para usarla como landing del área Backend.

---

## 🗂️ Índice por temas

### 1) Setup & proyecto
- [[Backend/Laravel/Basics|Laravel · Basics]] — estructura del framework, CLI, config .env, artisan básico.
- [[Backend/Laravel/Creating Laravel Project|Creating Laravel Project]] — requisitos, instalación, npm, composer.
- [[Backend/Laravel/Launching|Launching]] — levantar proyecto con Valet, MySQL, .env, dominio .test.

### 2) Capa HTTP
- [[Backend/Laravel/Controllers|Controllers]] — controladores normales, invocables, resource controllers y rutas.

### 3) Datos (Eloquent & DB)
- [[Backend/Laravel/Models|Models]] — modelos, relaciones, scopes, casts.
- [[Backend/Laravel/Migrations|Migrations]] — esquema, tips para migraciones seguras.
- [[Backend/Laravel/Factories|Factories]] — generación de datos de prueba.
- [[Backend/Laravel/Seeder|Seeder]] — estrategias de seed y orden de carga.

### 4) Integración con Frontend
- [[Backend/Laravel/Inertia.js|Inertia.js]] — SPA sin API tradicional, share props, layouts.
- [[Backend/Laravel/Alpine.js|Alpine.js]] — “sprinkles” de interactividad en vistas.

### 5) Lenguaje y utilidades
- [[Backend/Laravel/Tinker|Tinker]] — REPL interactivo: queries, agregados, relaciones.
- [[Backend/Laravel/Php Snippet|Php Snippet]] — switch de versiones PHP, funciones útiles.
- [[Backend/Switching versions PHP|Switching PHP Versions]] — comandos para cambiar versión de PHP.
- [[Backend/CORS|CORS]] — política CORS, configuración y debugging.

> Si tienes más notas dentro de `Backend/Laravel/` (p.ej. “Creating Laravel …”), añádelas al grupo correspondiente arriba para mantener el flujo.

---

## 🔎 Búsquedas comunes (copiar/pegar en el buscador de Obsidian)

- `path:"Backend/Laravel" tag:#todo` — pendientes dentro de Laravel
- `path:"Backend" CORS` — todo lo relacionado a CORS en Backend
- `path:"Backend/Laravel" migrations rollback` — recetas rápidas de migraciones

---

## ✅ Checklist de mantenimiento

- [ ] Cuando crees una nota nueva en `Backend/Laravel`, enlázala aquí en el grupo correcto.
- [ ] Mantén **Basics** y **Launching** al día (versiones de PHP/Laravel, pasos de arranque).
- [ ] Documenta cualquier convención especial (nombres de ramas, scripts de deploy) en **Launching**.

---

## ↩️ Volver al HOME
- [[HOME|🏠 HOME del Vault]]
