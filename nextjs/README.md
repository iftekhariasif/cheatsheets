# Next.js

Next.js 15 with App Router, Server Actions, and production deployment patterns.

## Table of Contents

- [App Router Architecture](#app-router-architecture)
- [Routing and Layouts](#routing-and-layouts)
- [Data Fetching and Caching](#data-fetching-and-caching)
- [Server Actions](#server-actions)
- [Middleware](#middleware)
- [Rendering Strategies](#rendering-strategies)
- [Styling](#styling)
- [Authentication Patterns](#authentication-patterns)
- [Deployment and Production](#deployment-and-production)

---

## App Router Architecture

### Directory structure

```
app/
├── layout.tsx          # Root layout (wraps entire app)
├── page.tsx            # Home page (/)
├── loading.tsx         # Loading UI for /
├── error.tsx           # Error UI for /
├── not-found.tsx       # 404 page
├── global-error.tsx    # Error boundary for root layout
├── template.tsx        # Re-mounts on navigation (unlike layout)
│
├── about/
│   └── page.tsx        # /about
│
├── blog/
│   ├── layout.tsx      # Nested layout for /blog/*
│   ├── page.tsx         # /blog
│   └── [slug]/
│       ├── page.tsx    # /blog/my-post
│       └── loading.tsx # Loading for individual post
│
├── (marketing)/        # Route group — no URL impact
│   ├── layout.tsx      # Shared layout for marketing pages
│   ├── pricing/
│   │   └── page.tsx    # /pricing
│   └── features/
│       └── page.tsx    # /features
│
└── api/
    └── webhook/
        └── route.ts    # API route: POST /api/webhook
```

### File conventions

| File | Purpose |
|---|---|
| `page.tsx` | UI for a route — makes the route publicly accessible |
| `layout.tsx` | Shared UI that wraps children — preserves state on navigation |
| `template.tsx` | Like layout but re-mounts on every navigation |
| `loading.tsx` | Loading UI — auto-wraps page in Suspense |
| `error.tsx` | Error UI — auto-wraps page in Error Boundary |
| `not-found.tsx` | 404 UI — shown when `notFound()` is called |
| `route.ts` | API endpoint (GET, POST, etc.) — cannot coexist with page.tsx |
| `default.tsx` | Fallback for parallel routes when no match |

### Root layout (required)

```tsx
// app/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "My App",
  description: "Built with Next.js",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

### Route groups `(folder)`

Group routes without affecting the URL. Useful for:
- Applying different layouts to different sections
- Organizing code

```
app/
├── (shop)/
│   ├── layout.tsx        # Shop layout with cart sidebar
│   ├── products/page.tsx # /products
│   └── cart/page.tsx     # /cart
│
├── (dashboard)/
│   ├── layout.tsx        # Dashboard layout with sidebar
│   ├── overview/page.tsx # /overview
│   └── settings/page.tsx # /settings
```

---

## Routing and Layouts

### Dynamic segments

```tsx
// app/blog/[slug]/page.tsx
export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.title}</article>;
}

// Generate static pages at build time
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}
```

### Catch-all segments

```tsx
// app/docs/[...slug]/page.tsx — matches /docs/a, /docs/a/b, /docs/a/b/c
export default async function Docs({ params }: { params: Promise<{ slug: string[] }> }) {
  const { slug } = await params;
  // slug = ["a", "b", "c"] for /docs/a/b/c
}

// app/docs/[[...slug]]/page.tsx — OPTIONAL catch-all
// Also matches /docs (slug = undefined)
```

### Nested layouts

Layouts persist across navigations — great for shared UI:

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex">
      <Sidebar />
      <main className="flex-1">{children}</main>
    </div>
  );
}

// app/dashboard/page.tsx — rendered inside DashboardLayout
// app/dashboard/settings/page.tsx — also rendered inside DashboardLayout
```

### Parallel routes

Render multiple pages in the same layout simultaneously:

```
app/
├── layout.tsx
├── page.tsx
├── @analytics/
│   ├── page.tsx
│   └── default.tsx
└── @team/
    ├── page.tsx
    └── default.tsx
```

```tsx
// app/layout.tsx
export default function Layout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  return (
    <div>
      {children}
      <div className="grid grid-cols-2">
        {analytics}
        {team}
      </div>
    </div>
  );
}
```

### Intercepting routes

Show a route in a modal while preserving context:

```
app/
├── feed/
│   ├── page.tsx              # /feed — the feed page
│   └── (..)photo/[id]/
│       └── page.tsx          # Intercepts /photo/[id] — shows as modal
└── photo/
    └── [id]/
        └── page.tsx          # /photo/[id] — full page (direct URL or refresh)
```

| Convention | Matches |
|---|---|
| `(.)` | Same level |
| `(..)` | One level up |
| `(..)(..)` | Two levels up |
| `(...)` | Root level |

### Navigation

```tsx
// Link component — prefetches by default
import Link from "next/link";

<Link href="/about">About</Link>
<Link href={`/blog/${post.slug}`}>Read More</Link>
<Link href="/dashboard" prefetch={false}>Dashboard</Link>

// Programmatic navigation
"use client";
import { useRouter } from "next/navigation";

function LogoutButton() {
  const router = useRouter();

  function handleLogout() {
    logout();
    router.push("/login");    // navigate
    router.replace("/login");  // navigate without history entry
    router.refresh();          // re-fetch server components
    router.back();             // go back
    router.forward();          // go forward
  }
}

// redirect() — in Server Components or Server Actions
import { redirect } from "next/navigation";

async function checkAuth() {
  const session = await getSession();
  if (!session) redirect("/login"); // throws, so no return needed
}
```

---

## Data Fetching and Caching

### Server Component data fetching

```tsx
// Just use async/await — no useEffect, no loading states
async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await fetch(`https://api.example.com/products/${id}`);
  const data = await product.json();

  return <ProductDetail product={data} />;
}
```

### Caching behavior (Next.js 15)

Next.js 15 changed defaults — `fetch` is **no longer cached by default**.

```tsx
// NOT cached (default in Next.js 15)
const data = await fetch("https://api.example.com/data");

// Cached — opt in explicitly
const data = await fetch("https://api.example.com/data", {
  cache: "force-cache",
});

// Revalidate every 60 seconds (ISR)
const data = await fetch("https://api.example.com/data", {
  next: { revalidate: 60 },
});

// Tag-based revalidation
const data = await fetch("https://api.example.com/data", {
  next: { tags: ["products"] },
});
```

### Revalidation

```tsx
// On-demand revalidation in Server Actions
"use server";

import { revalidatePath, revalidateTag } from "next/cache";

export async function createProduct(formData: FormData) {
  await db.products.create({ /* ... */ });

  // Revalidate specific path
  revalidatePath("/products");

  // Revalidate all fetches tagged with "products"
  revalidateTag("products");

  // Revalidate a specific layout
  revalidatePath("/products", "layout");
}
```

### `unstable_cache` for non-fetch data

Cache database queries, computations, etc.:

```tsx
import { unstable_cache } from "next/cache";

const getUser = unstable_cache(
  async (userId: string) => {
    return await db.users.findUnique({ where: { id: userId } });
  },
  ["user"],            // cache key parts
  {
    revalidate: 3600,   // revalidate every hour
    tags: ["users"],    // tag for on-demand revalidation
  }
);

// Usage in a Server Component
const user = await getUser("123");
```

### Streaming with Suspense

```tsx
import { Suspense } from "react";

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>

      {/* These stream in independently */}
      <Suspense fallback={<RevenueChartSkeleton />}>
        <RevenueChart /> {/* Slow query — streams in when ready */}
      </Suspense>

      <Suspense fallback={<RecentOrdersSkeleton />}>
        <RecentOrders />
      </Suspense>
    </div>
  );
}

async function RevenueChart() {
  const data = await getRevenueData(); // 3s query
  return <Chart data={data} />;
}
```

---

## Server Actions

### Defining Server Actions

```tsx
// Option 1: In a separate file
// app/actions.ts
"use server";

export async function createPost(formData: FormData) {
  const title = formData.get("title") as string;
  const content = formData.get("content") as string;

  await db.posts.create({ data: { title, content } });
  revalidatePath("/posts");
}

// Option 2: Inline in a Server Component
export default function Page() {
  async function handleSubmit(formData: FormData) {
    "use server";
    await db.posts.create({
      data: { title: formData.get("title") as string },
    });
    revalidatePath("/posts");
  }

  return (
    <form action={handleSubmit}>
      <input name="title" />
      <button type="submit">Create</button>
    </form>
  );
}
```

### With validation and error handling

```tsx
// actions.ts
"use server";

import { z } from "zod";
import { revalidatePath } from "next/cache";

const PostSchema = z.object({
  title: z.string().min(1, "Title is required").max(100),
  content: z.string().min(10, "Content must be at least 10 characters"),
});

type State = {
  errors?: { title?: string[]; content?: string[] };
  message?: string;
};

export async function createPost(prevState: State, formData: FormData): Promise<State> {
  const validated = PostSchema.safeParse({
    title: formData.get("title"),
    content: formData.get("content"),
  });

  if (!validated.success) {
    return { errors: validated.error.flatten().fieldErrors };
  }

  try {
    await db.posts.create({ data: validated.data });
    revalidatePath("/posts");
    return { message: "Post created!" };
  } catch (e) {
    return { message: "Database error. Please try again." };
  }
}
```

### Non-form usage

```tsx
"use client";

import { deletePost } from "./actions";

function DeleteButton({ postId }: { postId: string }) {
  async function handleDelete() {
    const confirmed = window.confirm("Delete this post?");
    if (confirmed) {
      await deletePost(postId);
    }
  }

  return <button onClick={handleDelete}>Delete</button>;
}
```

### Security considerations

```tsx
"use server";

import { auth } from "@/lib/auth";

export async function deletePost(postId: string) {
  // ALWAYS verify auth — Server Actions are public HTTP endpoints
  const session = await auth();
  if (!session) throw new Error("Unauthorized");

  // ALWAYS verify ownership
  const post = await db.posts.findUnique({ where: { id: postId } });
  if (post.authorId !== session.user.id) throw new Error("Forbidden");

  // ALWAYS validate input
  if (typeof postId !== "string") throw new Error("Invalid input");

  await db.posts.delete({ where: { id: postId } });
  revalidatePath("/posts");
}
```

---

## Middleware

Runs **before** every request. Executes on the Edge Runtime.

### Basic middleware

```tsx
// middleware.ts (in project root, NOT inside app/)
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  // Read
  const url = request.nextUrl;
  const pathname = url.pathname;
  const searchParams = url.searchParams;
  const cookie = request.cookies.get("session");
  const header = request.headers.get("authorization");

  // Redirect
  if (pathname === "/old-page") {
    return NextResponse.redirect(new URL("/new-page", request.url));
  }

  // Rewrite (URL stays the same, content changes)
  if (pathname.startsWith("/api/v1")) {
    return NextResponse.rewrite(new URL(pathname.replace("/v1", "/v2"), request.url));
  }

  // Set headers
  const response = NextResponse.next();
  response.headers.set("x-request-id", crypto.randomUUID());
  return response;
}

// Only run on specific paths
export const config = {
  matcher: [
    // Match all paths except static files and _next
    "/((?!_next/static|_next/image|favicon.ico).*)",
  ],
};
```

### Auth middleware

```tsx
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

const protectedRoutes = ["/dashboard", "/settings", "/profile"];
const authRoutes = ["/login", "/signup"];

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const sessionToken = request.cookies.get("session-token")?.value;

  // Redirect unauthenticated users to login
  const isProtected = protectedRoutes.some((route) => pathname.startsWith(route));
  if (isProtected && !sessionToken) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("callbackUrl", pathname);
    return NextResponse.redirect(loginUrl);
  }

  // Redirect authenticated users away from auth pages
  const isAuthRoute = authRoutes.some((route) => pathname.startsWith(route));
  if (isAuthRoute && sessionToken) {
    return NextResponse.redirect(new URL("/dashboard", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*", "/settings/:path*", "/profile/:path*", "/login", "/signup"],
};
```

### Matcher patterns

```tsx
export const config = {
  matcher: [
    "/about/:path*",                    // /about and all sub-paths
    "/dashboard/:path*",
    "/((?!api|_next/static|favicon.ico).*)",  // All except API and static
  ],
};

// Conditional matching in the function itself
export function middleware(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith("/admin")) {
    // admin-specific logic
  }
}
```

---

## Rendering Strategies

### Comparison table

| Strategy | When HTML is Generated | Data Freshness | Use Case |
|---|---|---|---|
| **SSG** (Static) | Build time | Stale until rebuild | Blog posts, docs, marketing |
| **ISR** (Incremental Static) | Build + revalidation | Configurable staleness | Product pages, listings |
| **SSR** (Dynamic) | Every request | Always fresh | User dashboards, search results |
| **PPR** (Partial Prerendering) | Shell at build, dynamic at request | Mixed | Best of both worlds |

### Static Site Generation (SSG)

Default for pages with no dynamic data:

```tsx
// This page is automatically static
export default function About() {
  return <h1>About Us</h1>;
}

// Static with dynamic params
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.content}</article>;
}
```

### Incremental Static Regeneration (ISR)

```tsx
// Revalidate every 60 seconds
export const revalidate = 60;

