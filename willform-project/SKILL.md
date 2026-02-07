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

When a user asks for an app, work through these six questions BEFORE writing any code:

### 1. What pages does this need?

Think about every screen a user would expect to see.

- Home/landing page
- Main content pages (list + detail)
- Create/edit forms
- Admin/management pages (password-protected)
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

### 5. Auth Strategy (v1)

Every app needs admin protection. In v1, use simple password-based admin access — NO NextAuth, NO OAuth, NO user registration.

**How it works:**

1. Admin visits `/admin` → sees password input form
2. Enters password → server action validates → sets HTTP-only cookie
3. All `/admin/*` pages check cookie via `checkAdminAuth()`
4. Wrong password or no cookie → redirect to `/admin/login`

**Implementation:**

```typescript
// lib/auth.ts
import { cookies } from "next/headers";

const ADMIN_PASSWORD = process.env.ADMIN_PASSWORD || "admin1234";
const COOKIE_NAME = "admin_session";
const COOKIE_VALUE = "authenticated"; // simple static token for v1

export async function checkAdminPassword(password: string): Promise<boolean> {
  return password === ADMIN_PASSWORD;
}

export async function setAdminCookie() {
  const cookieStore = await cookies();
  cookieStore.set(COOKIE_NAME, COOKIE_VALUE, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    maxAge: 60 * 60 * 24, // 24 hours
    path: "/",
  });
}

export async function checkAdminAuth(): Promise<boolean> {
  const cookieStore = await cookies();
  return cookieStore.get(COOKIE_NAME)?.value === COOKIE_VALUE;
}

export async function clearAdminCookie() {
  const cookieStore = await cookies();
  cookieStore.delete(COOKIE_NAME);
}
```

**Admin login page (`app/admin/login/page.tsx`):**
- Simple centered form: password input + submit button
- Server action calls `checkAdminPassword()` → `setAdminCookie()` → redirect to `/admin`
- Show error message on wrong password

**Admin layout (`app/admin/layout.tsx`):**
- Check `checkAdminAuth()` at the top
- If not authenticated and path is not `/admin/login`, redirect to `/admin/login`
- Include admin navigation and logout button

**Default password:** `admin1234` (set via `ADMIN_PASSWORD` env var)
Include this in seed data instructions to the user: "관리자 비밀번호는 admin1234 입니다."

### 6. What to Skip in v1

Not everything needs to be in v1. Skip these unless specifically requested:

- **Payment processing** — no PG integration, no card payments. Orders are just records.
- **Email notifications** — no SMTP, no email sending. Admin checks the dashboard.
- **File upload** — no file upload UI. Use `imageUrl` text field instead. (See Image Handling)
- **SEO optimization** — no meta tags, no sitemap, no structured data.
- **Multi-language (i18n)** — Korean only. No language switcher.
- **User registration/login** — visitors are anonymous. Only admin has a password.
- **Analytics** — no tracking, no page view counts.
- **Real-time features** — no WebSocket, no live updates. Refresh to see changes.

## App Type Reference

Use these as starting points, NOT templates. Every app should be tailored to the user's specific description. Add or remove features based on what the user actually needs.

### Shopping / E-commerce ("쇼핑몰", "스토어", "파는 사이트")

Pages:
- `/` — Home: hero banner, featured products, category shortcuts
- `/products` — Product list: grid with search bar, category filter, price range filter, pagination
- `/products/[id]` — Product detail: images, description, price, stock status, add-to-cart button
- `/cart` — Cart: item list with quantity controls, total calculation, checkout button
- `/checkout` — Checkout: order form (name, phone, address, delivery notes), order summary
- `/orders/[id]` — Order confirmation: order number, status, item summary
- `/admin/login` — Admin login (password)
- `/admin` — Admin dashboard: order count today, total revenue, low stock alerts
- `/admin/products` — Product CRUD: list with search, add/edit/delete
- `/admin/products/new` — Product form: name, price, description, imageUrl, category, stock
- `/admin/products/[id]/edit` — Product edit form
- `/admin/categories` — Category CRUD: name, description, display order
- `/admin/orders` — Order management: list with status filter, status update buttons
- `/admin/seed` — Seed data page: "샘플 데이터 생성" button

