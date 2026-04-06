## REPOSITORY CONTENTS

A repository contains the latest cheatsheets, command etc related to Docker, JavaScript, React.js, and Next.js. to help finding commands, shortcodes, boilerplates etc during work.

##### UPDATED ON: APRIL 06, 2026
---
## TOPICS

| # | Topic | Description |
|---|-------|-------------|
| 1 | [JavaScript](./javascript-primer/README.md) | Modern JS fundamentals, ES2024+ features, async patterns |
| 2 | [React.js](./reactjs/README.md) | React 19, Server Components, hooks, patterns |
| 3 | [Next.js](./nextjs/README.md) | App Router, Server Actions, caching, deployment |
| 4 | [Docker](./docker/README.md) | Containerization, multi-stage builds, Compose V2 |
---
## HOW TO USE

Work through the topics in order — each builds on the previous. If you're already comfortable with a section, skip to the parts marked with advanced tags.

---

### PHASE 01: JAVASCRIPT

| # | Section | What to Do |
|---|---------|------------|
| 1 | ES2024+ Features | Write examples for `Array.groupBy`, `Promise.withResolvers`, `Object.groupBy`. Show before/after comparisons. |
| 2 | Modules: ESM vs CJS | Show `import`/`export` vs `require`/`module.exports`, explain `"type": "module"` in package.json, cover interop pitfalls. |
| 3 | Async Patterns | Write examples for `async/await`, `Promise.all/allSettled/race/any`, error handling with try/catch, `AbortController`. |
| 4 | Destructuring & Spread | Nested destructuring, default values, rest params, shallow copy gotchas. |
| 5 | Array & Object Methods | `map`, `filter`, `reduce`, `flatMap`, `Object.entries/fromEntries`, `structuredClone`. |
| 6 | Template Literals | Multi-line strings, tagged templates (mention styled-components/GraphQL usage). |
| 7 | Optional Chaining & Nullish Coalescing | `?.` and `??` with real-world examples, compare `??` vs `\|\|`. |
| 8 | Closures & Scope | Lexical scope, stale closures in React (sets up the React section). |
| 9 | Tooling: Node.js 22+ | Built-in test runner, `--watch`, native `.env`, permission model. |

---

### PHASE 02: REACT.JS

| # | Section | What to Do |
|---|---------|------------|
| 1 | React 19 — What's New | Document `useActionState`, `useOptimistic`, `use()` API, ref-as-prop, metadata support. Include migration notes. |
| 2 | Server vs Client Components | Explain `"use client"` / `"use server"` directives, serialization boundaries, decision tree for when to use which. |
| 3 | Hooks Deep Dive | Cover all major hooks with examples. Focus on `useTransition`, `useDeferredValue`. |
| 4 | Suspense & Error Boundaries | Data fetching with Suspense, nested boundaries, fallback strategies. |
| 5 | State Management | When context is enough vs when to reach for Zustand/Jotai. Show `useReducer` patterns. |
| 6 | Performance | React Compiler (Forget), when `memo` actually helps, `lazy()` + code splitting, profiling with DevTools. |
| 7 | Forms & Actions | `useActionState`, `useFormStatus`, progressive enhancement. |
| 8 | Testing | React Testing Library, `user-event`, mocking patterns. |

---

### PHASE 03: NEXT.JS

| # | Section | What to Do |
|---|---------|------------|
| 1 | App Router Architecture | Explain `app/` directory, `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, route groups `(group)`. |
| 2 | Routing & Layouts | Nested layouts, parallel routes `@slot`, intercepting routes `(.)`, dynamic `[slug]`, catch-all `[...slug]`. |
| 3 | Data Fetching & Caching | `fetch()` caching behavior, `revalidatePath`/`revalidateTag`, ISR, streaming with Suspense. |
| 4 | Server Actions | `"use server"` in forms, revalidation after mutation, error handling, security considerations. |
| 5 | Middleware | `matcher` config, redirects, rewrites, auth checks at the edge. |
| 6 | Rendering Strategies | SSR vs SSG vs ISR vs PPR (Partial Prerendering) — decision guide with trade-offs. |
| 7 | Styling | CSS Modules, Tailwind, CSS-in-JS limitations in RSC. |
| 8 | Authentication | Auth.js integration, middleware-based auth, session handling. |
| 9 | Deployment | Vercel, self-hosted Node.js, Docker deployment (links to Docker section), `output: "standalone"`. |

---

### PHASE 04: DOCKER

| # | Section | What to Do |
|---|---------|------------|
| 1 | Core Concepts | Images vs containers, layers, registries, tags, build context. |
| 2 | Dockerfile Best Practices | Layer ordering, `.dockerignore`, image size reduction, pinning versions. |
| 3 | Multi-Stage Builds | Write a real multi-stage Dockerfile for a Next.js app (ties Phase 3 together). |
| 4 | Docker Compose V2 | `compose.yaml` with services, `depends_on` + healthcheck, profiles, watch mode. |
| 5 | Networking | Bridge vs host, service discovery, port exposure. |
| 6 | Volumes & Persistence | Named volumes, bind mounts, dev vs prod patterns. |
| 7 | Health Checks | `HEALTHCHECK` instruction, compose integration. |
| 8 | Node.js in Docker | Base image choice (alpine vs slim), `npm ci`, non-root user, signal handling with tini. |
| 9 | Security | Non-root, read-only fs, secrets, image scanning (trivy/scout). |
| 10 | Production Patterns | Logging, graceful shutdown, resource limits, orchestration mention. |
