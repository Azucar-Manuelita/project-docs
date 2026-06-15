---
name: vue-architecture
description: Vue 3 Composition API con arquitectura de 3 capas (Views, Components, Services). Incluye buenas prácticas de desarrollo, guías de arquitectura, y reglas MUST/SHOULD para revisiones de código limpio.
metadata:
  author: Ingesoft - Arquitectura de Capas
  version: "2026.1.31"
  source: Generated from https://github.com/vuejs/docs, customized for layered architecture
  applyTo:
    - "**/*.vue"
    - "**/*.ts"
    - "**/*.tsx"
---

# Vue 3 - Arquitectura de 3 Capas

> Basado en Vue 3.5. Siempre usar Composition API con `<script setup lang="ts">` siguiendo arquitectura de 3 capas: Views, Components y Services.

---

## 1. ARQUITECTURA DE CAPAS

### 1.1 Estructura General del Proyecto

```
src/
├── views/                 # Capa de Presentación (Páginas/Pantallas)
│   ├── admin/
│   │   ├── AdminDashboard.vue
│   │   └── UserManagement.vue
│   ├── user/
│   │   ├── TrainingPlan.vue
│   │   └── ProgressTracking.vue
│   └── shared/
│       └── NotFound.vue
├── components/            # Capa de Componentes Reutilizables
│   ├── common/
│   │   ├── Header.vue
│   │   ├── Sidebar.vue
│   │   └── Modal.vue
│   ├── forms/
│   │   ├── LoginForm.vue
│   │   ├── UserForm.vue
│   │   └── TrainingForm.vue
│   ├── tables/
│   │   ├── UsersTable.vue
│   │   └── TrainingsTable.vue
│   └── charts/
│       ├── ProgressChart.vue
│       └── StatisticsChart.vue
├── services/              # Capa de Lógica de Negocio (APIs)
│   ├── api/
│   │   ├── apiClient.ts
│   │   ├── userApi.ts
│   │   ├── adminApi.ts
│   │   └── trainingApi.ts
│   ├── composables/       # Composables reutilizables
│   │   ├── useUser.ts
│   │   ├── useAuth.ts
│   │   ├── useTraining.ts
│   │   └── useNotification.ts
│   └── store/             # Estado global (si aplica)
│       ├── modules/
│       │   ├── user.ts
│       │   ├── auth.ts
│       │   └── training.ts
│       └── index.ts
├── types/                 # Tipos TypeScript compartidos
│   ├── index.ts
│   └── models.ts
├── utils/                 # Utilidades compartidas
│   ├── formatters.ts
│   ├── validators.ts
│   └── constants.ts
└── main.ts
```

### 1.2 Flujo de Dependencias (Arquitectura Limpia)

```
Views (Presentación)
  ↓
  ├─→ Components (UI Reutilizable)
  │    ↓
  │    └─→ Composables (Lógica Compartida)
  │         ↓
  └─→ Composables (Lógica Compartida)
       ↓
       ├─→ Services/API (Llamadas HTTP)
       │    ↓
       │    └─→ API Client (Configuración)
       │
       ├─→ Types (Modelos Compartidos)
       │
       └─→ Utils (Funciones de Apoyo)

REGLA: Las capas superiores pueden depender de las inferiores,
       NUNCA lo contrario. Los Services NUNCA importan Views o Components.
```

### 1.3 Responsabilidades por Capa

#### **CAPA 1: VIEWS (Presentación/Páginas)**

**Responsabilidades:**
- Orquestar flujo de la página
- Integrar componentes
- Manejar estados específicos de la página
- Llamar composables para obtener lógica

**Prohibiciones:**
- NO hacer llamadas HTTP directas
- NO tener lógica de negocio compleja
- NO renderizar listas largas sin virtualización
- NO duplicar código entre vistas

