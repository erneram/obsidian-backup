La capa de **infrastructure** contiene los repositorios que manejan la comunicación con APIs externas. Implementa el patrón **Repository** para abstraer el acceso a datos.

## 📁 Ubicación

```
modules/
└── mi-modulo/
    └── infrastructure/
        ├── index.ts              # Exporta todos los repositories
        ├── FeatureRepository.ts  # Repository principal
        └── OtherRepository.ts    # Otros repositories del módulo
```

## 🎯 ¿Qué es un Repository?

Un **Repository** es una clase o objeto que:
- Encapsula la lógica de acceso a datos (HTTP, localStorage, etc.)
- Abstrae los detalles de implementación de la API
- Proporciona una interfaz limpia para los composables
- Facilita testing mediante mocking

## 📝 Estructura básica

```typescript
// infrastructure/FeatureRepository.ts

import Api from '@/services/Api'
import type {
  Feature,
  FeatureListResponse,
  CreateFeatureDto,
  UpdateFeatureDto
} from '../interfaces/Feature'

export interface IFeatureRepository {
  index(params?: Record<string, any>): Promise<Feature[]>
  show(id: number): Promise<Feature>
  store(data: CreateFeatureDto): Promise<Feature>
  update(id: number, data: UpdateFeatureDto): Promise<Feature>
  destroy(id: number): Promise<void>
}

class FeatureRepositoryImpl implements IFeatureRepository {
  private readonly basePath = '/features'

  async index(params?: Record<string, any>): Promise<Feature[]> {
    const response = await Api.get<FeatureListResponse>(this.basePath, { params })
    return response.data.data
  }

  async show(id: number): Promise<Feature> {
    const response = await Api.get<Feature>(`${this.basePath}/${id}`)
    return response.data
  }

  async store(data: CreateFeatureDto): Promise<Feature> {
    const response = await Api.post<Feature>(this.basePath, data)
    return response.data
  }

  async update(id: number, data: UpdateFeatureDto): Promise<Feature> {
    const response = await Api.put<Feature>(`${this.basePath}/${id}`, data)
    return response.data
  }

  async destroy(id: number): Promise<void> {
    await Api.delete(`${this.basePath}/${id}`)
  }
}

// Exportar instancia singleton
export const FeatureRepository = new FeatureRepositoryImpl()
```

## 🔧 Patrones de implementación

### 1. Repository con métodos CRUD estándar

```typescript
class CrudRepository<T, CreateDto, UpdateDto> {
  constructor(private basePath: string) {}

  async index(params?: Record<string, any>): Promise<T[]> {
    const { data } = await Api.get(this.basePath, { params })
    return data.data || data
  }

  async show(id: number): Promise<T> {
    const { data } = await Api.get(`${this.basePath}/${id}`)
    return data
  }

  async store(dto: CreateDto): Promise<T> {
    const { data } = await Api.post(this.basePath, dto)
    return data
  }

  async update(id: number, dto: UpdateDto): Promise<T> {
    const { data } = await Api.put(`${this.basePath}/${id}`, dto)
    return data
  }

  async destroy(id: number): Promise<void> {
    await Api.delete(`${this.basePath}/${id}`)
  }
}
```

### 2. Repository con métodos personalizados

```typescript
class OrderRepositoryImpl implements IOrderRepository {
  private readonly basePath = '/orders'

  // Métodos CRUD estándar...

  // Métodos personalizados
  async cancel(orderId: number, reason: string): Promise<Order> {
    const { data } = await Api.post(`${this.basePath}/${orderId}/cancel`, { reason })
    return data
  }

  async getByStatus(status: string): Promise<Order[]> {
    const { data } = await Api.get(this.basePath, { params: { status } })
    return data.data
  }

  async exportToPdf(orderId: number): Promise<Blob> {
    const { data } = await Api.get(`${this.basePath}/${orderId}/export`, {
      responseType: 'blob'
    })
    return data
  }
}
```

### 3. Repository para endpoints de selección

```typescript
interface SelectOption {
  value: number | string
  label: string
}

class SelectRepositoryImpl {
  async getGrades(search?: string): Promise<SelectOption[]> {
    const { data } = await Api.get('/select/grades', { params: { search } })
    return data
  }

  async getStudents(search?: string): Promise<SelectOption[]> {
    const { data } = await Api.get('/select/students', { params: { search } })
    return data
  }

  async getTeachers(search?: string): Promise<SelectOption[]> {
    const { data } = await Api.get('/select/teachers', { params: { search } })
    return data
  }
}

export const SelectRepository = new SelectRepositoryImpl()
```

### 4. Repository con manejo de body en DELETE

```typescript
async destroy(data: DeleteDto): Promise<DeleteResponse> {
  const response = await Api.delete<DeleteResponse>(
    this.basePath,
    data  // Axios permite enviar body en DELETE
  )
  return response.data
}
```

## 🏗️ Estructura de respuestas comunes

```typescript
// Respuesta de listado con paginación
interface PaginatedResponse<T> {
  data: T[]
  meta: {
    current_page: number
    last_page: number
    per_page: number
    total: number
  }
}

// Respuesta simple de listado
interface ListResponse<T> {
  data: T[]
}

// Respuesta de recurso individual
interface ResourceResponse<T> {
  data: T
}
```

## ✅ Buenas prácticas

1. **Usar interfaces**: Definir `IRepository` para tipar métodos
2. **Singleton pattern**: Exportar una instancia única del repository
3. **Nombres descriptivos**: `store()` para crear, `destroy()` para eliminar
4. **Tipar respuestas**: Siempre usar generics con tipos específicos
5. **Separar por dominio**: Un repository por entidad/recurso
6. **No incluir lógica de negocio**: Solo comunicación con API

## ❌ Anti-patrones

```typescript
// ❌ MAL: Lógica de negocio en el repository
async store(data: CreateDto): Promise<Feature> {
  // Validaciones deben estar en composable o componente
  if (!data.name) throw new Error('Name required')
  // ...
}

// ❌ MAL: Manipular UI desde repository
async destroy(id: number): Promise<void> {
  await Api.delete(`${this.basePath}/${id}`)
  toast.success('Eliminado') // NO hacer esto aquí
}

// ✅ BIEN: Solo comunicación con API
async destroy(id: number): Promise<void> {
  await Api.delete(`${this.basePath}/${id}`)
}
```

## 📤 Archivo index.ts

```typescript
// infrastructure/index.ts
export { FeatureRepository } from './FeatureRepository'
export type { IFeatureRepository } from './FeatureRepository'
export { SelectRepository } from './SelectRepository'
```

## 🔗 Uso en composables

```typescript
// composables/useFeature.ts
import { FeatureRepository } from '../infrastructure/FeatureRepository'

export function useFeature() {
  const fetchItems = async () => {
    items.value = await FeatureRepository.index()
  }

  const createItem = async (data: CreateFeatureDto) => {
    await FeatureRepository.store(data)
    await fetchItems()
  }

  // ...
}
```

## 🧪 Testing

Los repositories facilitan el testing al poder mockearlos:

```typescript
// __tests__/useFeature.spec.ts
import { vi } from 'vitest'
import { FeatureRepository } from '../infrastructure/FeatureRepository'

vi.mock('../infrastructure/FeatureRepository', () => ({
  FeatureRepository: {
    index: vi.fn().mockResolvedValue([{ id: 1, name: 'Test' }]),
    store: vi.fn().mockResolvedValue({ id: 2, name: 'New' })
  }
}))
```
