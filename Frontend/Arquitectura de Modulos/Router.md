El directorio **router** contiene la configuración de rutas del módulo. Define las URLs, componentes asociados, guards y metadatos de navegación.

## 📁 Ubicación

```
modules/
└── mi-modulo/
    └── router/
        ├── index.ts          # Exporta las rutas del módulo
        ├── guards.ts         # Guards de navegación (opcional)
        └── middlewares.ts    # Middlewares de ruta (opcional)
```

## 🎯 ¿Qué va en el router?

- **Definición de rutas**: Paths, componentes, nombres
- **Guards**: Validaciones antes de acceder a rutas
- **Metadatos**: Breadcrumbs, permisos, títulos
- **Lazy loading**: Importación dinámica de componentes

## 📝 Estructura básica

```typescript
// router/index.ts

import type { RouteRecordRaw } from 'vue-router'

export const featureRoutes: RouteRecordRaw[] = [
  {
    path: '/features',
    name: 'features.index',
    component: () => import('../views/FeatureIndex.vue'),
    meta: {
      requiresAuth: true,
      breadcrumb: { label: 'Características' }
    }
  },
  {
    path: '/features/create',
    name: 'features.create',
    component: () => import('../views/FeatureCreate.vue'),
    meta: {
      requiresAuth: true,
      breadcrumb: { label: 'Crear' }
    }
  },
  {
    path: '/features/:id',
    name: 'features.show',
    component: () => import('../views/FeatureShow.vue'),
    meta: {
      requiresAuth: true,
      breadcrumb: { label: 'Detalle' }
    }
  },
  {
    path: '/features/:id/edit',
    name: 'features.edit',
    component: () => import('../views/FeatureEdit.vue'),
    meta: {
      requiresAuth: true,
      breadcrumb: { label: 'Editar' }
    }
  }
]
```

## 🔧 Patrones comunes

### 1. Rutas con parámetros dinámicos

```typescript
{
  // Parámetro simple
  path: '/students/:id',
  name: 'students.show',
  component: () => import('../views/StudentShow.vue'),
  props: true  // Pasar params como props
}

{
  // Múltiples parámetros
  path: '/schools/:schoolId/grades/:gradeId/students',
  name: 'school.grade.students',
  component: () => import('../views/StudentList.vue'),
  props: (route) => ({
    schoolId: Number(route.params.schoolId),
    gradeId: Number(route.params.gradeId)
  })
}
```

### 2. Rutas anidadas (children)

```typescript
{
  path: '/dashboard',
  component: () => import('../layouts/DashboardLayout.vue'),
  children: [
    {
      path: '',  // /dashboard
      name: 'dashboard.home',
      component: () => import('../views/DashboardHome.vue')
    },
    {
      path: 'analytics',  // /dashboard/analytics
      name: 'dashboard.analytics',
      component: () => import('../views/DashboardAnalytics.vue')
    },
    {
      path: 'settings',  // /dashboard/settings
      name: 'dashboard.settings',
      component: () => import('../views/DashboardSettings.vue')
    }
  ]
}
```

### 3. Guards de navegación

```typescript
// router/guards.ts

import type { NavigationGuardNext, RouteLocationNormalized } from 'vue-router'

/**
 * Guard que verifica si el recurso existe
 */
export const resourceExistsGuard = async (
  to: RouteLocationNormalized,
  from: RouteLocationNormalized,
  next: NavigationGuardNext
) => {
  const resourceName = to.params.resourceName
  
  if (resourceName === 'allowed-resource') {
    next()
  } else {
    next({ name: 'not-found' })
  }
}

/**
 * Guard de permisos
 */
export const permissionGuard = (permission: string) => {
  return (to: RouteLocationNormalized, from: RouteLocationNormalized, next: NavigationGuardNext) => {
    const userPermissions = getUserPermissions()
    
    if (userPermissions.includes(permission)) {
      next()
    } else {
      next({ name: 'unauthorized' })
    }
  }
}
```

### 4. Uso de guards en rutas

```typescript
import { resourceExistsGuard, permissionGuard } from './guards'

export const featureRoutes: RouteRecordRaw[] = [
  {
    path: '/features/:id',
    name: 'features.show',
    component: () => import('../views/FeatureShow.vue'),
    beforeEnter: resourceExistsGuard
  },
  {
    path: '/admin/features',
    name: 'admin.features',
    component: () => import('../views/AdminFeatures.vue'),
    beforeEnter: permissionGuard('manage-features')
  },
  {
    path: '/special/:type',
    name: 'special',
    component: () => import('../views/Special.vue'),
    beforeEnter: (to, from, next) => {
      // Guard inline
      const allowedTypes = ['type-a', 'type-b']
      if (allowedTypes.includes(to.params.type as string)) {
        next()
      } else {
        next(false)
      }
    }
  }
]
```

