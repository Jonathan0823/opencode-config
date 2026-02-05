---
name: ts-vue-svelte
description: TypeScript, Vue 3, and Svelte 5 patterns and best practices
---

# TypeScript Vue Svelte Skill

## Overview

This skill provides guidelines for Vue 3 and Svelte 5 development with TypeScript, focusing on composition API, stores, and type-safe patterns.

## Vue 3 Patterns

### 1. Composition API

```vue
<!-- DO: Use <script setup> with TypeScript -->
<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue';
import type { User, UserFilters } from '@/types';

// Props with type safety
interface Props {
  user: User;
  editable?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
  editable: false,
});

// Emits with type safety
const emit = defineEmits<{
  (e: 'update', user: User): void;
  (e: 'delete', id: string): void;
}>();

// Reactive state
const isEditing = ref(false);
const formData = ref<Partial<User>>({ ...props.user });
const errors = ref<Record<string, string>>({});

// Computed
const fullName = computed(() => {
  return `${props.user.firstName} ${props.user.lastName}`;
});

const isValid = computed(() => {
  return formData.value.email?.includes('@') && formData.value.firstName;
});

// Watch
watch(() => props.user, (newUser) => {
  formData.value = { ...newUser };
}, { deep: true });

// Methods
function handleSubmit() {
  if (!isValid.value) return;
  emit('update', formData.value as User);
  isEditing.value = false;
}

// Lifecycle
onMounted(() => {
  console.log('UserCard mounted:', props.user.id);
});
</script>
```

### 2. Composables

```typescript
// composables/useUsers.ts
import { ref, computed } from 'vue';
import type { User, UserFilters, Pagination } from '@/types';

interface UseUsersOptions {
  pageSize?: number;
  filters?: UserFilters;
}

export function useUsers(options: UseUsersOptions = {}) {
  const { pageSize = 10, filters = {} } = options;
  
  // State
  const users = ref<User[]>([]);
  const loading = ref(false);
  const error = ref<Error | null>(null);
  const pagination = ref<Pagination>({
    page: 1,
    total: 0,
    pageSize,
  });
  
  // Computed
  const hasMore = computed(() => {
    return pagination.value.page * pagination.value.pageSize < pagination.value.total;
  });
  
  // Methods
  async function fetchUsers() {
    loading.value = true;
    error.value = null;
    
    try {
      const response = await fetch('/api/users', {
        method: 'POST',
        body: JSON.stringify({
          page: pagination.value.page,
          pageSize: pagination.value.pageSize,
          filters,
        }),
      });
      
      if (!response.ok) throw new Error('Failed to fetch');
      
      const data = await response.json();
      users.value = data.users;
      pagination.value.total = data.total;
    } catch (e) {
      error.value = e as Error;
    } finally {
      loading.value = false;
    }
  }
  
  function nextPage() {
    if (hasMore.value) {
      pagination.value.page++;
      fetchUsers();
    }
  }
  
  function previousPage() {
    if (pagination.value.page > 1) {
      pagination.value.page--;
      fetchUsers();
    }
  }
  
  return {
    users: readonly(users),
    loading: readonly(loading),
    error: readonly(error),
    pagination: readonly(pagination),
    hasMore,
    fetchUsers,
    nextPage,
    previousPage,
  };
}

// composables/useLocalStorage.ts
import { ref, watch } from 'vue';

export function useLocalStorage<T>(key: string, defaultValue: T) {
  const stored = localStorage.getItem(key);
  const data = ref<T>(stored ? JSON.parse(stored) : defaultValue);
  
  watch(data, (newValue) => {
    localStorage.setItem(key, JSON.stringify(newValue));
  }, { deep: true });
  
  return data;
}
```

### 3. Pinia Store

