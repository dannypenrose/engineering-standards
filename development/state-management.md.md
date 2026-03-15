# State Management Standards

> Authoritative state management standards for consistent, performant data handling across frontend applications.

## Purpose

Define clear patterns for where state lives, how it flows, and which tools manage it — eliminating the common source of complexity, bugs, and performance problems in frontend applications.

## Core Principles

1. **Categorise before choosing a tool** — Different state types need different solutions
2. **Server state is not client state** — Data from APIs needs specialised handling (caching, revalidation)
3. **Minimise global state** — Most state is local; lift only when truly shared
4. **URL is state** — If a user should be able to share or bookmark it, it belongs in the URL
5. **Single source of truth** — Each piece of state has exactly one owner

## State Categories

Every piece of state in an application falls into one of these categories. Categorise first, then choose the right tool.

| Category | Definition | Examples | Tool |
| -------- | ---------- | -------- | ---- |
| **Server state** | Data owned by the backend, cached locally | Users, products, API responses | TanStack Query, SWR |
| **Client state** | UI state not from the server | Theme, sidebar open/closed, selected tab | React state, Zustand, Context |
| **URL state** | State that should be shareable/bookmarkable | Filters, sort order, pagination, active tab | `useSearchParams`, nuqs |
| **Form state** | Ephemeral state during data entry | Field values, validation errors, dirty flags | React Hook Form |
| **Derived state** | Computed from other state | Filtered lists, totals, formatted values | `useMemo`, selectors |

### Decision Tree

```
Where does this data come from?
├── Server/API → Server State (TanStack Query)
│
├── User input in a form → Form State (React Hook Form)
│
├── Should this be in the URL (bookmarkable, shareable)?
│   └── Yes → URL State (useSearchParams / nuqs)
│
├── Is this shared across multiple components?
│   ├── No → Local State (useState)
│   └── Yes
│       ├── Parent + children only → Lift state up / Context
│       └── Many distant components → Client State Store (Zustand)
│
└── Is this computed from other state? → Derived State (useMemo)
```

## Server State

Server state is the most common source of state management complexity. Use a dedicated library — do not use `useState` + `useEffect` for API data.

### TanStack Query (React Query)

```typescript
// Query configuration
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,        // Data fresh for 5 min
      gcTime: 30 * 60 * 1000,           // Keep in cache 30 min
      retry: 2,
      refetchOnWindowFocus: false,
      refetchOnReconnect: true,
    },
  },
});

// Fetching data
function useUsers(filters: UserFilters) {
  return useQuery({
    queryKey: ['users', filters],
    queryFn: () => api.getUsers(filters),
    placeholderData: keepPreviousData,  // Keep old data while fetching new
  });
}

// Mutations with optimistic updates
function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: UpdateUserDto) => api.updateUser(data),
    onMutate: async (newData) => {
      await queryClient.cancelQueries({ queryKey: ['users', newData.id] });
      const previous = queryClient.getQueryData(['users', newData.id]);
      queryClient.setQueryData(['users', newData.id], (old: User) => ({
        ...old,
        ...newData,
      }));
      return { previous };
    },
    onError: (_err, _vars, context) => {
      queryClient.setQueryData(['users', context?.previous?.id], context?.previous);
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}
```

### Query Key Conventions

```typescript
// Structure: [entity, ...identifiers, ...filters]
const QUERY_KEYS = {
  users: {
    all:      ['users'] as const,
    lists:    () => [...QUERY_KEYS.users.all, 'list'] as const,
    list:     (filters: UserFilters) => [...QUERY_KEYS.users.lists(), filters] as const,
    details:  () => [...QUERY_KEYS.users.all, 'detail'] as const,
    detail:   (id: string) => [...QUERY_KEYS.users.details(), id] as const,
  },
  posts: {
    all:      ['posts'] as const,
    byUser:   (userId: string) => [...QUERY_KEYS.posts.all, 'user', userId] as const,
  },
};

// Usage
useQuery({ queryKey: QUERY_KEYS.users.detail(userId), ... });

// Invalidate all user queries
queryClient.invalidateQueries({ queryKey: QUERY_KEYS.users.all });
```

## Client State

### Local State (useState)

Use for state that belongs to a single component or small component tree:

```typescript
// Toggle, selection, local UI state
const [isOpen, setIsOpen] = useState(false);
const [selectedTab, setSelectedTab] = useState<'overview' | 'details'>('overview');
```

### Zustand (Shared Client State)

Use for client state shared across many components that is not server data and not URL state:

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface UIStore {
  sidebarOpen: boolean;
  theme: 'light' | 'dark' | 'system';
  toggleSidebar: () => void;
  setTheme: (theme: 'light' | 'dark' | 'system') => void;
}

export const useUIStore = create<UIStore>()(
  persist(
    (set) => ({
      sidebarOpen: true,
      theme: 'system',
      toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
      setTheme: (theme) => set({ theme }),
    }),
    { name: 'ui-preferences' } // Persists to localStorage
  )
);