### 5. Metadatos de ruta

```typescript
{
  path: '/features',
  name: 'features.index',
  component: () => import('../views/FeatureIndex.vue'),
  meta: {
    // Autenticación
    requiresAuth: true,
    
    // Permisos
    permissions: ['view-features'],
    
    // Breadcrumb
    breadcrumb: {
      label: 'Características',
      icon: 'feature-icon'
    },
    
    // Título de página
    title: 'Lista de Características',
    
    // Layout a usar
    layout: 'main',
    
    // Cache de componente
    keepAlive: true,
    
    // Datos personalizados
    customData: {
      category: 'admin'
    }
  }
}
```

### 6. Redirecciones

```typescript
{
  path: '/old-path',
  redirect: '/new-path'
}

{
  path: '/features',
  redirect: { name: 'features.index' }
}

{
  path: '/user/:id/profile',
  redirect: to => ({
    name: 'profile',
    params: { userId: to.params.id }
  })
}
```

### 7. Rutas con alias

```typescript
{
  path: '/features',
  alias: ['/caracteristicas', '/f'],  // Múltiples URLs
  name: 'features.index',
  component: () => import('../views/FeatureIndex.vue')
}
```

## 📦 Integración con router principal

```typescript
// src/router/index.ts

import { createRouter, createWebHistory } from 'vue-router'
import { featureRoutes } from '@/modules/features/router'
import { studentRoutes } from '@/modules/students/router'
import { authRoutes } from '@/modules/auth/router'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    ...authRoutes,
    ...featureRoutes,
    ...studentRoutes,
    // Ruta 404
    {
      path: '/:pathMatch(.*)*',
      name: 'not-found',
      component: () => import('@/views/NotFound.vue')
    }
  ]
})

export default router
```

## 🗂️ Archivos adicionales

### guards.ts

```typescript
// router/guards.ts

export const authGuard = (to, from, next) => {
  const isAuthenticated = checkAuth()
  if (to.meta.requiresAuth && !isAuthenticated) {
    next({ name: 'login' })
  } else {
    next()
  }
}

export const roleGuard = (allowedRoles: string[]) => {
  return (to, from, next) => {
    const userRole = getUserRole()
    if (allowedRoles.includes(userRole)) {
      next()
    } else {
      next({ name: 'unauthorized' })
    }
  }
}
```

### middlewares.ts

```typescript
// router/middlewares.ts

export const logNavigationMiddleware = (to, from) => {
  console.log(`Navigating from ${from.path} to ${to.path}`)
}

export const setPageTitleMiddleware = (to) => {
  document.title = to.meta.title || 'Mi Aplicación'
}
```

## ✅ Buenas prácticas

1. **Lazy loading**: Siempre usar `() => import()` para componentes
2. **Nombres únicos**: Usar namespace como `module.action` (ej: `students.create`)
3. **Props sobre params**: Preferir pasar props en lugar de acceder a `$route.params`
4. **Centralizar guards**: Reutilizar guards en archivo separado
5. **Documentar metadatos**: Definir qué metadatos usa la app

## ❌ Anti-patrones

```typescript
// ❌ MAL: Importación directa (aumenta bundle inicial)
import FeatureIndex from '../views/FeatureIndex.vue'
{
  path: '/features',
  component: FeatureIndex
}

// ✅ BIEN: Lazy loading
{
  path: '/features',
  component: () => import('../views/FeatureIndex.vue')
}

// ❌ MAL: Lógica compleja en la definición de ruta
{
  path: '/features',
  beforeEnter: async (to, from, next) => {
    const data = await fetch('/api/check')
    const json = await data.json()
    if (json.valid) {
      const user = await fetch('/api/user')
      // ... mucha lógica
    }
  }
}

// ✅ BIEN: Extraer a guard reutilizable
{
  path: '/features',
  beforeEnter: checkValidityGuard
}
```

## 📤 Exportación

```typescript
// router/index.ts

export { featureRoutes } from './index'
export { authGuard, roleGuard } from './guards'
```