```typescript
// stores/user.ts
import { defineStore } from 'pinia';
import type { User } from '@/types';

export const useUserStore = defineStore('user', {
  state: () => ({
    currentUser: null as User | null,
    isAuthenticated: false,
    loading: false,
    error: null as Error | null,
  }),
  
  getters: {
    isAdmin: (state) => state.currentUser?.role === 'admin',
    userName: (state) => state.currentUser?.name ?? 'Guest',
  },
  
  actions: {
    async login(email: string, password: string) {
      this.loading = true;
      this.error = null;
      
      try {
        const response = await fetch('/api/login', {
          method: 'POST',
          body: JSON.stringify({ email, password }),
        });
        
        if (!response.ok) throw new Error('Login failed');
        
        const data = await response.json();
        this.currentUser = data.user;
        this.isAuthenticated = true;
        localStorage.setItem('token', data.token);
      } catch (e) {
        this.error = e as Error;
      } finally {
        this.loading = false;
      }
    },
    
    logout() {
      this.currentUser = null;
      this.isAuthenticated = false;
      localStorage.removeItem('token');
    },
  },
});

// stores/index.ts - Setup store alternative
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';

export const useCounterStore = defineStore('counter', () => {
  // State
  const count = ref(0);
  
  // Getters
  const doubleCount = computed(() => count.value * 2);
  const isPositive = computed(() => count.value > 0);
  
  // Actions
  function increment() {
    count.value++;
  }
  
  function decrement() {
    count.value--;
  }
  
  return { count, doubleCount, isPositive, increment, decrement };
});
```

### 4. Component Patterns

```vue
<!-- DO: Scoped slots for flexible composition -->
<script setup lang="ts">
import { computed } from 'vue';
import type { TableColumn, TableItem } from '@/types';

interface Props<T extends TableItem> {
  items: T[];
  columns: TableColumn<T>[];
  loading?: boolean;
}

const props = defineProps<Props<any>>();

const sortedItems = computed(() => {
  // Sorting logic
  return props.items;
});
</script>

<template>
  <table class="data-table">
    <thead>
      <tr>
        <th v-for="col in columns" :key="col.key">
          {{ col.label }}
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="item in sortedItems" :key="item.id">
        <td v-for="col in columns" :key="col.key">
          <slot :name="col.key" :item="item" :value="item[col.key]">
            {{ item[col.key] }}
          </slot>
        </td>
      </tr>
    </tbody>
    <slot v-if="loading" name="loading">
      <tr><td :colspan="columns.length">Loading...</td></tr>
    </slot>
  </table>
</template>

<!-- Usage -->
<DataTable :items="users" :columns="userColumns">
  <template #status="{ item }">
    <StatusBadge :status="item.status" />
  </template>
  <template #actions="{ item }">
    <button @click="edit(item)">Edit</button>
  </template>
</DataTable>
```

## Svelte 5 Patterns

### 1. Runes (Svelte 5)

```svelte
<script lang="ts">
  import type { User } from './types';
  
  // Props
  interface Props {
    user: User;
    editable?: boolean;
  }
  
  let { user, editable = false }: Props = $props();
  
  // Reactive state with $state
  let isEditing = $state(false);
  let formData = $state({ ...user });
  
  // Derived state with $derived
  let fullName = $derived(`${user.firstName} ${user.lastName}`);
  let isValid = $derived(
    formData.email?.includes('@') && formData.firstName?.length > 0
  );
  
  // Effects with $effect
  $effect(() => {
    console.log('User changed:', user.id);
    // Cleanup function
    return () => {
      console.log('Cleaning up for user:', user.id);
    };
  });
  
  // Effect pre-run with $effect.pre (before DOM update)
  $effect.pre(() => {
    // Access DOM before it's updated
    const element = document.getElementById('user-form');
  });
  
  // Functions
  function handleSubmit() {
    if (!isValid) return;
    // Submit logic
    isEditing = false;
  }
</script>

<h2>{fullName}</h2>
{#if editable}
  <button onclick={() => isEditing = !isEditing}>
    {isEditing ? 'Cancel' : 'Edit'}
  </button>
{/if}
```

### 2. Stores

