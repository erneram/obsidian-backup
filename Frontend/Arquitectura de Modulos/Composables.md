Los **composables** son funciones reutilizables que encapsulan lógica de negocio usando la Composition API de Vue 3. Siguen el patrón de composición y permiten extraer y reutilizar lógica stateful entre componentes.

## 📁 Ubicación

```
modules/
└── mi-modulo/
    └── composables/
        ├── index.ts           # Exporta todos los composables
        ├── useFeature.ts      # Composable principal
        ├── useFeatureList.ts  # Composable para listados
        └── useFeatureForm.ts  # Composable para formularios
```

## 🎯 ¿Cuándo crear un composable?

- Cuando necesitas **reutilizar lógica** entre múltiples componentes
- Para **separar la lógica de negocio** de la UI
- Cuando manejas **estado reactivo** que necesita ser compartido
- Para encapsular llamadas a **APIs/repositories**

## 📝 Estructura básica

```typescript
// composables/useFeature.ts

import { ref, computed, onMounted } from 'vue'
import { FeatureRepository } from '../infrastructure/FeatureRepository'
import type { Feature } from '../interfaces/Feature'

export function useFeature() {
  // 1. Estado reactivo
  const items = ref<Feature[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)

  // 2. Computed properties
  const isEmpty = computed(() => items.value.length === 0)
  const totalItems = computed(() => items.value.length)

  // 3. Métodos/Acciones
  const fetchItems = async () => {
    loading.value = true
    error.value = null
    try {
      items.value = await FeatureRepository.index()
    } catch (err: any) {
      error.value = err?.message || 'Error al cargar datos'
      console.error('Error:', err)
    } finally {
      loading.value = false
    }
  }

  const createItem = async (data: Partial<Feature>) => {
    loading.value = true
    error.value = null
    try {
      await FeatureRepository.store(data)
      await fetchItems() // Recargar lista
    } catch (err: any) {
      error.value = err?.message || 'Error al crear'
      throw err
    } finally {
      loading.value = false
    }
  }

  const deleteItem = async (id: number) => {
    loading.value = true
    error.value = null
    try {
      await FeatureRepository.destroy(id)
      await fetchItems()
    } catch (err: any) {
      error.value = err?.message || 'Error al eliminar'
      throw err
    } finally {
      loading.value = false
    }
  }

  // 4. Retornar estado y métodos
  return {
    // Estado
    items,
    loading,
    error,
    // Computed
    isEmpty,
    totalItems,
    // Métodos
    fetchItems,
    createItem,
    deleteItem
  }
}
```

## 🔧 Patrones comunes

### 1. Composable con parámetros

```typescript
export function useFeatureById(featureId: number) {
  const feature = ref<Feature | null>(null)
  const loading = ref(false)

  const fetchFeature = async () => {
    loading.value = true
    try {
      feature.value = await FeatureRepository.show(featureId)
    } finally {
      loading.value = false
    }
  }

  return { feature, loading, fetchFeature }
}
```

### 2. Composable con parámetros de ruta

```typescript
import { useRoute } from 'vue-router'

export function useFeatureFromRoute() {
  const route = useRoute()
  
  const featureId = computed(() => {
    const id = route.params.id || route.params.resourceId
    return Number(id)
  })

  // ... resto de la lógica usando featureId.value

  return { featureId, /* ... */ }
}
```

### 3. Composable con TanStack Query

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/vue-query'

export function useFeature() {
  const queryClient = useQueryClient()

  const { data: items, isLoading, error } = useQuery({
    queryKey: ['features'],
    queryFn: () => FeatureRepository.index()
  })

  const createMutation = useMutation({
    mutationFn: (data: Partial<Feature>) => FeatureRepository.store(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['features'] })
    }
  })

  return {
    items,
    isLoading,
    error,
    createItem: createMutation.mutate
  }
}
```

### 4. Composable con filtros

```typescript
export function useFeatureList() {
  const items = ref<Feature[]>([])
  const filters = ref({
    search: '',
    status: null,
    page: 1
  })

  const fetchItems = async () => {
    items.value = await FeatureRepository.index(filters.value)
  }

  // Re-fetch automático cuando cambian filtros
  watch(filters, fetchItems, { deep: true })

  return { items, filters, fetchItems }
}
```

## ✅ Buenas prácticas

1. **Nombrar con prefijo `use`**: `useFeature`, `useAuth`, `useNotifications`
2. **Un composable = una responsabilidad**: No mezclar lógica no relacionada
3. **Llamar en setup**: Los composables deben llamarse en el contexto de setup
4. **Manejar errores**: Siempre incluir manejo de errores
5. **Documentar**: Agregar JSDoc para describir qué hace y qué retorna

## ❌ Anti-patrones

```typescript
// ❌ MAL: Llamar useRoute/useRouter fuera del setup
function getRouteId() {
  const route = useRoute() // Error: debe estar en setup
  return route.params.id
}

// ✅ BIEN: Llamar dentro del composable que se usa en setup
export function useFeature() {
  const route = useRoute() // OK: el composable se llama en setup
  const id = computed(() => route.params.id)
  // ...
}
```

## 📤 Archivo index.ts

```typescript
// composables/index.ts
export { useFeature } from './useFeature'
export { useFeatureList } from './useFeatureList'
export { useFeatureForm } from './useFeatureForm'
```

## 🔗 Uso en componentes

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useFeature } from '../composables/useFeature'

const { items, loading, error, fetchItems, deleteItem } = useFeature()

onMounted(() => {
  fetchItems()
})
</script>

<template>
  <div v-if="loading">Cargando...</div>
  <div v-else-if="error">{{ error }}</div>
  <ul v-else>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
      <button @click="deleteItem(item.id)">Eliminar</button>
    </li>
  </ul>
</template>
```
