Las **interfaces** definen la estructura de los datos (tipos TypeScript) utilizados en el módulo. Proporcionan type-safety y documentación del código.

## 📁 Ubicación

```
modules/
└── mi-modulo/
    └── interfaces/
        ├── index.ts          # Exporta todas las interfaces
        ├── Feature.ts        # Interfaces de la entidad principal
        ├── FeatureDto.ts     # DTOs (Data Transfer Objects)
        └── FeatureEnums.ts   # Enums relacionados
```

## 🎯 ¿Qué debe ir en interfaces?

- **Entidades**: Modelos de datos del dominio
- **DTOs**: Objetos para crear/actualizar recursos
- **Respuestas de API**: Estructura de lo que devuelve el backend
- **Props de componentes**: Tipos para props complejas
- **Enums**: Valores constantes con nombre

## 📝 Estructura básica

```typescript
// interfaces/Feature.ts

/**
 * Entidad principal del dominio
 */
export interface Feature {
  id: number
  name: string
  description: string | null
  status: FeatureStatus
  createdAt: string
  updatedAt: string
}

/**
 * Estados posibles de una Feature
 */
export enum FeatureStatus {
  ACTIVE = 'active',
  INACTIVE = 'inactive',
  PENDING = 'pending'
}

/**
 * DTO para crear una Feature
 */
export interface CreateFeatureDto {
  name: string
  description?: string
  status?: FeatureStatus
}

/**
 * DTO para actualizar una Feature
 */
export interface UpdateFeatureDto {
  name?: string
  description?: string
  status?: FeatureStatus
}

/**
 * Respuesta del listado de Features
 */
export interface FeatureListResponse {
  data: Feature[]
  meta?: PaginationMeta
}

/**
 * Metadatos de paginación
 */
export interface PaginationMeta {
  current_page: number
  last_page: number
  per_page: number
  total: number
}
```

## 🔧 Tipos de interfaces comunes

### 1. Entidades (Models)

```typescript
/**
 * Representa un usuario en el sistema
 */
export interface User {
  id: number
  email: string
  firstName: string
  lastName: string
  role: UserRole
  avatar?: string
  createdAt: string
  updatedAt: string
}

/**
 * Representa un estudiante con relaciones
 */
export interface Student {
  id: number
  firstName: string
  lastName: string
  grade: Grade        // Relación
  stops: Stop[]       // Relación múltiple
  status: StudentStatus
}
```

### 2. DTOs (Data Transfer Objects)

```typescript
/**
 * Datos requeridos para crear un estudiante
 */
export interface CreateStudentDto {
  first_name: string      // snake_case para API
  last_name: string
  grade_id: number
  email?: string
}

/**
 * Datos para actualizar un estudiante (todos opcionales)
 */
export interface UpdateStudentDto {
  first_name?: string
  last_name?: string
  grade_id?: number
  email?: string
}

/**
 * Datos para eliminar (cuando se requiere body)
 */
export interface DeleteStudentDto {
  id: number
  reason?: string
}
```

### 3. Respuestas de API

```typescript
/**
 * Respuesta genérica de un recurso
 */
export interface ApiResponse<T> {
  data: T
  message?: string
}

/**
 * Respuesta de listado con paginación
 */
export interface ApiIndexResponse<T> {
  data: T[]
  meta: {
    current_page: number
    last_page: number
    per_page: number
    total: number
    from: number
    to: number
  }
  links?: {
    first: string
    last: string
    prev: string | null
    next: string | null
  }
}

/**
 * Respuesta de error
 */
export interface ApiErrorResponse {
  message: string
  errors?: Record<string, string[]>
}

/**
 * Respuesta de eliminación exitosa
 */
export interface DeleteResponse {
  message: string
  success: boolean
}
```

### 4. Opciones para selects

```typescript
/**
 * Opción genérica para selects/dropdowns
 */
export interface SelectOption {
  value: number | string
  label: string
}

/**
 * Opción con datos adicionales
 */
export interface SelectOptionExtended<T = unknown> {
  value: number | string
  label: string
  data?: T
  disabled?: boolean
}
```