**Estructura típica:**
```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useUser } from '@/services/composables/useUser'
import { useNotification } from '@/services/composables/useNotification'
import UserTable from '@/components/tables/UsersTable.vue'
import UserForm from '@/components/forms/UserForm.vue'

// Importar composables, NO servicios directamente
const { users, loading, fetchUsers } = useUser()
const { showNotification } = useNotification()

// Ciclo de vida
onMounted(async () => {
  await fetchUsers()
})

// Handlers de eventos
const handleUserCreated = async (userData: CreateUserDTO) => {
  try {
    // El composable maneja la lógica
    const newUser = await useUser().createUser(userData)
    showNotification('Usuario creado exitosamente', 'success')
    await fetchUsers()
  } catch (error) {
    showNotification('Error al crear usuario', 'error')
  }
}
</script>

<template>
  <div class="user-management">
    <UserForm @submit="handleUserCreated" />
    <UserTable :users="users" :loading="loading" />
  </div>
</template>
```

#### **CAPA 2: COMPONENTS (Componentes Reutilizables)**

**Responsabilidades:**
- Presentar datos mediante props
- Emitir eventos para cambios
- Usar v-model para datos bidireccionales
- NO contener lógica de negocio
- Ser reutilizables en múltiples contextos

**Prohibiciones:**
- NO hacer llamadas HTTP
- NO acceder a servicios o APIs directamente
- NO tener estado global interno complejo
- NO conocer sobre el dominio de negocio

**Estructura típica:**
```vue
<script setup lang="ts">
import { computed } from 'vue'
import type { User, UserDTO } from '@/types'

// Props fuertemente tipados
interface Props {
  users: User[]
  loading?: boolean
  pageSize?: number
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  pageSize: 10,
})

// Emits fuertemente tipados
interface Emits {
  update: [user: User]
  delete: [userId: number]
}

const emit = defineEmits<Emits>()

// v-model para datos bidireccionales
const selectedUser = defineModel<User | null>('selected')

// Lógica de presentación SOLO
const paginatedUsers = computed(() =>
  props.users.slice(0, props.pageSize)
)

// Handlers que emiten eventos
const onUserClick = (user: User) => {
  selectedUser.value = user
  emit('update', user)
}

const onDelete = (user: User) => {
  emit('delete', user.id)
}
</script>

<template>
  <div class="users-table">
    <table v-if="!loading">
      <thead>
        <tr>
          <th>ID</th>
          <th>Nombre</th>
          <th>Email</th>
          <th>Acciones</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="user in paginatedUsers"
          :key="user.id"
          @click="onUserClick(user)"
          :class="{ 'is-selected': selectedUser?.id === user.id }"
        >
          <td>{{ user.id }}</td>
          <td>{{ user.name }}</td>
          <td>{{ user.email }}</td>
          <td>
            <button @click.stop="onDelete(user)">Eliminar</button>
          </td>
        </tr>
      </tbody>
    </table>
    <div v-else class="loading">Cargando...</div>
  </div>
</template>

<style scoped>
.users-table {
  width: 100%;
}

.is-selected {
  background-color: #f0f0f0;
}

.loading {
  padding: 2rem;
  text-align: center;
}
</style>
```

#### **CAPA 3: SERVICES/COMPOSABLES (Lógica de Negocio)**

**Composables de Lógica:**
Contienen la lógica de negocio reutilizable sin estar ligados a una vista específica.