export default async function Products() {
  const products = await fetch("https://api.example.com/products", {
    next: { revalidate: 60 },
  });
  return <ProductList products={await products.json()} />;
}
```

### Dynamic rendering (SSR)

Opt into dynamic rendering:

```tsx
// Force dynamic rendering
export const dynamic = "force-dynamic";

// Or use dynamic functions — these automatically make the page dynamic
import { cookies, headers } from "next/headers";

export default async function Dashboard() {
  const cookieStore = await cookies();
  const token = cookieStore.get("session");

  const headersList = await headers();
  const userAgent = headersList.get("user-agent");

  const data = await fetchDashboard(token);
  return <DashboardUI data={data} />;
}
```

### Route segment config

```tsx
// Force static or dynamic per page/layout
export const dynamic = "auto" | "force-dynamic" | "error" | "force-static";
export const revalidate = false | 0 | number; // false = Infinity (static)
export const fetchCache = "auto" | "default-cache" | "force-cache" | "only-cache" | "default-no-store" | "force-no-store" | "only-no-store";
export const runtime = "nodejs" | "edge";
export const maxDuration = 30; // seconds (for serverless)
```

### Partial Prerendering (PPR)

Static shell served instantly, dynamic parts stream in:

```tsx
// next.config.js
module.exports = {
  experimental: {
    ppr: true,
  },
};

