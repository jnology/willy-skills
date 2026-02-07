---
name: willform-project
description: Interpret user requests and plan complete, production-quality apps. This is the core thinking skill — use it whenever a user asks to build something.
user-invocable: false
---

# Willform Project — App Planning & Interpretation

Turn user conversations into complete, production-quality applications.

## When to Use

- User asks to build something: "쇼핑몰 만들어줘", "블로그 만들어줘", "예약 시스템 필요해"
- User describes a feature: "상품 목록이 필요해", "댓글 기능 추가해줘"
- User gives a vague idea: "뭔가 파는 사이트", "회사 소개 페이지"
- User wants to modify an existing app: "장바구니 추가해줘", "디자인 바꿔줘"

## Core Principle: Build First, Iterate Later

Users are non-developers. They cannot describe every feature they need. Your job is to:

1. **Interpret** — understand what they really want, not just what they said
2. **Expand** — think of everything a complete version needs
3. **Build** — create a working app with all essential features
4. **Deliver** — give them a live URL, then refine based on feedback

Do NOT ask many questions upfront. Build a solid first version, then iterate.

## Completeness Thinking Framework

When a user asks for an app, work through these five questions BEFORE writing any code:

### 1. What pages does this need?

Think about every screen a user would expect to see.

- Home/landing page
- Main content pages (list + detail)
- Create/edit forms
- Admin/management pages
- About, contact, or info pages
- Error and empty states

### 2. What data models are needed?

Think about what information the app stores and how items relate to each other.

- Primary entities (products, posts, reservations, etc.)
- Supporting entities (categories, tags, reviews, etc.)
- User-generated content (comments, ratings, orders, etc.)
- Relationships between entities (one-to-many, many-to-many)

### 3. What user flows exist?

Think about what actions visitors take from start to finish.

- Browse/search/filter content
- View details of a specific item
- Create or submit something (order, comment, booking)
- Manage or edit their own content
- Admin operations (add, edit, delete items)

### 4. What makes it feel complete?

Think about the small details that separate a prototype from a real app.

- Responsive layout (mobile, tablet, desktop)
- Korean language UI throughout
- Empty states ("아직 상품이 없습니다")
- Loading indicators
- Consistent visual design (colors, spacing, typography)
- Navigation between all pages
- Proper back/home links

### 5. What can I skip for now?

Not everything needs to be in v1. Skip these unless specifically requested:

- User authentication/login (unless core to the app)
- Payment processing
- Email notifications
- Search engine optimization
- Analytics
- Multi-language support

## App Type Reference

Use these as starting points, NOT templates. Every app should be tailored to the user's specific description. Add or remove features based on what the user actually needs.

### Shopping / E-commerce ("쇼핑몰", "스토어", "파는 사이트")

Pages:
- Home — featured products, categories
- Product list — grid with filters (category, price)
- Product detail — images, description, price, add-to-cart
- Cart — items, quantities, total, checkout button
- Checkout — order form (name, phone, address, notes)
- Order complete — confirmation with order number
- Admin: product management — CRUD with image upload
- Admin: order management — list, status update

Data models:
- Category (name, description, order)
- Product (name, description, price, image, category, stock, featured)
- CartItem (tracked via client state or Context)
- Order (customerName, phone, address, status, total, items)
- OrderItem (product, quantity, price at time of order)

Key features:
- Product filtering by category and price range
- Cart with quantity controls (Context API for state)
- Order status tracking (pending, confirmed, shipped, delivered)
- Stock management in admin

### Blog / Content Site ("블로그", "글 쓰는 사이트")

Pages:
- Home — recent posts, categories sidebar
- Post list — by category or tag
- Post detail — full content, author, date, comments
- Write/edit post — rich text or markdown input
- Admin: post management — CRUD, publish/draft toggle
- Admin: category management

Data models:
- Category (name, slug)
- Post (title, slug, content, excerpt, category, published, featured, createdAt)
- Comment (post, author name, content, createdAt)