```ts
// services/composables/useUser.ts
import { ref, computed } from 'vue'
import { useNotification } from './useNotification'
import * as userApi from '@/services/api/userApi'
import type { User, CreateUserDTO, UpdateUserDTO } from '@/types'

/**
 * Composable para gestionar usuarios
 * Responsable de: obtener, crear, actualizar, eliminar usuarios
 * 
 * @example
 * const { users, loading, fetchUsers, createUser } = useUser()
 */
export function useUser() {
  const users = ref<User[]>([])
  const loading = ref(false)
  const currentUser = ref<User | null>(null)
  const { showNotification } = useNotification()

  // Computed properties
  const totalUsers = computed(() => users.value.length)
  const hasUsers = computed(() => users.value.length > 0)

  /**
   * Obtiene todos los usuarios del servidor
   */
  const fetchUsers = async (): Promise<void> => {
    loading.value = true
    try {
      const response = await userApi.getUsers()
      users.value = response.data
    } catch (error) {
      showNotification('Error al cargar usuarios', 'error')
      throw error
    } finally {
      loading.value = false
    }
  }

  /**
   * Crea un nuevo usuario
   * @param userData - Datos del usuario a crear
   * @returns Usuario creado
   */
  const createUser = async (userData: CreateUserDTO): Promise<User> => {
    loading.value = true
    try {
      const response = await userApi.createUser(userData)
      users.value.push(response.data)
      return response.data
    } catch (error) {
      showNotification('Error al crear usuario', 'error')
      throw error
    } finally {
      loading.value = false
    }
  }

  /**
   * Actualiza un usuario existente
   * @param id - ID del usuario a actualizar
   * @param userData - Datos actualizados
   */
  const updateUser = async (id: number, userData: UpdateUserDTO): Promise<User> => {
    try {
      const response = await userApi.updateUser(id, userData)
      const index = users.value.findIndex(u => u.id === id)
      if (index !== -1) {
        users.value[index] = response.data
      }
      return response.data
    } catch (error) {
      showNotification('Error al actualizar usuario', 'error')
      throw error
    }
  }

  /**
   * Elimina un usuario
   * @param id - ID del usuario a eliminar
   */
  const deleteUser = async (id: number): Promise<void> => {
    try {
      await userApi.deleteUser(id)
      users.value = users.value.filter(u => u.id !== id)
    } catch (error) {
      showNotification('Error al eliminar usuario', 'error')
      throw error
    }
  }

  return {
    users,
    loading,
    currentUser,
    totalUsers,
    hasUsers,
    fetchUsers,
    createUser,
    updateUser,
    deleteUser,
  }
}
```

**Services/API:**
Contienen la configuración de llamadas HTTP.

```ts
// services/api/apiClient.ts
import axios, { type AxiosInstance, type AxiosRequestConfig } from 'axios'

/**
 * Cliente HTTP centralizado para todas las llamadas API
 * Responsabilidades:
 * - Configuración base (URLs, headers, timeouts)
 * - Interceptores de request/response
 * - Manejo de errores global
 * - Autenticación (tokens)
 */
class ApiClient {
  private client: AxiosInstance

  constructor() {
    this.client = axios.create({
      baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8000/api',
      timeout: 10000,
      headers: {
        'Content-Type': 'application/json',
      },
    })

    this.setupInterceptors()
  }

  private setupInterceptors(): void {
    // Interceptor de request: agregar token
    this.client.interceptors.request.use((config) => {
      const token = localStorage.getItem('authToken')
      if (token) {
        config.headers.Authorization = `Bearer ${token}`
      }
      return config
    })

    // Interceptor de response: manejo de errores
    this.client.interceptors.response.use(
      response => response,
      (error) => {
        if (error.response?.status === 401) {
          // Token inválido, redirigir a login
          window.location.href = '/login'
        }
        return Promise.reject(error)
      }
    )
  }

  /**
   * GET request
   */
  get<T>(url: string, config?: AxiosRequestConfig) {
    return this.client.get<T>(url, config)
  }

  /**
   * POST request
   */
  post<T>(url: string, data?: any, config?: AxiosRequestConfig) {
    return this.client.post<T>(url, data, config)
  }

  /**
   * PUT request
   */
  put<T>(url: string, data?: any, config?: AxiosRequestConfig) {
    return this.client.put<T>(url, data, config)
  }

  /**
   * DELETE request
   */
  delete<T>(url: string, config?: AxiosRequestConfig) {
    return this.client.delete<T>(url, config)
  }
}

export const apiClient = new ApiClient()
```