// The static parts render instantly, Suspense boundaries stream in
export default function Page() {
  return (
    <div>
      <StaticHeader />         {/* Instant — pre-rendered */}
      <StaticHero />           {/* Instant — pre-rendered */}

      <Suspense fallback={<CartSkeleton />}>
        <ShoppingCart />       {/* Streams in — dynamic */}
      </Suspense>

      <StaticFooter />         {/* Instant — pre-rendered */}
    </div>
  );
}
```

---

## Styling

### CSS Modules

Scoped CSS — no class name collisions:

```css
/* app/components/Button.module.css */
.button {
  padding: 8px 16px;
  border-radius: 4px;
  font-weight: 600;
}

.primary {
  background: #0070f3;
  color: white;
}

.secondary {
  background: #eaeaea;
  color: #333;
}
```

```tsx
import styles from "./Button.module.css";

export function Button({ variant = "primary", children }) {
  return (
    <button className={`${styles.button} ${styles[variant]}`}>
      {children}
    </button>
  );
}
```

### Tailwind CSS

```bash
npx create-next-app@latest --tailwind
# or add to existing project:
npm install tailwindcss @tailwindcss/postcss postcss
```

```css
/* app/globals.css */
@import "tailwindcss";
```

```tsx
export function Card({ title, children }) {
  return (
    <div className="rounded-lg border bg-white p-6 shadow-sm hover:shadow-md transition-shadow">
      <h3 className="text-lg font-semibold text-gray-900">{title}</h3>
      <div className="mt-2 text-gray-600">{children}</div>
    </div>
  );
}
```

### Global styles

```tsx
// app/layout.tsx
import "./globals.css"; // imported once in root layout

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

