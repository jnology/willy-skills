---
name: willform-coding
description: Full-stack coding patterns for Willform apps. Use when writing any code, building features, or setting up project structure.
user-invocable: false
---

# Willform Coding Patterns

Build complete, production-quality apps on the Willform platform.

## When to Use

- Modifying a skeleton app to match user requirements
- Adding features to existing project
- Writing components or pages
- Setting up database models
- Any code generation task

## Success Criteria

Every generated app must satisfy ALL of these before committing:

1. All files compile without TypeScript errors
2. Every `import` resolves to an existing file you created
3. Every route in navigation links to an existing page
4. Prisma schema includes all models referenced in code
5. Server Actions return `{ error }` on all validation failures
6. `package.json` has `"build": "next build"` only (no prisma commands)
7. `next.config.ts` includes `output: "standalone"`
8. **ALL UI text MUST be in English** — Korean, Japanese, Chinese, or any non-English text in UI is a build-blocking violation
9. Home page has a visually impactful hero section (not just plain text)
10. All list cards have hover effects and consistent rounded-2xl styling
11. Every user action (submit, delete, save) shows toast feedback
12. Data-dependent views have skeleton loading states (never blank screen)
13. Destructive actions (delete, reset) require confirmation modal

## Default Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Next.js (App Router, Server Components) |
| Language | TypeScript |
| Styling | Tailwind CSS with CSS variable design tokens |
| UI Components | Pure Tailwind — no shadcn/ui, MUI, Chakra, Ant Design |
| Database | PostgreSQL + Prisma |
| Config | `output: "standalone"` in next.config.ts |

IMPORTANT: Do NOT use Vite, Express, or React without Next.js. The platform build pipeline is designed for Next.js standalone output only.

IMPORTANT: Do NOT install any external UI component library (shadcn/ui, MUI, Chakra UI, Ant Design, Headless UI, Radix, DaisyUI). Use pure Tailwind CSS + React hooks for ALL UI patterns including toast, modal, and loading states.

## Project Structure

```
app/
├── app/
│   ├── layout.tsx          # Root layout (nav, providers)
│   ├── page.tsx            # Home page (Server Component)
│   ├── globals.css         # Tailwind imports
│   ├── [feature]/
│   │   └── page.tsx        # Feature page
│   └── api/
│       └── [resource]/
│           └── route.ts    # API route (if needed)
├── components/
│   ├── [Component].tsx     # Reusable components
│   └── [ClientComp].tsx    # Client components ('use client')
├── lib/
│   ├── db.ts               # Prisma singleton
│   ├── actions.ts          # Server Actions
│   └── utils.ts            # Helpers
├── prisma/
│   └── schema.prisma       # Database schema
├── next.config.ts          # Must include output: "standalone"
├── tailwind.config.ts
├── package.json
├── Dockerfile              # See DATABASE.md for template
└── tsconfig.json
```

## Code Patterns

### Server Component (default)

```tsx
import { prisma } from '@/lib/db'

export const dynamic = 'force-dynamic'

export default async function ProductList() {
  const products = await prisma.product.findMany({
    orderBy: { createdAt: 'desc' }
  })

  return (
    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
      {products.map(product => (
        <div key={product.id} className="bg-card rounded-xl border border-border p-4">
          <h3 className="font-semibold text-foreground">{product.name}</h3>
          <p className="text-muted-foreground">{product.description}</p>
        </div>
      ))}
    </div>
  )
}
```

### Dynamic Route Page (Next.js 15 — CRITICAL)

In Next.js 15, `params` is a **Promise** and MUST be awaited. Using synchronous params will cause a TypeScript build error: `Type '{ params: { id: string; }; }' does not satisfy the constraint 'PageProps'`.

```tsx
import { prisma } from '@/lib/db'
import { notFound } from 'next/navigation'

export const dynamic = 'force-dynamic'

// CORRECT — params is a Promise in Next.js 15
export default async function PostDetailPage({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params

  const post = await prisma.post.findUnique({ where: { id } })
  if (!post) notFound()

  return <div>{post.title}</div>
}
```

Similarly, `searchParams` is also a Promise in Next.js 15:

```tsx
export default async function PostsPage({
  searchParams,
}: {
  searchParams: Promise<{ category?: string }>
}) {
  const { category } = await searchParams
  // use category...
}
```