```ts
// services/api/userApi.ts
import { apiClient } from './apiClient'
import type { User, CreateUserDTO, UpdateUserDTO } from '@/types'

/**
 * API de Usuario
 * Responsable de: todas las llamadas HTTP relacionadas con usuarios
 */

export const getUsers = () =>
  apiClient.get<{ data: User[] }>('/users')

export const getUserById = (id: number) =>
  apiClient.get<{ data: User }>(`/users/${id}`)

export const createUser = (data: CreateUserDTO) =>
  apiClient.post<{ data: User }>('/users', data)

export const updateUser = (id: number, data: UpdateUserDTO) =>
  apiClient.put<{ data: User }>(`/users/${id}`, data)

export const deleteUser = (id: number) =>
  apiClient.delete(`/users/${id}`)
```

---

## 2. BUENAS PRÁCTICAS DE DESARROLLO

### 2.1 TypeScript - Guía Básica

**MUST HAVE:**
- ✅ Tipado fuerte en props, emits y estados
- ✅ Usar `interface` o `type` para DTOs y modelos
- ✅ Nunca usar `any` - reemplazar con tipos genéricos o `unknown`
- ✅ Exportar tipos desde `types/index.ts`
- ✅ Documentar tipos complejos con comentarios JSDoc

**SHOULD HAVE:**
- 📌 Usar `type` para objetos simples, `interface` para objetos complejos
- 📌 Crear tipos genéricos para respuestas de API

**Ejemplo correcto:**
```ts
// types/index.ts
export interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user' | 'trainer'
  createdAt: Date
  updatedAt: Date
}

export type CreateUserDTO = Omit<User, 'id' | 'createdAt' | 'updatedAt'>
export type UpdateUserDTO = Partial<CreateUserDTO>

export interface ApiResponse<T> {
  data: T
  status: number
  message?: string
}
```

### 2.2 Vue 3 Composition API

**MUST HAVE:**
- ✅ Usar `<script setup lang="ts">` SIEMPRE
- ✅ Definir `Props` e `Emits` como interfaces tipadas
- ✅ Usar `ref` para estados mutables
- ✅ Usar `computed` para valores derivados
- ✅ Usar `watch` solo cuando sea necesario monitorear cambios
- ✅ Documentar composables con JSDoc

**SHOULD HAVE:**
- 📌 Preferir `shallowRef` si no necesitas reactividad profunda
- 📌 Usar `readonly` para exponer datos sin mutación
- 📌 Extraer lógica repetida a composables

**Ejemplo correcto:**
```vue
<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'

interface Props {
  initialCount?: number
}

const props = withDefaults(defineProps<Props>(), {
  initialCount: 0,
})

interface Emits {
  update: [value: number]
}

const emit = defineEmits<Emits>()

const count = ref(props.initialCount)
const doubled = computed(() => count.value * 2)

watch(() => props.initialCount, (newVal) => {
  count.value = newVal
})

const increment = () => {
  count.value++
  emit('update', count.value)
}

onMounted(() => {
  console.log('Component mounted with initial count:', props.initialCount)
})
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Doubled: {{ doubled }}</p>
    <button @click="increment">Increment</button>
  </div>
</template>
```

### 2.3 Manejo de Estados

**MUST HAVE:**
- ✅ Estados locales en `ref` dentro del componente
- ✅ Estados compartidos en composables
- ✅ Estados globales en store (Pinia) si es necesario
- ✅ Nunca mutar directamente props
- ✅ Usar `v-model` para bindings bidireccionales

**Estructura correcta:**
```
Local State (ref) → Componente individual
Shared State (Composable) → Múltiples componentes en el mismo flujo
Global State (Pinia) → Toda la aplicación
```

---

## 3. BUENAS PRÁCTICAS DE ARQUITECTURA

### 3.1 Separación de Responsabilidades

**MUST HAVE:**
- ✅ Views NO hacen llamadas HTTP directas
- ✅ Components NO contienen lógica de negocio
- ✅ Services NO importan Views ni Components
- ✅ Cada archivo tiene UNA responsabilidad principal
- ✅ Reutilizar código mediante composables

