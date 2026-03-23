## 🏗️ Estructura completa de un módulo

```
modules/
└── mi-modulo/
    ├── components/       # Piezas de UI reutilizables
    ├── composables/      # Lógica de negocio reutilizable
    ├── infrastructure/   # Comunicación con APIs
    ├── interfaces/       # Definiciones de tipos
    ├── views/            # Páginas completas
    ├── router/           # Configuración de rutas
    └── docs/             # Documentación
```

---

## 🔄 Analogía: Restaurante

Imagina que tu módulo es un **restaurante**:

| Carpeta | Analogía | Descripción |
|---------|----------|-------------|
| **views/** | Salón del restaurante | Donde el cliente ve todo junto (la página completa) |
| **components/** | Platos individuales | Piezas de comida que se pueden servir en diferentes mesas |
| **composables/** | Recetas | Instrucciones de cómo preparar algo (lógica reutilizable) |
| **infrastructure/** | Proveedores | Quien trae los ingredientes del exterior (APIs) |
| **interfaces/** | Menú escrito | Define qué ingredientes y platos existen (tipos) |
| **router/** | Sistema de reservas | Decide quién puede sentarse dónde (rutas y permisos) |

---

## 📊 Tabla comparativa

| Aspecto | Components | Composables | Infrastructure | Interfaces | Views | Router |
|---------|------------|-------------|----------------|------------|-------|--------|
| **Contiene** | `.vue` files | `.ts` files | `.ts` files | `.ts` files | `.vue` files | `.ts` files |
| **Propósito** | UI visual | Lógica | API calls | Tipos | Páginas | Navegación |
| **Reutilizable** | ✅ Mucho | ✅ Mucho | ✅ Moderado | ✅ Siempre | ❌ No | ❌ No |
| **Tiene estado** | Local | Compartido | No | No | Orquesta | No |
| **Depende de** | Props | Repository | Interfaces | Nada | Todo | Views |

---

## 🎯 ¿Cuándo usar cada uno?

### Components (Componentes)
**Pregunta clave**: "¿Necesito mostrar algo en pantalla que pueda usar en varios lugares?"

```
✅ Usar cuando:
- Tienes UI que se repite (botones, cards, listas)
- Quieres encapsular estilos y comportamiento visual
- Necesitas recibir datos via props y emitir eventos

❌ No usar cuando:
- Es una página completa (usa views/)
- Solo es lógica sin UI (usa composables/)
```

**Ejemplo vida real**: Un **formulario de contacto** que aparece en la página de inicio, en el footer, y en una página dedicada.

---

### Composables
**Pregunta clave**: "¿Necesito reutilizar lógica de negocio en varios componentes?"

```
✅ Usar cuando:
- Tienes lógica que se repite en múltiples componentes
- Necesitas estado reactivo compartido
- Quieres separar la lógica de la UI

❌ No usar cuando:
- La lógica es específica de un solo componente
- Solo necesitas llamar una API sin estado (puede ir en repository)
```

**Ejemplo vida real**: Un **carrito de compras** - la lógica de añadir/quitar productos se usa en la página de producto, en el mini-carrito del header, y en la página de checkout.

---

### Infrastructure (Repositories)
**Pregunta clave**: "¿Necesito comunicarme con una API externa?"

```
✅ Usar cuando:
- Haces llamadas HTTP a un backend
- Necesitas abstraer endpoints de API
- Quieres centralizar la comunicación externa

❌ No usar cuando:
- La lógica no involucra APIs externas
- Necesitas manipular datos locales (usa composables/)
```

**Ejemplo vida real**: Un **servicio de mensajería** - todas las llamadas a la API de mensajes (enviar, recibir, marcar como leído) están en un solo lugar.

---

### Interfaces
**Pregunta clave**: "¿Qué forma tienen mis datos?"

```
✅ Usar cuando:
- Defines la estructura de una entidad
- Creas DTOs para enviar/recibir datos
- Necesitas tipar respuestas de API

❌ No usar cuando:
- Son tipos muy simples (string, number)
- El tipo solo se usa en un archivo
```

**Ejemplo vida real**: Un **formulario de registro** - defines qué campos tiene un usuario (nombre, email, contraseña) y qué campos son obligatorios.

---

### Views
**Pregunta clave**: "¿Esto es una página completa que el usuario visita?"

```
✅ Usar cuando:
- Es una ruta en la aplicación
- Combina múltiples componentes
- Orquesta la lógica de una página

❌ No usar cuando:
- Es un componente reutilizable (usa components/)
- Es solo lógica (usa composables/)
```

**Ejemplo vida real**: La **página de perfil de usuario** - combina el componente de avatar, el formulario de edición, la lista de actividad reciente, etc.

---

### Router
**Pregunta clave**: "¿Cómo llegan los usuarios a esta funcionalidad?"

```
✅ Usar cuando:
- Defines URLs de la aplicación
- Configuras guards de navegación
- Estableces metadatos de ruta (breadcrumbs, permisos)

❌ No usar cuando:
- No es una página navegable
- Es un modal o drawer (no necesita ruta)
```

**Ejemplo vida real**: Un **GPS de la aplicación** - define todas las "direcciones" (URLs) a las que puede ir el usuario y las reglas para llegar (autenticación, permisos).

---

## 🔀 Flujo de datos típico

```
Usuario hace clic
       ↓
   [Component]     ← UI recibe evento
       ↓
   [View]          ← Página orquesta la acción
       ↓
   [Composable]    ← Lógica procesa la acción
       ↓
   [Repository]    ← Llama a la API
       ↓
   [Interface]     ← Tipos validan los datos
       ↓
   Backend API     ← Responde
       ↓
   (camino inverso hasta actualizar la UI)
```

---

## 🆚 Comparativas específicas

### Component vs View

| Component | View |
|-----------|------|
| Pieza pequeña y reutilizable | Página completa |
| Recibe datos por props | Obtiene datos de composables |
| Puede usarse en múltiples views | Una view por ruta |
| Sin conexión a router | Conectada a una ruta |
| Ejemplo: `UserCard.vue` | Ejemplo: `UserProfile.vue` |

### Composable vs Repository

| Composable | Repository |
|------------|------------|
| Lógica de negocio | Comunicación con API |
| Tiene estado reactivo (ref, computed) | Sin estado, solo funciones |
| Puede llamar repositories | No debe llamar composables |
| Maneja loading/error | Solo hace fetch y retorna |
| Ejemplo: `useCart()` | Ejemplo: `CartRepository.index()` |

### Component vs Composable

| Component | Composable |
|-----------|------------|
| Tiene template HTML | Solo TypeScript |
| UI visual | Lógica reutilizable |
| Props + Events | Retorna refs y funciones |
| Se importa en `<template>` | Se llama en `<script setup>` |
| Archivo `.vue` | Archivo `.ts` |

---

## 📋 Checklist de decisión

Cuando crees un nuevo archivo, pregúntate:

1. **¿Tiene HTML/template?**
   - Sí → `components/` o `views/`
   - No → `composables/`, `infrastructure/`, o `interfaces/`

2. **¿Es una página completa con ruta?**
   - Sí → `views/`
   - No → `components/`

3. **¿Hace llamadas HTTP?**
   - Sí → `infrastructure/`
   - No → `composables/`

4. **¿Define la forma de los datos?**
   - Sí → `interfaces/`

5. **¿Configura navegación?**
   - Sí → `router/`

---

## 🌳 Ejemplo práctico: Módulo de Productos

```
modules/
└── products/
    ├── components/
    │   ├── ProductCard.vue         # Card de un producto
    │   ├── ProductList.vue         # Lista de productos
    │   ├── ProductFilters.vue      # Filtros de búsqueda
    │   └── ProductForm.vue         # Formulario crear/editar
    │
    ├── composables/
    │   ├── useProducts.ts          # CRUD de productos
    │   ├── useProductFilters.ts    # Lógica de filtros
    │   └── useProductCart.ts       # Añadir al carrito
    │
    ├── infrastructure/
    │   └── ProductRepository.ts    # API de productos
    │
    ├── interfaces/
    │   ├── Product.ts              # Interface Product
    │   └── ProductDto.ts           # DTOs crear/actualizar
    │
    ├── views/
    │   ├── ProductIndex.vue        # /products
    │   ├── ProductShow.vue         # /products/:id
    │   └── ProductCreate.vue       # /products/create
    │
    └── router/
        └── index.ts                # Rutas del módulo
```

---

## 💡 Regla de oro

> **"Cada archivo debe tener una sola responsabilidad clara"**

Si no puedes describir en una oración qué hace un archivo, probablemente hace demasiado y debería dividirse.

---

## ↩️ Navegación
- [[Frontend/Frontend|🎨 Frontend]] → [[HOME|🏠 HOME]]