### 5. Filtros y parámetros

```typescript
/**
 * Parámetros de búsqueda/filtrado
 */
export interface FeatureFilters {
  search?: string
  status?: FeatureStatus
  startDate?: string
  endDate?: string
  page?: number
  perPage?: number
  sortBy?: string
  sortOrder?: 'asc' | 'desc'
}
```

### 6. Enums

```typescript
/**
 * Estados de un pedido
 */
export enum OrderStatus {
  PENDING = 'pending',
  PROCESSING = 'processing',
  SHIPPED = 'shipped',
  DELIVERED = 'delivered',
  CANCELLED = 'cancelled'
}

/**
 * Roles de usuario
 */
export enum UserRole {
  ADMIN = 'admin',
  TEACHER = 'teacher',
  PARENT = 'parent',
  STUDENT = 'student'
}

/**
 * Días de la semana
 */
export enum Weekday {
  SUNDAY = 0,
  MONDAY = 1,
  TUESDAY = 2,
  WEDNESDAY = 3,
  THURSDAY = 4,
  FRIDAY = 5,
  SATURDAY = 6
}
```

## 🏗️ Convenciones de nomenclatura

| Tipo | Convención | Ejemplo |
|------|------------|---------|
| Entidad | PascalCase singular | `Student`, `Feature` |
| DTO crear | `Create` + Entidad + `Dto` | `CreateStudentDto` |
| DTO actualizar | `Update` + Entidad + `Dto` | `UpdateStudentDto` |
| DTO eliminar | `Delete` + Entidad + `Dto` | `DeleteStudentDto` |
| Respuesta lista | Entidad + `ListResponse` | `StudentListResponse` |
| Respuesta única | Entidad + `Response` | `StudentResponse` |
| Enum | PascalCase | `StudentStatus` |
| Type alias | PascalCase | `StudentId` |

## ✅ Buenas prácticas

1. **Documentar con JSDoc**: Describir qué representa cada interface
2. **Usar `?` para opcionales**: `description?: string`
3. **Preferir interfaces sobre types**: Para objetos, usar `interface`
4. **Separar DTOs de entidades**: No mezclar lo que envías vs lo que recibes
5. **Usar enums para valores fijos**: Status, roles, tipos, etc.
6. **Exportar todo desde index.ts**: Facilita imports

## ❌ Anti-patrones

```typescript
// ❌ MAL: Interface con cualquier tipo
export interface Feature {
  data: any  // Evitar any
}

// ❌ MAL: Mezclar entidad con DTO
export interface Student {
  id: number           // Propiedad de entidad
  first_name: string   // snake_case de API
  firstName: string    // camelCase de frontend (duplicado)
}

// ✅ BIEN: Separar claramente
export interface Student {
  id: number
  firstName: string
}

export interface CreateStudentDto {
  first_name: string  // Lo que espera la API
}
```

## 📤 Archivo index.ts

```typescript
// interfaces/index.ts

// Entidades
export type { Feature, FeatureWithRelations } from './Feature'
export type { Student, StudentWithGrade } from './Student'

// DTOs
export type { CreateFeatureDto, UpdateFeatureDto, DeleteFeatureDto } from './Feature'

// Respuestas
export type { FeatureListResponse, FeatureResponse } from './Feature'

// Enums
export { FeatureStatus } from './Feature'
export { UserRole } from './User'
```

## 🔗 Uso en otros archivos

```typescript
// En repository
import type { Feature, CreateFeatureDto, FeatureListResponse } from '../interfaces'

// En composable
import type { Feature } from '../interfaces/Feature'

// En componente
import type { Feature } from '@/modules/feature/interfaces'
```

## 💡 Tip: Type vs Interface

```typescript
// Usar INTERFACE para objetos
interface User {
  id: number
  name: string
}

// Usar TYPE para uniones, primitivos, o tipos complejos
type UserId = number
type UserStatus = 'active' | 'inactive'
type UserOrNull = User | null
type UserArray = User[]
```