Key features:
- Published/draft status
- Category filtering
- Comment system
- Reading time estimate
- Formatted dates in Korean

### Reservation / Booking ("예약", "예약 시스템", "예약 사이트")

Pages:
- Home — service overview, CTA to book
- Service list — available services/times
- Booking form — date, time, service, customer info
- Booking confirmation — summary with reference number
- Admin: booking list — by date, status filter
- Admin: service management — CRUD

Data models:
- Service (name, description, duration, price)
- Booking (service, date, time, customerName, phone, email, status, notes)
- TimeSlot (service, dayOfWeek, startTime, endTime, maxBookings)

Key features:
- Calendar/date picker for booking
- Time slot availability checking
- Booking status (pending, confirmed, cancelled)
- Admin notification of new bookings

### Portfolio / Company Site ("포트폴리오", "회사 소개", "홈페이지")

Pages:
- Home — hero section, about summary, featured work
- About — company/person info
- Portfolio/work — project grid
- Project detail — images, description
- Contact — form with name, email, message
- Admin: content management

Data models:
- Project (title, description, images, category, featured, order)
- ContactMessage (name, email, message, read, createdAt)

Key features:
- Hero section with large visual impact
- Responsive image grid
- Contact form with storage (not email)
- Clean, professional design

### Task / Todo App ("할 일", "할일 관리", "프로젝트 관리")

Pages:
- Dashboard — task overview, stats
- Task list — filterable by status, priority, category
- Task detail — full info, status changes
- Create/edit task

Data models:
- Category (name, color)
- Task (title, description, category, priority, status, dueDate)

Key features:
- Status workflow (todo, in-progress, done)
- Priority levels (low, medium, high)
- Due date tracking
- Filter and sort options

## Building Process

### Step 1: Plan (internal, not shown to user)

Work through the Completeness Thinking Framework above. List:
- All pages with their purpose
- All data models with fields
- All user flows

### Step 2: Tell the user you're starting

Send a brief, friendly message:
- "요청하신 쇼핑몰을 만들고 있습니다..."
- "블로그를 만들기 시작했습니다..."

### Step 3: Build ALL files

Create every file the app needs. NEVER commit with missing files.

Build order:
1. `prisma/schema.prisma` — all data models
2. `lib/db.ts` — Prisma singleton
3. `lib/actions.ts` — all Server Actions
4. `lib/utils.ts` — helpers (formatters, etc.)
5. `app/layout.tsx` — root layout with navigation
6. `app/globals.css` — Tailwind imports
7. `app/page.tsx` — home page
8. All feature pages (e.g., `app/products/page.tsx`, `app/products/[id]/page.tsx`)
9. All admin pages (e.g., `app/admin/page.tsx`, `app/admin/products/page.tsx`)
10. Shared components (e.g., `components/ProductCard.tsx`)
11. `package.json` — with all dependencies
12. `next.config.ts` — with `output: "standalone"`
13. `Dockerfile` — copy from DATABASE.md template (NEVER write your own)
14. `tsconfig.json`
15. `tailwind.config.ts`

### Step 4: Verify completeness

Before committing, check:
- Every `import` statement references a file that exists
- Every component referenced in JSX has a corresponding file
- `prisma/schema.prisma` includes ALL models used in actions
- `package.json` includes ALL npm packages used in code
- `Dockerfile` exists and follows the DATABASE.md template exactly

### Step 5: Commit & deliver

Use the willform-forgejo skill to commit and push. Then follow the progress protocol from SOUL.md.

## Feature Addition

When a user asks to add a feature to an existing app:

1. **Read current code** — understand existing structure, models, patterns
2. **Plan the addition** — what new files, what modified files, what new models
3. **Update schema first** — if new models are needed, add them to prisma/schema.prisma BEFORE writing code. Use `npx prisma db push` to apply. Be careful: adding required fields to existing models with data can cause data loss. Use optional fields or defaults for existing tables.
4. **Build consistently** — match existing code style, naming, design patterns
5. **Update navigation** — add new pages to the navigation if needed
6. **Test integration** — ensure new feature works with existing features
7. **Commit everything together** — new AND modified files in one commit

## Design Principles

### Visual Consistency
- Use a consistent color palette (grays + one accent color)
- Rounded corners (`rounded-xl` for cards, `rounded-lg` for buttons)
- Consistent spacing (`p-4`, `p-6`, `gap-4`, `gap-6`)
- Clean typography (font-semibold for headings, text-gray-500 for secondary text)

### Mobile First
- Single column on mobile, multi-column on desktop
- Use Tailwind breakpoints: `sm:`, `md:`, `lg:`
- Touch-friendly button sizes (min `py-2 px-4`)
- Navigation that works on small screens (collapsible menu or simple stack)

### Korean UI
- All labels, buttons, and messages in Korean
- Date format: Korean style (2024년 1월 15일)
- Currency: Korean Won with comma separators (12,000원)
- Empty states in Korean ("아직 등록된 상품이 없습니다")
- Form validation messages in Korean ("이름을 입력해주세요")

## User Communication

### What to Say

| Situation | Message |
|-----------|---------|
| Starting to build | "요청하신 [앱이름]을 만들고 있습니다..." |
| Adding a feature | "[기능명]을 추가하고 있습니다..." |
| Almost done coding | "거의 다 됐습니다! 마무리 중이에요." |
| Preparing to go live | "준비 중입니다... (보통 2-3분 걸려요)" |
| App is live | "완료되었습니다! [URL]에서 확인하실 수 있습니다." |
| Asking for feedback | "어떠세요? 수정하거나 추가하고 싶은 부분이 있으면 말씀해주세요." |

### What to NEVER Say

FORBIDDEN words in messages to the user:
Docker, image, container, Harbor, pod, namespace, ArgoCD,
git, commit, push, build, deploy, API, endpoint, schema,
migration, database, server, node, Prisma, pipeline, workflow,
registry, tag, SHA, branch, merge, CI/CD, route, middleware,
Dockerfile, config, standalone, module, dependency, runtime,
Kubernetes, cluster, ingress, manifest, YAML, JSON, CLI,
frontend, backend, fullstack, ORM, query, SQL, seed,
DNS, SSL, certificate

### How to Describe Features

Good (feature-focused):
- "상품을 등록하고 관리할 수 있는 관리자 페이지를 만들었습니다."
- "장바구니에 담고 주문할 수 있는 기능을 추가했습니다."
- "카테고리별로 글을 분류할 수 있게 했습니다."

Bad (technical):
- "Product 모델에 category relation을 추가했습니다."
- "Server Action으로 CRUD를 구현했습니다."
- "App Router에 동적 라우트를 설정했습니다."

## Handling Vague Requests

When the user's request is unclear:

| User Says | Interpretation | Action |
|-----------|---------------|--------|
| "사이트 만들어줘" | Generic website | Ask: "어떤 종류의 사이트를 만들어드릴까요? 예를 들어 쇼핑몰, 블로그, 회사 소개 등이 가능합니다." |
| "뭔가 파는 사이트" | E-commerce | Build a shopping site with products, cart, checkout |
| "심플한 블로그" | Minimal blog | Build a clean blog with posts and categories (skip comments, tags) |
| "예약 받을 수 있는 사이트" | Booking system | Build a reservation system with services and time slots |
| "상품 추가해줘" | Add product feature | Check if admin page exists, add product CRUD if not |
| "디자인 바꿔줘" | Redesign | Ask: "어떤 느낌으로 바꿔드릴까요? 더 모던하게, 컬러풀하게, 심플하게?" |

Only ask a question when the request is truly ambiguous. If you can make a reasonable interpretation, build it and let the user refine.
