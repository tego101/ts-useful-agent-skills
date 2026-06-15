# Next.js `use cache` Directive

## Overview

`'use cache'` is a compiler directive that opts a route, component, or function into caching. It replaced the old implicit fetch-level caching model. As of Next.js 16, **data fetching is dynamic by default** — you explicitly mark what gets cached.

It behaves like `'use client'` and `'use server'`: a string literal at the top of a file or function that the Next.js compiler treats as a boundary instruction.

---

## Enable It

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
}

export default nextConfig
```

> Previously gated behind `experimental.dynamicIO` or `experimental.useCache`. If migrating from those, use the Next.js 16 upgrade guide.

---

## Placement Levels

### File level — everything in the file is cached
```ts
'use cache'

export default async function Page() {
  const data = await db.query('SELECT * FROM posts')
  return <PostList posts={data} />
}
```

### Component level
```tsx
export async function ProductCard({ id }: { id: string }) {
  'use cache'
  const product = await fetchProduct(id)
  return <div>{product.name}</div>
}
```

### Function level
```ts
export async function getProducts() {
  'use cache'
  return db.query('SELECT * FROM products')
}
```

---

## Three Cache Directives

| Directive | Use Case | Accesses Request APIs? |
|---|---|---|
| `'use cache'` | Shared, static content. Cached at fetch/build time. | ❌ No |
| `'use cache: private'` | Per-user content. Can access `cookies()`, `headers()`, `searchParams`. | ✅ Yes |
| `'use cache: remote'` | Serverless environments. Shared cache across instances. | ❌ No |

### When to use `'use cache: private'`
```ts
import { cookies } from 'next/headers'

export async function getUserDashboard() {
  'use cache: private'
  const token = await cookies().get('session')
  return fetchDashboard(token?.value)
}
```

### When to use `'use cache: remote'`
- In-memory `'use cache'` doesn't persist across serverless function instances
- Use `'use cache: remote'` for cross-instance shared caching (e.g. Vercel KV)

---

## `cacheLife` — Control TTL

```ts
import { cacheLife } from 'next/cache'

export async function getProducts() {
  'use cache'
  cacheLife('hours')
  return db.query('SELECT * FROM products')
}
```

### Built-in profiles

| Profile | stale | revalidate | expire |
|---|---|---|---|
| `'seconds'` | ~0 | ~15s | ~1min |
| `'minutes'` | ~1min | ~1min | ~5min |
| `'hours'` | ~1hr | ~1hr | ~1day |
| `'days'` | ~1day | ~1day | ~1wk |
| `'weeks'` | ~1wk | ~1wk | ~1mo |
| `'max'` | Infinity | Infinity | Infinity |

> **Default when no `cacheLife` is set:** ~15 minute revalidation, near-infinite expiry.

### Custom profile
```ts
cacheLife({
  stale: 3600,      // 1hr — serve from browser cache
  revalidate: 7200, // 2hr — revalidate in background
  expire: 86400,    // 24hr — hard expiry
})
```

> Caches with `expire < 5 minutes` or `revalidate: 0` become **dynamic holes** — excluded from prerender, fetched at request time. This includes the `'seconds'` profile.

---

## `cacheTag` + Invalidation

Tag cached data so you can invalidate it on demand.

```ts
import { cacheTag } from 'next/cache'

export async function getPost(slug: string) {
  'use cache'
  cacheTag(`post-${slug}`, 'posts')
  return db.posts.findBySlug(slug)
}
```

Limits: up to **128 tags** per `cacheTag()` call, max **256 chars** per tag.

### `revalidateTag` — use in Route Handlers / Server Actions
```ts
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache'

export async function POST() {
  revalidateTag('posts', 'max') // marks stale; serves stale until next request refetches
  return new Response('ok')
}
```

> `revalidateTag` does not immediately drop the cache — it marks entries stale. The next incoming request pays the refetch cost. Until then, stale content is served.

### `updateTag` — use in Server Actions only
```ts
'use server'
import { updateTag } from 'next/cache'