### CSS-in-JS limitations

CSS-in-JS libraries that need runtime JS (styled-components, Emotion) **don't work in Server Components**. Options:

| Library | Server Components? | Notes |
|---|---|---|
| CSS Modules | Yes | Recommended |
| Tailwind CSS | Yes | Recommended |
| Sass/SCSS | Yes | `.module.scss` for scoping |
| styled-components | Client only | Need `"use client"` + config |
| Emotion | Client only | Need `"use client"` + config |
| Panda CSS | Yes | Zero-runtime alternative |
| Vanilla Extract | Yes | Zero-runtime, type-safe |

---

## Authentication Patterns

### Auth.js (NextAuth.js v5)

```bash
npm install next-auth@beta
```

```tsx
// auth.ts
import NextAuth from "next-auth";
import GitHub from "next-auth/providers/github";
import Credentials from "next-auth/providers/credentials";

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    GitHub({
      clientId: process.env.GITHUB_ID,
      clientSecret: process.env.GITHUB_SECRET,
    }),
    Credentials({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      authorize: async (credentials) => {
        const user = await getUser(credentials.email);
        if (user && await verify(credentials.password, user.passwordHash)) {
          return user;
        }
        return null;
      },
    }),
  ],
  callbacks: {
    authorized: async ({ auth }) => {
      return !!auth; // return true if authenticated
    },
  },
});
```

