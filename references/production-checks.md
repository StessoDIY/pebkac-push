# Production-Readiness Checks Reference

Run these checks during Phase 3 Step 7 of the PEBKAC Push workflow. These patterns catch code that works perfectly on a developer's machine but can cause real problems at scale — unnecessary API calls, memory leaks, bloated bundles, and other things that only surface when real users show up.

Only flag what's actually present in the changed files. If nothing is found, skip this step silently.

---

## Expensive Per-Request Operations

These patterns mean something expensive happens on every page load or every API call, when it probably shouldn't.

### API/fetch calls in render paths

```bash
# React: fetch/API calls that aren't in useEffect, loaders, or server-side functions
git diff --cached -U0 | grep -E '^\+.*(fetch\(|axios\.|\.get\(|\.post\()' | grep -v 'useEffect\|getServerSideProps\|getStaticProps\|loader\|action\|server'
```

**What to look for**: A `fetch()` call sitting directly in a React component body (not inside `useEffect`, a server component, or a data loader). This means it fires on every render — potentially dozens of times per page view.

**What to tell the user**: "This API call at line [N] runs every time the page renders. At 10K users/day, that's potentially 50K+ requests to this service. Should we move it to a loader or cache the result?"

### Database queries inside loops (N+1)

```bash
# SQL or ORM calls that appear inside loop constructs
git diff --cached -U3 | grep -B3 -E '(SELECT|INSERT|UPDATE|DELETE|findBy|findAll|findOne|query\(|\.find\(|\.where\()' | grep -E '(for\s*\(|\.forEach|\.map\(|while\s*\(|for\s+.*\s+in\s+|for\s+.*\s+of\s+)'
```

**What to look for**: A database query (SQL, ORM call, or DAC method) that runs inside a `for` loop, `.map()`, or `.forEach()`. This is an N+1 query — it fires once per item instead of fetching all items in one query.

**What to tell the user**: "This database call is inside a loop — so if there are 100 items, it makes 100 separate database queries. That's going to be slow. Want me to refactor it to fetch everything in one query?"

### Heavy computation without memoization

```bash
# Expensive operations in React render paths without useMemo/useCallback
git diff --cached -U5 | grep -E '^\+.*(\.sort\(|\.filter\(|\.reduce\(|JSON\.parse|JSON\.stringify)' | grep -v 'useMemo\|useCallback\|memo\('
```

**What to look for**: Array sorting, filtering, or complex object manipulation that runs on every render without `useMemo`. Fine for small arrays, but if the data could grow, it'll cause jank.

**What to tell the user**: "This sort/filter operation runs every time the component re-renders. If the list gets long, this could cause noticeable slowdown. Want me to wrap it in `useMemo`?"

### SSR/Hydration hazards

```bash
# Non-deterministic values in render paths that can break server/client matching
git diff --cached -U0 | grep -E '^\+.*(new Date\(\)|Date\.now\(\)|Math\.random\(\)|crypto\.randomUUID)' | grep -v 'useEffect\|useState\|server\|api\|handler'
```

**What to look for**: `new Date()`, `Math.random()`, or `crypto.randomUUID()` called during render. These produce different values on the server vs. client, causing hydration mismatches and React warnings.

**What to tell the user**: "This `new Date()` call runs during rendering, which can cause a mismatch between the server and browser. Want me to move it into a `useEffect` so it only runs in the browser?"

---

## Missing Caching

### Static data fetched on every request

```bash
# Fetch calls that don't include cache headers or revalidation
git diff --cached -U3 | grep -A3 'fetch(' | grep -v 'cache\|revalidate\|stale'
```

**What to look for**: API calls that fetch data which rarely changes (pricing tiers, feature flags, navigation menus) but don't specify any caching strategy. Every visitor pays the latency cost.

**What to tell the user**: "This data looks like it doesn't change often, but it's being fetched fresh on every request. Adding a cache header (or using `revalidate` in Next.js) would make the page load faster for everyone."

### Missing revalidation on Next.js pages

```bash
# Next.js pages without revalidation that fetch data
git diff --cached --name-only | grep -E 'page\.(tsx?|jsx?)$' | while read f; do
  grep -L 'revalidate\|dynamic\|generateStaticParams' "$f" 2>/dev/null && echo "$f: no revalidation strategy"
done
```

**What to tell the user**: "This page fetches data but doesn't set a revalidation interval. That means it either rebuilds on every request (slow) or never updates (stale). Want to add a `revalidate` value?"

---

## Expensive Build/Deploy Patterns

### Full library imports

```bash
# Importing entire libraries when tree-shaking may not help
git diff --cached -U0 | grep -E "^\+.*import .+ from '(lodash|moment|date-fns|rxjs|@fortawesome|icons)'"
git diff --cached -U0 | grep -E '^\+.*require\(.*(lodash|moment)\)'
```

