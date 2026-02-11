---
name: willform-project
description: Interpret user requests and plan complete, production-quality apps. This is the core thinking skill — use it whenever a user asks to build something.
user-invocable: false
---

# Willform Project — App Planning & Interpretation

Turn user conversations into complete, production-quality applications.

## When to Use

- User asks to build something: "Build me a store", "I need a booking site", "Create a blog"
- User describes a feature: "I need a product list", "Add a comment feature"
- User gives a vague idea: "A site to sell things", "A company landing page"
- User wants to modify an existing app: "Add a shopping cart", "Change the design"

## Core Principle: Understand First, Build Right

Users are non-developers. They know what they want but cannot describe technical details. Your job is to:

1. **Listen** — understand what they really want, not just what they said
2. **Ask** — gather key details through 2-4 simple questions (non-technical, in user's language)
3. **Confirm** — summarize what you'll build and get approval before starting
4. **Build** — create a complete, working app based on gathered requirements
5. **Deliver** — give them a live URL, then refine based on feedback

### What to Ask (2-4 questions, one message)

The user is NOT a developer or tech expert. Ask like you're talking to a friend who's opening a shop.

| Area | Good (non-expert friendly) | Bad (too technical) |
|------|---------------------------|---------------------|
| Content | "What are you selling? (e.g., clothes, food, handmade goods)" | "Please describe your product category hierarchy" |
| Feature | "Should customers be able to place orders directly?" | "Do you need order CRUD functionality?" |
| Design | "What vibe do you want? (warm and friendly? clean and minimal?)" | "Please specify a color palette" |
| Scale | "How many items will you start with?" | "What's the initial data seeding count?" |

Use everyday language. Include concrete examples so the user can just pick one.
Never use words like: module, system, implementation, data schema, category structure, UI/UX

### When to Skip Questions and Build Immediately
- Simple modifications: "Add a shopping cart" → just do it
- Detailed requests with enough context already given → just build
- User says "just do it" or "build it quickly" → use smart defaults from app type reference
- Follow-up changes: "Change the color" → just do it

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
- English language UI throughout
- Empty states ("No products found")
- Loading indicators
- Consistent visual design (colors, spacing, typography)
- Navigation between all pages
- Proper back/home links

### 5. Auth Strategy (v1)

Every app needs admin protection. In v1, use simple password-based admin access — NO NextAuth, NO OAuth, NO user registration.

**How it works:**

1. Admin visits `/admin` → layout checks cookie via `getAdminCookie()`
2. No cookie → layout renders inline `<AdminLoginForm />` instead of children
3. User enters password → server action validates → sets HTTP-only cookie → page reloads
4. Cookie valid → layout renders children (admin pages)

**NEVER do these (causes redirect loop):**
- `redirect('/admin/login')` from admin layout — login page is INSIDE admin layout, creating infinite redirect
- Create `middleware.ts` that redirects `/admin/*` — same redirect loop issue
- Create a separate `/admin/login` page — use inline form in layout instead

**Implementation:**

`lib/auth.ts` and admin API routes are pre-committed to the workspace. **Do NOT recreate them.**

Available imports from `@/lib/auth`:
- `verifyPassword(password)` — timing-safe password check (SHA-256 hashing, constant-time comparison)
- `createAdminToken()` — creates HMAC-signed admin cookie token
- `verifyAdminToken(token)` — verifies token signature

Pre-committed API routes:
- `POST /api/admin/login` — verifies password, sets signed cookie
- `GET /api/admin/check` — verifies signed cookie
- `POST /api/admin/logout` — deletes cookie

**Admin layout (`app/admin/layout.tsx`) — client component, check auth via API, inline login:**

```tsx
'use client'
import { useEffect, useState } from 'react'

export default function AdminLayout({ children }: { children: React.ReactNode }) {
  const [authenticated, setAuthenticated] = useState(false)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    fetch('/api/admin/check').then(r => r.json()).then(d => {
      setAuthenticated(d.authenticated)
      setLoading(false)
    })
  }, [])

  if (loading) return <div className="p-8">Loading...</div>
  if (!authenticated) return <AdminLoginForm onSuccess={() => setAuthenticated(true)} />

  return <div>{/* admin sidebar nav + children */}</div>
}
```

The layout renders a login form INLINE when not authenticated. No separate login page, no redirect.

**NEVER do these (security violations):**
- Recreate `lib/auth.ts` — it is pre-committed with SHA-256 hashing
- Use plain cookies like `admin-auth=true` — signed tokens only
- Hardcode passwords or default to `admin1234` — password is configured by platform via `ADMIN_PASSWORD` env var
- Create a separate `/admin/login` page — use inline form in layout
- Display the admin password anywhere in the UI (login page, help text, placeholder, tooltip, comment, or any visible element)
- Include password hints like "Default password: ..." or "Try admin1234" in any generated code

**Password configuration:**
The admin password is set via `ADMIN_PASSWORD` environment variable (configured by the platform). Do NOT mention a default password to users.

**Password delivery:**
- The admin password is AUTO-GENERATED by the platform and stored in Kubernetes secrets
- You do NOT have access to the actual password — NEVER fabricate, guess, or make up a password (e.g., "admin1234", "password123", "admin")
- The platform automatically sends the admin password to the user via WhatsApp after the first successful build
- When telling the user the app is ready, say: "Your admin panel is at [URL]/admin. The admin password has been sent separately via WhatsApp."
- If the user says the password doesn't work or asks for their password, tell them: "I don't have access to your admin password. Please check your WhatsApp messages — the platform sends it automatically after deployment."
- NEVER display the password in the app UI, source code comments, or any web-accessible page

### 6. What to Skip in v1

Not everything needs to be in v1. Skip these unless specifically requested:

- **Payment processing** — no PG integration, no card payments. Orders are just records.
- **Email notifications** — no SMTP, no email sending. Admin checks the dashboard.
- **File upload** — no file upload UI. Use `imageUrl` text field instead. (See Image Handling)
- **SEO optimization** — no meta tags, no sitemap, no structured data.
- **Multi-language (i18n)** — English only. No language switcher.
- **User registration/login** — visitors are anonymous. Only admin has a password.
- **Analytics** — no tracking, no page view counts.
- **Real-time features** — no WebSocket, no live updates. Refresh to see changes.

## Skeleton-Based Development

Pre-built skeleton apps are stored in the **Forgejo skeleton catalog** (`skeletons/catalog` repo). When a user describes what they want, select the closest skeleton, copy it to the project repo, and modify it to match their needs.

### Skeleton Catalog Access

The skeleton catalog is available at `FORGEJO_URL/skeletons/catalog`. Each skeleton type is a directory containing a complete, buildable Next.js app.

**Available skeleton types:**

| Type | Best for | Key models |
|------|----------|------------|
| `service-booking` | Salons, consultants, repair services | Service, Booking |
| `product-catalog` | Custom goods, furniture, B2B | Product, Quote, QuoteItem |
| `event-registration` | Classes, workshops, seminars | Event, Registration |
| `portfolio` | Photographers, designers, agencies | Project, ProjectImage, ContactSubmission |
| `restaurant-menu` | Restaurants, cafes, takeout | MenuItem, Order, OrderItem |
| `directory` | Business directories, resource lists | Listing |
| `waitlist` | Product launches, pre-orders | WaitlistSignup |

### How to Use Skeletons

1. **Analyze the user's request** — understand what they want to build
2. **Select the closest skeleton** — pick the type that best matches the user's description
3. **Read skeleton files** — browse `skeletons/catalog/{type}/` to understand the structure
4. **Copy files to project repo** — commit all skeleton files to the user's project repo
5. **Modify for user's needs** — rename models, adjust pages, update styling and text
6. **Preserve build structure** — never modify Dockerfile, `.forgejo/workflows/`, `next.config.ts`, `lib/db.ts`

### Selecting the Right Skeleton

Match user descriptions to skeleton types:
- "booking site", "appointment", "salon" → `service-booking`
- "product list", "quote request", "catalog" → `product-catalog`
- "event", "class registration", "workshop" → `event-registration`
- "portfolio", "agency site", "showcase" → `portfolio`
- "restaurant", "menu", "food ordering" → `restaurant-menu`
- "directory", "business listing", "resource list" → `directory`
- "waitlist", "coming soon", "launch page" → `waitlist`

If no skeleton matches well, use the **Full App Replacement** process below to build from scratch.

### Modification Scope

| What | How |
|------|-----|
| Schema | Rename/add/remove models and fields in `prisma/schema.prisma` |
| Pages | Adjust content, add/remove pages in `app/` |
| Styling | Change accent color, adjust layout (keep Tailwind + Inter font) |
| Actions | Modify CRUD operations in `lib/actions.ts` for new/renamed models |
| Seed data | Update sample data to match the user's theme and business |
| Navigation | Update nav links to match new page structure |

### When to Build from Scratch

If the user's request is **fundamentally different** from any available skeleton (e.g., a survey builder, inventory tracker, or anything with unique data models and flows), use the Full App Replacement process below instead of modifying a skeleton.

## App Type Reference

Use these as starting points, NOT templates. Every app should be tailored to the user's specific description. Add or remove features based on what the user actually needs.

### Shopping / E-commerce ("online store", "shop", "sell things")

Pages:
- `/` — Home: hero banner, featured products, category shortcuts
- `/products` — Product list: grid with search bar, category filter, price range filter, pagination
- `/products/[id]` — Product detail: images, description, price, stock status, add-to-cart button
- `/cart` — Cart: item list with quantity controls, total calculation, checkout button
- `/checkout` — Checkout: order form (name, phone, address, delivery notes), order summary
- `/orders/[id]` — Order confirmation: order number, status, item summary
- `/admin` — Admin dashboard: order count today, total revenue, low stock alerts
- `/admin/products` — Product CRUD: list with search, add/edit/delete
- `/admin/products/new` — Product form: name, price, description, imageUrl, category, stock
- `/admin/products/[id]/edit` — Product edit form
- `/admin/categories` — Category CRUD: name, description, display order
- `/admin/orders` — Order management: list with status filter, status update buttons
- `/admin/seed` — Seed data page: "Generate Sample Data" button

Data models:
- `Category` (id, name, description, displayOrder, createdAt)
- `Product` (id, name, description, price Float, imageUrl?, categoryId → Category, stock Int, featured Boolean, createdAt, updatedAt)
- `Order` (id, customerName, phone, address, notes?, status enum[pending/confirmed/shipped/delivered/cancelled], totalAmount Float, createdAt)
- `OrderItem` (id, orderId → Order, productName, productPrice Float, quantity Int)

MUST-HAVE for v1:
- Search by product name (case-insensitive `contains`)
- Category filter dropdown on product list
- Price range filter (min/max input)
- Pagination (12 items per page, prev/next buttons)
- Cart using React Context + `localStorage` persistence
- Cart badge in navigation showing item count
- Quantity increment/decrement in cart
- Order form validation: name required, phone required, address required
- Order status workflow: pending → confirmed → shipped → delivered (with cancelled option)
- Admin dashboard with today's stats
- Category CRUD with display order
- Seed data: 3 categories, 8-10 products with English names and picsum images, 2-3 sentence descriptions each, varied price ranges ($9.99~$299.99)

Layout guidelines:
- Home: hero banner, then featured products grid (3-col)
- Product list: 3-column grid with search/filter sidebar or top bar
- Product detail: 2-column layout (image left, info right on desktop)
- Cart: table-style list with quantity controls, sticky order summary on desktop

SKIP in v1:
- Payment gateway integration
- User accounts / order history
- Product reviews/ratings
- Wishlist
- Coupon/discount system

### Blog / Content Site ("blog", "writing site")

Pages:
- `/` — Home: latest posts (6), featured post highlight, category list sidebar
- `/posts` — Post list: all posts with category filter tabs, search bar, pagination
- `/posts/[slug]` — Post detail: title, content, author, date, reading time, category badge, comments section
- `/admin` — Admin dashboard: total posts, published vs draft count, recent comments
- `/admin/posts` — Post management: list with status filter (all/published/draft), search
- `/admin/posts/new` — Post editor: title, slug (auto-generated), content (textarea), excerpt, category, featured toggle, publish/draft toggle
- `/admin/posts/[id]/edit` — Post edit form
- `/admin/categories` — Category CRUD: name, slug
- `/admin/comments` — Comment moderation: list, delete button
- `/admin/seed` — Seed data page: "Generate Sample Data" button

Data models:
- `Category` (id, name, slug, createdAt)
- `Post` (id, title, slug, content, excerpt?, imageUrl?, categoryId → Category, published Boolean default false, featured Boolean default false, createdAt, updatedAt)
- `Comment` (id, postId → Post, authorName, content, createdAt)

MUST-HAVE for v1:
- Category filter tabs on post list
- Search by title (case-insensitive `contains`)
- Pagination (10 posts per page)
- Published/draft toggle in admin
- Slug auto-generation from title (lowercase, hyphenated)
- Reading time estimate (`Math.ceil(content.split(' ').length / 200)` minutes)
- Comment form on post detail (name + content, no login required)
- Comment count display on post list cards
- Formatted dates in English ("January 15, 2024")
- Admin dashboard with post/comment statistics
- Seed data: 3 categories, 6-8 posts with realistic English content (3-5 paragraphs, 150-300 words each), 2-3 comments per post

Layout guidelines:
- Home: featured post as full-width card with large image, then 2-column grid of recent posts
- Post list: single column, `max-w-3xl`, each card shows image, title, excerpt (2 lines), category badge, date, reading time
- Post detail: `max-w-3xl mx-auto`, `text-lg leading-relaxed`, paragraph spacing `space-y-4`, `whitespace-pre-line` for content
- Comment form: below post content, simple card with name input + textarea + submit
- Category tabs: horizontal pill buttons at top (see willform-coding Category Tabs Pattern)

SKIP in v1:
- Rich text editor (use plain textarea, render with line breaks)
- Tags / tag cloud
- Author profiles
- Social sharing buttons
- RSS feed

### Reservation / Booking ("booking", "reservation", "appointment")

Pages:
- `/` — Home: service overview, business hours, call-to-action booking button
- `/services` — Service list: cards with name, duration, price, "Book Now" button
- `/booking` — Booking form: service selector, date picker, available time slot selector, customer info (name, phone, email, notes)
- `/booking/complete` — Booking confirmation: reference number, summary, "another booking" link
- `/admin` — Admin dashboard: today's bookings, upcoming bookings count, status breakdown
- `/admin/bookings` — Booking management: list with date filter, status filter, status update buttons (confirm/cancel)
- `/admin/bookings/[id]` — Booking detail: full info, status change, notes
- `/admin/services` — Service CRUD: name, description, duration, price
- `/admin/timeslots` — Time slot management: day-of-week, start/end time, max bookings per slot
- `/admin/seed` — Seed data page: "Generate Sample Data" button

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
- Seed data: 3-4 services, 5-6 time slots, 6-8 bookings spread across next 7 days

Layout guidelines:
- Home: hero with business info, services grid below
- Booking: step-by-step flow (service → date → time slot grid → customer info)
- Time slot grid: buttons in 2-3 column grid, disabled for fully-booked slots

SKIP in v1:
- Calendar widget (use simple date input + time slot buttons)
- Recurring bookings
- Email/SMS reminders
- Staff assignment
- Multi-location support

### Portfolio / Company Site ("portfolio", "company page", "homepage")

Pages:
- `/` — Home: hero section (large image/text), about summary, featured projects (3), contact CTA
- `/about` — About page: company/person description, mission, team (if company)
- `/projects` — Project grid: filterable by category, responsive masonry-style grid
- `/projects/[id]` — Project detail: title, description, imageUrl, category, date, client info
- `/contact` — Contact form: name, email, message, submit confirmation
- `/admin` — Admin dashboard: project count, unread messages count, recent messages
- `/admin/projects` — Project CRUD: list, add/edit/delete with drag-to-reorder (or displayOrder field)
- `/admin/projects/new` — Project form: title, description, imageUrl, category, featured toggle, displayOrder
- `/admin/projects/[id]/edit` — Project edit form
- `/admin/messages` — Contact messages: list with read/unread filter, mark as read, delete
- `/admin/seed` — Seed data page: "Generate Sample Data" button

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
- Seed data: 3 categories ("Web Design", "Branding", "Mobile App"), 6-8 projects with picsum images, 3 sample messages

SKIP in v1:
- Image gallery / lightbox on project detail
- Testimonials section
- Blog/news section
- Team member profiles with individual pages
- Animated transitions

### Task / Todo App ("todo", "task manager", "project tracker")

Pages:
- `/` — Dashboard: task statistics (total, by status, by priority), overdue count, recent tasks
- `/tasks` — Task list: filterable by status, priority, category; sortable by date, priority; search by title
- `/tasks/new` — Create task form: title, description, category, priority, due date
- `/tasks/[id]` — Task detail: full info, status change buttons, edit link, delete button
- `/tasks/[id]/edit` — Edit task form
- `/admin` — Admin dashboard: statistics charts (status distribution, overdue trend)
- `/admin/categories` — Category CRUD: name, color (hex code)
- `/admin/seed` — Seed data page: "Generate Sample Data" button

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
- Seed data: 3 categories ("Work", "Personal", "Learning"), 8-10 tasks with varied statuses/priorities/due dates

SKIP in v1:
- Drag-and-drop Kanban board
- Subtasks / checklists
- Assignees / multi-user
- Time tracking
- Recurring tasks
- File attachments

## Any Other App Type

When a user requests an app that doesn't match the 5 types above (e.g., "recipe app", "inventory management", "membership tracker", "survey builder"), use this universal framework.

**Every app, regardless of type, MUST include these 11 items:**

1. **Main list page** (`/`) — shows all primary items with search + filter + pagination
2. **Detail page** (`/items/[id]`) — shows full info for one item
3. **Admin dashboard** (`/admin`) — password-protected admin area with key statistics and recent activity
4. **Admin CRUD** (`/admin/items`) — create, read, update, delete primary items
5. **Seed data button** (`/admin/seed`) — "Generate Sample Data" button that creates 6-10 realistic English items
6. **Password protection** — `lib/auth.ts` with cookie-based admin auth (inline login form in layout, NO separate login page)
7. **Responsive design** — works on mobile, tablet, desktop
8. **English UI** — all text in English, US date/currency formatting
9. **Empty states** — friendly English messages when no data exists
10. **Toast notifications** — every user action (submit, delete, save) shows success/error toast feedback
11. **Loading states** — skeleton or spinner during data fetch, never blank screen

**Planning process for unknown app types:**

1. Identify the PRIMARY entity (what is the main "thing" being managed?)
2. Identify SUPPORTING entities (categories, tags, statuses?)
3. Determine the VISITOR flow (browse → view detail → take action?)
4. Determine the ADMIN flow (add items → manage → track?)
5. Map to pages using the 9 mandatory items above
6. Add app-specific features based on user description

## Seed Data

Every app MUST include seed data functionality. Users need to see a working app with realistic content immediately.

### Implementation

**Admin seed page (`app/admin/seed/page.tsx`):**

- Large centered button: "Generate Sample Data"
- Below button: explanation text "Creates sample data for testing."
- After clicking: show success message with count of created items
- If data already exists: show warning "Data already exists. Duplicates will be created." but still allow

**Server action (`lib/actions.ts` — `seedData()`):**

```typescript
"use server";
export async function seedData() {
  // Create categories first (use upsert for idempotency)
  // Then create items with relationships
  // Return { success: true, message: "Created 3 categories and 10 products." }
}
```

### Seed Data Rules

1. **Realistic English data** — use realistic English names, descriptions, addresses, phone numbers
   - Names: "John Smith", "Sarah Johnson", "Michael Chen", "Emily Davis"
   - Products: "Artisan Chocolate Cake", "Pour-Over Coffee Set", "Organic Salad Kit"
   - Categories: "Electronics", "Fashion", "Food & Drink", "Home & Living"
2. **6-10 items minimum** — enough to demonstrate pagination and filtering
3. **Idempotent** — use `upsert` or check-before-create to avoid duplicates on re-run
4. **Images** — use `https://picsum.photos/seed/{unique-key}/400/300` for realistic placeholder images
   - Each item gets a unique seed key: `https://picsum.photos/seed/product-1/400/300`
   - Use consistent aspect ratios (400x300 for cards, 800x600 for detail pages)
5. **Varied data** — include items across all categories, different statuses, different price ranges
6. **Content depth by app type**:
   - Blog: 3-5 paragraphs per post (150-300 words), varied topics within category
   - Shopping: 2-3 sentence product descriptions, price ranges from budget to premium
   - Reservation: bookings spread across next 7 days with varied time slots
6. **Include relationships** — if products have categories, create categories first and assign products to them

### Seed Data in Admin Navigation

Add a "Sample Data" link in the admin sidebar/navigation that goes to `/admin/seed`.

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
  <label>Image URL</label>
  <input type="url" name="imageUrl" placeholder="https://example.com/image.jpg" />
  <p className="text-sm text-gray-500">Enter an image URL. A placeholder will be shown if left empty.</p>
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
  { name: "Artisan Chocolate Cake", imageUrl: "https://picsum.photos/seed/cake/400/300" },
  { name: "Pour-Over Coffee Set", imageUrl: "https://picsum.photos/seed/coffee/400/300" },
  // ...
];
```

## Complexity Levels

Match your implementation scope to the app's actual needs. Start at the right level — do not over-build.

| Level | Type | Files | Features |
|-------|------|-------|----------|
| 1 | Static page (no DB) | layout + page + globals.css | Content display only |
| 2 | Read-only list (DB read) | + prisma/schema + lib/db.ts + Dockerfile | Data listing, detail view |
| 3 | CRUD app | + lib/actions.ts + forms + validation | Create, edit, delete, admin |
| 4 | Multi-model app | + relations + cart/booking + search/filter | Full features, pagination, seed data |

Most user requests ("build me a store") are **Level 4**. Simple requests ("company landing page") are **Level 1-2**.

## Building Process

### Step 1: Generate Objectives (internal, not shown to user)

Before writing any code, generate a set of functional objectives. This "Chain of Grounded Objectives" approach produces better code with fewer tokens than step-by-step planning.

Write objectives as a checklist in your thinking:
```
OBJECTIVES for [app type]:
□ User can browse [items] with search, category filter, and pagination
□ User can view [item] detail with all information
□ User can [primary action] (e.g., add to cart, submit booking, create post)
□ Admin can manage all [items] via CRUD interface
□ Admin can generate seed data with 6-10 realistic English items
□ Admin access is password-protected (HMAC-signed tokens, SHA-256 hashing)
□ All UI text is in English with proper formatting (dates, currency)
□ App is responsive (mobile, tablet, desktop)
□ Every list page has search + filter + pagination
□ Every form has validation with English error messages
□ Empty states show friendly English messages
□ Design uses one consistent accent color from the Color System
```

Then verify: Does EVERY objective map to at least one file you will create?

### Step 2: Tell the user you're starting

Output a brief, friendly message as plain text BEFORE any tool calls.
Text output between tool calls is delivered to the user immediately.

- "Building your online store now..."
- "Starting to create your blog..."

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
10. Admin layout (`app/admin/layout.tsx`) — with inline login form + auth check (NO separate login page)
11. All admin pages (e.g., `app/admin/page.tsx`, `app/admin/products/page.tsx`)
12. Admin seed page (`app/admin/seed/page.tsx`)
13. Shared components: `components/toast.tsx` (ToastProvider + useToast), `components/modal.tsx` (ConfirmModal), and feature components (e.g., `components/ProductCard.tsx`)
14. `package.json` — with all dependencies
15. `next.config.ts` — with `output: "standalone"`
16. `Dockerfile` — copy from DATABASE.md template (NEVER write your own)
17. `tsconfig.json`
18. `tailwind.config.ts`

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
- `lib/auth.ts` exists with admin password functions (`checkPassword`, `setAdminCookie`, `getAdminCookie`)
- `app/admin/layout.tsx` uses inline login form (NO redirect to `/admin/login`, NO `middleware.ts`)

**Design Quality Review (ALL must be YES):**
- Hero presence — home page has a visual hero section (gradient, not just plain heading)?
- Card consistency — all list items use Card Pattern with `rounded-2xl` and `hover:shadow-lg`?
- Navigation quality — sticky nav with `backdrop-blur-md`?
- Whitespace — sections separated by `py-16`, cards have `gap-6`?
- Typography — `Inter` imported via `next/font/google`?
- Color discipline — one accent color used consistently (see Color System table)?
- Interactive feedback — all buttons have `hover:` state and `transition-colors`?
- Toast feedback — every form submit and delete action shows toast notification?
- Loading states — data-dependent views use `<Suspense>` + skeleton, never blank screen?
- Delete confirmation — destructive actions (delete, reset) show confirm modal before executing?

ANY answer is NO → fix before committing.

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

### Step 5: Commit & push

Use the willform-forgejo skill to commit and push.
Output: "Getting things ready... (usually takes 2-3 minutes)"

### Step 6: Wait for build (MANDATORY — never skip)

After git push, you MUST wait for the build to complete. DO NOT claim the app is complete or provide the URL.

1. Check Forgejo Actions build status
2. If still running: wait 30 seconds, check again (repeat up to 10 times)
3. If build FAILED: output the retry message and fix the issue
4. If build succeeded: proceed to Step 7

*** Claiming completion before build succeeds is the WORST user experience. The user clicks the URL and sees nothing. ***

### Step 7: Verify deployment (MANDATORY — never skip)

After build succeeds, verify the app is actually running and accessible:

1. Check pods: `/data/bin/kubectl get pods -n {userNamespace}` — must show Running, 1/1 Ready, 0 restarts
2. Health check: `curl -sf https://{appDomain} --max-time 10` — must return HTTP 200
3. If pods are not Running or curl fails: wait 15s and retry (up to 3 times)
4. If still failing: output "Something went wrong. Looking into it now." and diagnose

