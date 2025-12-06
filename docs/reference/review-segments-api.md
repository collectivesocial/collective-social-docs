# Review Segments API

Review segments allow users to post incremental reviews and thoughts as they consume content (e.g., chapter-by-chapter for books, episode-by-episode for TV shows).

## Endpoints

### POST /reviewsegments
Create a new review segment.

**Request Body:**
```json
{
  "text": "string (required) - The review segment content",
  "percentage": "number (required, 0-100) - How far through the content (e.g., 25 for 25%)",
  "title": "string (optional) - A title for this segment (e.g., 'Chapter 5 thoughts')",
  "mediaItemId": "number (optional) - The media item ID this segment is about",
  "mediaType": "string (optional) - Type of media (book, movie, tv, etc.)",
  "listItem": "string (optional) - URI of the list item this segment belongs to"
}
```

**Response:**
```json
{
  "success": true,
  "uri": "at://did:plc:xxx/app.collectivesocial.feed.reviewsegment/xxx",
  "cid": "bafyxxx",
  "reviewSegment": {
    "uri": "at://did:plc:xxx/app.collectivesocial.feed.reviewsegment/xxx",
    "cid": "bafyxxx",
    "value": {
      "text": "Really enjoying this chapter...",
      "percentage": 25,
      "title": "Chapter 5 thoughts",
      "mediaItemId": 123,
      "mediaType": "book",
      "listItem": "at://did:plc:xxx/app.collectivesocial.feed.listitem/xxx",
      "createdAt": "2024-01-01T00:00:00.000Z"
    }
  }
}
```

### PUT /reviewsegments/:uri
Update an existing review segment.

**URL Parameters:**
- `uri` - The URI of the review segment (URL encoded)

**Request Body:** Same as POST, all fields are optional except `text` and `percentage` cannot be empty if provided.

**Response:** Same as POST

### DELETE /reviewsegments/:uri
Delete a review segment.

**URL Parameters:**
- `uri` - The URI of the review segment (URL encoded)

**Response:**
```json
{
  "success": true
}
```

### GET /reviewsegments/media/:mediaItemId
Get all review segments for a specific media item (current user only).

**URL Parameters:**
- `mediaItemId` - The media item ID

**Response:**
```json
{
  "segments": [
    {
      "uri": "at://did:plc:xxx/app.collectivesocial.feed.reviewsegment/xxx",
      "cid": "bafyxxx",
      "value": {
        "text": "Great start!",
        "percentage": 10,
        "title": "Chapter 1",
        "mediaItemId": 123,
        "createdAt": "2024-01-01T00:00:00.000Z"
      }
    },
    {
      "uri": "at://did:plc:xxx/app.collectivesocial.feed.reviewsegment/yyy",
      "cid": "bafyyyy",
      "value": {
        "text": "Plot twist!",
        "percentage": 50,
        "title": "Midpoint",
        "mediaItemId": 123,
        "createdAt": "2024-01-02T00:00:00.000Z"
      }
    }
  ]
}
```

*Note: Results are sorted by percentage in ascending order (chronological consumption order).*

### GET /reviewsegments/list/:listItemUri
Get all review segments for a specific list item (current user only).

**URL Parameters:**
- `listItemUri` - The list item URI (URL encoded)

**Response:** Same as `/media/:mediaItemId`

### GET /reviewsegments/user/:did
Get all review segments for any user (for viewing others' segments).

**URL Parameters:**
- `did` - The DID of the user whose segments to retrieve

**Query Parameters:**
- `mediaItemId` (optional) - Filter segments for a specific media item

**Response:** Same as `/media/:mediaItemId`

## Usage Examples

### Creating a segment while reading a book
```javascript
const response = await fetch('http://localhost:3000/reviewsegments', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',
  body: JSON.stringify({
    text: 'The world-building in this chapter is incredible. The author really brings the setting to life.',
    percentage: 15,
    title: 'Chapter 3 - World Building',
    mediaItemId: 456,
    mediaType: 'book',
    listItem: 'at://did:plc:abc123/app.collectivesocial.feed.listitem/xyz789'
  })
});
```

### Updating a segment to fix a typo
```javascript
const segmentUri = encodeURIComponent('at://did:plc:abc123/app.collectivesocial.feed.reviewsegment/xyz789');
const response = await fetch(`http://localhost:3000/reviewsegments/${segmentUri}`, {
  method: 'PUT',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',
  body: JSON.stringify({
    text: 'The world-building in these chapters is incredible. The author really brings the setting to life.'
  })
});
```

### Getting all segments for a book you're reading
```javascript
const response = await fetch('http://localhost:3000/reviewsegments/media/456', {
  credentials: 'include'
});
const data = await response.json();
console.log(data.segments); // Array sorted by percentage
```

### Viewing someone else's reading journey
```javascript
const did = 'did:plc:otherperson123';
const mediaItemId = 456;
const response = await fetch(`http://localhost:3000/reviewsegments/user/${did}?mediaItemId=${mediaItemId}`, {
  credentials: 'include'
});
const data = await response.json();
console.log(data.segments); // Their segments for this book
```

## Notes

- All endpoints require authentication
- Users can only create/update/delete their own segments
- Users can view any user's segments (public by default via AT Protocol)
- Percentage field helps readers avoid spoilers by knowing "how far in" a segment was written
- Segments are automatically sorted by percentage for chronological reading
- The `listItem` field connects segments to specific entries in collections
