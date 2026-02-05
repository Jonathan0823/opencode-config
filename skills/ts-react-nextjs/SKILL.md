---
name: ts-react-nextjs
description: TypeScript, React 19, and Next.js 15 patterns and best practices
---

# TypeScript React Next.js Skill

## Overview

This skill provides guidelines for modern React 19 and Next.js 15 development with TypeScript, focusing on Server Components, the App Router, and type-safe patterns.

## Core Principles

### 1. TypeScript Fundamentals

```typescript
// DO: Use strict TypeScript configuration
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}

// DO: Define explicit types for props
interface UserCardProps {
  user: {
    id: string;
    name: string;
    email: string;
    avatar?: string;
  };
  onSelect?: (id: string) => void;
  isLoading?: boolean;
}

// DO: Use discriminated unions for complex state
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function DataDisplay<T>({ state }: { state: RequestState<T> }) {
  switch (state.status) {
    case 'idle': return <p>Ready to load</p>;
    case 'loading': return <Spinner />;
    case 'success': return <DataView data={state.data} />;
    case 'error': return <ErrorMessage error={state.error} />;
  }
}
```

### 2. React 19 Patterns

```tsx
// DO: Use Server Components by default (no 'use client')
// app/users/page.tsx - Server Component
async function UsersPage() {
  const users = await fetchUsers(); // Direct fetch, no useEffect
  return (
    <div>
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}

// DO: Mark Client Components explicitly
'use client';

import { useState } from 'react';

function SearchInput({ onSearch }: { onSearch: (query: string) => void }) {
  const [query, setQuery] = useState('');
  
  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      onKeyDown={(e) => e.key === 'Enter' && onSearch(query)}
    />
  );
}

// DO: Use new React 19 'use' hook
import { use, Suspense } from 'react';

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

function UserProfileWrapper() {
  return (
    <Suspense fallback={<Skeleton />}>
      <UserProfile userPromise={fetchUser()} />
    </Suspense>
  );
}
```

### 3. Next.js 15 App Router Patterns

```tsx
// DO: Use proper file structure
// app/
// ├── layout.tsx          # Root layout
// ├── page.tsx            # Home page
// ├── loading.tsx         # Loading UI
// ├── error.tsx           # Error boundary
// ├── not-found.tsx       # 404 page
// ├── users/
// │   ├── page.tsx        # /users
// │   ├── layout.tsx      # Users layout
// │   ├── loading.tsx     # Users loading state
// │   ├── [id]/
// │   │   ├── page.tsx    # /users/:id
// │   │   └── edit/
// │   │       └── page.tsx # /users/:id/edit
// │   └── new/
// │       └── page.tsx    # /users/new
// └── api/
//     └── users/
//         └── route.ts    # API route

// DO: Dynamic routes with type-safe params
// app/users/[id]/page.tsx
interface PageProps {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
}

async function UserPage({ params, searchParams }: PageProps) {
  const { id } = await params;
  const { tab } = await searchParams;
  
  const user = await fetchUser(id);
  return <UserProfile user={user} activeTab={tab} />;
}

// DO: API Routes
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = searchParams.get('page') ?? '1';
  
  const users = await fetchUsers({ page: parseInt(page) });
  return NextResponse.json(users);
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  
  const validated = userSchema.safeParse(body);
  if (!validated.success) {
    return NextResponse.json(
      { errors: validated.error.flatten() },
      { status: 400 }
    );
  }
  
  const user = await createUser(validated.data);
  return NextResponse.json(user, { status: 201 });
}

// DO: Parallel data fetching
async function DashboardPage() {
  const [usersPromise, ordersPromise, statsPromise] = [
    fetchUsers(),
    fetchOrders(),
    fetchStats(),
  ];
  
  return (
    <>
      <Suspense fallback={<UsersSkeleton />}>
        <UsersPanel usersPromise={usersPromise} />
      </Suspense>
      <Suspense fallback={<OrdersSkeleton />}>
        <OrdersPanel ordersPromise={ordersPromise} />
      </Suspense>
      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel statsPromise={statsPromise} />
      </Suspense>
    </>
  );
}
```

### 4. State Management

```tsx
// DO: Server State with React Query/TanStack Query
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}

function useCreateUser() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: createUser,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}

// DO: Form State with React Hook Form + Zod
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const userSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email'),
  age: z.number().min(18, 'Must be 18+').optional(),
});

type UserFormData = z.infer<typeof userSchema>;

function UserForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<UserFormData>({
    resolver: zodResolver(userSchema),
  });
  
  const createUser = useCreateUser();
  
  return (
    <form onSubmit={handleSubmit((data) => createUser.mutate(data))}>
      <input {...register('name')} />
      {errors.name && <span>{errors.name.message}</span>}
      
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}
      
      <input type="number" {...register('age', { valueAsNumber: true })} />
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create'}
      </button>
    </form>
  );
}

// DO: Client State with Zustand (minimal)
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface UIState {
  sidebarOpen: boolean;
  theme: 'light' | 'dark';
  toggleSidebar: () => void;
  setTheme: (theme: 'light' | 'dark') => void;
}

const useUIStore = create<UIState>()(
  devtools(
    persist(
      (set) => ({
        sidebarOpen: true,
        theme: 'light',
        toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
        setTheme: (theme) => set({ theme }),
      }),
      { name: 'ui-storage' }
    )
  )
);
```

### 5. Error Handling and Loading

```tsx
// DO: Error boundaries with error.tsx
// app/users/error.tsx
'use client';

import { useEffect } from 'react';

export default function ErrorBoundary({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    console.error('Error:', error);
  }, [error]);
  
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// DO: Loading states with loading.tsx
// app/users/loading.tsx
export default function Loading() {
  return (
    <div className="animate-pulse">
      <div className="h-8 bg-gray-200 rounded mb-4"></div>
      <div className="space-y-3">
        {[1, 2, 3].map((i) => (
          <div key={i} className="h-20 bg-gray-200 rounded"></div>
        ))}
      </div>
    </div>
  );
}

// DO: Not found pages
// app/users/[id]/not-found.tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <div>
      <h2>User Not Found</h2>
      <p>Could not find the requested user.</p>
      <Link href="/users">Return to Users</Link>
    </div>
  );
}

// DO: Using notFound()
import { notFound } from 'next/navigation';

async function UserPage({ params }: { params: { id: string } }) {
  const user = await fetchUser(params.id);
  
  if (!user) {
    notFound();
  }
  
  return <UserProfile user={user} />;
}
```

### 6. Performance Patterns

```tsx
// DO: Code splitting with dynamic imports
import { Suspense, lazy } from 'react';

const HeavyChart = lazy(() => import('./HeavyChart'));

function Dashboard() {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <HeavyChart />
    </Suspense>
  );
}

// DO: Memoization for expensive calculations
'use client';

import { useMemo, memo } from 'react';

function ExpensiveList({ items, filter }: { items: Item[]; filter: string }) {
  const filtered = useMemo(() => {
    return items.filter(item => item.name.includes(filter));
  }, [items, filter]);
  
  return (
    <ul>
      {filtered.map(item => (
        <ListItem key={item.id} item={item} />
      ))}
    </ul>
  );
}

// DO: Memoize components that receive stable props
const UserCard = memo(function UserCard({ user }: { user: User }) {
  return (
    <div>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
});
```

## When to Use

Use this skill when:
- Building React 19 applications
- Working with Next.js 15 App Router
- Setting up TypeScript configuration
- Designing component architecture
- Implementing Server Components
- Managing state (server, form, client)
- Optimizing performance