Data models:
- `Category` (id, name, description, displayOrder, createdAt)
- `Product` (id, name, description, price Int, imageUrl?, categoryId → Category, stock Int, featured Boolean, createdAt, updatedAt)
- `Order` (id, customerName, phone, address, notes?, status enum[pending/confirmed/shipped/delivered/cancelled], totalAmount Int, createdAt)
- `OrderItem` (id, orderId → Order, productName, productPrice Int, quantity Int)

MUST-HAVE for v1:
- Search by product name (case-insensitive `contains`)
- Category filter dropdown on product list
- Price range filter (min/max input)
- Pagination (12 items per page, prev/next buttons)
- Cart using React Context + `localStorage` persistence
- Cart badge in navigation showing item count
- Quantity increment/decrement in cart
- Order form validation: name required, phone required (digits only), address required
- Order status workflow: pending → confirmed → shipped → delivered (with cancelled option)
- Admin dashboard with today's stats
- Category CRUD with display order
- Seed data: 3 categories, 8-10 products with Korean names and picsum images

SKIP in v1:
- Payment gateway integration
- User accounts / order history
- Product reviews/ratings
- Wishlist
- Coupon/discount system

### Blog / Content Site ("블로그", "글 쓰는 사이트")

Pages:
- `/` — Home: latest posts (6), featured post highlight, category list sidebar
- `/posts` — Post list: all posts with category filter tabs, search bar, pagination
- `/posts/[slug]` — Post detail: title, content, author, date, reading time, category badge, comments section
- `/admin/login` — Admin login (password)
- `/admin` — Admin dashboard: total posts, published vs draft count, recent comments
- `/admin/posts` — Post management: list with status filter (all/published/draft), search
- `/admin/posts/new` — Post editor: title, slug (auto-generated), content (textarea), excerpt, category, featured toggle, publish/draft toggle
- `/admin/posts/[id]/edit` — Post edit form
- `/admin/categories` — Category CRUD: name, slug
- `/admin/comments` — Comment moderation: list, delete button
- `/admin/seed` — Seed data page: "샘플 데이터 생성" button

Data models:
- `Category` (id, name, slug, createdAt)
- `Post` (id, title, slug, content, excerpt?, imageUrl?, categoryId → Category, published Boolean default false, featured Boolean default false, createdAt, updatedAt)
- `Comment` (id, postId → Post, authorName, content, createdAt)

MUST-HAVE for v1:
- Category filter tabs on post list
- Search by title (case-insensitive `contains`)
- Pagination (10 posts per page)
- Published/draft toggle in admin
- Slug auto-generation from title (Korean → transliterated or timestamp-based)
- Reading time estimate (`Math.ceil(content.length / 500)` minutes)
- Comment form on post detail (name + content, no login required)
- Comment count display on post list cards
- Formatted dates in Korean ("2024년 1월 15일")
- Admin dashboard with post/comment statistics
- Seed data: 3 categories, 6-8 posts with realistic Korean content, 2-3 comments per post

SKIP in v1:
- Rich text editor (use plain textarea, render with line breaks)
- Tags / tag cloud
- Author profiles
- Social sharing buttons
- RSS feed

### Reservation / Booking ("예약", "예약 시스템", "예약 사이트")