**What to look for**: `import _ from 'lodash'` instead of `import debounce from 'lodash/debounce'`. The full import can add 70KB+ (lodash) or 300KB+ (moment.js) to the bundle.

**What to tell the user**: "You're importing all of [library] ([size]) but only using [function]. Want me to switch to a smaller import? It'll make the page load faster for your users."

**Common alternatives to suggest**:
- `moment` → `date-fns` or `dayjs` (2-7KB vs 300KB)
- `lodash` → specific imports (`lodash/debounce`) or native JS
- `@fortawesome` → specific icon imports

### Large static assets in the repo

```bash
# Images over 500KB
git diff --cached --name-only | while read f; do
  echo "$f" | grep -iE '\.(png|jpg|jpeg|gif|svg|webp|avif|bmp|ico)$' && \
  [ -f "$f" ] && size=$(stat -f%z "$f" 2>/dev/null || stat -c%s "$f" 2>/dev/null) && \
  [ "$size" -gt 524288 ] && echo "  ⚠️ $(($size/1024))KB — consider optimizing or using a CDN"
done
```

**What to tell the user**: "This image is [size]. For a web app, images should usually be under 200KB. Want me to compress it, convert to WebP, or should it live on a CDN instead?"

### Unoptimized images in web apps

```bash
# img tags without optimization in React/Next.js (should use next/image or srcset)
git diff --cached -U0 | grep -E '^\+.*<img ' | grep -v 'next/image\|Image \|srcset\|loading='
```

**What to tell the user**: "This `<img>` tag doesn't use lazy loading or responsive sizing. For a web app, using `next/image` (or adding `loading=\"lazy\"` and `srcset`) helps pages load faster on slow connections."

---

## Scaling Red Flags

### Timers without cleanup

```bash
# setTimeout/setInterval without corresponding cleanup
git diff --cached -U0 | grep -E '^\+.*(setTimeout|setInterval)\(' | grep -v 'clearTimeout\|clearInterval\|cleanup\|return'
```

**What to look for**: `setTimeout` or `setInterval` in a React component without a cleanup function in `useEffect`. These keep running after the component unmounts, causing memory leaks.

**What to tell the user**: "This timer will keep running even after the user navigates away from this page. Want me to add cleanup so it stops when it should?"

### Event listeners without removal

```bash
# addEventListener without corresponding removeEventListener
git diff --cached -U0 | grep -E '^\+.*addEventListener\(' | grep -v 'removeEventListener'
```

**What to tell the user**: "This event listener gets added but never removed. Over time, this can slow down the page. Want me to add cleanup?"

### Unbounded growth

```bash
# Arrays or Maps that grow without bounds (push without size checks)
git diff --cached -U3 | grep -E '^\+.*(\.push\(|\.set\(|\.add\()' | grep -v 'splice\|delete\|clear\|slice\|limit\|max'
```

**What to look for**: Arrays, Maps, or Sets that keep growing (`.push()`, `.set()`, `.add()`) without any size limit, eviction, or cleanup. Fine for small, bounded data — problematic for anything that grows with usage.

**What to tell the user**: "This array keeps growing over time but is never trimmed. If this runs for a while, it could use more and more memory. Should we add a size limit?"

### Synchronous file I/O in request handlers

```bash
# Sync file operations in server code
git diff --cached -U0 | grep -E '^\+.*(readFileSync|writeFileSync|existsSync|readdirSync)' | grep -v 'config\|setup\|init\|bootstrap'
```

**What to look for**: `readFileSync`, `writeFileSync`, etc. in code that runs per-request. Synchronous I/O blocks the entire Node.js event loop — fine during startup, catastrophic under load.

**What to tell the user**: "This reads a file synchronously, which blocks the server while it's reading. That's fine for startup code, but if this runs on every request, it'll slow down all other requests too. Want me to switch to the async version?"

---

## How to Present Findings

Collect all findings and present them as a batch, just like the safety checks. Group by severity:

> "I noticed a few things that work fine locally but could cause problems in production:"
>
> **Performance:**
> - `src/pricing.ts:42` — API call on every page load. At scale, this adds up fast.
> - `src/components/List.tsx:18` — Sorting a potentially large array on every render.
>
> **Bundle size:**
> - `src/utils/format.ts:1` — Importing all of moment.js (300KB). Only using `format()`.
>
> **Memory:**
> - `src/hooks/useTracking.ts:12` — Event listener added but never cleaned up.

For each item, offer to fix it. Most of these are quick changes. If the user declines, note it in the PR description under reviewer notes.

If nothing is found, don't mention this step at all. No news is good news.
