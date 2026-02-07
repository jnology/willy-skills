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

### Client Component

```tsx
'use client'

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)
  return (
    <button onClick={() => setCount(c => c + 1)} className="px-4 py-2 bg-gray-900 text-white rounded-lg">
      {count}
    </button>
  )
}
```

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

```typescript
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ?? new PrismaClient()

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

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

## Dockerfile

DO NOT write your own Dockerfile. Copy the template from DATABASE.md exactly.
It handles: isolated prisma stage, confbox conflict, standalone output, non-root user.

## Response to User

When code is complete:
1. Describe what was built in simple, friendly terms (like talking to a friend)
2. Commit and trigger build (or ask for approval based on level)
3. After deployment: provide the app URL
4. Never show code, file paths, or technical details to user

FORBIDDEN words in user messages:
Docker, image, container, Harbor, pod, namespace, ArgoCD,
git, commit, push, build, deploy, API, endpoint, schema,
migration, database, server, node, Prisma, pipeline, workflow,
registry, tag, SHA, branch, merge, CI/CD, route, middleware,
Dockerfile, config, standalone, module, dependency, runtime,
Kubernetes, cluster, ingress, manifest, YAML, JSON, CLI,
frontend, backend, fullstack, ORM, query, SQL, seed,
DNS, SSL, certificate

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

Report EACH phase separately. NEVER combine or skip phases.

| Phase | Message to User |
|-------|-----------------|
| Coding start | "요청하신 [앱이름]을 만들고 있습니다..." |
| Feature added | "[기능명]을 추가하고 있습니다..." |
| Code ready | "거의 다 됐습니다! 마무리 중이에요." |
| Building | "준비 중입니다... (보통 2-3분 걸려요)" |
| Almost done | "거의 완성됐습니다! 조금만 기다려주세요." |
| Live | "완료되었습니다! [URL]에서 확인하실 수 있습니다." |

IMPORTANT: "거의 다 됐습니다" is NOT "완료". Build and deploy must complete first.

## Completion Wording (STRICT)

| Stage | Allowed Message | FORBIDDEN |
|-------|----------------|-----------|
| Code committed | "거의 다 됐습니다! 마무리 중이에요." | "만들었습니다", "완료", URL 언급 |
| Build complete | "거의 완성됐습니다! 조금만 기다려주세요." | "완료되었습니다", URL 언급 |
| Deploy healthy + URL verified | "완료되었습니다! [URL]에서 확인하실 수 있습니다." | - |

NEVER say "배포가 완료되면 ~에서 확인하실 수 있습니다" (future tense with URL).
The URL MUST be given ONLY after confirming the app is live and accessible.