```typescript
// stores/user.ts
import { writable, derived, readonly } from 'svelte/store';
import type { User } from './types';

// Writable store
function createUserStore() {
  const { subscribe, set, update } = writable<User | null>(null);
  
  return {
    subscribe,
    set,
    update,
    login: async (email: string, password: string) => {
      const response = await fetch('/api/login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      });
      
      if (!response.ok) throw new Error('Login failed');
      
      const data = await response.json();
      set(data.user);
      localStorage.setItem('token', data.token);
    },
    logout: () => {
      set(null);
      localStorage.removeItem('token');
    },
  };
}

export const user = createUserStore();

// Derived store
export const isAdmin = derived(user, $user => $user?.role === 'admin');
export const isAuthenticated = derived(user, $user => $user !== null);

// Custom store
function createLocalStorage<T>(key: string, initialValue: T) {
  const stored = localStorage.getItem(key);
  const { subscribe, set } = writable<T>(
    stored ? JSON.parse(stored) : initialValue
  );
  
  return {
    subscribe,
    set: (value: T) => {
      localStorage.setItem(key, JSON.stringify(value));
      set(value);
    },
  };
}

export const theme = createLocalStorage<'light' | 'dark'>('theme', 'light');
```

### 3. Component Patterns

```svelte
<!-- DO: Component events -->
<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import type { User } from './types';
  
  interface Props {
    user: User;
    editable?: boolean;
  }
  
  let { user, editable = false }: Props = $props();
  
  const dispatch = createEventDispatcher<{
    update: User;
    delete: string;
  }>();
  
  function handleUpdate() {
    dispatch('update', user);
  }
  
  function handleDelete() {
    dispatch('delete', user.id);
  }
</script>

<div class="user-card">
  <h3>{user.name}</h3>
  {#if editable}
    <button onclick={handleUpdate}>Update</button>
    <button onclick={handleDelete}>Delete</button>
  {/if}
</div>

<!-- Usage -->
<UserCard
  {user}
  editable={true}
  on:update={(e) => console.log('Update:', e.detail)}
  on:delete={(e) => console.log('Delete:', e.detail)}
/>
```

```svelte
<!-- DO: Slot patterns -->
<script lang="ts">
  import type { Snippet } from 'svelte';
  
  interface Props {
    title: string;
    children: Snippet;
    actions?: Snippet;
  }
  
  let { title, children, actions }: Props = $props();
</script>

<div class="card">
  <header>
    <h2>{title}</h2>
    {#if actions}
      <div class="actions">
        {@render actions()}
      </div>
    {/if}
  </header>
  <div class="content">
    {@render children()}
  </div>
</div>

<!-- Usage -->
<Card title="User Profile">
  {#snippet actions()}
    <button>Edit</button>
    <button>Delete</button>
  {/snippet}
  
  <p>User content here</p>
</Card>
```

### 4. Transitions and Animations

```svelte
<script lang="ts">
  import { fade, fly, slide, scale } from 'svelte/transition';
  import { flip } from 'svelte/animate';
  import type { Todo } from './types';
  
  let todos = $state<Todo[]>([]);
  let showCompleted = $state(true);
  
  function addTodo(text: string) {
    todos = [...todos, { id: crypto.randomUUID(), text, completed: false }];
  }
  
  function toggleTodo(id: string) {
    todos = todos.map(t =>
      t.id === id ? { ...t, completed: !t.completed } : t
    );
  }
</script>

<ul>
  {#each todos.filter(t => showCompleted || !t.completed) as todo (todo.id)}
    <li
      animate:flip={{ duration: 200 }}
      transition:slide={{ duration: 300 }}
    >
      <label>
        <input
          type="checkbox"
          checked={todo.completed}
          onchange={() => toggleTodo(todo.id)}
        />
        <span class:completed={todo.completed}>
          {todo.text}
        </span>
      </label>
    </li>
  {/each}
</ul>

{#if todos.length === 0}
  <p transition:fade>No todos yet</p>
{/if}

<style>
  .completed {
    text-decoration: line-through;
    opacity: 0.6;
  }
</style>
```

## When to Use

Use this skill when:
- Building Vue 3 applications with Composition API
- Working with Svelte 5 and runes
- Creating reusable composables/composables
- Managing state with Pinia (Vue) or Svelte stores
- Designing component architecture
- TypeScript integration with Vue/Svelte