Pages:
- `/` — Home: service overview, business hours, call-to-action booking button
- `/services` — Service list: cards with name, duration, price, "예약하기" button
- `/booking` — Booking form: service selector, date picker, available time slot selector, customer info (name, phone, email, notes)
- `/booking/complete` — Booking confirmation: reference number, summary, "another booking" link
- `/admin/login` — Admin login (password)
- `/admin` — Admin dashboard: today's bookings, upcoming bookings count, status breakdown
- `/admin/bookings` — Booking management: list with date filter, status filter, status update buttons (confirm/cancel)
- `/admin/bookings/[id]` — Booking detail: full info, status change, notes
- `/admin/services` — Service CRUD: name, description, duration, price
- `/admin/timeslots` — Time slot management: day-of-week, start/end time, max bookings per slot
- `/admin/seed` — Seed data page: "샘플 데이터 생성" button

Data models:
- `Service` (id, name, description, duration Int minutes, price Int, imageUrl?, active Boolean default true, createdAt)
- `TimeSlot` (id, serviceId → Service, dayOfWeek Int 0-6, startTime String "09:00", endTime String "10:00", maxBookings Int default 1)
- `Booking` (id, serviceId → Service, date DateTime, timeSlot String "09:00", customerName, phone, email?, notes?, status enum[pending/confirmed/cancelled/completed], createdAt)

MUST-HAVE for v1:
- Date picker (HTML native `input[type="date"]`, min=today)
- Time slot availability check: count existing bookings for selected date+time vs maxBookings
- Disable fully-booked time slots in the UI
- Booking status workflow: pending → confirmed → completed (with cancelled option)
- Calendar-style view in admin (group bookings by date)
- Service active/inactive toggle
- Booking reference number (formatted: `BK-{YYMMDD}-{count}`)
- Admin dashboard with today's schedule
- Seed data: 3-4 services, 5-6 time slots, 6-8 bookings across different dates

SKIP in v1:
- Calendar widget (use simple date input + time slot buttons)
- Recurring bookings
- Email/SMS reminders
- Staff assignment
- Multi-location support

### Portfolio / Company Site ("포트폴리오", "회사 소개", "홈페이지")

Pages:
- `/` — Home: hero section (large image/text), about summary, featured projects (3), contact CTA
- `/about` — About page: company/person description, mission, team (if company)
- `/projects` — Project grid: filterable by category, responsive masonry-style grid
- `/projects/[id]` — Project detail: title, description, imageUrl, category, date, client info
- `/contact` — Contact form: name, email, message, submit confirmation
- `/admin/login` — Admin login (password)
- `/admin` — Admin dashboard: project count, unread messages count, recent messages
- `/admin/projects` — Project CRUD: list, add/edit/delete with drag-to-reorder (or displayOrder field)
- `/admin/projects/new` — Project form: title, description, imageUrl, category, featured toggle, displayOrder
- `/admin/projects/[id]/edit` — Project edit form
- `/admin/messages` — Contact messages: list with read/unread filter, mark as read, delete
- `/admin/seed` — Seed data page: "샘플 데이터 생성" button

Data models:
- `Project` (id, title, description, imageUrl?, category String, featured Boolean default false, displayOrder Int default 0, createdAt)
- `ContactMessage` (id, name, email, message, read Boolean default false, createdAt)

MUST-HAVE for v1:
- Hero section with large visual impact (full-width, gradient overlay on image)
- Project category filter (buttons/tabs, not dropdown)
- Responsive project grid (1 col mobile, 2 cols tablet, 3 cols desktop)
- Image fallback: first letter of title + colored background when no imageUrl
- Contact form with client-side validation (email format, required fields)
- Contact form stores to database (NOT email sending)
- Success message after form submission
- Unread message indicator in admin navigation
- Mark message as read functionality
- Seed data: 3 categories ("웹 디자인", "브랜딩", "모바일 앱"), 6-8 projects with picsum images, 3 sample messages

SKIP in v1:
- Image gallery / lightbox on project detail
- Testimonials section
- Blog/news section
- Team member profiles with individual pages
- Animated transitions

### Task / Todo App ("할 일", "할일 관리", "프로젝트 관리")

