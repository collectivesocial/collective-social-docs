# Collective Social - Knowledge Priming Document

> This document provides curated project context for AI assistants and new contributors.
> Following the [Knowledge Priming](https://martinfowler.com/articles/reduce-friction-ai/knowledge-priming.html) pattern:
> share architectural decisions, conventions, and examples _before_ asking for code generation.

---

## 1. Architecture Overview

Collective Social is a **book/media tracking and review platform** built on the AT Protocol (Bluesky ecosystem). It enables users to create collections of media, rate and review items, and share recommendations socially.

### System Components

```
┌─────────────────────┐     ┌─────────────────────┐     ┌──────────────────┐
│  collective-social  │     │  collective-social  │     │   AT Protocol    │
│        -web         │────▶│        -api         │────▶│   (Bluesky PDS)  │
│   (React frontend)  │     │  (Express backend)  │     │                  │
└─────────────────────┘     └────────┬────────────┘     └──────────────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │   PostgreSQL DB   │
                            │  (aggregates &    │
                            │   local cache)    │
                            └──────────────────┘
```

### Data Flow

1. **User data** lives in AT Protocol repos (decentralized, user-owned)
2. **API** reads/writes AT Protocol records AND maintains a PostgreSQL cache for aggregates (ratings, counts, search)
3. **Frontend** communicates only with the API; never directly with AT Protocol

### Key Architectural Decisions

- **Decentralized storage**: Collections and reviews are AT Protocol records in user repos, not just in our database
- **PostgreSQL as cache**: The DB stores aggregates (rating distributions, counts) and app-specific data (media metadata, admin state) but is NOT the source of truth for user content
- **Session-based auth**: OAuth via AT Protocol, sessions via iron-session (not JWT/localStorage)
- **No global state management**: React local state + prop drilling (no Redux/Zustand/Context for data)

---

## 2. Tech Stack and Versions

### Frontend (collective-social-web)

| Technology | Version | Notes |
|-----------|---------|-------|
| React | 19.0.0 | Latest major version |
| TypeScript | 5.x | Strict mode |
| Vite | 6.0.1 | Build tool, HMR in dev |
| Chakra UI | v3.30.0 | **Not v2** - significant API changes |
| React Router DOM | 7.1.1 | File-based concepts but configured routes |
| Emotion | latest | CSS-in-JS via Chakra |
| Lucide React | via react-icons/lu | Icon library |

### Backend (collective-social-api)

| Technology | Version | Notes |
|-----------|---------|-------|
| Node.js | 20.x LTS | Runtime |
| Express.js | 5.x | **Not 4.x** - async error handling built-in |
| TypeScript | 5.x | Strict mode |
| Kysely | 0.28.8 | Type-safe SQL query builder (not an ORM) |
| PostgreSQL | 15+ | Primary database |
| @atproto/oauth-client-node | latest | AT Protocol OAuth |
| iron-session | latest | Encrypted cookie sessions |
| Pino | latest | Structured logging |

### Critical Version Notes

- **Chakra UI v3**: `Modal` is now `Dialog`, `isOpen` is now `open`, `colorScheme` is now `colorPalette`
- **Express 5**: Async errors auto-propagate (no need for `next(err)` in most cases)
- **Kysely**: Use `sql` tagged template literal, NOT `ctx.db.raw()` (does not exist)

---

## 3. Project Structure

### Frontend

```
collective-social-web/src/
├── App.tsx                    # Routes + auth state + API URL config
├── main.tsx                   # Entry: Chakra Provider setup
├── theme.ts                   # Chakra createSystem() theme config
├── components/
│   ├── LoginButton.tsx        # AT Protocol OAuth login form
│   ├── ReviewSegments.tsx     # Chapter-by-chapter review (edit mode)
│   ├── ReviewSegmentsViewer.tsx # Chapter reviews (read-only)
│   ├── FeedList.tsx           # Activity feed with profile lookups
│   ├── ui/                    # Generated Chakra UI primitives
│   └── ...
├── pages/
│   ├── GroupItemDetailPage.tsx # Item within a group (has discussions)
│   ├── GroupListDetailPage.tsx # List within a group
│   ├── ItemDetailsPage.tsx    # Media item details + reviews
│   └── ...
├── types/                     # Shared TypeScript interfaces
└── utils/
    └── textUtils.tsx          # @mention, #hashtag, URL parsing
```

### Backend

```
collective-social-api/src/
├── index.ts                   # Express app setup, middleware, route mounting
├── config.ts                  # envalid environment config
├── context.ts                 # AppContext: db, oauthClient, logger
├── db.ts                      # Kysely setup + ALL migrations (inline)
├── auth/client.ts             # AT Protocol OAuth client config
├── models/                    # TypeScript interfaces for DB tables
├── routes/
│   ├── auth.ts                # /auth/login, /auth/callback, /auth/logout
│   ├── collections.ts         # Collections, items, reviews, ratings
│   ├── comments.ts            # Discussion threads and replies
│   ├── feed.ts                # Activity feed
│   ├── media.ts               # Media item CRUD, OpenLibrary search
│   └── ...
├── middleware/
│   └── trackUserActivity.ts   # Last-active timestamp tracking
└── lexicons/                  # AT Protocol schema definitions
```

---

## 4. Naming Conventions

### Files

- **Components**: PascalCase (`LoginButton.tsx`, `StarRating.tsx`)
- **Pages**: PascalCase with `Page` suffix (`ItemDetailsPage.tsx`)
- **Utilities**: camelCase (`textUtils.tsx`)
- **Routes (API)**: camelCase, noun-based (`collections.ts`, `comments.ts`)
- **Models**: camelCase, singular (`user.ts`, `media.ts`)

### Code

- **React components**: PascalCase function components (`function LoginButton()`)
- **Hooks**: `use` prefix (`useState`, `useEffect`, `useNavigate`)
- **API fetch functions**: verb-noun (`fetchComments`, `createReview`)
- **Event handlers**: `handle` prefix (`handleSubmit`, `handleBack`)
- **Boolean state**: `is`/`has` prefix (`isLoading`, `hasError`)
- **Constants**: camelCase for module-level (`apiUrl`), SCREAMING_SNAKE for true constants

### Database

- **Tables**: snake_case plural (`media_items`, `review_segments`)
- **Columns**: camelCase in Kysely interfaces, snake_case in actual PostgreSQL
- **Migrations**: Sequential numbered strings (`'001'`, `'002'`, ... `'013'`)

### AT Protocol

- **Lexicons**: reverse-domain (`app.collectivesocial.list`, `app.collectivesocial.listitem`)
- **Record keys (rkey)**: nanoid-generated short IDs

---

## 5. Key Patterns and Examples

### Frontend: API Data Fetching (current pattern)

```tsx
// Standard pattern: local state + useEffect + fetch
const [data, setData] = useState<ItemType[]>([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState<string | null>(null);

useEffect(() => {
  const fetchData = async () => {
    try {
      const response = await fetch(`${apiUrl}/endpoint`, {
        credentials: 'include',  // ALWAYS include for auth
      });
      if (!response.ok) throw new Error('Failed to fetch');
      const result = await response.json();
      setData(result);
    } catch (err) {
      setError('Failed to load data');
    } finally {
      setLoading(false);
    }
  };
  fetchData();
}, [apiUrl]);
```

### Frontend: Chakra UI v3 Dialog

```tsx
// CORRECT for Chakra v3
<DialogRoot open={isOpen} onOpenChange={(e) => setIsOpen(e.open)}>
  <DialogContent>
    <DialogHeader>Title</DialogHeader>
    <DialogBody>Content</DialogBody>
  </DialogContent>
</DialogRoot>

// WRONG - these are v2 patterns
// <Modal isOpen={isOpen} onClose={onClose}>
```

### Backend: Route Handler with Auth

```typescript
router.get('/endpoint', async (req, res) => {
  try {
    const agent = await ctx.oauthClient.restore(req.session.sessionId);
    if (!agent) {
      return res.status(401).json({ error: 'Not authenticated' });
    }

    const results = await ctx.db
      .selectFrom('table_name')
      .select(['id', 'title', 'createdAt'])  // Prefer explicit columns
      .where('userDid', '=', agent.did!)
      .orderBy('createdAt', 'desc')
      .execute();

    res.json(results);
  } catch (err) {
    ctx.logger.error({ err }, 'Failed to fetch data');
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

### Backend: AT Protocol Record Operations

```typescript
// Create a record in user's AT Protocol repo
await agent.api.com.atproto.repo.putRecord({
  repo: agent.did!,
  collection: 'app.collectivesocial.listitem',
  rkey: nanoid(13),
  record: {
    $type: 'app.collectivesocial.listitem',
    list: listUri,
    title: 'Book Title',
    mediaType: 'book',
    status: 'want',
    createdAt: new Date().toISOString(),
  },
});
```

---

## 6. Anti-Patterns (Do NOT Do These)

| Anti-Pattern | Why | Do This Instead |
|-------------|-----|-----------------|
| `ctx.db.raw(...)` | Does not exist in Kysely | Use `sql` tagged template |
| `<Modal isOpen={...}>` | Chakra v2 API | Use `<DialogRoot open={...}>` |
| `colorScheme="teal"` | Chakra v2 prop | Use `colorPalette="teal"` |
| `history.push('/path')` | React Router v5 | Use `navigate('/path')` from `useNavigate()` |
| JWT in localStorage | Security risk, not our pattern | Sessions via iron-session cookies |
| `fetch()` without `credentials: 'include'` | Auth cookies won't be sent | Always include credentials |
| Class-based components | Not used in this codebase | Use function components with hooks |
| Redux/Zustand/Context for server state | Over-engineering for current scale | Local state (future: TanStack Query) |
| `.selectAll()` in hot paths | Over-fetches columns, larger payloads | Select only needed columns |
| Per-row external API calls in loops | N+1 performance problem | Batch or pre-fetch data |

---

## 7. Domain-Specific Knowledge

### Rating System

- Range: 0 to 5 in 0.5 increments (11 possible values)
- Distribution tracked per-column: `rating0`, `rating0_5`, `rating1`, ... `rating5`
- Helper: `getRatingColumnName(2.5)` returns `"rating2_5"`
- Aggregates: `totalRatings`, `totalReviews`, `averageRating`

### Collections and Lists

- A **collection** (list) is an AT Protocol record of type `app.collectivesocial.list`
- A **list item** is an AT Protocol record of type `app.collectivesocial.listitem`
- Items link to their parent list via an `at://` URI
- Items can reference a `mediaItemId` in the local PostgreSQL for aggregate data

### Groups

- Groups are collaborative spaces where members share lists
- Each group has lists, and each list has items
- Items within groups can have **discussions** (comment threads)
- URL pattern: `/groups/{did}/lists/{listId}/items/{itemId}`

### Review Segments

- Allow chapter-by-chapter or section-by-section reviews
- Only show when item status is `in-progress` AND user is owner
- Support media-type-specific length units (pages, episodes, minutes)

### Recommendations

- Track who recommended an item (by DID or handle)
- Handles are auto-resolved to DIDs via AT Protocol
- Multiple recommenders per item supported

---

## 8. Curated Knowledge Sources

### Official Documentation

| Topic | Source | Why |
|-------|--------|-----|
| AT Protocol | https://atproto.com/docs | Authoritative for lexicon/record patterns |
| Chakra UI v3 | https://www.chakra-ui.com/docs | Must use v3 docs, not v2 |
| Kysely | https://kysely.dev/docs/getting-started | Query builder patterns |
| React Router v7 | https://reactrouter.com/home | Routing patterns |
| Vite | https://vite.dev/guide/ | Build config |

### Internal Documentation

| Topic | Path | What It Covers |
|-------|------|----------------|
| Custom Lexicons | `docs/reference/custom-lexicons.md` | AT Protocol record schemas |
| Review Segments | `docs/reference/review-segments-quick-reference.md` | Segment component usage |
| Recommendations | `docs/reference/recommendations.md` | Recommender feature |
| OpenLibrary | `docs/reference/openlibrary-integration.md` | Book search/metadata |
| Review Segments API | `docs/reference/review-segments-api.md` | Full API reference |

### Key Articles

| Concept | Source | Relevance |
|---------|--------|-----------|
| Knowledge Priming | https://martinfowler.com/articles/reduce-friction-ai/knowledge-priming.html | This document's methodology |
| AT Protocol Overview | https://atproto.com/guides/overview | Why we use decentralized storage |

---

## 9. Development Workflow

### Getting Started

```bash
# Frontend
cd collective-social-web
npm install
npm run dev          # http://localhost:5173

# Backend
cd collective-social-api
npm run docker:up    # Start PostgreSQL
cp .env.example .env # Configure environment
npm run dev          # http://localhost:3000 (auto-runs migrations)
```

### Testing (Manual)

- No automated test suite yet (planned)
- Frontend: Browser DevTools + React DevTools + Network tab
- Backend: API client (Postman/Insomnia) + direct PostgreSQL queries
- Integration: Run both locally, test end-to-end flows

### Code Quality

```bash
# Frontend
npm run lint         # ESLint
npm run build        # TypeScript check + production build

# Backend
npm run format:check # Prettier check
npm run format       # Auto-format
```

---

## 10. Known Performance Considerations

### Current Pain Points

1. **No code splitting**: All pages are eagerly imported in App.tsx
2. **N+1 profile fetches**: Comments and feed fetch profiles per-row from Bluesky API
3. **No client-side caching**: Every navigation re-fetches data
4. **Auth check blocks render**: Initial `/users/me` call blocks the entire app

### Recommended Improvements (Prioritized)

1. Add `React.lazy()` + `Suspense` for route-level code splitting
2. Batch or cache AT Protocol profile lookups
3. Introduce TanStack Query for client-side caching and deduplication
4. Narrow `.selectAll()` to specific columns in hot API paths
5. Add database indexes for frequently-filtered columns
6. Implement skeleton loading states instead of spinners
