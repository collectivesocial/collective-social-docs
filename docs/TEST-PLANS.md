# Test Plans - Collective Social Bug Fixes

## Issue #27: Login Back Button Does Not Reset Form

### Automated Tests (src/test/LoginButton.test.tsx)
- [x] Form resets on `pageshow` with `persisted=true` (back button)
- [x] Form does NOT reset on `pageshow` with `persisted=false` (normal load)
- [x] Handle normalized to lowercase before submission
- [x] Loading state shows during submission

### Manual Validation
1. Navigate to login page
2. Type a handle (e.g., "test.bsky.social")
3. Click "Login with ATProto" (redirects to OAuth provider)
4. Press browser back button
5. **Expected**: Form is empty, button says "Login with ATProto" (not "Redirecting...")
6. Verify no console errors

### Edge Cases
- Double back button press
- Back button after OAuth error redirect
- Mobile Safari bfcache behavior (known to be aggressive)

---

## Issue #28: Case-Insensitive Form Matching

### Automated Tests
- [x] LoginButton lowercases handle before submission (LoginButton.test.tsx)
- [ ] API: Media search matches case-insensitively (integration test recommended)

### Manual Validation - Login
1. Enter handle as "MyHandle.BSky.Social"
2. Submit login
3. **Expected**: Resolves correctly (same as "myhandle.bsky.social")

### Manual Validation - Media Matching
1. Add a book titled "The Hobbit" to a collection
2. Search for "the hobbit" (all lowercase)
3. **Expected**: Shows aggregate stats (ratings, reviews) from existing database entry
4. Search for "THE HOBBIT" (all caps)
5. **Expected**: Same result

### Integration Test (Recommended Addition)
```typescript
// test/integration/media-search.test.ts
it('matches books case-insensitively by title and creator', async () => {
  // Insert a media item with mixed case
  await db.insertInto('media_items').values({
    title: 'The Great Gatsby',
    creator: 'F. Scott Fitzgerald',
    mediaType: 'book',
    totalRatings: 5,
  }).execute();

  // Search with different case
  const res = await request(app).get('/media/search?q=great+gatsby&type=book');
  expect(res.body.results[0].inDatabase).toBe(true);
  expect(res.body.results[0].totalRatings).toBe(5);
});
```

---

## Issue #29: Comments Not Showing

### Automated Tests (test/buildThreadsWithProfiles.test.ts)
- [x] Replies threaded under parent posts
- [x] Deeply nested reply chains work
- [x] Multiple top-level posts with separate reply trees
- [x] Empty input returns empty array
- [x] Posts with no replies get empty replies array
- [x] Unknown author DIDs get fallback profile
- [x] Posts sorted chronologically