Pages:
- `/` — Dashboard: task statistics (total, by status, by priority), overdue count, recent tasks
- `/tasks` — Task list: filterable by status, priority, category; sortable by date, priority; search by title
- `/tasks/new` — Create task form: title, description, category, priority, due date
- `/tasks/[id]` — Task detail: full info, status change buttons, edit link, delete button
- `/tasks/[id]/edit` — Edit task form
- `/admin/login` — Admin login (password)
- `/admin` — Admin dashboard: statistics charts (status distribution, overdue trend)
- `/admin/categories` — Category CRUD: name, color (hex code)
- `/admin/seed` — Seed data page: "샘플 데이터 생성" button

Data models:
- `Category` (id, name, color String default "#6B7280", createdAt)
- `Task` (id, title, description?, categoryId → Category?, priority enum[low/medium/high], status enum[todo/in_progress/done], dueDate DateTime?, createdAt, updatedAt)

MUST-HAVE for v1:
- Status workflow: todo → in_progress → done (click to advance, or select)
- Priority levels with visual indicators: low (green), medium (yellow), high (red)
- Due date with overdue highlighting (red text/badge when past due)
- Filter by status (tabs: all/todo/in-progress/done)
- Filter by priority (dropdown or buttons)
- Filter by category (dropdown)
- Sort by: newest, oldest, priority (high first), due date (soonest first)
- Search by task title
- Statistics dashboard: total tasks, completion rate %, tasks by status (bar or pie), overdue count
- Category color displayed as dot/badge next to task
- Bulk status change (select multiple → change status)
- Seed data: 3 categories ("업무", "개인", "학습"), 8-10 tasks with varied statuses/priorities/due dates

SKIP in v1:
- Drag-and-drop Kanban board
- Subtasks / checklists
- Assignees / multi-user
- Time tracking
- Recurring tasks
- File attachments

## Any Other App Type

When a user requests an app that doesn't match the 5 types above (e.g., "레시피 앱", "재고 관리", "회원 관리", "설문조사"), use this universal framework.

**Every app, regardless of type, MUST include these 10 items:**

1. **Main list page** (`/`) — shows all primary items with search + filter + pagination
2. **Detail page** (`/items/[id]`) — shows full info for one item
3. **Admin login** (`/admin/login`) — password-protected admin area
4. **Admin dashboard** (`/admin`) — key statistics and recent activity
5. **Admin CRUD** (`/admin/items`) — create, read, update, delete primary items
6. **Seed data button** (`/admin/seed`) — "샘플 데이터 생성" button that creates 6-10 realistic Korean items
7. **Password protection** — `lib/auth.ts` with cookie-based admin auth
8. **Responsive design** — works on mobile, tablet, desktop
9. **Korean UI** — all text in Korean, Korean date/currency formatting
10. **Empty states** — friendly Korean messages when no data exists

**Planning process for unknown app types:**

1. Identify the PRIMARY entity (what is the main "thing" being managed?)
2. Identify SUPPORTING entities (categories, tags, statuses?)
3. Determine the VISITOR flow (browse → view detail → take action?)
4. Determine the ADMIN flow (add items → manage → track?)
5. Map to pages using the 10 mandatory items above
6. Add app-specific features based on user description

## Seed Data

Every app MUST include seed data functionality. Users need to see a working app with realistic content immediately.

### Implementation

**Admin seed page (`app/admin/seed/page.tsx`):**

- Large centered button: "샘플 데이터 생성"
- Below button: explanation text "테스트용 샘플 데이터를 생성합니다."
- After clicking: show success message with count of created items
- If data already exists: show warning "이미 데이터가 있습니다. 중복 생성됩니다." but still allow

**Server action (`lib/actions.ts` — `seedData()`):**

```typescript
"use server";
export async function seedData() {
  // Create categories first (use upsert for idempotency)
  // Then create items with relationships
  // Return { success: true, message: "카테고리 3개, 상품 10개가 생성되었습니다." }
}
```

