Las **views** son componentes Vue que representan páginas completas de la aplicación. Están asociadas a rutas y actúan como contenedores que orquestan componentes más pequeños.

## 📁 Ubicación

```
modules/
└── mi-modulo/
    └── views/
        ├── FeatureIndex.vue    # Listado/índice
        ├── FeatureShow.vue     # Detalle de un recurso
        ├── FeatureCreate.vue   # Formulario de creación
        ├── FeatureEdit.vue     # Formulario de edición
        └── FeatureSettings.vue # Configuraciones
```

## 🎯 ¿Qué es una View?

Una **view** es:
- Un componente Vue asociado a una **ruta**
- La **página principal** que ve el usuario
- Un **contenedor** que usa componentes y composables
- Responsable de **orquestar** la lógica de la página

## 📝 Estructura básica

```vue
<!-- views/FeatureIndex.vue -->

<script setup lang="ts">
import { onMounted } from 'vue'
import { useI18n } from 'vue-i18n'

// Composables del módulo
import { useFeatureList } from '../composables/useFeatureList'

// Componentes del módulo
import FeatureCard from '../components/FeatureCard.vue'
import FeatureFilters from '../components/FeatureFilters.vue'

// Componentes compartidos
import { Button } from '@/modules/shared/components/ui/button'
import { Card } from '@/modules/shared/components/ui/card'
import MainLayout from '@/modules/shared/layouts/MainLayout.vue'

const { t } = useI18n()

// Usar composable
const {
  items,
  loading,
  error,
  filters,
  fetchItems,
  deleteItem
} = useFeatureList()

// Cargar datos al montar
onMounted(() => {
  fetchItems()
})
</script>

<template>
  <MainLayout>
    <div class="container mx-auto p-6">
      <!-- Header -->
      <div class="flex justify-between items-center mb-6">
        <h1 class="text-2xl font-bold">{{ t('features') }}</h1>
        <RouterLink :to="{ name: 'features.create' }">
          <Button>{{ t('create') }}</Button>
        </RouterLink>
      </div>

      <!-- Filtros -->
      <FeatureFilters v-model="filters" class="mb-4" />

      <!-- Estados -->
      <div v-if="loading" class="flex justify-center py-8">
        <span class="loading loading-spinner loading-lg"></span>
      </div>

      <div v-else-if="error" class="alert alert-error">
        {{ error }}
      </div>

      <div v-else-if="items.length === 0" class="text-center py-8 text-gray-500">
        {{ t('noItemsFound') }}
      </div>

      <!-- Lista -->
      <div v-else class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        <FeatureCard
          v-for="item in items"
          :key="item.id"
          :feature="item"
          @delete="deleteItem(item.id)"
        />
      </div>
    </div>
  </MainLayout>
</template>
```

## 🔧 Tipos de views

### 1. Index View (Listado)

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useFeatureList } from '../composables/useFeatureList'
import FeatureTable from '../components/FeatureTable.vue'

const { items, loading, fetchItems, deleteItem } = useFeatureList()

onMounted(fetchItems)
</script>

<template>
  <div>
    <header class="flex justify-between mb-4">
      <h1>Listado</h1>
      <RouterLink :to="{ name: 'features.create' }">
        <Button>Crear nuevo</Button>
      </RouterLink>
    </header>
    
    <FeatureTable
      :items="items"
      :loading="loading"
      @delete="deleteItem"
    />
  </div>
</template>
```

### 2. Show View (Detalle)

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useRoute } from 'vue-router'
import { useFeatureById } from '../composables/useFeatureById'
import FeatureDetail from '../components/FeatureDetail.vue'

const route = useRoute()
const featureId = Number(route.params.id)

const { feature, loading, error, fetchFeature } = useFeatureById(featureId)

onMounted(fetchFeature)
</script>

<template>
  <div>
    <div v-if="loading">Cargando...</div>
    <div v-else-if="error">{{ error }}</div>
    <FeatureDetail v-else-if="feature" :feature="feature" />
  </div>
</template>
```

### 3. Create View (Formulario de creación)

```vue
<script setup lang="ts">
import { useRouter } from 'vue-router'
import { toast } from 'vue-sonner'
import { useFeatureCreate } from '../composables/useFeatureCreate'
import FeatureForm from '../components/FeatureForm.vue'

const router = useRouter()
const { createFeature, loading, error } = useFeatureCreate()

const handleSubmit = async (data) => {
  try {
    await createFeature(data)
    toast.success('Creado exitosamente')
    router.push({ name: 'features.index' })
  } catch (err) {
    toast.error('Error al crear')
  }
}
</script>

<template>
  <div>
    <h1>Crear Feature</h1>
    <FeatureForm
      :loading="loading"
      @submit="handleSubmit"
    />
  </div>
</template>
```

### 4. Edit View (Formulario de edición)

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { toast } from 'vue-sonner'
import { useFeatureEdit } from '../composables/useFeatureEdit'
import FeatureForm from '../components/FeatureForm.vue'

const route = useRoute()
const router = useRouter()
const featureId = Number(route.params.id)

const { feature, loading, fetchFeature, updateFeature } = useFeatureEdit(featureId)