**Violación común - INCORRECTO:**
```vue
<!-- Views/Dashboard.vue - ¡NO HACER ESTO! -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import axios from 'axios'

const users = ref([])

// ❌ Lógica de negocio en la vista
onMounted(async () => {
  const response = await axios.get('http://api.com/users')
  users.value = response.data.filter(u => u.active)
  users.value.sort((a, b) => a.name.localeCompare(b.name))
})
</script>
```

**Corrección - HACER ESTO:**
```ts
// services/composables/useUser.ts
export function useUser() {
  const users = ref<User[]>([])

  const fetchActiveUsers = async () => {
    const response = await userApi.getUsers()
    users.value = response.data
      .filter(u => u.active)
      .sort((a, b) => a.name.localeCompare(b.name))
  }

  return { users, fetchActiveUsers }
}
```

```vue
<!-- Views/Dashboard.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'
import { useUser } from '@/services/composables/useUser'

const { users, fetchActiveUsers } = useUser()

onMounted(() => fetchActiveUsers())
</script>
```

### 3.2 Manejo de Errores y Notificaciones

**MUST HAVE:**
- ✅ Try-catch en toda operación async
- ✅ Notificaciones centralizadas (composable useNotification)
- ✅ Logging de errores para debugging
- ✅ Mensajes amigables al usuario

**Estructura:**
```ts
export function useNotification() {
  const notifications = ref<Notification[]>([])

  const showNotification = (
    message: string,
    type: 'success' | 'error' | 'warning' | 'info' = 'info',
    duration = 3000
  ) => {
    const id = Date.now()
    notifications.value.push({ id, message, type })

    if (duration > 0) {
      setTimeout(() => {
        notifications.value = notifications.value.filter(n => n.id !== id)
      }, duration)
    }
  }

  return { notifications, showNotification }
}
```

### 3.3 Importaciones y Rutas

**MUST HAVE:**
- ✅ Usar path aliases (@ para src/)
- ✅ Importaciones organizadas: Vue → Librerias → Locales
- ✅ Agrupar imports por tipo
- ✅ NO usar rutas relativas complejas

**Correcto:**
```ts
// Importaciones de Vue
import { ref, computed, watch } from 'vue'
import { useRouter } from 'vue-router'

// Importaciones de librerías externas
import axios from 'axios'

// Importaciones locales
import UserTable from '@/components/tables/UsersTable.vue'
import { useUser } from '@/services/composables/useUser'
import type { User } from '@/types'
```

---

## 4. REGLAS DE REVISIÓN DE CÓDIGO - MUST & SHOULD

### 4.1 MUST HAVE (Obligatorio - El código NO pasa sin esto)

**Arquitectura:**
- ❌ No mezclar llamadas HTTP en Views/Components
- ❌ No importar Views o Components en Services
- ❌ No crear props/emits sin tipos
- ❌ No mutar props directamente
- ❌ No usar lógica condicional complicada en templates

**TypeScript:**
- ❌ No usar `any`
- ❌ No dejar tipos implícitos en funciones públicas
- ❌ No exportar tipos sin documentar
- ❌ No mezclar types e interfaces sin consistencia

**Código Limpio:**
- ❌ No duplicar lógica entre archivos
- ❌ No usar nombres genéricos (data, info, result)
- ❌ No dejar console.log en código de producción

**Rendimiento:**
- ❌ No crear computed properties en bucles
- ❌ No usar v-for en listas sin key
- ❌ No renderizar listas grandes sin virtualización
- ❌ No crear watchers que nunca se limpian

**Documentación:**
- ❌ No dejar funciones públicas sin comentario JSDoc
- ❌ No dejar props/emits sin descripción de tipo
- ❌ No dejar lógica compleja sin comentarios explicativos

### 4.2 SHOULD HAVE (Recomendado - Mejorar con cada revisión)

