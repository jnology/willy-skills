---
name: willform-coding
description: Full-stack coding patterns for Willform apps. Use when writing any code, building features, or setting up project structure.
user-invocable: false
---

# Willform Coding Patterns

Build complete, production-quality apps on the Willform platform.

## When to Use

- Creating new project from scratch
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
8. All UI text is in Korean

## Default Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Next.js 15 (App Router, Server Components) |
| Language | TypeScript |
| Styling | Tailwind CSS |
| Database | PostgreSQL + Prisma |
| Config | `output: "standalone"` in next.config.ts |

IMPORTANT: Do NOT use Vite, Express, or React without Next.js. The platform build pipeline is designed for Next.js standalone output only.

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
        <div key={product.id} className="bg-white rounded-xl border p-4">
          <h3 className="font-semibold">{product.name}</h3>
          <p className="text-gray-500">{product.description}</p>
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
  if (!name) return { error: '이름은 필수입니다' }

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

Simple password-based admin access using cookies.

```typescript
// lib/auth.ts — checkPassword, setAdminCookie (httpOnly, 1 day), getAdminCookie
import { cookies } from 'next/headers'
const ADMIN_PASSWORD = process.env.ADMIN_PASSWORD || 'admin1234'
export function checkPassword(pw: string) { return pw === ADMIN_PASSWORD }
export async function setAdminCookie() {
  (await cookies()).set('admin-auth', 'true', { httpOnly: true, maxAge: 86400 })
}
export async function getAdminCookie() {
  return (await cookies()).get('admin-auth')?.value === 'true'
}
```

```tsx
// app/admin/layout.tsx — check cookie, show login or children
import { getAdminCookie } from '@/lib/auth'
export default async function AdminLayout({ children }: { children: React.ReactNode }) {
  if (!(await getAdminCookie())) return <AdminLoginForm />
  return <div>{children}</div>
}
```

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
  if (!formData.get('name')) errors.name = '상품명은 필수입니다'
  if (Number(formData.get('price')) <= 0) errors.price = '가격은 0보다 커야 합니다'
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
  data: { name: '전자제품', products: { create: [{ name: '노트북', price: 1500000 }] } },
})
```

### Seed Data Server Action

Idempotent: check count first, use `createMany`, `revalidatePath` after.

```typescript
export async function seedSampleData() {
  if ((await prisma.product.count()) > 0) return { error: '이미 데이터가 있습니다' }
  await prisma.product.createMany({ data: [
    { name: '샘플 1', price: 10000, imageUrl: 'https://picsum.photos/seed/p1/400/300' },
    { name: '샘플 2', price: 20000, imageUrl: 'https://picsum.photos/seed/p2/400/300' },
  ]})
  revalidatePath('/'); return { success: true }
}
```

Admin button: `<button onClick={() => seedSampleData()}>샘플 데이터 추가</button>`

### Admin Layout

`app/admin/layout.tsx` — sidebar nav with `Link` components, `bg-gray-50` background.

```tsx
const NAV = [{ href: '/admin', label: '대시보드' }, { href: '/admin/products', label: '상품 관리' }]
// Layout: flex container, left nav (w-56, bg-white, border-r), right main (flex-1, p-8)
// Map NAV → <Link> with hover:bg-gray-100 styling
```

### Image URL Field

Schema: `imageUrl String?`. Display: `<img>` if exists, else fallback `<div>` with first letter + `bg-gray-200`.
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
- [ ] All referenced components exist as files

### Required Quality
- [ ] TypeScript for all files (`.tsx`, `.ts`)
- [ ] Responsive layout (mobile-first, Tailwind breakpoints)
- [ ] Korean language UI (labels, messages, empty states)
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
| **Styling** | Tailwind utilities, rounded-xl for cards, consistent gray palette |

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
| No `tailwind-merge` in deps | Add if using `cn()` utility |
| `"build": "prisma db push && next build"` in package.json | DB not available during build. Use `"build": "next build"` only |

## Dockerfile

DO NOT write your own Dockerfile. Copy the template from DATABASE.md exactly.
It handles: isolated prisma stage, confbox conflict, standalone output, non-root user.

## Visual Design Quality

Avoid generic "AI-generated" look. Each app should feel intentionally designed.

### Typography
- Use `Pretendard` or `Noto Sans KR` via next/font/google
- Headings: font-bold, text-2xl to text-4xl
- Body: text-sm to text-base, text-gray-600
- Price/numbers: font-semibold, tabular-nums

### Color System (per app type)

| App Type | Accent | Tailwind Class | Use For |
|----------|--------|----------------|---------|
| Shopping | Warm Orange | `orange-500` | CTA buttons, price highlights, cart badge |
| Blog | Deep Blue | `blue-600` | Links, category tags, headers |
| Reservation | Teal | `teal-500` | Available slots, confirm buttons |
| Portfolio | Slate/Neutral | `slate-700` | Minimal, content-focused |
| Task/Todo | Indigo | `indigo-500` | Priority indicators, action buttons |

Apply ONE accent color consistently across the entire app. Never mix accent colors.

### Layout Quality
- Hero sections: full-width with gradient or image background
- Card grids: `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6`
- Admin: sidebar navigation with `w-64` fixed sidebar
- Empty states: centered icon + message + action button
- Loading: skeleton placeholders (not spinners)
- Hover effects: `hover:shadow-lg transition-shadow` on cards

### Anti-Patterns (Never Do)
- Rainbow gradients or neon colors
- Multiple conflicting accent colors
- Default browser form styling (always style inputs)
- Walls of unstyled text
- Generic stock photo placeholders

## Response to User

When code is complete:
1. Describe what was built in simple, friendly terms (like talking to a friend)
2. Commit and trigger build (or ask for approval based on level)
3. After deployment: provide the app URL
4. Never show code, file paths, or technical details to user

FORBIDDEN words: See willform-guard SKILL.

Good user messages:
- "요청하신 쇼핑몰을 만들고 있습니다..."
- "상품 관리 기능을 추가하고 있습니다..."
- "거의 다 됐습니다! 마무리 중이에요."
- "완료되었습니다! [URL]에서 확인하실 수 있습니다."
- "문제가 있었지만 해결했습니다."

Bad user messages:
- "Docker 이미지를 빌드하고 Harbor에 푸시 중입니다"
- "Prisma 스키마를 생성하고 마이그레이션을 실행합니다"
- "Git에 커밋하고 빌드 파이프라인을 트리거합니다"

Internal (for commits and logs only):
- File paths and changes
- npm packages installed
- Environment variables configured

## Message Formatting Rules

**List formatting (REQUIRED):**
- List starts immediately after section header (no blank line)
- One blank line between sections
- List items grouped together
- No emojis — use plain text headers

**Correct:**
```
만들어진 기능:
- 상품 목록과 검색
- 장바구니
- 주문하기
- 관리자 페이지
```

**Wrong (forbidden):**
```
📋 구현된 페이지:

• 메인 페이지 (GET /api/products)
```

## Progress Reporting

Each progress message MUST be output as plain text BEFORE the tool calls for that phase.
Text output between tool calls is delivered to the user immediately via block streaming.

| Phase | Output text first | Then do |
|-------|-------------------|---------|
| Coding start | "요청하신 [앱이름]을 만들고 있습니다..." | Write code files |
| Feature added | "[기능명]을 추가하고 있습니다..." | Write feature files |
| Code ready | "거의 다 됐습니다! 마무리 중이에요." | Git commit and push |
| Building | "준비 중입니다... (보통 2-3분 걸려요)" | Check build status |
| Almost done | "거의 완성됐습니다! 조금만 기다려주세요." | Check pod status |
| Live | "완료되었습니다! [URL]에서 확인하실 수 있습니다." | (final message) |

IMPORTANT: "거의 다 됐습니다" is NOT "완료". Build and deploy must complete first.

## Completion Wording (STRICT)

| Stage | Allowed Message | FORBIDDEN |
|-------|----------------|-----------|
| Code committed | "거의 다 됐습니다! 마무리 중이에요." | "만들었습니다", "완료", URL 언급 |
| Build complete | "거의 완성됐습니다! 조금만 기다려주세요." | "완료되었습니다", URL 언급 |
| Deploy healthy + URL verified | "완료되었습니다! [URL]에서 확인하실 수 있습니다." | - |

NEVER say "배포가 완료되면 ~에서 확인하실 수 있습니다" (future tense with URL).
The URL MUST be given ONLY after confirming the app is live and accessible.