### Seed Data Rules

1. **Korean realistic data** — use realistic Korean names, descriptions, addresses, phone numbers
   - Names: "김민수", "이서연", "박지훈", "최수아"
   - Products: "수제 초콜릿 케이크", "핸드드립 커피 세트", "유기농 샐러드 키트"
   - Categories: "전자기기", "패션", "식품", "생활용품"
2. **6-10 items minimum** — enough to demonstrate pagination and filtering
3. **Idempotent** — use `upsert` or check-before-create to avoid duplicates on re-run
4. **Images** — use `https://picsum.photos/seed/{unique-key}/400/300` for realistic placeholder images
   - Each item gets a unique seed key: `https://picsum.photos/seed/product-1/400/300`
   - Use consistent aspect ratios (400x300 for cards, 800x600 for detail pages)
5. **Varied data** — include items across all categories, different statuses, different price ranges
6. **Include relationships** — if products have categories, create categories first and assign products to them

### Seed Data in Admin Navigation

Add a "샘플 데이터" link in the admin sidebar/navigation that goes to `/admin/seed`.

## Image Handling

All apps handle images via URL string fields — NEVER implement file upload.

### Schema Pattern

```prisma
model Product {
  id       String  @id @default(cuid())
  name     String
  imageUrl String? // URL to image, nullable
  // ... other fields
}
```

### Admin Form

Image input in admin forms is a simple text input:

```tsx
<div>
  <label>이미지 URL</label>
  <input type="url" name="imageUrl" placeholder="https://example.com/image.jpg" />
  <p className="text-sm text-gray-500">이미지 URL을 입력하세요. 비워두면 기본 이미지가 표시됩니다.</p>
</div>
```

### Display with Fallback

When no imageUrl is provided, show a fallback with the first letter of the item name:

```tsx
function ItemImage({ name, imageUrl, className }: { name: string; imageUrl?: string | null; className?: string }) {
  if (imageUrl) {
    return <img src={imageUrl} alt={name} className={className} />;
  }
  return (
    <div className={`bg-gray-200 flex items-center justify-center ${className}`}>
      <span className="text-2xl font-bold text-gray-400">
        {name.charAt(0)}
      </span>
    </div>
  );
}
```

### Seed Data Images

Use picsum.photos with unique seed keys for consistent, reproducible images:

```typescript
const products = [
  { name: "수제 초콜릿 케이크", imageUrl: "https://picsum.photos/seed/cake/400/300" },
  { name: "핸드드립 커피 세트", imageUrl: "https://picsum.photos/seed/coffee/400/300" },
  // ...
];
```

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
3. `lib/auth.ts` — admin password auth (cookie-based)
4. `lib/actions.ts` — all Server Actions (including seedData)
5. `lib/utils.ts` — helpers (formatters, etc.)
6. `app/layout.tsx` — root layout with navigation
7. `app/globals.css` — Tailwind imports
8. `app/page.tsx` — home page
9. All feature pages (e.g., `app/products/page.tsx`, `app/products/[id]/page.tsx`)
10. Admin login page (`app/admin/login/page.tsx`)
11. Admin layout (`app/admin/layout.tsx`) — with auth check
12. All admin pages (e.g., `app/admin/page.tsx`, `app/admin/products/page.tsx`)
13. Admin seed page (`app/admin/seed/page.tsx`)
14. Shared components (e.g., `components/ProductCard.tsx`)
15. `package.json` — with all dependencies
16. `next.config.ts` — with `output: "standalone"`
17. `Dockerfile` — copy from DATABASE.md template (NEVER write your own)
18. `tsconfig.json`
19. `tailwind.config.ts`

### Step 4: Verify completeness