onMounted(fetchFeature)

const handleSubmit = async (data) => {
  try {
    await updateFeature(data)
    toast.success('Actualizado exitosamente')
    router.push({ name: 'features.show', params: { id: featureId } })
  } catch (err) {
    toast.error('Error al actualizar')
  }
}
</script>

<template>
  <div>
    <h1>Editar Feature</h1>
    <div v-if="loading && !feature">Cargando...</div>
    <FeatureForm
      v-else
      :initial-data="feature"
      :loading="loading"
      @submit="handleSubmit"
    />
  </div>
</template>
```

### 5. Dashboard/Composite View

```vue
<script setup lang="ts">
import { useFeatureStats } from '../composables/useFeatureStats'
import StatsCard from '../components/StatsCard.vue'
import RecentFeatures from '../components/RecentFeatures.vue'
import FeatureChart from '../components/FeatureChart.vue'

const { stats, recentItems, chartData } = useFeatureStats()
</script>

<template>
  <div class="grid grid-cols-12 gap-4">
    <!-- Stats en una fila -->
    <div class="col-span-12 grid grid-cols-4 gap-4">
      <StatsCard
        v-for="stat in stats"
        :key="stat.label"
        :label="stat.label"
        :value="stat.value"
        :icon="stat.icon"
      />
    </div>
    
    <!-- Gráfica y lista lado a lado -->
    <div class="col-span-8">
      <FeatureChart :data="chartData" />
    </div>
    
    <div class="col-span-4">
      <RecentFeatures :items="recentItems" />
    </div>
  </div>
</template>
```

## 🎨 Convención de estructura

```vue
<script setup lang="ts">
// 1. Imports de Vue
import { ref, computed, onMounted, watch } from 'vue'

// 2. Imports de vue-router
import { useRoute, useRouter } from 'vue-router'

// 3. Imports de librerías externas
import { useI18n } from 'vue-i18n'
import { toast } from 'vue-sonner'

// 4. Imports de composables del módulo
import { useFeature } from '../composables/useFeature'

// 5. Imports de componentes del módulo
import FeatureList from '../components/FeatureList.vue'
import FeatureForm from '../components/FeatureForm.vue'

// 6. Imports de componentes compartidos
import { Button } from '@/modules/shared/components/ui/button'
import MainLayout from '@/modules/shared/layouts/MainLayout.vue'

// 7. Props y emits (si aplica)
const props = defineProps<{
  initialTab?: string
}>()

// 8. Composables y hooks
const { t } = useI18n()
const route = useRoute()
const router = useRouter()

// 9. Estado local
const activeTab = ref(props.initialTab || 'list')

// 10. Composables del módulo
const { items, loading, fetchItems } = useFeature()

// 11. Computed properties
const pageTitle = computed(() => t('features'))

// 12. Watchers
watch(activeTab, (newTab) => {
  console.log('Tab changed:', newTab)
})

// 13. Métodos/handlers
const handleCreate = () => {
  router.push({ name: 'features.create' })
}

// 14. Lifecycle hooks
onMounted(() => {
  fetchItems()
})
</script>

<template>
  <!-- Template -->
</template>

<style scoped>
/* Estilos si son necesarios */
</style>
```

## ✅ Buenas prácticas

1. **Delegar lógica a composables**: La view orquesta, no implementa
2. **Usar componentes pequeños**: Dividir UI en componentes reutilizables
3. **Manejar estados**: loading, error, empty, success
4. **Lazy load pesado**: Para secciones que no son visibles inicialmente
5. **Internacionalización**: Usar `t()` para textos

## ❌ Anti-patrones

```vue
<!-- ❌ MAL: Lógica de negocio en la view -->
<script setup>
const items = ref([])

const fetchItems = async () => {
  const response = await axios.get('/api/items')  // Debería estar en repository
  items.value = response.data.map(item => ({      // Debería estar en composable
    ...item,
    fullName: `${item.firstName} ${item.lastName}`
  }))
}
</script>

<!-- ✅ BIEN: Delegar a composable -->
<script setup>
import { useItems } from '../composables/useItems'
const { items, fetchItems } = useItems()
</script>
```

```vue
<!-- ❌ MAL: Template gigante sin componentizar -->
<template>
  <div>
    <div class="header">...</div>
    <div class="filters">
      <!-- 100 líneas de filtros -->
    </div>
    <div class="table">
      <!-- 200 líneas de tabla -->
    </div>
    <div class="pagination">...</div>
  </div>
</template>

<!-- ✅ BIEN: Componentizar -->
<template>
  <div>
    <PageHeader :title="title" />
    <FeatureFilters v-model="filters" />
    <FeatureTable :items="items" />
    <Pagination v-model="page" :total="total" />
  </div>
</template>
```

## 🔗 Relación con rutas

```typescript
// router/index.ts
{
  path: '/features',
  name: 'features.index',
  component: () => import('../views/FeatureIndex.vue')  // ← View
}
```

La view `FeatureIndex.vue` se carga cuando el usuario navega a `/features`.

---

## ↩️ Navegación
- [[Frontend/Frontend|🎨 Frontend]] → [[HOME|🏠 HOME]]
