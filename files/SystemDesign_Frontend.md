# Frontend System Design — Complete Guide
### Every Concept, Pattern & Trade-off You Need

---

## Table of Contents

1. [What is Frontend System Design?](#1-what-is-frontend-system-design)
2. [Rendering Strategies](#2-rendering-strategies)
3. [Application Architecture Patterns](#3-application-architecture-patterns)
4. [Component Architecture & Design Systems](#4-component-architecture--design-systems)
5. [State Management at Scale](#5-state-management-at-scale)
6. [Data Fetching Patterns](#6-data-fetching-patterns)
7. [Caching Strategies](#7-caching-strategies)
8. [Performance Optimization](#8-performance-optimization)
9. [Code Splitting & Lazy Loading](#9-code-splitting--lazy-loading)
10. [Micro-Frontends](#10-micro-frontends)
11. [Real-Time Communication](#11-real-time-communication)
12. [CDN & Asset Delivery](#12-cdn--asset-delivery)
13. [Security](#13-security)
14. [Authentication & Authorization](#14-authentication--authorization)
15. [API Design from the Frontend](#15-api-design-from-the-frontend)
16. [Accessibility (a11y)](#16-accessibility-a11y)
17. [Internationalization (i18n)](#17-internationalization-i18n)
18. [Build Systems & Bundlers](#18-build-systems--bundlers)
19. [Progressive Web Apps (PWA)](#19-progressive-web-apps-pwa)
20. [Testing Strategy](#20-testing-strategy)
21. [Monitoring & Observability](#21-monitoring--observability)
22. [Error Handling at Scale](#22-error-handling-at-scale)
23. [Analytics & Tracking Architecture](#23-analytics--tracking-architecture)
24. [Scalability Patterns](#24-scalability-patterns)
25. [Design Interview: How to Answer](#25-design-interview-how-to-answer)
26. [Classic FE System Design Problems — Solved](#26-classic-fe-system-design-problems--solved)

---

## 1. What is Frontend System Design?

Frontend system design is the process of defining **the architecture, components, modules, interfaces, and data flow** of a large-scale frontend application. It covers everything from how pixels render on screen to how 10 million users simultaneously read and write data without the app falling apart.

### What interviewers actually evaluate

```
❌ They don't care about: which CSS framework you use
✓ They care about:
  - How you decompose a complex problem into manageable pieces
  - How you handle scale (10M users, 50 engineers, 5 platforms)
  - The trade-offs you choose and WHY
  - Performance, reliability, security, maintainability
```

### The 4 Pillars of FE System Design

```
┌─────────────────┬─────────────────┬──────────────────┬──────────────────┐
│   PERFORMANCE   │   SCALABILITY   │   RELIABILITY    │  MAINTAINABILITY │
│                 │                 │                  │                  │
│ How fast does   │ How does it     │ How does it      │ How do 50+ devs  │
│ it load/run?    │ handle 10M      │ behave when      │ work on it       │
│                 │ users?          │ things fail?     │ without chaos?   │
└─────────────────┴─────────────────┴──────────────────┴──────────────────┘
```

---

## 2. Rendering Strategies

Choosing the right rendering strategy is one of the most impactful architectural decisions. It affects SEO, performance, infrastructure cost, and developer experience.

---

### CSR — Client-Side Rendering

```
Browser downloads empty HTML → Downloads JS bundle → JS runs → Renders UI

Server response:
<html><body><div id="root"></div><script src="bundle.js"></script></body></html>

Nothing visible until JS executes.
```

**How it works:**
1. Server sends a nearly empty HTML shell
2. Browser downloads the JS bundle (can be 500KB - 5MB)
3. JS executes, React hydrates, UI renders
4. Subsequent navigation: instant (no server roundtrip, client-side routing)

**Performance profile:**
```
TTFB (Time to First Byte):  Fast  (server just sends HTML shell)
FCP  (First Contentful Paint): Slow  (waits for JS to download + run)
TTI  (Time to Interactive):  Slow  (same as FCP for CSR)
Subsequent navigation:        Fast  (client-side routing)
```

**When to use:**
- Dashboards behind authentication (SEO doesn't matter)
- Highly interactive apps (admin panels, SaaS tools, figma-like tools)
- When you have a BFF or GraphQL layer

**When NOT to use:**
- Public-facing pages (blog, landing page, e-commerce) — SEO suffers
- Users on slow networks / low-end devices
- Core Web Vitals are a priority

---

### SSR — Server-Side Rendering

```
Browser requests page → Server runs JS, renders HTML → Sends full HTML → Browser displays
→ JS downloads → React hydrates (attaches event listeners)
```

**How it works:**
1. Server receives request
2. Server fetches data (DB / API)
3. Server renders React components to HTML string
4. Sends full HTML — user sees content immediately
5. JS downloads in background
6. React "hydrates" — attaches event handlers to server-rendered HTML

**Performance profile:**
```
TTFB:  Slower (server does work before responding)
FCP:   Fast   (HTML has content, no JS needed to see UI)
TTI:   Medium (must wait for hydration)
Subsequent navigation: Slower than CSR (needs server roundtrip)
```

**The Hydration Problem:**
```
Server renders: <button>Count: 0</button>
Client hydrates: React takes over, attaches onClick, syncs state

Problem: If server HTML doesn't match client render → hydration mismatch error
         React discards server HTML and re-renders entirely (worse than CSR)
```

**When to use:**
- E-commerce product pages (SEO critical + personalization)
- Social media feeds (content changes per user)
- News articles (SEO + fresh content)
- Any page where first-load experience matters AND content is dynamic

---

### SSG — Static Site Generation

```
BUILD TIME: Server renders all pages → Saves as static HTML files
REQUEST TIME: CDN serves the pre-built HTML instantly
```

**How it works:**
1. At build time, framework generates all pages as `.html` files
2. These files are uploaded to CDN
3. User requests a page → CDN serves the file immediately (no server computation)
4. Fastest possible TTFB and FCP

**Performance profile:**
```
TTFB:  Fastest (CDN edge, no computation)
FCP:   Fastest
TTI:   Fast
Freshness: Stale (requires rebuild to update)
```

**When to use:**
- Marketing pages, landing pages
- Documentation sites
- Blogs with infrequent updates
- Pages where data doesn't change per user

**When NOT to use:**
- User-specific content (dashboards, feeds)
- Frequently updating data (stock prices, live scores)
- Sites with 100,000+ pages (build time becomes hours)

---

### ISR — Incremental Static Regeneration

```
First request → CDN has no cache → Server renders → Saves to CDN
Next request (within 60s) → CDN serves cached HTML (stale-while-revalidate)
After 60s → Next request triggers background re-render → Cache updated
```

**How it works:**
- Hybrid of SSG and SSR
- Pages are statically generated but can be **revalidated in the background** after a TTL
- Uses `stale-while-revalidate` strategy: serve old content immediately, refresh in background

```javascript
// Next.js ISR
export async function getStaticProps() {
  const product = await fetchProduct(id);
  return {
    props: { product },
    revalidate: 60, // re-generate this page at most once every 60 seconds
  };
}
```

**When to use:**
- Product pages (mostly static but price/stock changes)
- Blog posts (stable but can be updated)
- Any page where "slightly stale" is acceptable

---

### Partial Hydration & Islands Architecture

**Problem with SSR:** You SSR the entire page, then hydrate the entire page. But 80% of the page might be static text that doesn't need JS.

**Islands Architecture (Astro, Fresh):**
```
┌────────────────────────────────────────────────┐
│  Static HTML (no JS, no hydration needed)       │
│                                                 │
│  ┌──────────────┐     ┌────────────────────┐   │
│  │  Interactive │     │    Interactive     │   │
│  │   Island     │     │      Island        │   │
│  │ (hydrated)   │     │    (hydrated)      │   │
│  └──────────────┘     └────────────────────┘   │
│                                                 │
│  Static HTML continues...                       │
└────────────────────────────────────────────────┘
```

Only the "islands" (interactive components) are hydrated with JS. Everything else is static HTML. Result: 90% less JS shipped.

---

### Streaming SSR (React 18)

Traditional SSR blocks the entire response until all data is fetched. Streaming SSR sends HTML in **chunks** as they're ready.

```
Traditional SSR:
  Wait for ALL data → render ALL HTML → send response
  User waits for the slowest API call

Streaming SSR:
  Send HTML shell immediately →
  Stream header HTML →
  Stream above-fold content →
  Stream below-fold content (as data arrives)
  User sees content progressively
```

```jsx
// React 18 Suspense boundaries enable streaming
<Suspense fallback={<HeaderSkeleton />}>
  <Header />          {/* streamed first */}
</Suspense>

<Suspense fallback={<FeedSkeleton />}>
  <NewsFeed />        {/* streamed when data ready */}
</Suspense>

<Suspense fallback={<SidebarSkeleton />}>
  <Recommendations /> {/* streamed last (slowest API) */}
</Suspense>
```

---

### Rendering Strategy Decision Matrix

| Scenario | Strategy | Why |
|---|---|---|
| Marketing landing page | SSG | SEO + fastest load, rarely changes |
| Blog post | SSG or ISR | SEO + content rarely changes |
| Product listing page | ISR (60s) | SEO + price/stock changes |
| Product detail page | ISR (30s) | SEO + inventory updates |
| User dashboard | CSR | Personalized, SEO irrelevant |
| News feed | SSR | Fresh content, SEO matters |
| Admin panel | CSR | No SEO, highly interactive |
| E-commerce cart | CSR | Personalized, real-time |
| Search results | SSR | SEO + dynamic results |

---

## 3. Application Architecture Patterns

### Monolithic Frontend

```
One codebase → one build → one deployment

Example: A single Next.js app that serves all pages

Pros:
  - Simple to develop, build, deploy
  - Easy to share code between features
  - One CI/CD pipeline

Cons:
  - One team's changes can break everything
  - Bundle grows with every feature
  - Slow builds as app grows
  - All teams deploy together (coordination bottleneck)
```

---

### Layered Architecture (Most Common)

```
┌────────────────────────────────────────────┐
│            PRESENTATION LAYER              │
│   Components, Pages, Layouts, Styles       │
│   (knows nothing about data fetching)      │
├────────────────────────────────────────────┤
│            APPLICATION LAYER               │
│   Business logic, state management,        │
│   use cases, event handlers                │
├────────────────────────────────────────────┤
│            DATA ACCESS LAYER               │
│   API clients, data fetching hooks,        │
│   cache management, transformers           │
├────────────────────────────────────────────┤
│            INFRASTRUCTURE LAYER            │
│   HTTP client (axios), storage,            │
│   analytics, error reporting               │
└────────────────────────────────────────────┘
```

**Why this matters:** If you mix API calls directly inside components, you can't test them, can't reuse them, and can't replace the API without touching every component.

---

### Feature-Based Architecture (Vertical Slicing)

Instead of organising by type (all components together, all hooks together), organise by **feature**.

```
❌ Type-based (doesn't scale):
src/
  components/
    UserCard.tsx
    ProductCard.tsx
    OrderCard.tsx
  hooks/
    useUser.ts
    useProducts.ts
    useOrders.ts
  api/
    userApi.ts
    productApi.ts
    orderApi.ts

✓ Feature-based (scales well):
src/
  features/
    auth/
      components/LoginForm.tsx
      hooks/useAuth.ts
      api/authApi.ts
      store/authSlice.ts
      types/auth.types.ts
    products/
      components/ProductCard.tsx
      hooks/useProducts.ts
      api/productApi.ts
    orders/
      components/OrderCard.tsx
      hooks/useOrders.ts
      api/orderApi.ts
  shared/
    components/Button.tsx (used across features)
    utils/
    types/
```

**The rule:** A feature folder should be **deletable** without breaking other features.

---

### Flux / Unidirectional Data Flow

The architectural pattern behind Redux, Zustand, Context — all of React's state management.

```
┌──────────┐    action     ┌──────────┐    new state   ┌──────────┐
│          │ ──────────▶   │          │ ─────────────▶  │          │
│   View   │               │  Store   │                 │   View   │
│          │ ◀────────────  │ (Reducer)│ ◀─────────────  │          │
└──────────┘   re-render   └──────────┘   dispatch      └──────────┘
```

- **Action:** Describes WHAT happened (`{ type: 'ADD_TO_CART', payload: product }`)
- **Reducer:** Pure function — given current state + action, returns new state
- **Store:** Holds the state, notifies subscribers when it changes
- **View:** Reads from store, dispatches actions on user interaction

**Why unidirectional?** Bidirectional data flow (two-way binding) makes it impossible to trace where a state change originated. Unidirectional flow means you can always follow the action → reducer → state → view chain.

---

### Atomic Design (Component Architecture Philosophy)

```
Atoms → Molecules → Organisms → Templates → Pages

Atoms: smallest UI units (Button, Input, Icon, Label)
         ↓ compose into
Molecules: simple groups of atoms (SearchBar = Input + Button)
         ↓ compose into
Organisms: complex sections (Header = Logo + Nav + SearchBar + UserMenu)
         ↓ compose into
Templates: page layouts (no real data, just slots)
         ↓ filled with real data to make
Pages: final rendered views
```

---

## 4. Component Architecture & Design Systems

### What is a Design System?

A design system is a **single source of truth** for UI: tokens, components, patterns, and guidelines that all teams use to build consistent UIs.

```
Design System
├── Tokens (variables)
│   ├── Colors: primary-500 = #6200EE
│   ├── Typography: heading-xl = 32px, bold, -0.5px
│   ├── Spacing: space-4 = 16px
│   ├── Border radius: radius-md = 8px
│   └── Shadows: shadow-card = 0 2px 8px rgba(0,0,0,0.1)
├── Components
│   ├── Primitive (Button, Input, Badge)
│   ├── Composite (Modal, Dropdown, DatePicker)
│   └── Layout (Grid, Stack, Divider)
├── Patterns
│   ├── Forms (validation, error states)
│   ├── Empty states
│   └── Loading states (skeleton screens)
└── Documentation (Storybook)
```

---

### Component API Design

A well-designed component API is the difference between a component that's used everywhere and one that's forked into 5 slightly different versions.

#### Composition over Configuration
```tsx
// ❌ Prop explosion — one config prop per feature
<Button
  hasIcon
  iconName="arrow"
  iconPosition="left"
  hasLoading
  loadingText="Saving..."
  hasTooltip
  tooltipText="Save your work"
/>

// ✓ Composable — consumers control the structure
<Button>
  <Icon name="arrow" />
  Save
</Button>
```

#### Polymorphic Components (The `as` prop)
```tsx
// The component renders as whatever element you tell it to
interface ButtonProps<T extends React.ElementType = 'button'> {
  as?: T;
  children: React.ReactNode;
  variant?: 'primary' | 'secondary';
}

const Button = <T extends React.ElementType = 'button'>({
  as,
  children,
  variant = 'primary',
  ...rest
}: ButtonProps<T> & React.ComponentPropsWithoutRef<T>) => {
  const Component = as ?? 'button';
  return <Component className={styles[variant]} {...rest}>{children}</Component>;
};

// Usage:
<Button>Click me</Button>                      // renders <button>
<Button as="a" href="/signup">Sign up</Button>  // renders <a href="/signup">
<Button as={Link} to="/home">Home</Button>      // renders React Router Link
```

#### Controlled vs Uncontrolled Components
```tsx
// Uncontrolled: component manages its own state
const UncontrolledInput = ({ defaultValue, onChange }) => {
  const [value, setValue] = useState(defaultValue ?? '');
  const handleChange = (e) => {
    setValue(e.target.value);
    onChange?.(e.target.value);
  };
  return <input value={value} onChange={handleChange} />;
};

// Controlled: parent manages state
const ControlledInput = ({ value, onChange }) => (
  <input value={value} onChange={(e) => onChange(e.target.value)} />
);

// ✓ Best: support BOTH (like Radix UI, Headless UI)
const SmartInput = ({ value: controlledValue, defaultValue, onChange }) => {
  const [uncontrolledValue, setUncontrolledValue] = useState(defaultValue ?? '');
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : uncontrolledValue;

  const handleChange = (newValue) => {
    if (!isControlled) setUncontrolledValue(newValue);
    onChange?.(newValue);
  };

  return <input value={value} onChange={(e) => handleChange(e.target.value)} />;
};
```

---

### Headless Components / Renderless Architecture

Separate **behaviour** from **appearance**. The component provides logic and accessibility — you provide the markup.

```tsx
// Headless Dropdown (logic only, no styles)
const { isOpen, selectedItem, getToggleProps, getMenuProps, getItemProps } =
  useDropdown({ items, onChange });

// Consumer decides how it looks:
<div>
  <button {...getToggleProps()}>
    {selectedItem?.label ?? 'Select...'}
  </button>
  {isOpen && (
    <ul {...getMenuProps()} className="my-custom-menu">
      {items.map((item, i) => (
        <li key={item.id} {...getItemProps({ item, index: i })}>
          {item.label}
        </li>
      ))}
    </ul>
  )}
</div>

// Used by: Radix UI, Headless UI, Downshift, React Aria
```

**Why this matters at scale:** Your design team can restyle the dropdown without touching any logic. Your engineering team can update the keyboard navigation without touching any styles.

---

## 5. State Management at Scale

### Types of State

Every piece of data in your app falls into one of these categories — and each needs a different solution:

```
┌─────────────────────────────────────────────────────────────────┐
│  STATE TYPE          │  WHAT IT IS            │  TOOL           │
├─────────────────────────────────────────────────────────────────┤
│  Local UI State      │  Modal open/closed,    │  useState        │
│                      │  form input value,     │  useReducer      │
│                      │  tooltip hover         │                  │
├─────────────────────────────────────────────────────────────────┤
│  Shared UI State     │  Selected tab, active  │  Zustand         │
│                      │  sidebar item, theme   │  Context         │
├─────────────────────────────────────────────────────────────────┤
│  Server State        │  Data from API,        │  React Query     │
│                      │  paginated lists,      │  SWR             │
│                      │  user profile          │  Apollo (GQL)    │
├─────────────────────────────────────────────────────────────────┤
│  URL / Navigation    │  Current route,        │  React Router    │
│  State               │  query params,         │  Next.js Router  │
│                      │  pagination page       │                  │
├─────────────────────────────────────────────────────────────────┤
│  Form State          │  Field values,         │  React Hook Form │
│                      │  validation errors,    │  Formik          │
│                      │  submission state      │                  │
└─────────────────────────────────────────────────────────────────┘
```

**Most common mistake:** Using Redux/Zustand for **server state**. Server state (API data) has unique properties — it's async, can go stale, is owned by the server — and needs a dedicated tool like React Query.

---

### State Colocation Principle

State should live **as close to where it's used** as possible. Only lift state up when multiple components need it.

```
❌ Over-centralized state (everything in Redux/global store):
  - 10 components share state none of them actually share
  - Every change re-renders the world
  - Hard to debug because mutations come from anywhere

✓ Colocated state:
  - Modal open/close state → inside the Modal component
  - Form field value → inside the form component
  - User profile → global store (used across app)
  - Active cart items → global store (used across app)
```

---

### Normalizing State (The Database Pattern)

When storing collections of items, normalize them to avoid duplication and stale data.

```javascript
// ❌ Denormalized (duplicated data, stale references)
{
  posts: [
    { id: 1, title: "Hello", author: { id: 42, name: "Aashik" } },
    { id: 2, title: "World", author: { id: 42, name: "Aashik" } },
  ]
}
// If Aashik changes their name, you must update every post

// ✓ Normalized (single source of truth)
{
  posts: {
    byId: { 1: { id: 1, title: "Hello", authorId: 42 }, 2: {...} },
    allIds: [1, 2]
  },
  users: {
    byId: { 42: { id: 42, name: "Aashik" } },
    allIds: [42]
  }
}
// Update user once, all posts automatically reflect the change
```

```javascript
// Selectors — compute derived data from normalized state
const selectPostWithAuthor = (state, postId) => {
  const post = state.posts.byId[postId];
  const author = state.users.byId[post.authorId];
  return { ...post, author };
};
```

---

### The Selector Pattern (Memoized Derived State)

```javascript
import { createSelector } from 'reselect'; // or re-reselect for React

// Without memoization: recomputes on every render
const getCompletedTasks = (tasks) => tasks.filter(t => t.completed);

// With memoization: only recomputes when tasks array changes
const selectCompletedTasks = createSelector(
  (state) => state.tasks.items,
  (tasks) => tasks.filter(t => t.completed)
);

// Parameterized selector
const makeSelectTasksByUser = () => createSelector(
  (state) => state.tasks.items,
  (_, userId) => userId,
  (tasks, userId) => tasks.filter(t => t.assigneeId === userId)
);

const selectAashiksTasks = makeSelectTasksByUser();
const tasks = selectAashiksTasks(state, 'user_42');
```

---

## 6. Data Fetching Patterns

### REST vs GraphQL vs tRPC

```
REST:
  GET /users/42          → { id, name, email, address, preferences, ... }
  GET /users/42/posts    → [{ id, title, content, authorId, ... }]
  GET /users/42/followers → [{ id, name, ... }]

  Problem: 3 round trips, over-fetching (got address when you only needed name)

GraphQL:
  POST /graphql
  query {
    user(id: 42) {
      name                  ← only what you need
      posts(limit: 5) {
        title               ← only what you need
      }
    }
  }

  Result: 1 round trip, exactly the data you asked for

tRPC:
  Type-safe RPC calls — functions on server called as if they're local
  const user = await trpc.user.getById.query({ id: 42 });
  Fully inferred TypeScript types — no schema, no codegen
```

---

### React Query — Server State Management

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Fetching
const { data, isLoading, isError, error, isFetching } = useQuery({
  queryKey: ['user', userId],          // cache key
  queryFn: () => api.get(`/users/${userId}`),
  staleTime: 5 * 60 * 1000,           // data is fresh for 5 minutes
  gcTime: 10 * 60 * 1000,             // keep in cache 10 min after unused
  retry: 3,                            // retry failed requests
  retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000), // exponential backoff
  select: (data) => data.data,         // transform/pick from response
  enabled: !!userId,                   // conditional fetching
});

// Mutation with optimistic update
const queryClient = useQueryClient();
const mutation = useMutation({
  mutationFn: (newTitle) => api.patch(`/tasks/${taskId}`, { title: newTitle }),

  onMutate: async (newTitle) => {
    await queryClient.cancelQueries({ queryKey: ['task', taskId] });
    const previousTask = queryClient.getQueryData(['task', taskId]);

    // Optimistic update
    queryClient.setQueryData(['task', taskId], (old) => ({ ...old, title: newTitle }));

    return { previousTask }; // context for rollback
  },

  onError: (err, newTitle, context) => {
    // Rollback on error
    queryClient.setQueryData(['task', taskId], context.previousTask);
  },

  onSettled: () => {
    // Invalidate to refetch fresh data regardless of success/failure
    queryClient.invalidateQueries({ queryKey: ['task', taskId] });
  },
});
```

---

### Infinite Scrolling vs Pagination

```jsx
// Infinite Scroll with React Query
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
} = useInfiniteQuery({
  queryKey: ['posts'],
  queryFn: ({ pageParam = 1 }) => api.get(`/posts?page=${pageParam}&limit=20`),
  getNextPageParam: (lastPage) => lastPage.hasMore ? lastPage.nextPage : undefined,
});

// All pages flattened
const posts = data?.pages.flatMap(page => page.items) ?? [];

// Intersection Observer for auto-load
const { ref: loaderRef } = useIntersectionObserver({
  onChange: (isVisible) => { if (isVisible && hasNextPage) fetchNextPage(); },
});

return (
  <>
    {posts.map(post => <PostCard key={post.id} post={post} />)}
    <div ref={loaderRef} />
    {isFetchingNextPage && <Spinner />}
  </>
);
```

**When to use Infinite Scroll vs Pagination:**

| Infinite Scroll | Pagination |
|---|---|
| Social feeds | Search results |
| Content consumption (news, images) | Data tables |
| Mobile UX | When users need to jump to specific pages |
| "Discovery" browsing | When sharing links to specific pages |

---

### Prefetching & Background Fetching

```jsx
// Prefetch on hover — data is ready before user clicks
const prefetchUser = (userId) => {
  queryClient.prefetchQuery({
    queryKey: ['user', userId],
    queryFn: () => api.get(`/users/${userId}`),
    staleTime: 60000,
  });
};

<UserCard
  onMouseEnter={() => prefetchUser(user.id)}  // prefetch on hover
  onFocus={() => prefetchUser(user.id)}        // accessibility: prefetch on focus
  onClick={() => navigate(`/users/${user.id}`)}
/>
```

---

## 7. Caching Strategies

### The 5 Types of Frontend Cache

```
┌────────────────────────────────────────────────────────────────────┐
│ 1. Browser HTTP Cache    │ GET /api/users HTTP/1.1                  │
│                          │ Cache-Control: max-age=300               │
│                          │ Browser won't re-request for 5 minutes   │
├────────────────────────────────────────────────────────────────────┤
│ 2. Service Worker Cache  │ Intercepts fetch(), serves from Cache API │
│                          │ Works offline, stale-while-revalidate     │
├────────────────────────────────────────────────────────────────────┤
│ 3. In-Memory API Cache   │ React Query / SWR cache                   │
│                          │ Lives in JS memory, cleared on refresh    │
├────────────────────────────────────────────────────────────────────┤
│ 4. CDN Cache             │ Static assets (JS, CSS, images) at edge   │
│                          │ Cache-Control: immutable (1 year)          │
├────────────────────────────────────────────────────────────────────┤
│ 5. LocalStorage / IDB    │ Persistent across sessions                │
│                          │ For expensive-to-compute data             │
└────────────────────────────────────────────────────────────────────┘
```

---

### HTTP Cache Headers

```
Cache-Control: max-age=300
  → Browser caches for 5 minutes. No server request during this period.

Cache-Control: no-cache
  → Browser must revalidate with server every time (ETag/Last-Modified).
  → Does NOT mean "don't cache" — it means "always check if stale."

Cache-Control: no-store
  → Never cache. Every request goes to server. (Sensitive data)

Cache-Control: public, max-age=31536000, immutable
  → CDN + browser can cache for 1 year. Never revalidate.
  → Used for hashed static assets (bundle.abc123.js) that never change.

ETag: "abc123"
  → Server fingerprint of the response.
  → Browser sends: If-None-Match: "abc123"
  → Server: 304 Not Modified (no body) OR 200 with new content + new ETag
```

---

### Stale-While-Revalidate

```
Request 1 (cache empty): fetch from server → store in cache → return fresh data
Request 2 (within maxAge): return cached data immediately (stale: 0 revalidations)
Request 3 (after maxAge): return stale cached data immediately (user sees content instantly)
                          SIMULTANEOUSLY fetch fresh data in background
                          → Update cache when background fetch completes
Request 4: fresh data returned
```

```javascript
// Service Worker implementing stale-while-revalidate
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open('api-cache').then(async (cache) => {
      const cachedResponse = await cache.match(event.request);
      const fetchPromise = fetch(event.request).then((networkResponse) => {
        cache.put(event.request, networkResponse.clone());
        return networkResponse;
      });
      return cachedResponse ?? fetchPromise; // return cache, but also update it
    })
  );
});
```

---

### Cache Invalidation Strategies

```
1. TTL (Time-To-Live):       Expire after N seconds
   Simple but data can be stale for N seconds

2. Event-based invalidation: A mutation triggers cache purge
   queryClient.invalidateQueries(['posts'])
   More accurate but requires wiring up every mutation

3. Cache tagging:            Tag cached items, purge by tag
   POST /products → invalidate tag: 'products'
   Used by CDNs (Fastly, Cloudflare)

4. Versioned URLs:           Change URL when content changes
   /bundle.a1b2c3.js → /bundle.d4e5f6.js
   Old URL cached forever, new content = new URL
   Used for static assets
```

---

## 8. Performance Optimization

### Core Web Vitals (Google's Performance Metrics)

```
LCP — Largest Contentful Paint
  What: Time until the largest visible element is rendered
  Good: < 2.5s   Needs Improvement: 2.5-4s   Poor: > 4s
  Fix: SSR, preload hero image, optimize image sizes

FID / INP — Interaction to Next Paint (INP replaced FID in 2024)
  What: Time from user interaction to browser response
  Good: < 200ms  Needs Improvement: 200-500ms  Poor: > 500ms
  Fix: Break up long tasks, defer non-critical JS, use web workers

CLS — Cumulative Layout Shift
  What: Sum of unexpected layout shifts (content jumping around)
  Good: < 0.1    Needs Improvement: 0.1-0.25   Poor: > 0.25
  Fix: Reserve space for images/ads, avoid inserting above content

TTFB — Time to First Byte
  What: Time until first byte of response received
  Good: < 800ms
  Fix: CDN, server-side caching, optimize server response time
```

---

### Critical Rendering Path

```
HTML → DOM
CSS  → CSSOM    }→ Render Tree → Layout → Paint → Composite
                                             ↓
                                      Pixels on screen

Blocking resources = anything that pauses rendering:
  - <link rel="stylesheet"> blocks rendering (browser needs CSSOM before paint)
  - <script> without defer/async blocks HTML parsing AND rendering
```

```html
<!-- ❌ Render-blocking -->
<head>
  <link rel="stylesheet" href="styles.css" />
  <script src="analytics.js"></script>
</head>

<!-- ✓ Optimized -->
<head>
  <!-- Critical CSS inline (no network request) -->
  <style>/* above-the-fold styles only */</style>

  <!-- Non-critical CSS loaded async -->
  <link rel="preload" href="styles.css" as="style" onload="this.rel='stylesheet'" />

  <!-- Scripts deferred (parsed after HTML, executed after DOMContentLoaded) -->
  <script defer src="app.js"></script>

  <!-- Async: downloaded in parallel, executed as soon as downloaded -->
  <script async src="analytics.js"></script>

  <!-- Preconnect to external origins (DNS + TCP + TLS before request) -->
  <link rel="preconnect" href="https://api.myapp.com" />
  <link rel="preconnect" href="https://fonts.googleapis.com" />
</head>
```

---

### Image Optimization

Images are typically 50-70% of a page's total byte size.

```html
<!-- Modern image optimization -->
<picture>
  <!-- AVIF: best compression, newer browsers -->
  <source srcset="hero.avif" type="image/avif" />
  <!-- WebP: good compression, wide support -->
  <source srcset="hero.webp" type="image/webp" />
  <!-- Fallback -->
  <img
    src="hero.jpg"
    alt="Hero image"
    width="1200"
    height="600"
    loading="lazy"           /* defer off-screen images */
    decoding="async"         /* decode in background */
    fetchpriority="high"     /* for LCP image: load first */
  />
</picture>
```

```jsx
// Responsive images — serve right size for screen
<img
  src="product-800.jpg"
  srcset="product-400.jpg 400w, product-800.jpg 800w, product-1200.jpg 1200w"
  sizes="(max-width: 600px) 100vw, (max-width: 1200px) 50vw, 33vw"
  alt="Product"
/>
```

---

### JavaScript Bundle Optimization

```
Bundle Size Budget:
  Mobile (3G):  < 100KB initial JS (gzipped)
  Desktop:      < 300KB initial JS (gzipped)

Techniques:
  1. Tree Shaking:    Remove unused exports at build time
  2. Code Splitting:  Split bundle into chunks, load on demand
  3. Minification:    Terser removes whitespace, shortens names
  4. Compression:     Brotli (better) or Gzip at CDN level
  5. Lazy Loading:    Dynamic import() for non-critical routes/features
  6. Dependency audit: Replace large deps with smaller alternatives
     moment.js (67KB) → date-fns (only import what you use, tree-shakeable)
     lodash (72KB)    → lodash-es (tree-shakeable) or individual functions
```

```bash
# Analyze your bundle
npx webpack-bundle-analyzer stats.json
npx source-map-explorer build/static/js/*.js
```

---

### Virtual Scrolling (Windowing)

Rendering 10,000 list items in the DOM is catastrophic. Virtualization renders only what's visible.

```jsx
import { FixedSizeList } from 'react-window';

// Only renders ~15-20 items in DOM at any time
// regardless of how many items exist in the data array
<FixedSizeList
  height={600}
  width="100%"
  itemCount={10000}
  itemSize={72}
>
  {({ index, style }) => (
    <div style={style}>  {/* style provides position (top, left, height) */}
      <UserRow user={users[index]} />
    </div>
  )}
</FixedSizeList>

// Variable height items
import { VariableSizeList } from 'react-window';
```

---

### Web Workers (Offload CPU Work)

JavaScript is single-threaded. Heavy computation blocks the UI thread (causes jank).

```javascript
// worker.js (runs in separate thread)
self.onmessage = (event) => {
  const { data } = event;
  const result = heavyComputation(data); // doesn't block UI
  self.postMessage(result);
};

// main.js
const worker = new Worker(new URL('./worker.js', import.meta.url));

worker.postMessage(largeDataset);
worker.onmessage = (event) => {
  setResult(event.data); // update UI with result
};

// Use cases:
// - Parsing large JSON responses
// - Image processing / filters
// - Encryption / hashing
// - Search indexing (Fuse.js on large datasets)
// - Running ML models (TensorFlow.js)
```

---

### React Performance Patterns

```jsx
// 1. React.memo — prevent re-render when props haven't changed
const ExpensiveComponent = React.memo(({ data, onAction }) => {
  return <ComplexChart data={data} onAction={onAction} />;
}, (prevProps, nextProps) => {
  // Custom comparison: return true if props are equal (skip re-render)
  return prevProps.data.id === nextProps.data.id;
});

// 2. useMemo — memoize expensive computation
const sortedData = useMemo(
  () => [...data].sort((a, b) => b.score - a.score).slice(0, 100),
  [data]
);

// 3. useCallback — stable function reference for child components
const handleDelete = useCallback((id) => {
  dispatch(deleteItem(id));
}, [dispatch]); // only recreated if dispatch changes (never in practice)

// 4. Avoid object/array literals in JSX (new reference every render)
// ❌ style={{ color: 'red' }} → new object every render
// ✓ style={styles.error}     → stable reference from StyleSheet.create

// 5. Context split — don't put frequently-changing values with stable ones
// ❌ Context with { user, theme, cart } → cart change re-renders all consumers
// ✓ Split into UserContext, ThemeContext, CartContext
```

---

## 9. Code Splitting & Lazy Loading

### Route-Based Code Splitting

```jsx
import { lazy, Suspense } from 'react';

// Each route is a separate chunk — only loaded when user visits that route
const HomePage     = lazy(() => import('./pages/Home'));
const ProductsPage = lazy(() => import('./pages/Products'));
const CheckoutPage = lazy(() => import('./pages/Checkout'));
const AdminPage    = lazy(() => import('./pages/Admin'));

const Router = () => (
  <Suspense fallback={<PageSkeleton />}>
    <Routes>
      <Route path="/"          element={<HomePage />} />
      <Route path="/products"  element={<ProductsPage />} />
      <Route path="/checkout"  element={<CheckoutPage />} />
      <Route path="/admin"     element={<AdminPage />} />
    </Routes>
  </Suspense>
);
```

---

### Component-Level Code Splitting

```jsx
// Heavy components loaded only when needed
const RichTextEditor = lazy(() =>
  import(/* webpackChunkName: "rich-text-editor" */ '@monaco-editor/react')
);

const VideoPlayer = lazy(() =>
  import(/* webpackChunkName: "video-player" */ './VideoPlayer')
);

// Feature flags — don't ship code for disabled features
const { isAdminEnabled } = useFeatureFlags();
const AdminPanel = isAdminEnabled
  ? lazy(() => import('./AdminPanel'))
  : null;
```

---

### Prefetching Routes

```jsx
// Prefetch the next likely page while user is on current page
const prefetchCheckout = () => {
  import('./pages/Checkout'); // starts downloading chunk in background
};

// On product page, user is likely going to checkout
<AddToCartButton
  onMouseEnter={prefetchCheckout}  // prefetch on hover
  onClick={handleAddToCart}
/>
```

---

## 10. Micro-Frontends

### What are Micro-Frontends?

Apply microservices thinking to the frontend. Different teams own different **vertical slices** of the UI, deployed independently.

```
Monolith:                    Micro-Frontend:
One codebase                 Team A: /products  (React)
One build                    Team B: /checkout  (Vue)
One deployment               Team C: /account   (Angular)
All teams block each other   Each team deploys independently
```

---

### Module Federation (Webpack 5)

The most popular micro-frontend implementation. One app can **dynamically load code from another app at runtime**.

```javascript
// products-app/webpack.config.js — the REMOTE
{
  plugins: [
    new ModuleFederationPlugin({
      name: 'products',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductCard': './src/components/ProductCard',
        './ProductList': './src/components/ProductList',
      },
      shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
    }),
  ],
}

// shell-app/webpack.config.js — the HOST
{
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        products: 'products@https://products.myapp.com/remoteEntry.js',
        checkout: 'checkout@https://checkout.myapp.com/remoteEntry.js',
      },
    }),
  ],
}

// shell-app: consume remote components as if they're local
const ProductCard = lazy(() => import('products/ProductCard'));
const CheckoutFlow = lazy(() => import('checkout/CheckoutFlow'));
```

---

### Micro-Frontend Trade-offs

```
Pros:
  + Teams deploy independently (no release trains)
  + Technology diversity (each team can use different stack)
  + Smaller bundles per team (faster builds)
  + Better fault isolation (one team's bug doesn't break others)

Cons:
  - Duplicated dependencies (each MFE ships its own react)
  - Consistency challenges (different teams build inconsistent UIs)
  - Runtime integration complexity
  - Shared state between MFEs is hard
  - Testing the composed app is harder

When to use:
  - Large org (5+ frontend teams working on same product)
  - Independent deployment cadence is a hard requirement
  - Teams need technology autonomy

When NOT to use:
  - Small team (< 3 frontend devs)
  - Tight coupling between features
  - Bundle size is critical (MFEs increase overall bundle)
```

---

## 11. Real-Time Communication

### WebSockets

Persistent, **full-duplex** connection. Both client and server can send messages at any time.

```javascript
// Client
const ws = new WebSocket('wss://api.myapp.com/ws');

ws.onopen = () => {
  ws.send(JSON.stringify({ type: 'subscribe', channels: ['price:AAPL'] }));
};

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  updatePrice(msg.symbol, msg.price);
};

ws.onclose = (event) => {
  // Auto-reconnect with exponential backoff
  scheduleReconnect(attempt);
};

// Heartbeat to detect zombie connections
const heartbeat = setInterval(() => {
  if (ws.readyState === WebSocket.OPEN) ws.send(JSON.stringify({ type: 'ping' }));
}, 30000);
```

**Use cases:** Chat, live trading, collaborative editing, multiplayer games, live sports scores.

---

### Server-Sent Events (SSE)

**One-way** stream from server to client. Simpler than WebSockets, built on HTTP.

```javascript
// Client
const eventSource = new EventSource('/api/live-feed', {
  withCredentials: true, // send cookies for auth
});

eventSource.onmessage = (event) => {
  const update = JSON.parse(event.data);
  appendToFeed(update);
};

// Named events
eventSource.addEventListener('order-status', (event) => {
  updateOrderStatus(JSON.parse(event.data));
});

// SSE auto-reconnects on connection drop (built-in, unlike WebSockets)
eventSource.onerror = () => {
  // Browser automatically retries using retry: value from server
};
```

**Use cases:** Live notifications, news feeds, order tracking, log streaming.

---

### Long Polling

Simulates real-time by keeping the HTTP request open until new data arrives.

```javascript
const longPoll = async () => {
  while (isPolling) {
    try {
      const response = await fetch(`/api/notifications?since=${lastId}`, {
        signal: AbortSignal.timeout(30000), // 30s timeout
      });
      const data = await response.json();
      if (data.notifications.length) {
        processNotifications(data.notifications);
        lastId = data.lastId;
      }
    } catch (err) {
      await sleep(2000); // brief pause before retry on error
    }
  }
};
```

**Use cases:** When WebSockets are blocked by firewall, legacy systems, simple notification polling.

---

### Comparison Table

| | WebSocket | SSE | Long Polling | Regular Polling |
|---|---|---|---|---|
| Direction | Bidirectional | Server → Client | Server → Client | Server → Client |
| Protocol | WS/WSS | HTTP | HTTP | HTTP |
| Auto-reconnect | No (manual) | Yes | Manual | N/A |
| Firewall friendly | Sometimes blocked | Yes | Yes | Yes |
| Complexity | Medium | Low | Low | Lowest |
| Use case | Chat, games | Notifications, feeds | Fallback | Simple updates |

---

## 12. CDN & Asset Delivery

### What a CDN Does

```
Without CDN:
  User in Mumbai → request → Server in Virginia → 200ms round trip

With CDN:
  User in Mumbai → request → CDN edge in Mumbai → 10ms round trip
  (CDN edge has cached the asset from origin server)
```

---

### Cache-Control for Static Assets

```
Hashed assets (bundle.a1b2c3.js):
  Cache-Control: public, max-age=31536000, immutable
  → Cache for 1 year. When content changes, filename changes. ✓

HTML files (index.html):
  Cache-Control: no-cache
  → Always revalidate. HTML changes → new asset URLs in HTML. ✓

API responses (GET /products):
  Cache-Control: public, max-age=300, stale-while-revalidate=600
  → Fresh for 5min, serve stale up to 10min while revalidating ✓
```

---

### Font Optimization

```html
<!-- Preconnect to font provider -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- Use font-display: swap to prevent invisible text -->
<style>
  @font-face {
    font-family: 'Inter';
    src: url('/fonts/Inter.woff2') format('woff2');
    font-display: swap; /* show system font until custom font loads */
    font-weight: 400 700; /* variable font range */
  }
</style>
```

---

## 13. Security

### XSS — Cross-Site Scripting

Attacker injects malicious JavaScript that runs in victim's browser.

```
Reflected XSS:  Malicious URL → server reflects script → victim's browser executes it
Stored XSS:     Attacker stores script in DB → rendered for all users
DOM XSS:        JS reads attacker-controlled data and writes it to DOM
```

```javascript
// ❌ Vulnerable: directly setting innerHTML
element.innerHTML = userInput; // if userInput = "<script>stealCookies()</script>"

// ✓ Safe alternatives
element.textContent = userInput;        // treats input as text, never HTML
element.innerHTML = sanitize(userInput); // DOMPurify before inserting HTML

import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(untrustedHTML, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
  ALLOWED_ATTR: ['href'],
});

// React is safe by default (JSX escapes automatically)
<div>{userInput}</div>  // ✓ safe — React escapes
<div dangerouslySetInnerHTML={{ __html: userInput }} />  // ❌ bypasses escaping
```

---

### Content Security Policy (CSP)

HTTP header that tells the browser which sources are trusted.

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.myapp.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' https: data:;
  connect-src 'self' https://api.myapp.com wss://api.myapp.com;
  font-src 'self' https://fonts.gstatic.com;
  frame-ancestors 'none';
  upgrade-insecure-requests;
```

**What this does:** Even if an XSS attack injects `<script src="https://evil.com/steal.js">`, the browser blocks it because `evil.com` isn't in `script-src`.

---

### CSRF — Cross-Site Request Forgery

Tricks logged-in user into performing actions on another site.

```
1. User is logged into bank.com (session cookie is set)
2. User visits evil.com
3. evil.com has: <img src="https://bank.com/transfer?to=attacker&amount=10000" />
4. Browser sends request to bank.com WITH session cookie
5. Bank executes transfer (thinks it's legitimate user request)
```

**Defenses:**
```javascript
// 1. SameSite cookies (modern defense)
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly
// SameSite=Strict: cookie only sent for same-site requests
// SameSite=Lax: cookie sent for top-level navigation, not embedded requests

// 2. CSRF Token (traditional defense)
// Server generates a random token per session
// Frontend includes it in every mutating request:
axios.defaults.headers['X-CSRF-Token'] = getCsrfToken();

// 3. Double-submit cookie
// Token in cookie + token in request header must match
// An attacker can't read the cookie to forge the header
```

---

### Secrets in Frontend Code

```
❌ NEVER hardcode in frontend:
  - API secret keys
  - Database credentials
  - Private keys
  - Internal service URLs

✓ What IS safe in frontend:
  - Public API keys (Stripe publishable key, Google Maps API key — restrict by domain)
  - Public OAuth client IDs
  - Feature flag values

Environment variables in Vite/CRA:
  VITE_API_URL = "https://api.myapp.com"      ← safe (public endpoint)
  STRIPE_SECRET_KEY = "sk_live_..."           ← NEVER (your .env is bundled)
```

---

## 14. Authentication & Authorization

### Token Storage — Where and Why

```
localStorage:
  + Accessible from JS
  + Persists across tabs
  - Vulnerable to XSS (any script can read localStorage)
  ❌ Do NOT store auth tokens here

sessionStorage:
  + Cleared when tab closes
  - Still vulnerable to XSS
  ❌ Do NOT store auth tokens here

HttpOnly Cookie:
  + JS CANNOT read it (immune to XSS token theft)
  + Automatically sent with requests
  + SameSite=Strict prevents CSRF
  ✓ Best option for web auth tokens

Memory (JS variable):
  + Immune to XSS persistence (cleared on page refresh)
  - Access token lost on refresh (need refresh flow)
  ✓ Good for short-lived access tokens + HttpOnly cookie for refresh token
```

---

### The Dual Token Pattern

```
Access Token:
  - Short-lived (15 minutes)
  - Stored in memory (JS variable)
  - Sent in Authorization header
  - Lost on page refresh (by design)

Refresh Token:
  - Long-lived (30 days)
  - Stored in HttpOnly cookie
  - Used only to get new access token
  - Never accessible to JS

Flow:
  App loads → no access token in memory
  → Send refresh request (HttpOnly cookie automatically attached)
  → Server validates refresh token
  → Returns new access token (in response body)
  → Store access token in memory
  → Use for API calls
  → After 15min: token expires → interceptor requests new one silently
```

---

### Role-Based Access Control (RBAC) on Frontend

```jsx
// PermissionGate component
const permissions = {
  admin:   ['read', 'write', 'delete', 'manage_users'],
  editor:  ['read', 'write'],
  viewer:  ['read'],
};

const usePermission = (action) => {
  const { user } = useAuth();
  return permissions[user.role]?.includes(action) ?? false;
};

const PermissionGate = ({ action, children, fallback = null }) => {
  const can = usePermission(action);
  return can ? children : fallback;
};

// Usage
<PermissionGate action="delete" fallback={<DisabledButton />}>
  <DeleteButton onClick={handleDelete} />
</PermissionGate>

// ⚠️ Frontend permission checks are UX only — never security
// Always enforce permissions on the backend
// A clever user can bypass frontend checks via DevTools
```

---

## 15. API Design from the Frontend

### BFF — Backend for Frontend

A server specifically designed to serve one frontend client's needs.

```
Without BFF:
  Mobile App  ──────────────────────→  User Service
  Mobile App  ──────────────────────→  Orders Service
  Mobile App  ──────────────────────→  Products Service
  3 round trips, over-fetching, each service returns its full model

With BFF:
  Mobile App  → Mobile BFF  → User Service
                           → Orders Service
                           → Products Service
  1 round trip, BFF aggregates + shapes data for mobile
```

```javascript
// BFF endpoint: exactly what mobile home screen needs
// GET /bff/mobile/home
{
  "user": { "name": "Aashik", "avatarUrl": "..." },         // from User Service
  "activeOrders": [{ "id": "...", "status": "shipped" }],  // from Orders Service
  "recommendations": [{ "id": "...", "title": "..." }],    // from Products Service
  "unreadCount": 3                                          // from Notifications Service
}
```

---

### GraphQL on the Frontend

```jsx
// Apollo Client setup
const client = new ApolloClient({
  uri: '/graphql',
  cache: new InMemoryCache({
    typePolicies: {
      // Normalize Product by id (like Redux normalize)
      Product: { keyFields: ['id'] },
      // Paginated queries — merge pages instead of replace
      Query: {
        fields: {
          products: relayStylePagination(),
        },
      },
    },
  }),
});

// Query with fragments (reusable field selections)
const PRODUCT_FIELDS = gql`
  fragment ProductFields on Product {
    id title price imageUrl rating reviewCount
  }
`;

const GET_PRODUCTS = gql`
  ${PRODUCT_FIELDS}
  query GetProducts($category: String!, $page: Int!) {
    products(category: $category, page: $page) {
      items { ...ProductFields }
      hasNextPage
      totalCount
    }
  }
`;

const { data, loading, fetchMore } = useQuery(GET_PRODUCTS, {
  variables: { category: 'electronics', page: 1 },
});
```

---

## 16. Accessibility (a11y)

### Why a11y is a System Design Concern

- Legal requirement in many countries (ADA, WCAG 2.1)
- 15% of people have some form of disability
- Good a11y often improves UX for everyone (captions help in noisy environments)

---

### The 4 POUR Principles

```
Perceivable:   Information must be perceivable (alt text, captions, contrast)
Operable:      UI must be operable (keyboard nav, no seizure-inducing content)
Understandable: Content must be understandable (clear language, error messages)
Robust:        Must work with assistive tech (screen readers, voice control)
```

---

### Semantic HTML

```html
<!-- ❌ div soup — screen reader has no context -->
<div class="header">
  <div class="nav">
    <div class="nav-item" onclick="navigate()">Home</div>
  </div>
</div>
<div class="main">
  <div class="article">
    <div class="title">My Post</div>
  </div>
</div>

<!-- ✓ Semantic — screen reader understands structure -->
<header>
  <nav aria-label="Main navigation">
    <ul>
      <li><a href="/">Home</a></li>
    </ul>
  </nav>
</header>
<main>
  <article>
    <h1>My Post</h1>
  </article>
</main>
```

---

### ARIA Attributes

```jsx
// When semantic HTML isn't enough, ARIA fills the gap
<button
  aria-label="Close modal"  // overrides visible text for screen readers
  aria-expanded={isOpen}    // communicates state
  aria-controls="menu-id"   // links button to what it controls
  onClick={closeModal}
>
  ✕
</button>

// Live regions — announce dynamic content to screen readers
<div aria-live="polite" aria-atomic="true">
  {statusMessage} {/* announced when it changes */}
</div>

// Custom interactive components
<div
  role="slider"
  aria-valuemin={0}
  aria-valuemax={100}
  aria-valuenow={value}
  aria-label="Volume"
  tabIndex={0}
  onKeyDown={handleKeyDown}
/>
```

---

### Focus Management

```jsx
// Trap focus inside modal (keyboard users can't escape)
import { createFocusTrap } from 'focus-trap';

useEffect(() => {
  if (isOpen) {
    const trap = createFocusTrap(modalRef.current, {
      escapeDeactivates: true,
      returnFocusOnDeactivate: true,
    });
    trap.activate();
    return () => trap.deactivate();
  }
}, [isOpen]);

// Restore focus to trigger element when modal closes
const triggerRef = useRef(null);
const openModal = () => {
  setPreviousFocus(document.activeElement);
  setIsOpen(true);
};
const closeModal = () => {
  setIsOpen(false);
  previousFocus?.focus(); // return focus to trigger
};
```

---

## 17. Internationalization (i18n)

### Architecture of an i18n System

```
App
├── Detect locale (browser preference / user setting / URL prefix)
├── Load translation bundle for that locale (lazy load)
├── Provide locale context to all components
└── Components: use t('key') instead of hardcoded strings

Translation files:
  en.json: { "welcome": "Welcome, {{name}}!", "items": "{{count}} items" }
  hi.json: { "welcome": "नमस्ते, {{name}}!", "items": "{{count}} आइटम" }
  ar.json: { "welcome": "مرحبا، {{name}}!", "items": "{{count}} عناصر" }
```

```jsx
import i18n from 'i18next';
import { useTranslation } from 'react-i18next';

// Usage
const { t, i18n } = useTranslation();

<Text>{t('welcome', { name: user.name })}</Text>
<Text>{t('items', { count: cart.length })}</Text>

// Pluralization (i18next handles this automatically)
// en.json: { "items_one": "{{count}} item", "items_other": "{{count}} items" }
<Text>{t('items', { count: 3 })}</Text>  // "3 items"
<Text>{t('items', { count: 1 })}</Text>  // "1 item"
```

---

### RTL (Right-to-Left) Layout Support

```jsx
const { i18n } = useTranslation();
const isRTL = i18n.dir() === 'rtl'; // Arabic, Hebrew, Farsi

// HTML
document.documentElement.dir = isRTL ? 'rtl' : 'ltr';

// CSS Logical Properties (work for both LTR and RTL)
.element {
  margin-inline-start: 16px;  /* left in LTR, right in RTL */
  padding-inline-end: 8px;    /* right in LTR, left in RTL */
  border-inline-start: 2px solid;
}

// Instead of:
.element {
  margin-left: 16px;   /* doesn't flip in RTL */
}
```

---

## 18. Build Systems & Bundlers

### What a Bundler Does

```
Source files (many):            Output (few):
  src/app.tsx          →        dist/bundle.js    (all JS, minified)
  src/components/*.tsx →        dist/bundle.css   (all CSS, minified)
  src/styles/*.css     →        dist/assets/...   (images, fonts)
  node_modules/...
```

---

### Vite vs Webpack

```
Webpack (battle-tested, flexible):
  Uses CommonJS bundling for production
  Dev server: rebundles entire app on change (slow for large apps)
  Config: complex but highly flexible
  Ecosystem: massive, every plugin exists

Vite (modern, fast):
  Dev server: native ES modules — no bundling! (instant HMR)
  Production: Rollup under the hood
  Config: simple, sensible defaults
  Speed: 10-100x faster dev server than Webpack for large apps

Rule of thumb:
  New project: use Vite
  Existing large project: migrate gradually (vite-plugin-commonjs can help)
  Need micro-frontends with Module Federation: Webpack
```

---

### Tree Shaking

```javascript
// ❌ CommonJS import — bundlers can't tree-shake
const _ = require('lodash');
const result = _.isEmpty(obj); // entire lodash shipped (72KB)

// ✓ ES Module import — bundler removes unused exports
import { isEmpty } from 'lodash-es'; // only isEmpty shipped (~1KB)

// Your own code: use named exports, not default object exports
// ❌
export default { formatDate, parseDate, isWeekend }; // can't tree-shake

// ✓
export const formatDate = () => {};
export const parseDate = () => {};
export const isWeekend = () => {};
// Consumer: import { formatDate } from './utils/date'
// Only formatDate is bundled
```

---

## 19. Progressive Web Apps (PWA)

### What Makes a PWA

```
1. Service Worker:   Intercepts network requests, enables offline
2. Web App Manifest: Makes app "installable" on home screen
3. HTTPS:            Required for service worker
```

---

### Service Worker — The Core of PWA

```javascript
// sw.js — registered in main app:
// navigator.serviceWorker.register('/sw.js')

const CACHE_VERSION = 'v2';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const DYNAMIC_CACHE = `dynamic-${CACHE_VERSION}`;

// Install: pre-cache critical assets
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then((cache) =>
      cache.addAll([
        '/',
        '/index.html',
        '/bundle.js',
        '/styles.css',
        '/offline.html',
      ])
    )
  );
});

// Activate: delete old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(keys.filter(k => k !== STATIC_CACHE && k !== DYNAMIC_CACHE)
                      .map(k => caches.delete(k)))
    )
  );
});

// Fetch: intercept all network requests
self.addEventListener('fetch', (event) => {
  const { request } = event;

  // Strategy 1: Cache First (for static assets)
  if (request.url.includes('/static/')) {
    event.respondWith(
      caches.match(request).then(cached => cached ?? fetch(request))
    );
    return;
  }

  // Strategy 2: Network First with offline fallback (for HTML/API)
  event.respondWith(
    fetch(request)
      .then((response) => {
        const clone = response.clone();
        caches.open(DYNAMIC_CACHE).then(c => c.put(request, clone));
        return response;
      })
      .catch(() =>
        caches.match(request) ??
        caches.match('/offline.html')
      )
  );
});
```

---

## 20. Testing Strategy

### The Testing Trophy (Frontend)

```
          ╱‾‾‾‾‾‾‾‾‾‾╲
         /  E2E Tests  \      ← Few, slow, high confidence
        /  (Cypress,    \       Test critical user journeys
       /   Playwright)   \
      /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
     /  Integration Tests  \  ← Many, moderate speed
    /  (React Testing Lib)  \   Test component + hooks + API together
   /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
  /      Unit Tests           \ ← Most, fast
 /  (Jest, Vitest)             \  Test pure functions, utils, reducers
╱‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾╲
        Static Analysis          ← TypeScript, ESLint (free, always on)
```

---

### Integration Tests with React Testing Library

```jsx
// Testing what the USER sees, not implementation details
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('LoginForm', () => {
  it('shows error for invalid email', async () => {
    render(<LoginForm onSubmit={jest.fn()} />);

    await userEvent.type(screen.getByLabelText('Email'), 'not-an-email');
    await userEvent.type(screen.getByLabelText('Password'), 'password123');
    await userEvent.click(screen.getByRole('button', { name: 'Log in' }));

    expect(screen.getByText('Please enter a valid email')).toBeInTheDocument();
  });

  it('calls onSubmit with credentials on valid submission', async () => {
    const onSubmit = jest.fn().mockResolvedValue({ token: 'abc' });
    render(<LoginForm onSubmit={onSubmit} />);

    await userEvent.type(screen.getByLabelText('Email'), 'user@test.com');
    await userEvent.type(screen.getByLabelText('Password'), 'password123');
    await userEvent.click(screen.getByRole('button', { name: 'Log in' }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        email: 'user@test.com',
        password: 'password123',
      });
    });
  });
});
```

---

### Visual Regression Testing

```javascript
// Storybook + Chromatic
// Every component is a "story" — a visual snapshot
// Chromatic detects pixel-level changes in CI

// Button.stories.tsx
export default { title: 'Button', component: Button };

export const Primary = { args: { variant: 'primary', children: 'Click me' } };
export const Disabled = { args: { variant: 'primary', disabled: true, children: 'Disabled' } };
export const Loading  = { args: { isLoading: true, children: 'Loading...' } };

// CI: chromatic --project-token=$CHROMATIC_TOKEN
// If a CSS change accidentally makes buttons 1px taller, Chromatic flags it
```

---

## 21. Monitoring & Observability

### Real User Monitoring (RUM)

```javascript
// Report Core Web Vitals from real users
import { onCLS, onFID, onLCP, onINP, onTTFB } from 'web-vitals';

const sendToAnalytics = ({ name, value, rating, navigationType }) => {
  fetch('/api/vitals', {
    method: 'POST',
    body: JSON.stringify({ metric: name, value, rating, navigationType }),
    keepalive: true, // survives page unload
  });
};

onCLS(sendToAnalytics);
onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

---

### Frontend Error Monitoring

```javascript
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: 'https://xxx@sentry.io/xxx',
  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration({
      maskAllText: true,    // GDPR: mask user input in replays
      blockAllMedia: false,
    }),
  ],
  tracesSampleRate: 0.1,   // 10% of transactions
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0, // 100% of sessions with errors
});

// Add user context for better error grouping
Sentry.setUser({ id: user.id, email: user.email });

// Custom error context
Sentry.captureException(error, {
  tags: { component: 'CheckoutFlow', step: 'payment' },
  extra: { orderId, cartValue },
});
```

---

## 22. Error Handling at Scale

### Error Boundaries

```jsx
// Class component (only class components can be error boundaries)
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    Sentry.captureException(error, { extra: info });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <GenericErrorScreen onReset={() => this.setState({ hasError: false })} />;
    }
    return this.props.children;
  }
}

// Granular error boundaries — isolate failures to the smallest scope
<ErrorBoundary fallback={<HeaderFallback />}>
  <Header />
</ErrorBoundary>

<ErrorBoundary fallback={<FeedError />}>
  <NewsFeed />
</ErrorBoundary>
```

---

### Global Error Handling Strategy

```javascript
// 1. Unhandled promise rejections
window.addEventListener('unhandledrejection', (event) => {
  Sentry.captureException(event.reason);
  event.preventDefault(); // prevent console error
});

// 2. Global JS errors
window.addEventListener('error', (event) => {
  Sentry.captureException(event.error, {
    extra: { filename: event.filename, line: event.lineno },
  });
});

// 3. API error normalization
const normalizeApiError = (error) => {
  if (error.response) {
    // Server responded with error status
    return { type: 'API_ERROR', status: error.response.status, message: error.response.data?.message };
  } else if (error.request) {
    // Request made but no response (network error)
    return { type: 'NETWORK_ERROR', message: 'No internet connection' };
  } else {
    // Request setup error
    return { type: 'CLIENT_ERROR', message: error.message };
  }
};
```

---

## 23. Analytics & Tracking Architecture

### Event-Driven Analytics Design

```javascript
// analytics.ts — centralized analytics module
const analyticsClient = {
  track: (event, properties) => {
    // Send to multiple destinations
    window.analytics?.track(event, properties);  // Segment
    window.gtag?.('event', event, properties);    // Google Analytics
    posthog?.capture(event, properties);          // PostHog
  },
  identify: (userId, traits) => {
    window.analytics?.identify(userId, traits);
  },
  page: (name, properties) => {
    window.analytics?.page(name, properties);
  },
};

// Typed events (prevents typos in event names)
export const Analytics = {
  signUpStarted:  (method) => analyticsClient.track('Sign Up Started', { method }),
  signUpCompleted:(method) => analyticsClient.track('Sign Up Completed', { method }),
  productViewed:  (id, name, price) => analyticsClient.track('Product Viewed', { id, name, price }),
  addedToCart:    (id, price) => analyticsClient.track('Product Added', { id, price }),
  orderCompleted: (id, revenue) => analyticsClient.track('Order Completed', { id, revenue }),
};
```

---

### Privacy-First Analytics

```javascript
// GDPR: only track after consent
const { analyticsConsent } = useConsent();

useEffect(() => {
  if (analyticsConsent) {
    loadAnalyticsScript();
  }
}, [analyticsConsent]);

// IP anonymization in GA4:
gtag('config', 'G-XXXXXXXX', {
  anonymize_ip: true,
  cookie_flags: 'SameSite=None;Secure',
});
```

---

## 24. Scalability Patterns

### Feature Flags

Control feature rollout without deploying new code.

```javascript
import { useFeatureFlag } from '@growthbook/growthbook-react';

// Gradually roll out to users
const NewCheckoutFlow = () => {
  const isNewCheckout = useFeatureFlag('new-checkout-flow');

  return isNewCheckout
    ? <NewCheckout />
    : <LegacyCheckout />;
};

// A/B testing
const { value: ctaText } = useFeatureFlag('cta-text', 'Buy Now');
<Button>{ctaText}</Button>  // "Buy Now" or "Add to Cart" based on experiment
```

---

### Design Tokens at Scale

```javascript
// tokens.json — single source of truth, shared across web + mobile + design tools
{
  "color": {
    "brand": {
      "primary":   { "value": "#6200EE" },
      "secondary": { "value": "#03DAC6" }
    },
    "semantic": {
      "error":   { "value": "{color.red.500}" },
      "success": { "value": "{color.green.500}" }
    }
  },
  "spacing": {
    "1": { "value": "4px" },
    "2": { "value": "8px" },
    "4": { "value": "16px" },
    "8": { "value": "32px" }
  }
}

// Style Dictionary transforms tokens → CSS variables, iOS Swift, Android XML
// CSS output:
// :root { --color-brand-primary: #6200EE; --spacing-4: 16px; }
```

---

## 25. Design Interview: How to Answer

### The RADIO Framework

```
R — Requirements Clarification
  Ask before designing. Never assume.
  - Who are the users? (internal / consumer / mobile-first?)
  - What's the scale? (100 users / 10M users?)
  - What platforms? (web / mobile / both?)
  - What are the key user flows? (what does this thing need to DO?)
  - Are there non-functional requirements? (offline support? real-time? i18n?)

A — Architecture (High Level)
  Draw the overall structure:
  - Component hierarchy
  - Data flow
  - Client-server interaction
  - Key modules and their responsibilities

D — Data Model
  - What data does the app need?
  - How is it structured on the client?
  - How does it come from the server?
  - How is it cached?

I — Interface Design
  - API interface (what endpoints/queries are needed)
  - Component API (props, events)
  - State interface (what's in the store and its shape)

O — Optimizations
  - Performance (code splitting, caching, virtual scrolling)
  - Accessibility
  - Offline support
  - Error handling
  - Security considerations
```

---

## 26. Classic FE System Design Problems — Solved

---

### Design a News Feed (Facebook / Twitter)

```
Requirements clarified:
  - User sees posts from people they follow
  - Infinite scroll
  - Real-time updates for new posts
  - Like, comment, share actions

Architecture:
  Rendering: SSR for initial load (SEO + first paint) → CSR for interactions
  Data: React Query for paginated posts + WebSocket for real-time new posts

Component tree:
  FeedPage
  ├── FeedHeader (create post)
  ├── NewPostsBanner ("3 new posts — tap to load")
  └── FeedList (virtualized with react-window)
      └── PostCard (memo'd)
          ├── PostHeader (avatar, name, time)
          ├── PostContent (text, media)
          └── PostActions (like, comment, share)

Data flow:
  1. Initial: SSR fetches first 20 posts → renders
  2. Client hydrates, React Query takes over cache
  3. WebSocket subscribes to feed channel
  4. New post arrives via WebSocket → store in pendingPosts array
  5. Show "3 new posts" banner — user taps → prepend to feed
  6. Infinite scroll → fetchNextPage() on intersection

State:
  - posts: React Query infinite query (server state)
  - pendingPosts: useState (UI state — new posts not yet shown)
  - likedPostIds: optimistic update in React Query cache

Performance:
  - Virtual scrolling (only 15 posts rendered in DOM)
  - Image lazy loading + blurhash placeholder
  - PostCard wrapped in React.memo
  - Intersection Observer for scroll trigger (not scroll event listener)
```

---

### Design a Google Docs-like Collaborative Editor

```
Requirements:
  - Multiple users edit same document simultaneously
  - Changes reflected in real-time for all users
  - Works offline (queue edits, sync when back online)
  - Version history

Architecture:
  Real-time: WebSocket (Yjs CRDT library)
  Offline: Yjs works offline natively — changes stored locally, synced on reconnect
  Conflict resolution: CRDT (mathematical guarantee — no conflicts possible)

Key decisions:
  1. Don't send the entire document on every change
     Send operational transforms / CRDT updates (tiny diffs)

  2. Cursor positions of other users: ephemeral state via WebSocket
     Not persisted — if they disconnect, cursor disappears

  3. Autosave: debounced 2 seconds after last keystroke
     Yjs also syncs via WebSocket in real-time

  4. Offline: Yjs stores all changes in IndexedDB
     On reconnect: push local changes, pull remote changes
     CRDT merges them — no conflicts

Component:
  DocumentEditor
  ├── Toolbar (bold, italic, lists)
  ├── CollaboratorCursors (other users' positions)
  ├── Editor (contenteditable or ProseMirror)
  └── StatusBar (saving... / saved / offline)
```

---

### Design an Autocomplete Search

```
Requirements:
  - Suggestions as user types
  - <100ms perceived latency
  - Keyboard navigation
  - Works on mobile

Key decisions:

1. Debounce input (300ms):
   Don't fire API on every keystroke — wait until user pauses

2. Cache previous queries:
   useQuery with { staleTime: 60000 } — "react" always returns same results
   React Query caches by queryKey: ['search', query]

3. Prefetch first letter results:
   On input focus, prefetch results for likely first letters

4. Abort previous request:
   AbortController — if user types "re", then "rea" quickly,
   cancel the "re" request before "rea" request completes

5. Keyboard navigation:
   Track highlightedIndex with useReducer
   ArrowUp/Down: move highlight
   Enter: select highlighted result
   Escape: close dropdown, return focus to input

6. Accessibility:
   role="combobox" on input
   role="listbox" on dropdown
   aria-activedescendant pointing to highlighted option
   aria-expanded on input

const useSearch = (query) => {
  const debouncedQuery = useDebounce(query, 300);

  return useQuery({
    queryKey: ['search', debouncedQuery],
    queryFn: ({ signal }) => searchApi(debouncedQuery, { signal }),
    enabled: debouncedQuery.length >= 2,
    staleTime: 60 * 1000,
    placeholderData: keepPreviousData, // no flash of empty while typing
  });
};
```

---

### Design a Video Streaming UI (YouTube / Netflix)

```
Key decisions:

1. Adaptive Bitrate Streaming (ABR):
   HLS or DASH — server provides multiple quality levels
   Player (Video.js / HLS.js) automatically selects quality based on bandwidth

2. Thumbnail seeking:
   Generate sprite sheet of thumbnails at build time
   On hover: show thumbnail from sprite using CSS background-position

3. Buffering strategy:
   Buffer 30 seconds ahead (never buffer entire video)
   Show buffer progress in seek bar

4. Offline / PWA:
   Service Worker caches video chunks as user watches
   Next episode pre-fetched while watching current

5. Autoplay next episode:
   Start fetching next episode data 3 minutes before current ends
   10 second countdown modal — user can cancel or skip

6. Performance:
   Lazy load comments (not visible on initial render)
   Virtualize comment list
   Intersection Observer to pause video when not visible (mobile)
   requestVideoFrameCallback for precise thumbnail sync
```

---

## Quick Reference — When to Use What

| Problem | Solution |
|---|---|
| Data from API, caching, loading states | React Query / SWR |
| Client-only UI state, preferences | useState + MMKV |
| Complex shared state, many consumers | Zustand |
| Form validation and submission | React Hook Form |
| Real-time updates (chat, feeds) | WebSocket + React Query |
| SEO + fast first load | SSR or SSG |
| Dashboard behind auth | CSR |
| Large list (1000+ items) | react-window virtual scroll |
| Heavy computation | Web Worker |
| Route-based code splitting | React.lazy + Suspense |
| Offline support | Service Worker + IndexedDB |
| Component library | Headless components + tokens |
| Multi-team frontend at scale | Micro-frontends (Module Federation) |
| Type-safe full-stack | tRPC |
| Flexible data fetching, avoid over-fetching | GraphQL |

---

*Frontend system design is ultimately about making the right trade-offs — speed vs freshness, flexibility vs consistency, team autonomy vs product coherence. Every decision has a cost. Know the costs.*