Before committing, check:
- Every `import` statement references a file that exists
- Every component referenced in JSX has a corresponding file
- `prisma/schema.prisma` includes ALL models used in actions
- `package.json` includes ALL npm packages used in code
- `Dockerfile` exists and follows the DATABASE.md template exactly
- No leftover imports from a previous app (if this is a replacement)
- `layout.tsx` navigation links match current pages only
- No orphaned component files from old app in `components/`
- `lib/auth.ts` exists with admin password functions

**package.json rules:**
- `"build"` script MUST be `"next build"` ONLY — NO prisma commands in build script
  - Correct: `"build": "next build"`
  - WRONG: `"build": "prisma generate && next build"`
  - WRONG: `"build": "npx prisma db push && next build"`
- Required dependencies (always include):
  - `next`, `react`, `react-dom`
  - `prisma` (devDependencies), `@prisma/client`
  - `tailwindcss`, `@tailwindcss/postcss`, `postcss`
- Do NOT include packages that aren't imported in any file

### Step 5: Commit & deliver

Use the willform-forgejo skill to commit and push. Then follow the progress protocol from SOUL.md.

## Full App Replacement

When a user asks to build a COMPLETELY DIFFERENT app (e.g., "쇼핑몰 만들어줘" after already having a blog, or "블로그 만들어줘" after having a shopping mall):

### How to Detect

A full replacement is needed when:
- User asks for a different app TYPE (shopping → blog, blog → todo, etc.)
- User says "다시 만들어줘", "새로 만들어줘", "다른 거 만들어줘"
- The requested app has fundamentally different data models and pages

A full replacement is NOT needed when:
- User asks to add a feature ("장바구니 추가해줘") → use Feature Addition
- User asks to modify existing app ("디자인 바꿔줘") → use Feature Addition
- User asks to fix something ("이거 안 돼") → fix the existing code

### Cleanup Process (CRITICAL)

Before writing any new code, you MUST delete ALL old app files. Leftover files from the old app will cause build failures (webpack module-not-found errors from stale imports).

**Step 1: Delete old app files**

Remove these directories and files completely:

```bash
# Delete old app code
rm -rf app/
rm -rf components/
rm -f lib/actions.ts
rm -f lib/utils.ts
rm -f lib/auth.ts

# Delete old schema (will be replaced)
rm -f prisma/schema.prisma
```

**Step 2: Preserve platform files (NEVER delete these)**

```
.forgejo/workflows/    # Build workflow — managed by platform
Dockerfile             # Will be rewritten, but don't delete before creating new one
package.json           # Will be rewritten with new deps
next.config.ts         # Usually stays the same
tsconfig.json          # Usually stays the same
tailwind.config.ts     # Usually stays the same
lib/db.ts              # Prisma singleton — usually stays the same
.gitignore
.dockerignore
```

**Step 3: Build the new app from scratch**

Follow the standard "Building Process" above, starting from Step 1 (Plan).

**Step 4: Verify no leftovers**

Before committing, verify:
- `grep -r "from.*@/" app/ components/ lib/` — every import resolves to an existing file
- No references to old models (e.g., Product in a blog app, Post in a shopping app)
- No references to old components (e.g., CartProvider in a blog app)
- `layout.tsx` navigation matches the NEW app's pages, not the old one's

### Common Leftover Issues

| Old App | New App | Typical Leftovers |
|---------|---------|-------------------|
| Shopping | Blog | CartProvider, AdminProductForm, product routes |
| Blog | Shopping | PostCard, CategorySidebar, comment actions |
| Todo | Any | TaskCard, priority/status enums |
| Any | Any | Old `layout.tsx` navigation links, old admin pages |

**The #1 rule: When replacing an app, start with a CLEAN workspace. Delete first, then build.**

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
| Admin password | "관리자 페이지는 [URL]/admin 에서 접속하실 수 있어요. 비밀번호는 admin1234 입니다." |

### What to NEVER Say

FORBIDDEN words: See willform-guard SKILL.

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