**Arquitectura:**
- 📌 Usar composables para lógica reutilizable
- 📌 Centralizar configuraciones (API_URL, timeouts)
- 📌 Crear tipos compartidos en carpeta types/
- 📌 Mantener coherencia en estructura de componentes

**TypeScript:**
- 📌 Usar tipos genéricos en lugar de `unknown` cuando sea posible
- 📌 Crear tipos DTOs distintos de entidades de dominio
- 📌 Usar discriminated unions para tipos complejos

**Código Limpio:**
- 📌 Extraer métodos largos en funciones más pequeñas
- 📌 Usar nombres descriptivos (no u, i, x)
- 📌 Mantener máximo 3 niveles de anidación
- 📌 Usar early returns para simplificar condicionales

**Rendimiento:**
- 📌 Usar `shallowRef` cuando no necesites reactividad profunda
- 📌 Lazy load de componentes grandes
- 📌 Memoizar valores computados costosos

**Testing:**
- 📌 Escribir tests para composables críticos
- 📌 Usar mocks para servicios de API
- 📌 Mantener cobertura > 70%

---

## 5. CHECKLIST PREVIO A COMMIT

```markdown
### Arquitectura
- [ ] Las llamadas HTTP están en Services/Composables
- [ ] Views SOLO orquestan componentes y composables
- [ ] Components NO tienen lógica de negocio
- [ ] No hay imports cruzados entre capas prohibidas

### TypeScript
- [ ] Todos los tipos están explícitamente definidos
- [ ] Todos los props/emits están tipados
- [ ] No hay `any` en el código
- [ ] Los DTOs están separados de modelos de dominio

### Código Limpio
- [ ] Funciones menores a 30 líneas
- [ ] Componentes menores a 100 líneas
- [ ] No hay código duplicado
- [ ] Nombres descriptivos para variables/funciones
- [ ] No hay console.log en producción

### Documentación
- [ ] Composables tienen comentario JSDoc
- [ ] Props/Emits tienen descripción de tipo
- [ ] Lógica compleja tiene comentarios
- [ ] Archivo README.md actualizado si aplica

### Performance
- [ ] No hay computeds en bucles
- [ ] v-for tiene key único
- [ ] No hay watchers sin cleanup
- [ ] Listas grandes tienen virtualización

### Testing
- [ ] Composables nuevos tienen tests
- [ ] Services tienen tests unitarios
- [ ] Casos de error están contemplados
```

---

## 6. EJEMPLOS PRÁCTICOS COMPLETOS

### 6.1 Flujo Completo: Crear Usuario

**1. Tipos (types/index.ts):**
```ts
export interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user'
}

export type CreateUserDTO = Omit<User, 'id'>
```

**2. API Client (services/api/userApi.ts):**
```ts
import { apiClient } from './apiClient'
import type { User, CreateUserDTO } from '@/types'

export const createUser = (data: CreateUserDTO) =>
  apiClient.post<User>('/users', data)
```

**3. Composable (services/composables/useUser.ts):**
```ts
import { ref } from 'vue'
import * as userApi from '@/services/api/userApi'
import { useNotification } from './useNotification'
import type { User, CreateUserDTO } from '@/types'

export function useUser() {
  const loading = ref(false)
  const { showNotification } = useNotification()

  const createUser = async (data: CreateUserDTO): Promise<User> => {
    loading.value = true
    try {
      const response = await userApi.createUser(data)
      showNotification('Usuario creado exitosamente', 'success')
      return response.data
    } catch (error) {
      showNotification('Error al crear usuario', 'error')
      throw error
    } finally {
      loading.value = false
    }
  }

  return { loading, createUser }
}
```

