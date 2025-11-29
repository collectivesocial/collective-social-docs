# Custom Lexicons for Collective Social

This document describes the custom AT Protocol lexicons used by Collective Social to store collections and media reviews.

## Overview

Collective Social uses custom lexicons stored in users' AT Protocol repositories instead of the standard Bluesky graph lists. This provides:

1. **Application-specific data structure** - Optimized for media tracking and reviews
2. **Decentralized storage** - Data lives in each user's AT Protocol repo
3. **Protocol compliance** - Follows AT Protocol standards for custom record types
4. **Future extensibility** - Easy to add new fields and features

## Lexicon Definitions

### app.collectivesocial.list

**File:** `lexicons/app.collectivesocial.list.json`

Represents a collection (list) of media items.

**Record Type:** `app.collectivesocial.list`

**Fields:**
- `name` (required, string) - Display name for the collection
- `description` (optional, string) - Optional description of the collection
- `visibility` (optional, enum) - Visibility setting: `public` or `private` (defaults to `public`)
- `purpose` (required, string) - Purpose identifier (e.g., 'app.collectivesocial.defs#curatelist')
- `avatar` (optional, blob) - Optional avatar image for the collection
- `createdAt` (required, datetime) - Timestamp when collection was created

**Visibility:**
- `public` - Collection appears on user's profile and can be viewed by anyone
- `private` - Collection is only visible to the owner

**Example:**
```json
{
  "$type": "app.collectivesocial.list",
  "name": "My Favorite Books",
  "description": "Books I've read and loved",
  "visibility": "public",
  "purpose": "app.collectivesocial.defs#curatelist",
  "createdAt": "2025-11-28T12:00:00.000Z"
}
```

### app.collectivesocial.listitem

**File:** `lexicons/app.collectivesocial.listitem.json`

Represents a media item (review) within a collection.

**Record Type:** `app.collectivesocial.listitem`

**Fields:**
- `list` (required, at-uri) - Reference to the parent collection (AT-URI)
- `title` (required, string) - Title of the media item
- `creator` (optional, string) - Creator/author of the media (e.g., book author, film director)
- `mediaType` (optional, enum) - Type of media: `book`, `movie`, `tv`, `podcast`, `article`, `game`, `music`
- `status` (optional, enum) - Consumption status: `want`, `in-progress`, `completed`
- `rating` (optional, number) - Rating from 0 to 5, supports half-star increments (0.5 steps)
- `review` (optional, string) - Review or notes about the media item
- `createdAt` (required, datetime) - Timestamp when item was added

**Example:**
```json
{
  "$type": "app.collectivesocial.listitem",
  "list": "at://did:plc:abc123/app.collectivesocial.list/xyz789",
  "title": "The Hobbit",
  "creator": "J.R.R. Tolkien",
  "mediaType": "book",
  "status": "completed",
  "rating": 5,
  "review": "A timeless classic that started it all. Bilbo's journey is heartwarming and adventurous.",
  "createdAt": "2025-11-28T12:30:00.000Z"
}
```

## Usage in API

### Creating Collections

```typescript
const record: AppCollectiveSocialList.Record = {
  $type: 'app.collectivesocial.list',
  name: 'My Reading List',
  description: 'Books to read',
  visibility: 'public', // or 'private'
  purpose: 'app.collectivesocial.defs#curatelist',
  createdAt: new Date().toISOString(),
};

const response = await agent.api.com.atproto.repo.createRecord({
  repo: agent.did!,
  collection: 'app.collectivesocial.list',
  record: record as any,
});
```

### Listing Collections

```typescript
const response = await agent.api.com.atproto.repo.listRecords({
  repo: agent.did!,
  collection: 'app.collectivesocial.list',
});

const collections = response.data.records.map((record: any) => ({
  uri: record.uri,
  name: record.value.name,
  description: record.value.description,
  visibility: record.value.visibility || 'public',
  // ... other fields
}));
```

### Adding Items to Collections

```typescript
const listItemRecord: AppCollectiveSocialListitem.Record = {
  $type: 'app.collectivesocial.listitem',
  list: 'at://did:plc:abc123/app.collectivesocial.list/xyz789',
  title: 'Example Book',
  creator: 'Author Name',
  mediaType: 'book',
  status: 'completed',
  rating: 4.5,
  review: 'Great read!',
  createdAt: new Date().toISOString(),
};

const response = await agent.api.com.atproto.repo.createRecord({
  repo: agent.did!,
  collection: 'app.collectivesocial.listitem',
  record: listItemRecord as any,
});
```

### Listing Items in a Collection

```typescript
const response = await agent.api.com.atproto.repo.listRecords({
  repo: agent.did!,
  collection: 'app.collectivesocial.listitem',
});

// Filter items for a specific list
const items = response.data.records
  .filter((record: any) => record.value.list === listUri)
  .map((record: any) => ({
    uri: record.uri,
    title: record.value.title,
    creator: record.value.creator,
    // ... other fields
  }));
```

## TypeScript Types

TypeScript interfaces are defined in `src/types/lexicon.ts`:

```typescript
export namespace AppCollectiveSocialList {
  export interface Record {
    $type?: 'app.collectivesocial.list';
    name: string;
    description?: string;
    visibility?: 'public' | 'private';
    purpose: string;
    avatar?: {
      cid: string;
      mimeType: string;
    };
    createdAt: string;
  }
}

export namespace AppCollectiveSocialListitem {
  export interface Record {
    $type?: 'app.collectivesocial.listitem';
    list: string;
    title: string;
    creator?: string;
    mediaType?: 'book' | 'movie' | 'tv' | 'podcast' | 'article' | 'game' | 'music';
    status?: 'want' | 'in-progress' | 'completed';
    rating?: number;
    review?: string;
    createdAt: string;
  }
}
```

## Migration from Bluesky Graph Lists

The previous implementation used:
- `app.bsky.graph.list` for collections
- `app.bsky.graph.listitem` for items (with post references)

The new implementation:
- Uses `app.collectivesocial.list` for collections
- Uses `app.collectivesocial.listitem` for items (with embedded review data)
- Stores all data directly in the AT Protocol repo
- No longer creates posts for each review

## Public Collections Endpoint

To fetch public collections for display on a user's profile:

```typescript
const response = await fetch(`${apiUrl}/collections/public/${userDid}`, {
  credentials: 'include',
});

const { collections } = await response.json();
// Returns only collections where visibility === 'public'
```

This endpoint:
- Can be called with or without authentication
- Only returns collections with `visibility: 'public'`
- Suitable for profile pages and public discovery

## Benefits

1. **Self-contained data** - All review information stored in the listitem record
2. **No post pollution** - Reviews don't clutter the user's post feed
3. **Efficient queries** - Direct record queries without post lookups
4. **Richer metadata** - Custom fields for media type, status, ratings
5. **Privacy controls** - Public/private visibility for collections
6. **Application ownership** - Clear data ownership and structure

## Future Enhancements

Potential additions to the lexicons:
- Tags/labels for categorization
- Cover images for items
- External links (ISBN, IMDB, etc.)
- Collaborative collections (shared lists)
- Import/export functionality
- Activity tracking (start date, finish date)