**NEVER do this (will fail to build):**
```tsx
// WRONG — synchronous params causes TypeScript error in Next.js 15
export default async function Page({ params }: { params: { id: string } }) {
  const post = await prisma.post.findUnique({ where: { id: params.id } })

// WRONG — synchronous searchParams also causes TypeScript error
export default async function Page({ searchParams }: { searchParams: { q?: string } }) {
```

### Client Component

Add `'use client'` at the top. Use for interactive UI only (useState, onClick, useEffect). Keep Server Components as default for data fetching.

### Server Action (mutations)

```tsx
'use server'

import { prisma } from '@/lib/db'
import { revalidatePath } from 'next/cache'

export async function createItem(formData: FormData) {
  const name = formData.get('name') as string
  if (!name) return { error: 'Name is required' }

  await prisma.item.create({ data: { name } })
  revalidatePath('/')
  return { success: true }
}
```

### Prisma Model

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Item {
  id        String   @id @default(cuid())
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### Prisma Singleton (lib/db.ts)

Use the standard global singleton pattern to prevent multiple PrismaClient instances in development. `lib/db.ts` already exists in the workspace — reuse it, do not recreate.

### Admin Password Protection

`lib/auth.ts` and admin API routes are pre-committed to the workspace. **Do NOT recreate them.**

Available imports from `@/lib/auth`:
- `verifyPassword(password)` — timing-safe password check
- `createAdminToken()` — creates HMAC-signed admin cookie token
- `verifyAdminToken(token)` — verifies token signature

Pre-committed API routes:
- `POST /api/admin/login` — verifies password, sets signed cookie
- `GET /api/admin/check` — verifies signed cookie
- `POST /api/admin/logout` — deletes cookie

Admin layout pattern:
```tsx
// app/admin/layout.tsx — "use client", check /api/admin/check, show login or children
```

**Security rules:**
- NEVER recreate `lib/auth.ts` — import from existing file
- NEVER use plain cookies like `admin-auth=true`
- NEVER hardcode default passwords
- NEVER include password hints, default password text, or any password-related help text in the login form UI (e.g., "Default password: admin1234", "Password: admin1234", placeholder="admin1234")
- NEVER generate code that displays, logs, or exposes the admin password in any UI element, HTML comment, or console output
- NEVER fabricate, guess, or make up an admin password in conversation (e.g., "admin1234") — the password is auto-generated by the platform and you do NOT have access to it
- The login form should show ONLY a password input field and a submit button — no hints, no defaults, no helper text about what the password might be
- ALWAYS use `verifyAdminToken()` in admin server actions:
```typescript
import { verifyAdminToken } from "@/lib/auth";
import { cookies } from "next/headers";

export async function adminAction() {
  const token = (await cookies()).get("admin_token")?.value ?? "";
  if (!verifyAdminToken(token)) throw new Error("Unauthorized");
  // ... action logic
}
```

**NEVER do these (causes redirect loop):**
- `redirect('/admin/login')` from admin layout
- Create `middleware.ts` for admin auth redirect
- Create a separate `/admin/login` page

### Cart Context + localStorage

`lib/cart-context.tsx` — `'use client'` component with `CartProvider` and `useCart` hook.

Key structure:
- `CartItem = { id, name, price, quantity }`
- `useState<CartItem[]>` + two `useEffect`: load on mount (SSR-safe: check `typeof window`), sync on change
- Functions: `addItem` (find existing → increment, else push qty 1), `removeItem`, `updateQuantity`, `clearCart`
- `totalAmount` = `items.reduce((sum, i) => sum + i.price * i.quantity, 0)`
- Wrap in root layout: `<CartProvider>{children}</CartProvider>`

### Form Validation + Error Display

Server Action returns `{ errors: Record<string, string> }`, client displays per field.

```tsx
// Server Action — validate, return { errors } or create + revalidatePath
export async function createProduct(_prev: any, formData: FormData) {
  const errors: Record<string, string> = {}
  if (!formData.get('name')) errors.name = 'Product name is required'
  if (Number(formData.get('price')) <= 0) errors.price = 'Price must be greater than 0'
  if (Object.keys(errors).length > 0) return { errors }
  // ... create + revalidatePath → return { errors: {} }
}
// Client: useActionState(createProduct, { errors: {} })
// Display: {state.errors.name && <p className="text-red-500 text-sm">...</p>}
```

### Pagination (Server-Side)

```tsx
const PAGE_SIZE = 12
export default async function Page({ searchParams }: { searchParams: Promise<{ page?: string }> }) {
  const { page } = await searchParams
  const currentPage = Math.max(1, Number(page) || 1)
  const [items, total] = await Promise.all([
    prisma.product.findMany({ skip: (currentPage - 1) * PAGE_SIZE, take: PAGE_SIZE, orderBy: { createdAt: 'desc' } }),
    prisma.product.count(),
  ])
  const totalPages = Math.ceil(total / PAGE_SIZE)
  // Render items + page navigation Links: <Link href={`?page=${i+1}`}>
}
```

### Search + Category Filter

`'use client'` component using `useSearchParams` + `router.push` to update URL params.

```tsx
// components/SearchFilter.tsx — 'use client'
const update = (key: string, value: string) => {
  const params = new URLSearchParams(searchParams.toString())
  value ? params.set(key, value) : params.delete(key)
  router.push(`?${params.toString()}`)
}
// Render: input for search (onChange → update('q', ...)), select for category filter
```

IMPORTANT: Wrap in `<Suspense>` when used: `<Suspense fallback={null}><SearchFilter /></Suspense>`
Server page reads `searchParams` to filter: `prisma.product.findMany({ where: { name: { contains: q } } })`

### Prisma Relations

1:N with cascade delete, include queries, nested create.

```prisma
model Category {
  id String @id @default(cuid()); name String; products Product[]
}
model Product {
  id String @id @default(cuid()); name String; price Int
  category Category @relation(fields: [categoryId], references: [id], onDelete: Cascade)
  categoryId String
}
```

```typescript
const products = await prisma.product.findMany({ include: { category: true } })
await prisma.category.create({
  data: { name: 'Electronics', products: { create: [{ name: 'Laptop', price: 1200 }] } },
})
```

### Seed Data Server Action

Idempotent: check count first, use `createMany`, `revalidatePath` after.

```typescript
export async function seedSampleData() {
  if ((await prisma.product.count()) > 0) return { error: 'Data already exists' }
  await prisma.product.createMany({ data: [
    { name: 'Sample Product 1', price: 2999, imageUrl: 'https://picsum.photos/seed/p1/400/300' },
    { name: 'Sample Product 2', price: 4999, imageUrl: 'https://picsum.photos/seed/p2/400/300' },
  ]})
  revalidatePath('/'); return { success: true }
}
```

Admin button: `<button onClick={() => seedSampleData()}>Add Sample Data</button>`

### Admin Layout

`app/admin/layout.tsx` — sidebar nav with `Link` components, `bg-background` background.

```tsx
const NAV = [{ href: '/admin', label: 'Dashboard' }, { href: '/admin/products', label: 'Products' }]
// Layout: flex container, left nav (w-56, bg-card, border-r border-border), right main (flex-1, p-8)
// Map NAV → <Link> with hover:bg-accent styling
```

### Image URL Field

Schema: `imageUrl String?`. Display: `<img>` if exists, else fallback `<div>` with first letter + `bg-muted`.
Admin form: `<input type="url" name="imageUrl" placeholder="https://..." defaultValue={product?.imageUrl || ''} />`

## App Completeness Checklist

Every generated app MUST include ALL of the following. Missing any item = incomplete app.

### Required Structure
- [ ] `prisma/schema.prisma` with all models for the feature
- [ ] `lib/db.ts` — Prisma singleton
- [ ] `lib/actions.ts` — Server Actions for all CRUD operations
- [ ] `lib/utils.ts` — formatters, helpers
- [ ] `next.config.ts` with `output: "standalone"`
- [ ] `Dockerfile` — use template from DATABASE.md (NEVER write your own)
- [ ] `components/toast.tsx` — Toast context provider + useToast hook
- [ ] `components/modal.tsx` — Confirm modal component
- [ ] All referenced components exist as files

### Required Quality
- [ ] TypeScript for all files (`.tsx`, `.ts`)
- [ ] Responsive layout (mobile-first, Tailwind breakpoints)
- [ ] **English-only UI** — every label, message, placeholder, empty state, button, heading, tooltip MUST be in English. Zero non-English strings allowed.
- [ ] Loading/empty states for all data-dependent views
- [ ] Error handling in all Server Actions (return `{ error }` on failure)
- [ ] Proper Prisma relations with cascading deletes where appropriate
- [ ] Clean, professional design with consistent spacing and colors

### CRITICAL Build Rules
- [ ] Dockerfile uses isolated prisma-gen stage (see DATABASE.md)
- [ ] `mkdir -p public` in builder stage before `npm run build`
- [ ] CMD is `["node", "server.js"]` — NEVER include prisma commands
- [ ] node:20-alpine (NOT node:22)
- [ ] Never modify `.forgejo/workflows/`

## Best Practices

| Category | Practice |
|----------|----------|
| **Components** | Server Components default. `'use client'` only for interactivity |
| **Data Fetching** | Direct Prisma calls in Server Components |
| **Mutations** | Server Actions with `revalidatePath()` |
| **State** | useState for local UI, Context for cross-component (e.g. cart) |
| **Forms** | `<form action={serverAction}>` pattern |
| **Styling** | Tailwind CSS variable tokens (bg-primary, text-foreground, etc.), rounded-xl for cards |

## Common Pitfalls

| Mistake | Solution |
|---------|----------|
| Synchronous `params` in dynamic routes | Next.js 15 requires `params: Promise<{...}>` with `await params` — see Dynamic Route pattern above |
| `prisma db push` in package.json build | Build runs in isolated container without DB access. Use `"build": "next build"` only. prisma generate is in Dockerfile |
| Missing prisma/schema.prisma | ALWAYS create schema BEFORE components |
| Using Express/Vite | Use Next.js App Router — it's the only supported framework |
| Single-stage Dockerfile | Use multi-stage from DATABASE.md |
| Runtime prisma in CMD | NEVER — use `node server.js` only |
| Missing `output: "standalone"` | Required in next.config.ts |
| `'use client'` everywhere | Only for interactive components |
| Components referencing missing files | Create ALL files before committing |
| Leftover imports after app replacement | Delete ALL old files before writing new app (see willform-project Full App Replacement) |
| Old layout.tsx nav linking to deleted pages | Rewrite layout.tsx completely for the new app |
| Admin redirect loop | `/admin/login` inside admin layout causes infinite redirect | Use inline `<AdminLoginForm />` in layout, no separate login page |
| middleware.ts for admin auth | Middleware + layout both redirect = loop | No middleware.ts — use layout auth only |
| No `tailwind-merge` in deps | Add if using `cn()` utility |
| `"build": "prisma db push && next build"` in package.json | DB not available during build. Use `"build": "next build"` only |
| No toast after form submit | Add `ToastProvider` in layout + `useToast()` in forms — see Toast Pattern |
| No loading state during data fetch | Add skeleton with `animate-pulse` — see Loading Skeleton Pattern |
| External UI library (shadcn, MUI, etc.) | Use pure Tailwind patterns from this SKILL — zero external UI deps |

## Dockerfile

DO NOT write your own Dockerfile. Copy the template from DATABASE.md exactly.
It handles: isolated prisma stage, confbox conflict, standalone output, non-root user.

## Visual Design System

Concrete patterns to copy into generated apps. Use these exact className strings.

### Design Tokens

```tsx
// The skeleton provides this layout - DO NOT modify
<body className="font-sans antialiased bg-background text-foreground min-h-screen">
```

We use system `font-sans` — do NOT import `Inter` or any font from `next/font/google`. Google Fonts cause Docker build failures in air-gapped environments.

Typography weights: `font-normal` body, `font-medium` labels, `font-semibold` headings, `font-bold` hero.
Spacing rhythm: sections `py-16`, cards `p-6`, form gaps `space-y-4`, grid `gap-6`.

### Color System

All colors use CSS variable design tokens. NEVER use hardcoded Tailwind colors (e.g., `bg-blue-600`, `text-gray-900`).

| Token | Tailwind Class | Usage |
|-------|----------------|-------|
| primary | bg-primary, text-primary | Buttons, links, active states |
| primary-foreground | text-primary-foreground | Text on primary bg |
| background | bg-background | Page background |
| foreground | text-foreground | Main text |
| card | bg-card | Card/panel backgrounds |
| muted | bg-muted | Secondary backgrounds |
| muted-foreground | text-muted-foreground | Secondary text |
| border | border-border | Borders, dividers |
| input | border-input | Form input borders |
| ring | ring-ring | Focus rings |
| destructive | bg-destructive | Error/delete actions |
| success | bg-success, text-success | Success states, confirmations |
| warning | bg-warning, text-warning | Warning states, pending items |
| secondary | bg-secondary | Tertiary backgrounds |
| accent | bg-accent | Hover highlights |

Buttons: `text-primary-foreground rounded-lg px-6 py-3 font-semibold transition-colors`.

### Hero Pattern

```tsx
<section className="bg-gradient-to-br from-primary to-primary/80 text-primary-foreground py-20">
  <div className="max-w-6xl mx-auto px-6 text-center">
    <h1 className="text-4xl md:text-5xl font-bold mb-4">Title</h1>
    <p className="text-lg text-primary-foreground/80 mb-8 max-w-2xl mx-auto">Subtitle</p>
    <a href="/products" className="inline-block bg-card text-foreground font-semibold px-8 py-4 rounded-xl hover:bg-accent transition-colors">
      Browse
    </a>
  </div>
</section>
```

Every home page MUST have a hero.

### Card Pattern

```tsx
<div className="bg-card rounded-2xl border border-border overflow-hidden hover:shadow-lg transition-shadow">
  {imageUrl ? (
    <div className="aspect-[4/3] overflow-hidden">
      <img src={imageUrl} alt={name} className="w-full h-full object-cover hover:scale-105 transition-transform duration-300" />
    </div>
  ) : (
    <div className="aspect-[4/3] bg-muted flex items-center justify-center">
      <span className="text-4xl font-bold text-muted-foreground">{name.charAt(0)}</span>
    </div>
  )}
  <div className="p-5">
    <h3 className="text-foreground font-semibold text-lg mb-1">{name}</h3>
    <p className="text-muted-foreground text-sm line-clamp-2 mb-3">{description}</p>
    <span className="font-semibold text-primary">${price.toFixed(2)}</span>
  </div>
</div>
```

All list items MUST use this card structure with `rounded-2xl` and `hover:shadow-lg`.

### Navigation Pattern

```tsx
<nav className="sticky top-0 z-50 bg-card/80 backdrop-blur-md border-b border-border">
  <div className="max-w-6xl mx-auto px-6 h-16 flex items-center justify-between">
    <a href="/" className="font-bold text-xl text-foreground">Site Name</a>
    <div className="flex items-center gap-6">
      <a href="/products" className="text-muted-foreground hover:text-foreground transition-colors">Products</a>
      <a href="/cart" className="relative text-muted-foreground hover:text-foreground">
        Cart
        {count > 0 && <span className="absolute -top-2 -right-4 bg-primary text-primary-foreground text-xs w-5 h-5 rounded-full flex items-center justify-center">{count}</span>}
      </a>
    </div>
  </div>
</nav>
```

Navigation MUST be `sticky top-0` with `backdrop-blur-md`.

### Category Tabs Pattern

```tsx
<div className="flex gap-2 overflow-x-auto pb-2">
  {categories.map(cat => (
    <button
      key={cat.id}
      onClick={() => setActive(cat.id)}
      className={`px-4 py-2 rounded-full text-sm font-medium whitespace-nowrap transition-colors ${
        active === cat.id
          ? 'bg-primary text-primary-foreground'
          : 'bg-muted text-muted-foreground hover:bg-accent'
      }`}
    >
      {cat.name}
    </button>
  ))}
</div>
```

### Empty State Pattern

```tsx
<div className="text-center py-16">
  <div className="text-6xl mb-4 text-muted-foreground/50">📝</div>
  <h3 className="text-lg font-semibold text-muted-foreground mb-2">No items found</h3>
  <p className="text-muted-foreground/70 text-sm">Get started by adding your first item.</p>
</div>
```

### Admin Sidebar Pattern

```tsx
<aside className="w-64 bg-card border-r border-border min-h-screen p-4">
  <h2 className="font-bold text-lg mb-6 px-3 text-foreground">Admin</h2>
  <nav className="space-y-1">
    {NAV_ITEMS.map(item => (
      <a
        key={item.href}
        href={item.href}
        className={`block px-3 py-2.5 rounded-lg text-sm font-medium transition-colors ${
          isActive(item.href)
            ? 'bg-primary/10 text-primary'
            : 'text-muted-foreground hover:bg-accent hover:text-foreground'
        }`}
      >
        {item.label}
      </a>
    ))}
  </nav>
</aside>
```

### Form Input Pattern

```tsx
<div>
  <label className="block text-sm font-medium text-foreground mb-1">Product Name</label>
  <input
    type="text"
    name="name"
    className="w-full px-4 py-2.5 border border-input rounded-lg focus:ring-2 focus:ring-ring focus:border-transparent outline-none transition-all bg-card text-foreground"
    placeholder="Enter product name"
  />
  {errors?.name && <p className="mt-1 text-sm text-destructive">{errors.name}</p>}
</div>
```

All inputs MUST have `focus:ring-2` and `rounded-lg`. Never use unstyled browser inputs.

### Toast Notification Pattern

```tsx
// components/toast.tsx — 'use client'
'use client'
import { createContext, useContext, useState, useCallback } from 'react'

type Toast = { id: number; message: string; type: 'success' | 'error' }
const ToastCtx = createContext<{ toast: (msg: string, type?: 'success' | 'error') => void }>({ toast: () => {} })
export function useToast() { return useContext(ToastCtx) }

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([])
  const toast = useCallback((message: string, type: 'success' | 'error' = 'success') => {
    const id = Date.now()
    setToasts(prev => [...prev, { id, message, type }])
    setTimeout(() => setToasts(prev => prev.filter(t => t.id !== id)), 3000)
  }, [])
  return (
    <ToastCtx.Provider value={{ toast }}>
      {children}
      <div className="fixed bottom-6 right-6 z-50 space-y-2">
        {toasts.map(t => (
          <div key={t.id} className={`px-4 py-3 rounded-lg shadow-lg text-white text-sm font-medium animate-slide-up ${
            t.type === 'success' ? 'bg-success' : 'bg-destructive'}`}>{t.message}</div>
        ))}
      </div>
    </ToastCtx.Provider>
  )
}
```

`globals.css`: `@keyframes slide-up { from { opacity:0; transform:translateY(1rem) } to { opacity:1; transform:translateY(0) } } .animate-slide-up { animation: slide-up 0.2s ease-out; }`

Wrap in root layout: `<ToastProvider>{children}</ToastProvider>`. Usage: `const { toast } = useToast(); toast('Saved successfully');`

### Confirm Modal Pattern

```tsx
// components/modal.tsx — 'use client'
'use client'
import { useEffect, useCallback } from 'react'

export function ConfirmModal({ open, onConfirm, onCancel, title, description }: {
  open: boolean; onConfirm: () => void; onCancel: () => void; title: string; description?: string
}) {
  const onKey = useCallback((e: KeyboardEvent) => { if (e.key === 'Escape') onCancel() }, [onCancel])
  useEffect(() => {
    if (!open) return
    document.addEventListener('keydown', onKey)
    document.body.style.overflow = 'hidden'
    return () => { document.removeEventListener('keydown', onKey); document.body.style.overflow = '' }
  }, [open, onKey])
  if (!open) return null
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center">
      <div className="absolute inset-0 bg-black/50" onClick={onCancel} />
      <div className="relative bg-card rounded-2xl p-6 max-w-sm w-full mx-4 shadow-xl">
        <h3 className="text-lg font-semibold text-foreground mb-2">{title}</h3>
        {description && <p className="text-muted-foreground text-sm mb-6">{description}</p>}
        <div className="flex gap-3 justify-end">
          <button onClick={onCancel} className="px-4 py-2 text-sm font-medium text-foreground bg-secondary rounded-lg hover:bg-accent">Cancel</button>
          <button onClick={onConfirm} className="px-4 py-2 text-sm font-medium text-primary-foreground bg-destructive rounded-lg hover:bg-destructive/90">Confirm</button>
        </div>
      </div>
    </div>
  )
}
```

Usage: `useState(false)` for `open`. Show on delete click, execute on confirm.

### Loading Skeleton Pattern

```tsx
function CardSkeleton() {
  return (
    <div className="bg-card rounded-2xl border border-border overflow-hidden animate-pulse">
      <div className="aspect-[4/3] bg-muted" />
      <div className="p-5 space-y-3">
        <div className="h-5 bg-muted rounded w-3/4" />
        <div className="h-4 bg-muted rounded w-full" />
      </div>
    </div>
  )
}
// Usage: <Suspense fallback={<div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">{Array.from({length:6}).map((_,i)=><CardSkeleton key={i}/>)}</div>}>
```

### Button Loading State Pattern

```tsx
<button type="submit" disabled={pending}
  className="bg-primary text-primary-foreground px-6 py-3 rounded-lg font-semibold hover:bg-primary/90 transition-colors disabled:opacity-50 disabled:cursor-not-allowed flex items-center gap-2">
  {pending && <svg className="animate-spin h-4 w-4" viewBox="0 0 24 24" fill="none">
    <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" />
    <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" />
  </svg>}
  {pending ? 'Processing...' : 'Save'}
</button>
```

Button variants:
```tsx
<button className="bg-primary text-primary-foreground px-4 py-2 rounded-lg hover:bg-primary/90">
  Primary
</button>
<button className="bg-secondary text-foreground px-4 py-2 rounded-lg hover:bg-accent">
  Secondary
</button>
```

Every submit button MUST show loading state via `useActionState` or `useState`.

### Stats Card Pattern

```tsx
<div className="bg-card rounded-2xl border border-border p-6">
  <p className="text-sm font-medium text-muted-foreground mb-1">Total Orders</p>
  <p className="text-3xl font-bold text-foreground">128</p>
</div>
// Grid: grid grid-cols-2 lg:grid-cols-4 gap-6
```

### Badge Pattern

```tsx
function Badge({ status }: { status: string }) {
  const styles: Record<string, string> = {
    active: 'bg-green-100 text-green-700', pending: 'bg-yellow-100 text-yellow-700',
    inactive: 'bg-gray-100 text-gray-600', error: 'bg-red-100 text-red-700',
  }
  return <span className={`inline-flex px-2.5 py-0.5 rounded-full text-xs font-medium ${styles[status] || styles.inactive}`}>{status}</span>
}
```

### Table Pattern

```tsx
<div className="bg-card rounded-2xl border border-border overflow-hidden overflow-x-auto">
  <table className="w-full text-sm">
    <thead className="bg-secondary border-b border-border">
      <tr>
        <th className="text-left px-6 py-3 font-medium text-muted-foreground">Name</th>
        <th className="text-right px-6 py-3 font-medium text-muted-foreground">Amount</th>
      </tr>
    </thead>
    <tbody className="divide-y divide-border">
      {items.map(item => (
        <tr key={item.id} className="hover:bg-accent transition-colors">
          <td className="px-6 py-4 font-medium text-foreground">{item.name}</td>
          <td className="px-6 py-4 text-right text-foreground">${item.amount.toLocaleString()}</td>
        </tr>
      ))}
    </tbody>
  </table>
</div>
```

### Footer Pattern

```tsx
<footer className="bg-foreground text-background/70 mt-auto">
  <div className="max-w-6xl mx-auto px-6 py-12 flex flex-col md:flex-row justify-between gap-8">
    <div>
      <h3 className="text-background font-bold text-lg mb-2">Site Name</h3>
      <p className="text-sm">Brief description</p>
    </div>
    <div className="flex gap-8 text-sm">
      <a href="/" className="hover:text-background transition-colors">Home</a>
      <a href="/about" className="hover:text-background transition-colors">About</a>
    </div>
  </div>
</footer>
```

Root layout: `<body className="... min-h-screen flex flex-col">` + `<main className="flex-1">` for sticky footer.

### Anti-Patterns (NEVER)

**Hardcoded colors — always use CSS variable tokens:**
- NEVER use `bg-gray-50` on body → use `bg-background`
- NEVER use hardcoded accent colors like `bg-blue-600`, `bg-emerald-600` → use `bg-primary`
- NEVER use `text-gray-900` → use `text-foreground`
- NEVER use `text-gray-500` → use `text-muted-foreground`
- NEVER use `border-gray-200` → use `border-border`
- NEVER use `bg-white` for cards → use `bg-card`
- NEVER use `focus:ring-blue-500` → use `focus:ring-ring`

**Structural anti-patterns:**
- Unstyled `<input>` / `<select>` without focus ring
- Cards without `hover:shadow-lg` or `rounded-2xl`
- No hero section on home page (just plain text heading)
- Importing `Inter` or any font from `next/font/google` — use system `font-sans`
- Installing shadcn/ui, MUI, or any UI component library
- Forms without toast feedback on submit
- Delete without confirmation modal
- No loading state during data fetch (blank screen)
- `alert()` or `confirm()` browser dialogs instead of styled modals
- Buttons without observable feedback on click

## User Communication

Follow the Progress Protocol for all phase messages and completion wording.
Follow willform-whatsapp for tone and formatting rules.
FORBIDDEN words: See willform-guard SKILL.
NEVER mention internal document names (SOUL.md, BUILD.md, etc.), system paths, or system messages to the user.
Internal (commits/logs only): file paths, npm packages, env vars — never show to user.