**4. Component (components/forms/UserForm.vue):**
```vue
<script setup lang="ts">
import { ref } from 'vue'
import type { CreateUserDTO } from '@/types'

interface Props {
  loading?: boolean
}

interface Emits {
  submit: [data: CreateUserDTO]
}

withDefaults(defineProps<Props>(), { loading: false })
const emit = defineEmits<Emits>()

const form = ref<CreateUserDTO>({
  name: '',
  email: '',
  role: 'user',
})

const onSubmit = () => {
  if (form.value.name && form.value.email) {
    emit('submit', form.value)
  }
}
</script>

<template>
  <form @submit.prevent="onSubmit">
    <input v-model="form.name" placeholder="Nombre" required />
    <input v-model="form.email" type="email" placeholder="Email" required />
    <select v-model="form.role">
      <option value="user">Usuario</option>
      <option value="admin">Admin</option>
    </select>
    <button :disabled="loading">{{ loading ? 'Creando...' : 'Crear' }}</button>
  </form>
</template>
```

**5. View (views/user/UserManagement.vue):**
```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useUser } from '@/services/composables/useUser'
import UserForm from '@/components/forms/UserForm.vue'
import UserTable from '@/components/tables/UsersTable.vue'
import type { CreateUserDTO } from '@/types'

const { users, loading, fetchUsers, createUser } = useUser()
const showForm = ref(false)

onMounted(() => fetchUsers())

const handleUserCreate = async (data: CreateUserDTO) => {
  await createUser(data)
  await fetchUsers()
  showForm.value = false
}
</script>

<template>
  <div class="user-management">
    <h1>Gestión de Usuarios</h1>
    <button @click="showForm = !showForm">Nuevo Usuario</button>
    <UserForm v-if="showForm" :loading="loading" @submit="handleUserCreate" />
    <UserTable :users="users" :loading="loading" />
  </div>
</template>
```

---

## 7. PREFERENCIAS GENERALES

- Preferir TypeScript sobre JavaScript
- Preferir `<script setup lang="ts">` sobre `<script>`
- Preferir `shallowRef` sobre `ref` si no necesitas reactividad profunda
- Siempre usar Composition API
- Mantener componentes pequeños y enfocados
- Documentar código complejo

---

## 8. ERRORES COMUNES A EVITAR

| Error | ❌ INCORRECTO | ✅ CORRECTO |
|-------|-------------|-----------|
| Llamadas HTTP en Views | Hacer fetch en View | Usar composable en View |
| Lógica en Components | Componente con if/logic | Component solo renderiza |
| Props no tipados | `props: ['name']` | `defineProps<{ name: string }>()` |
| Sin manejo de errores | `await api.call()` | `try { await api.call() } catch (e) {}` |
| Código duplicado | Copiar código | Extraer a composable |
| Console.log en prod | `console.log(data)` | Remover antes de commit |
| Mutation de props | `props.value = new` | `emit('update', new)` |

---

## 9. QUICK REFERENCE

### Archivo Inicial (main.ts)
```ts
import { createApp } from 'vue'
import App from '@/App.vue'
import router from '@/router'

createApp(App)
  .use(router)
  .mount('#app')
```

### Composable Template
```ts
import { ref, computed } from 'vue'

/**
 * Descripción del composable
 * @returns {{ state, action, computed }}
 */
export function useFeature() {
  const state = ref<Type>(initialValue)
  const derived = computed(() => /* lógica */)
  
  const action = () => { /* implementación */ }

  return { state, derived, action }
}
```

### Component Template
```vue
<script setup lang="ts">
import { ref } from 'vue'
import type { Props } from '@/types'

interface Props { /* */ }
interface Emits { /* */ }

const props = withDefaults(defineProps<Props>(), { /* defaults */ })
const emit = defineEmits<Emits>()

// Logic
</script>

<template>
  <!-- Template -->
</template>

<style scoped>
/* Styles */
</style>
```

---

## 10. RECURSOS Y REFERENCIAS

- [Vue 3 Official Docs](https://vuejs.org/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [Clean Code Principles](https://www.oreilly.com/library/view/clean-code/9780136083238/)
- [Software Architecture Patterns](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/)