// Usage — only subscribe to what you need (automatic re-render optimization)
const sidebarOpen = useUIStore((s) => s.sidebarOpen);
const toggleSidebar = useUIStore((s) => s.toggleSidebar);
```

### React Context (Scoped Shared State)

Use for state shared within a subtree (parent and its descendants), not for global state:

```typescript
// Good use: theme, locale, feature flags, auth state (within a provider)
const AuthContext = createContext<AuthState | null>(null);

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}
```

**When NOT to use Context:**
- Data that changes frequently (causes re-renders for all consumers)
- Large state objects (every consumer re-renders on any change)
- State that most components don't need (use Zustand with selectors instead)

## URL State

If a user should be able to share a link and see the same view, that state belongs in the URL.

```typescript
// Next.js with useSearchParams
import { useSearchParams, useRouter, usePathname } from 'next/navigation';

function useFilters() {
  const searchParams = useSearchParams();
  const router = useRouter();
  const pathname = usePathname();

  const filters = {
    search: searchParams.get('q') ?? '',
    status: searchParams.get('status') ?? 'all',
    page: parseInt(searchParams.get('page') ?? '1', 10),
    sort: searchParams.get('sort') ?? 'createdAt',
  };

  const setFilter = (key: string, value: string) => {
    const params = new URLSearchParams(searchParams.toString());
    if (value) {
      params.set(key, value);
    } else {
      params.delete(key);
    }
    params.set('page', '1'); // Reset to page 1 on filter change
    router.push(`${pathname}?${params.toString()}`);
  };

  return { filters, setFilter };
}
```

### URL State Rules

- Filters, sort order, pagination, search terms → URL params
- Selected tab or accordion → URL param if the tab content is the main focus
- Modal open/closed → URL param only if it's a significant view (e.g., `/users?edit=123`)
- Ephemeral hover/focus state → never in URL

## Form State

### React Hook Form

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  email: z.string().email('Invalid email'),
  role: z.enum(['admin', 'user', 'viewer']),
});

type FormData = z.infer<typeof schema>;

function UserForm({ user, onSubmit }: Props) {
  const { register, handleSubmit, formState: { errors, isDirty, isSubmitting } } = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: {
      name: user?.name ?? '',
      email: user?.email ?? '',
      role: user?.role ?? 'user',
    },
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      {errors.name && <span>{errors.name.message}</span>}

      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}

      <button type="submit" disabled={!isDirty || isSubmitting}>
        Save
      </button>
    </form>
  );
}
```

## Performance

### Avoiding Unnecessary Re-Renders

```typescript
// ❌ Bad: Object created every render, causes child re-render
<UserList filters={{ status: 'active', page: 1 }} />

// ✅ Good: Memoize objects passed as props
const filters = useMemo(() => ({ status: 'active', page: 1 }), []);
<UserList filters={filters} />

// ❌ Bad: Reading entire store (re-renders on any store change)
const store = useUIStore();

// ✅ Good: Select only what you need
const theme = useUIStore((s) => s.theme);
```

### State Colocation

Keep state as close to where it is used as possible:

```
                  Global Store (Zustand)
                  Only for truly global UI state
                          │
                  Context Provider
                  For subtree-scoped state (auth, theme)
                          │
                  Page/Route Component
                  URL state, server state queries
                          │
                  Feature Component
                  Local UI state, form state
                          │
                  Leaf Component
                  Derived/computed values only
```

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
| ------------ | ------- | ---------------- |
| `useState` + `useEffect` for API data | No caching, no loading/error states, no deduplication | TanStack Query |
| Global Redux store for everything | Unnecessary complexity, re-render issues | Categorise state, use right tool |
| Prop drilling through 5+ levels | Fragile, hard to refactor | Context or Zustand |
| Duplicating server data in client store | Two sources of truth, sync bugs | TanStack Query is the cache |
| Storing derived data | Can go stale, wastes memory | `useMemo` or selectors |
| Filters in `useState` instead of URL | Not shareable, lost on refresh | `useSearchParams` |

## Checklist

### Architecture

- [ ] Every piece of state categorised (server, client, URL, form, derived)
- [ ] Server state managed by TanStack Query (not `useState` + `useEffect`)
- [ ] URL state used for filters, pagination, sort, search
- [ ] Form state managed by React Hook Form with Zod validation
- [ ] Client state in Zustand (global) or Context (scoped subtree)
- [ ] No duplicated state (single source of truth per datum)

### Performance

- [ ] Zustand stores use selectors (not full store subscription)
- [ ] Objects/arrays passed as props are memoised
- [ ] State colocated as close to usage as possible
- [ ] TanStack Query `staleTime` configured (not default 0)

### Testing

- [ ] Server state: mock at the API layer, not the query layer
- [ ] Client state: test through component interaction, not store internals
- [ ] URL state: test by checking URL params after user interaction
- [ ] Form state: test validation rules and submission flow

## References

- [TanStack Query Documentation](https://tanstack.com/query/latest)
- [Zustand Documentation](https://zustand-demo.pmnd.rs/)
- [React Hook Form Documentation](https://react-hook-form.com/)
- [Kent C. Dodds — Application State Management with React](https://kentcdodds.com/blog/application-state-management-with-react)
- [Bulletproof React — State Management](https://github.com/alan2207/bulletproof-react)