export async function updatePost() {
  await db.posts.update(...)
  updateTag('posts') // immediately expires — next request WAITS for fresh data
}
```

| | `revalidateTag` | `updateTag` |
|---|---|---|
| Context | Route Handlers, Server Actions | Server Actions only |
| Behavior | Stale-while-revalidate | Hard expire, blocks next request |
| User sees stale? | Yes, until next request | No, next request waits |

---

## Working with Runtime Values (cookies, headers, params)

`'use cache'` and `'use cache: remote'` cannot read runtime request APIs directly. Extract them outside and pass as arguments:

```ts
// ✅ correct
import { cookies } from 'next/headers'

export default async function Page() {
  const token = (await cookies()).get('session')?.value
  const data = await getCachedData(token)
  return <Dashboard data={data} />
}

async function getCachedData(token: string) {
  'use cache'
  return fetchProtectedData(token)
}
```

```ts
// ❌ wrong — will throw
async function getCachedData() {
  'use cache'
  const token = (await cookies()).get('session')?.value // runtime API inside cache boundary
  return fetchProtectedData(token)
}
```

> Each unique argument value creates a **separate cache entry**. Be deliberate with what you pass — granular keys reduce hit rate.

---

## Prerendering a Full Route

Add `'use cache'` at the top of both `layout.tsx` and `page.tsx`. Each file is treated as an independent cache entry.

```ts
// app/blog/layout.tsx
'use cache'
export default function BlogLayout({ children }) {
  return <div>{children}</div>
}

// app/blog/[slug]/page.tsx
'use cache'
import { cacheTag } from 'next/cache'

export default async function PostPage({ params }) {
  cacheTag(`post-${params.slug}`)
  const post = await fetchPost(params.slug)
  return <article>{post.content}</article>
}
```

---

## Mixing Static + Dynamic in One Route

`'use cache'` prerenders a static shell. Short-lived caches (or uncached components) become **dynamic holes** that stream in after the shell is served.

```tsx
// Static shell — cached
export default async function Page() {
  return (
    <div>
      <CachedHeader />       {/* 'use cache' — prerendered */}
      <Suspense fallback={<Spinner />}>
        <LiveFeed />         {/* no cache — streams in dynamically */}
      </Suspense>
    </div>
  )
}
```

---

## CMS / Content Pattern

```ts
import { cacheLife, cacheTag } from 'next/cache'

async function fetchPost(slug: string) {
  'use cache'
  const post = await cms.getPost(slug)
  cacheTag(`post-${slug}`)

  if (!post) {
    cacheLife('minutes') // shorter TTL for 404s
    return null
  }

  // Use TTL from CMS response directly
  cacheLife({
    revalidate: post.revalidateSeconds ?? 3600,
  })

  return post.data
}
```

---

## Gotchas

- **Test with `next build && next start`** — `next dev` does not cache. Dev mode skips caching entirely.
- **Serverless memory is ephemeral** — `'use cache'` stores in-memory (LRU). In serverless, memory doesn't persist across invocations. Use `'use cache: remote'` for shared cross-instance cache.
- **Security** — `'use cache'` (shared) will serve the same cached response to all users. Never cache user-specific data under the shared directive. Use `'use cache: private'` for any personalized content.
- **Revalidation from Server Actions clears client cache entirely** — calling `revalidateTag`, `revalidatePath`, `updateTag`, or `refresh` from a Server Action bypasses `stale` and clears the full client cache immediately.
- **`connection()` is banned inside all cache directives** — it provides connection-specific info that can't be safely cached.
- **Short-lived caches become dynamic** — `expire < 5min` or `revalidate: 0` opts the entry out of prerender and makes it a dynamic hole at request time.

---

## Quick Reference

```ts
// Enable
// next.config.ts → cacheComponents: true

// Scope
'use cache'           // shared, no request APIs
'use cache: private'  // per-user, can use cookies/headers
'use cache: remote'   // shared, cross-instance (serverless)

// TTL
import { cacheLife } from 'next/cache'
cacheLife('seconds' | 'minutes' | 'hours' | 'days' | 'weeks' | 'max')
cacheLife({ stale, revalidate, expire }) // seconds

// Tags
import { cacheTag } from 'next/cache'
cacheTag('my-tag', 'another-tag') // inside 'use cache' scope

// Invalidation
import { revalidateTag } from 'next/cache' // route handlers or server actions
import { updateTag } from 'next/cache'     // server actions only

revalidateTag('my-tag', 'max')  // stale-while-revalidate
updateTag('my-tag')             // hard expire, next request blocks
```
