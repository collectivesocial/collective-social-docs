# Open Library Integration

This document describes how Collective Social integrates with the Open Library API to enable book search and metadata enrichment.

## Overview

Collective Social uses the [Open Library API](https://openlibrary.org/developers/api) to provide book search functionality and fetch detailed book metadata. The integration enables users to:

1. **Search for books** by title or author
2. **Fetch book details** including cover images, descriptions, and publication information
3. **Store enriched metadata** in the local database
4. **Track aggregated ratings** across all user reviews

## Architecture

### Components

The Open Library integration consists of three main components:

1. **Service Layer** (`src/services/openlibrary.ts`) - Core API client and helper functions
2. **API Routes** (`src/routes/media.ts`) - HTTP endpoints for search and media management
3. **UI Component** (`src/components/MediaSearch.tsx`) - React component for book search interface

### Data Flow

```
User Search → MediaSearch Component → POST /media/search
                                           ↓
                                    OpenLibrary API
                                           ↓
                                    Database Lookup (enrichment)
                                           ↓
                                    Results with ratings
                                           ↓
User Selection → POST /media/add → Store in media_items table
                                           ↓
                                    Add to Collection (listitem)
```

## Open Library Service

### Location
`collective-social-api/src/services/openlibrary.ts`

### Functions

#### `searchBooks(query: string, limit?: number)`

Searches Open Library for books matching the query string.

**Parameters:**
- `query` - Search query (title, author, or combination)
- `limit` - Maximum number of results (default: 10)

**Returns:** Array of `OpenLibrarySearchResult`

**Example:**
```typescript
const results = await searchBooks('Lord of the Rings Tolkien', 10);
```

**API Endpoint:** `https://openlibrary.org/search.json?q={query}&limit={limit}`

#### `getBookByISBN(isbn: string)`

Fetches detailed book information using ISBN.

**Parameters:**
- `isbn` - ISBN-10 or ISBN-13

**Returns:** `OpenLibraryBook | null`

**Example:**
```typescript
const book = await getBookByISBN('9780547928227');
```

**API Endpoint:** `https://openlibrary.org/isbn/{isbn}.json`

#### `getBookByKey(key: string)`

Fetches book details using Open Library work or edition key.

**Parameters:**
- `key` - Open Library key (e.g., `/works/OL45883W`)

**Returns:** `OpenLibraryBook | null`

**Example:**
```typescript
const book = await getBookByKey('/works/OL45883W');
```

**API Endpoint:** `https://openlibrary.org{key}.json`

#### `getCoverUrl(coverId: number, size?: 'S' | 'M' | 'L')`

Generates cover image URL from cover ID.

**Parameters:**
- `coverId` - Open Library cover ID
- `size` - Image size: S (small), M (medium), L (large), default: M

**Returns:** Cover image URL string

**Example:**
```typescript
const coverUrl = getCoverUrl(12345, 'M');
// Returns: https://covers.openlibrary.org/b/id/12345-M.jpg
```

#### `extractISBN(result: OpenLibrarySearchResult | OpenLibraryBook)`

Extracts primary ISBN from search result or book data, preferring ISBN-13.

**Returns:** ISBN string or `undefined`

#### `extractDescription(book: OpenLibraryBook)`

Extracts description text from book data (handles both string and object formats).

**Returns:** Description string or `undefined`

## API Endpoints

### Location
`collective-social-api/src/routes/media.ts`

### POST /media/search

Search for books and enrich with database information.

**Authentication:** Required

**Request Body:**
```json
{
  "query": "The Hobbit",
  "mediaType": "book"
}
```

**Response:**
```json
{
  "results": [
    {
      "title": "The Hobbit",
      "author": "J.R.R. Tolkien",
      "publishYear": 1937,
      "isbn": "9780547928227",
      "coverImage": "https://covers.openlibrary.org/b/id/12345-M.jpg",
      "inDatabase": true,
      "totalReviews": 15,
      "averageRating": 4.7,
      "mediaItemId": 42
    }
  ]
}
```

**Flow:**
1. Search Open Library API
2. For each result, extract ISBN
3. Check local database for existing reviews
4. Enrich results with `totalReviews` and `averageRating`
5. Return combined data

### POST /media/add

Add a book to the local database.

**Authentication:** Required

**Request Body:**
```json
{
  "title": "The Hobbit",
  "creator": "J.R.R. Tolkien",
  "mediaType": "book",
  "isbn": "9780547928227",
  "coverImage": "https://covers.openlibrary.org/b/id/12345-M.jpg",
  "publishYear": 1937
}
```

**Response:**
```json
{
  "mediaItemId": 42,
  "existed": false
}
```

**Behavior:**
- Checks if book already exists by ISBN
- If exists, returns existing `mediaItemId`
- If new, fetches additional details from Open Library (description)
- Inserts into `media_items` table
- Returns new `mediaItemId`

### GET /media/:id

Fetch media item details from database.

**Authentication:** Optional

**Response:**
```json
{
  "id": 42,
  "mediaType": "book",
  "title": "The Hobbit",
  "creator": "J.R.R. Tolkien",
  "isbn": "9780547928227",
  "coverImage": "https://covers.openlibrary.org/b/id/12345-M.jpg",
  "description": "A great tale of adventure...",
  "publishedYear": 1937,
  "totalReviews": 15,
  "averageRating": 4.7
}
```

## Database Schema

### media_items Table

```sql
CREATE TABLE media_items (
  id SERIAL PRIMARY KEY,
  media_type VARCHAR(50) NOT NULL,
  title VARCHAR(500) NOT NULL,
  creator VARCHAR(255),
  isbn VARCHAR(20),
  external_id VARCHAR(255),
  cover_image TEXT,
  description TEXT,
  published_year INTEGER,
  total_reviews INTEGER DEFAULT 0,
  average_rating DECIMAL(3,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_media_items_isbn ON media_items(isbn);
CREATE INDEX idx_media_items_type ON media_items(media_type);
```

### Fields

- `id` - Auto-incrementing primary key
- `media_type` - Type of media (currently only 'book')
- `title` - Book title from Open Library
- `creator` - Author name(s)
- `isbn` - ISBN-10 or ISBN-13 (used for lookups)
- `external_id` - Reserved for other external identifiers
- `cover_image` - Full URL to cover image
- `description` - Book description from Open Library
- `published_year` - First publication year
- `total_reviews` - Count of all reviews (updated on add/delete)
- `average_rating` - Average rating across all reviews (updated on add/delete)
- `created_at` - Timestamp when record was created
- `updated_at` - Timestamp when record was last updated

## MediaSearch Component

### Location
`collective-social-web/src/components/MediaSearch.tsx`

### Usage

```tsx
import { MediaSearch } from '../components/MediaSearch';

function MyComponent() {
  const handleSelect = (result) => {
    console.log('Selected book:', result);
  };

  return (
    <MediaSearch 
      apiUrl="http://localhost:3000"
      onSelect={handleSelect}
    />
  );
}
```

### Features

1. **Media Type Selection** - Dropdown to choose media type (currently only books)
2. **Search Input** - Text field for query input
3. **Results Display** - Grid of search results with cover images
4. **Database Indicators** - Shows existing ratings when book is in database
5. **Auto-add to Database** - Automatically adds book to database when selected

### User Flow

1. User enters search query
2. Component calls `POST /media/search`
3. Results displayed with cover images
4. User clicks on a book
5. If not in database, `POST /media/add` is called automatically
6. `onSelect` callback fired with complete result (including `mediaItemId`)
7. Parent component can then add to collection

## Aggregated Ratings

### How It Works

The system maintains aggregated ratings in the `media_items` table:

#### On Review Creation

When a user adds a book to their collection with a rating:

```typescript
const currentAvg = currentItem.averageRating || 0;
const newTotalReviews = currentItem.totalReviews + 1;
const newAverage = 
  (currentAvg * currentItem.totalReviews + newRating) / newTotalReviews;

await db.updateTable('media_items')
  .set({
    totalReviews: newTotalReviews,
    averageRating: parseFloat(newAverage.toFixed(2)),
    updatedAt: new Date(),
  })
  .where('id', '=', mediaItemId)
  .execute();
```

#### On Review Deletion

When a user removes a book from their collection:

```typescript
const newTotalReviews = currentItem.totalReviews - 1;

if (newTotalReviews === 0) {
  // Reset to 0 if no more reviews
  await db.updateTable('media_items')
    .set({
      totalReviews: 0,
      averageRating: 0,
      updatedAt: new Date(),
    })
    .where('id', '=', mediaItemId)
    .execute();
} else {
  // Recalculate average without this rating
  const currentAvg = currentItem.averageRating || 0;
  const newAverage = 
    (currentAvg * currentItem.totalReviews - deletedRating) / newTotalReviews;

  await db.updateTable('media_items')
    .set({
      totalReviews: newTotalReviews,
      averageRating: parseFloat(newAverage.toFixed(2)),
      updatedAt: new Date(),
    })
    .where('id', '=', mediaItemId)
    .execute();
}
```

### Display

Aggregated ratings are displayed in search results and collection items:

```tsx
{item.mediaItem && item.mediaItem.totalReviews > 1 && (
  <div>
    Community: ⭐ {item.mediaItem.averageRating?.toFixed(1)} 
    ({item.mediaItem.totalReviews} reviews)
  </div>
)}
```

## Data Models

### TypeScript Interfaces

#### OpenLibrarySearchResult

```typescript
interface OpenLibrarySearchResult {
  key: string;                    // Work key, e.g., "/works/OL45883W"
  title: string;                  // Book title
  author_name?: string[];         // Array of author names
  first_publish_year?: number;    // First publication year
  isbn?: string[];                // Array of ISBNs
  cover_i?: number;              // Cover image ID
  publisher?: string[];          // Array of publisher names
}
```

#### OpenLibraryBook

```typescript
interface OpenLibraryBook {
  title: string;
  authors?: Array<{ name: string }>;
  publish_date?: string;
  publishers?: string[];
  isbn_13?: string[];
  isbn_10?: string[];
  covers?: number[];
  description?: string | { value: string };
  number_of_pages?: number;
}
```

#### MediaItem (Database)

```typescript
interface MediaItem {
  id: Generated<number>;
  mediaType: 'book' | 'movie' | 'tv' | 'podcast' | 'article' | 'game' | 'music';
  title: string;
  creator?: string;
  isbn?: string;
  externalId?: string;
  coverImage?: string;
  description?: string;
  publishedYear?: number;
  totalReviews: number;
  averageRating?: number;
  createdAt: Date;
  updatedAt: Date;
}
```

## Integration with Collections

### Adding Books to Collections

When a user adds a book to a collection:

1. Book is searched via MediaSearch component
2. User selects book from results
3. Book is automatically added to `media_items` table if not exists
4. User provides review data (status, rating, review text)
5. Listitem is created in AT Protocol repo with `mediaItemId` reference
6. Aggregated rating is updated in `media_items` table

### Collection Item Enrichment

When fetching collection items:

```typescript
// GET /collections/:listUri/items enriches items
const items = await Promise.all(
  listitems.map(async (listitem) => {
    const item = { ...listitem };
    
    if (listitem.mediaItemId) {
      const mediaItem = await db
        .selectFrom('media_items')
        .selectAll()
        .where('id', '=', listitem.mediaItemId)
        .executeTakeFirst();
      
      if (mediaItem) {
        item.mediaItem = mediaItem; // Includes cover, description, ratings
      }
    }
    
    return item;
  })
);
```

## Error Handling

### Open Library API Errors

- **404 Not Found** - Book doesn't exist, returns `null`
- **Network Errors** - Throws error, caught by calling code
- **Malformed Responses** - Handled with TypeScript type guards

### Database Errors

- **Duplicate ISBNs** - Checked before insert, existing record returned
- **Missing mediaItemId** - Gracefully handled, items display without enrichment
- **Rating calculation errors** - Logged and skipped, doesn't block operation

## Rate Limiting

Open Library doesn't have strict rate limits for the public API, but best practices:

- **Reasonable delays** between bulk operations
- **Cache results** in database to minimize API calls
- **Batch processing** for imports should be throttled

## Future Enhancements

### Planned Improvements

1. **Additional metadata**
   - Genres/subjects from Open Library
   - Multiple edition support
   - Publisher information

2. **Search improvements**
   - Advanced search filters (year, language)
   - Pagination for large result sets
   - Search history/suggestions

3. **Other media types**
   - Movies (TMDB API)
   - TV Shows (TMDB API)
   - Music (MusicBrainz API)

4. **Performance optimizations**
   - Redis caching for search results
   - Background jobs for metadata enrichment
   - CDN for cover images

5. **Enhanced matching**
   - Fuzzy matching for titles
   - Multiple ISBN support per book
   - Edition linking

## Resources

- [Open Library API Documentation](https://openlibrary.org/developers/api)
- [Open Library Search API](https://openlibrary.org/dev/docs/api/search)
- [Open Library Books API](https://openlibrary.org/dev/docs/api/books)
- [Open Library Covers API](https://openlibrary.org/dev/docs/api/covers)
- [Open Library Data Dumps](https://openlibrary.org/developers/dumps)

## Examples

### Complete Book Search and Add Flow

```typescript
// 1. Search for books
const searchResults = await searchBooks('Hobbit Tolkien');

// 2. User selects first result
const selectedBook = searchResults[0];

// 3. Add to database if not exists
let mediaItemId = selectedBook.mediaItemId;

if (!mediaItemId) {
  const isbn = extractISBN(selectedBook);
  const bookDetails = await getBookByISBN(isbn);
  const description = extractDescription(bookDetails);
  
  const result = await db.insertInto('media_items')
    .values({
      mediaType: 'book',
      title: selectedBook.title,
      creator: selectedBook.author_name?.[0],
      isbn: isbn,
      coverImage: getCoverUrl(selectedBook.cover_i, 'M'),
      description: description,
      publishedYear: selectedBook.first_publish_year,
      totalReviews: 0,
      averageRating: undefined,
    })
    .returning('id')
    .executeTakeFirstOrThrow();
    
  mediaItemId = result.id;
}

// 4. Create listitem with review
const listitem = await agent.api.com.atproto.repo.createRecord({
  repo: agent.did,
  collection: 'app.collectivesocial.listitem',
  record: {
    $type: 'app.collectivesocial.listitem',
    list: listUri,
    title: selectedBook.title,
    creator: selectedBook.author_name?.[0],
    mediaType: 'book',
    mediaItemId: mediaItemId,
    status: 'completed',
    rating: 5,
    review: 'Amazing book!',
    createdAt: new Date().toISOString(),
  },
});

// 5. Update aggregated rating
// (Handled automatically by POST /collections/:listUri/items endpoint)
```

### Displaying Books with Community Ratings

```typescript
// Fetch collection items with enrichment
const response = await fetch(
  `/collections/${encodeURIComponent(listUri)}/items`,
  { credentials: 'include' }
);

const { items } = await response.json();

// Render with community data
items.forEach(item => {
  console.log(`${item.title} by ${item.creator}`);
  console.log(`Your rating: ${item.rating} stars`);
  
  if (item.mediaItem?.totalReviews > 1) {
    console.log(
      `Community: ${item.mediaItem.averageRating} stars ` +
      `(${item.mediaItem.totalReviews} reviews)`
    );
  }
});
```