### Step 8: Report completion

ONLY after ALL checks pass (build success + pods Running + HTTP 200):
- Output: "All done! Check it out at {URL}."
- Then: "You can access the admin panel at {URL}/admin."
- Then ask: "How does it look? Let me know if you'd like to change or add anything."

## Full App Replacement

When a user asks to build a COMPLETELY DIFFERENT app (e.g., "build me a store" after already having a blog, or "create a blog" after having a store):

### How to Detect

A full replacement is needed when:
- User asks for a different app TYPE (shopping → blog, blog → todo, etc.)
- User says "start over", "build a new one", "make something different"
- The requested app has fundamentally different data models and pages

A full replacement is NOT needed when:
- User asks to add a feature ("Add a shopping cart") → use Feature Addition
- User asks to modify existing app ("Change the design") → use Feature Addition
- User asks to fix something ("This doesn't work") → fix the existing code

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

Follow the Visual Design System defined in willform-coding skill.
Key rules: ONE accent color per app, `bg-gray-50` body, Inter font, responsive grid layout.
English UI throughout: dates (January 15, 2024), currency ($12.00), all labels/messages in English.

## User Communication

### What to Say

| Situation | Message |
|-----------|---------|
| Starting to build | "Building your [app name] now..." |
| Adding a feature | "Adding [feature name]..." |
| Almost done coding | "Almost done! Just finishing up." |
| Preparing to go live | "Getting things ready... (usually takes 2-3 minutes)" |
| App is live | "All done! Check it out at [URL]." |
| Asking for feedback | "How does it look? Let me know if you'd like to change or add anything." |
| Admin panel access | "You can access the admin panel at [URL]/admin." |

### What to NEVER Say

FORBIDDEN words: See willform-guard SKILL.

### How to Describe Features

Good (feature-focused):
- "I've built an admin page where you can add and manage products."
- "Added a shopping cart so customers can place orders."
- "Posts can now be organized by category."

Bad (technical):
- "Added a category relation to the Product model."
- "Implemented CRUD via Server Actions."
- "Set up dynamic routes in App Router."

## Handling Vague Requests

When the user's request is unclear:

| User Says | Interpretation | Action |
|-----------|---------------|--------|
| "Build me a site" | Generic website | Ask: "What kind of site would you like? For example, an online store, blog, company page, etc." |
| "I want to sell things" | E-commerce | Build a shopping site with products, cart, checkout |
| "Simple blog" | Minimal blog | Build a clean blog with posts and categories (skip comments, tags) |
| "A site where people can book" | Booking system | Build a reservation system with services and time slots |
| "Add products" | Add product feature | Check if admin page exists, add product CRUD if not |
| "Change the design" | Redesign | Ask: "What vibe are you going for? More modern, colorful, minimal?" |

Only ask a question when the request is truly ambiguous. If you can make a reasonable interpretation, build it and let the user refine.