### Manual Validation
1. Navigate to a group item page with existing discussions
   (e.g., https://app.collectivesocial.app/groups/did%3Aplc%3Abnq5k43mh4ekjlqsktkzif3c/lists/3meftfj7qgb24/items/3mkqqjyk7zu2s)
2. Click "Discussion" on a chapter/segment
3. **Expected**: Top-level posts AND their replies render as a threaded tree
4. Post a new reply to an existing comment
5. **Expected**: Reply appears nested under the parent comment after refresh
6. Post a top-level comment
7. **Expected**: Appears at the top level with no indentation

### Regression Prevention
The root cause was passing already-threaded data to a function that re-threads.
The unit test explicitly validates that flat posts with `parentPostUri` are correctly
assembled into trees, preventing any future regression.

---

## Issue #30: Performance Investigation

### Before/After Testing Methodology

#### 1. Establish Baselines (Before Changes)

**Frontend Metrics (Browser DevTools)**
```bash
# Use Lighthouse CI for repeatable measurements
npm install -g @lhci/cli
lhci autorun --collect.url=https://app.collectivesocial.app
```

Key metrics to capture:
- **First Contentful Paint (FCP)**: Target < 1.5s
- **Largest Contentful Paint (LCP)**: Target < 2.5s
- **Time to Interactive (TTI)**: Target < 3.5s
- **Total Blocking Time (TBT)**: Target < 200ms
- **Bundle size**: `npx vite-bundle-visualizer`

**API Metrics**
```bash
# Use autocannon or k6 for API benchmarks
npx autocannon -d 10 -c 10 http://localhost:3000/groups/{did}/segments/{rkey}/posts
npx autocannon -d 10 -c 10 http://localhost:3000/feed/events
npx autocannon -d 10 -c 10 http://localhost:3000/media/search?q=hobbit&type=book
```

Capture:
- **p50/p95/p99 latency** for each endpoint
- **Requests per second** throughput
- **Response payload size** (bytes)

**Database Queries**
```sql
-- Enable pg_stat_statements for query performance tracking
-- Run EXPLAIN ANALYZE on hot queries
EXPLAIN ANALYZE SELECT * FROM media_items WHERE LOWER(title) = 'the hobbit';
EXPLAIN ANALYZE SELECT * FROM reviews WHERE "authorDid" = 'did:plc:xxx' ORDER BY "createdAt" DESC;
```

#### 2. Implement Changes (Quick Wins)

| Change | Expected Impact | Measurement |
|--------|----------------|-------------|
| React.lazy code splitting | Smaller initial bundle, faster FCP | Bundle size, FCP |
| Batch profile fetches | Fewer API calls, faster comment load | Comment endpoint p95 |
| TanStack Query caching | Fewer refetches, faster navigation | Network requests count, TTI on navigation |
| Narrow .selectAll() | Smaller payloads | Response size bytes |

#### 3. Measure After (Same Tools)

```bash
# Re-run identical benchmarks after each change
lhci autorun --collect.url=https://app.collectivesocial.app

npx autocannon -d 10 -c 10 http://localhost:3000/groups/{did}/segments/{rkey}/posts
```

#### 4. Automated Performance Tests (Recommended)

```typescript
// test/performance/comments-latency.test.ts
import { describe, it, expect } from 'vitest';

describe('Comments endpoint performance', () => {
  it('responds within 500ms for a segment with 20 posts', async () => {
    const start = Date.now();
    const res = await fetch(`${API_URL}/groups/${DID}/segments/${RKEY}/posts`, {
      headers: { Cookie: sessionCookie },
    });
    const elapsed = Date.now() - start;

    expect(res.ok).toBe(true);
    expect(elapsed).toBeLessThan(500);
  });

  it('response payload is under 50KB for 20 posts', async () => {
    const res = await fetch(`${API_URL}/groups/${DID}/segments/${RKEY}/posts`, {
      headers: { Cookie: sessionCookie },
    });
    const body = await res.text();
    expect(body.length).toBeLessThan(50 * 1024);
  });
});
```

#### 5. Monitoring in Production

Recommended tools:
- **Web Vitals** (via `web-vitals` npm package) reporting to analytics
- **Datadog RUM** or similar for real user monitoring
- **API response time** tracked via Pino logs with request duration
- **Bundle size CI check** via `bundlesize` or `size-limit` in CI pipeline

### Performance Quick Win Implementation Plan

| Priority | Change | Files | Estimated Impact |
|----------|--------|-------|-----------------|
| 1 | Code splitting with React.lazy | `src/App.tsx` | -40% initial bundle |
| 2 | Batch profile lookups | `src/services/userProfiles.ts` | -80% comment latency |
| 3 | Add TanStack Query | Multiple components | -50% repeat load times |
| 4 | Narrow selectAll() | `src/routes/feed.ts`, `comments.ts` | -20% payload size |
| 5 | Add DB indexes | `src/db.ts` migration | -60% query time on hot paths |

---

## Running All Tests

```bash
# API (all 66 tests)
cd collective-social-api && npm test

# Web (all 38 tests, 1 pre-existing skip)
cd collective-social-web && npm test

# Specific test files
cd collective-social-api && npx vitest run test/buildThreadsWithProfiles.test.ts
cd collective-social-web && npx vitest run src/test/LoginButton.test.tsx
```