```tsx
// app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/auth";
export const { GET, POST } = handlers;
```

### Using auth in Server Components

```tsx
import { auth } from "@/auth";

export default async function Dashboard() {
  const session = await auth();

  if (!session) {
    redirect("/login");
  }

  return (
    <div>
      <h1>Welcome, {session.user.name}</h1>
      <img src={session.user.image} alt="avatar" />
    </div>
  );
}
```

### Sign in / Sign out

```tsx
// Server Action approach
import { signIn, signOut } from "@/auth";

export function SignIn() {
  return (
    <form
      action={async () => {
        "use server";
        await signIn("github");
      }}
    >
      <button type="submit">Sign in with GitHub</button>
    </form>
  );
}

export function SignOut() {
  return (
    <form
      action={async () => {
        "use server";
        await signOut();
      }}
    >
      <button type="submit">Sign Out</button>
    </form>
  );
}
```

### Middleware-based route protection

```tsx
// middleware.ts
export { auth as middleware } from "@/auth";

export const config = {
  matcher: ["/dashboard/:path*", "/settings/:path*"],
};
```

### Protected API routes

```tsx
// app/api/admin/route.ts
import { auth } from "@/auth";
import { NextResponse } from "next/server";

export async function GET() {
  const session = await auth();

  if (!session) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  if (session.user.role !== "admin") {
    return NextResponse.json({ error: "Forbidden" }, { status: 403 });
  }

  const data = await getAdminData();
  return NextResponse.json(data);
}
```

---

## Deployment and Production

### Vercel (zero-config)

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel

# Deploy to production
vercel --prod
```

Environment variables: Set in Vercel Dashboard > Settings > Environment Variables.

### Self-hosting with Node.js

```bash
# Build
next build

# Start production server
next start -p 3000

# Or with standalone output (smaller, self-contained)
# next.config.js
module.exports = {
  output: "standalone",
};
```

```bash
# After build, run the standalone server
node .next/standalone/server.js
```

### Docker deployment

```dockerfile
# Dockerfile
FROM node:22-alpine AS base

# Install dependencies
FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Build
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Production
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

```yaml
# compose.yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### Environment variables

```bash
# .env.local — loaded in all environments (git-ignored)
DATABASE_URL=postgresql://localhost:5432/mydb
API_SECRET=super-secret

# .env.development — only in dev
NEXT_PUBLIC_API_URL=http://localhost:3000/api

# .env.production — only in production
NEXT_PUBLIC_API_URL=https://api.example.com
```

| Prefix | Available in | Exposed to browser? |
|---|---|---|
| `NEXT_PUBLIC_` | Server + Client | Yes |
| No prefix | Server only | No |

```tsx
// Server Component — access any env var
const secret = process.env.API_SECRET;

// Client Component — only NEXT_PUBLIC_ vars
const apiUrl = process.env.NEXT_PUBLIC_API_URL;
```

### Production checklist

```bash
# Analyze bundle size
npm install @next/bundle-analyzer
ANALYZE=true npm run build

# Check for issues
next lint

# Test production build locally
npm run build && npm run start
```

| Setting | Config |
|---|---|
| Image optimization | `next.config.js` → `images.remotePatterns` |
| Security headers | `next.config.js` → `headers()` |
| Redirects | `next.config.js` → `redirects()` |
| Compression | Enabled by default (gzip) |
| Static export | `output: "export"` for pure static sites |

```js
// next.config.js — common production settings
module.exports = {
  output: "standalone",
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "cdn.example.com" },
    ],
  },
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          { key: "X-Frame-Options", value: "DENY" },
          { key: "X-Content-Type-Options", value: "nosniff" },
          { key: "Referrer-Policy", value: "origin-when-cross-origin" },
        ],
      },
    ];
  },
};
```

---

## References

- [Next.js Docs](https://nextjs.org/docs)
- [Next.js GitHub](https://github.com/vercel/next.js)
- [Vercel Blog](https://vercel.com/blog)
- [Auth.js Docs](https://authjs.dev)
- [Next.js Deployment Docs](https://nextjs.org/docs/app/building-your-application/deploying)
