# Item Recommendations

## Overview

The Collective Social platform supports tracking who recommended items in lists. This feature allows users to record when someone suggests a book or media item, making it easy to remember who to reach out to for discussions after experiencing the item.

## Data Structure

### Lexicon Schema

Each list item can include a `recommendations` array with the following structure:

```json
{
  "recommendations": [
    {
      "did": "did:plc:xxxxx",
      "suggestedAt": "2024-11-28T12:00:00.000Z"
    }
  ]
}
```

### Fields

- **did** (string, required): The DID (Decentralized Identifier) of the user who recommended the item
- **suggestedAt** (string, required): ISO 8601 timestamp when the recommendation was made

### Multiple Recommenders

Items can have multiple recommenders. Each recommendation is tracked separately with its own timestamp, allowing users to see when different people suggested the same item.

## API Usage

### Adding an Item with Recommendations

When creating a list item via `POST /collections/:listUri/items`, include the `recommendedBy` field:

```json
{
  "title": "The Hitchhiker's Guide to the Galaxy",
  "creator": "Douglas Adams",
  "mediaType": "book",
  "status": "want",
  "recommendedBy": "brittanyellich.com"
}
```

The `recommendedBy` field accepts:
- A single DID: `"did:plc:xxxxx"`
- A single handle: `"brittanyellich.com"` (automatically resolved to DID)
- An array of DIDs: `["did:plc:xxxxx", "did:plc:yyyyy"]`
- An array of handles: `["brittanyellich.com", "friend.bsky.social"]`

### Handle Resolution

When a handle is provided instead of a DID, the API automatically resolves it using the AT Protocol's `resolveHandle` method. If resolution fails, the original value is logged as a warning but still stored.

### Response Format

When fetching list items via `GET /collections/:listUri/items`, the response includes:

```json
{
  "items": [
    {
      "uri": "at://did:plc:xxxxx/app.collectivesocial.listitem/xxx",
      "title": "The Hitchhiker's Guide to the Galaxy",
      "creator": "Douglas Adams",
      "recommendations": [
        {
          "did": "did:plc:4hodhjl2kposuchzvpiviwps",
          "suggestedAt": "2024-11-28T12:00:00.000Z"
        }
      ],
      "createdAt": "2024-11-28T12:00:00.000Z"
    }
  ]
}
```

## Frontend Display

### Showing Recommendations

In the collection details view, recommendations are displayed below the item details with:
- A 💡 emoji to indicate recommendations
- Truncated DID (first 24 characters) as a clickable link to PDSLS
- Formatted date when the recommendation was made

Example display:
```
💡 Recommended by:
  did:plc:4hodhjl2kposuchwvp... (11/28/2024)
```

### Adding Recommendations

When adding items through the UI, users can optionally specify who recommended the item by entering:
- A DID (e.g., `did:plc:xxxxx`)
- A handle (e.g., `brittanyellich.com` or `friend.bsky.social`)

The field includes a helpful placeholder and description to guide users.

## Use Cases

### Personal Reading Lists

Track books friends have recommended so you can:
- Remember who suggested what
- Reach out to discuss after reading
- Build reading accountability with friends
- Discover patterns in whose recommendations you enjoy

### Book Clubs

- Record when members suggest books
- Track suggestion history for rotation planning
- Acknowledge contributors in discussions

### Gift Ideas

- Remember who mentioned wanting to read something
- Track recommendations for others in your network

## Implementation Details

### Backend Changes

1. **Lexicon Update** (`lexicons/app.collectivesocial.listitem.json`):
   - Added `recommendations` array property
   - Defined recommendation object schema with `did` and `suggestedAt`

2. **Type Definitions** (`src/types/lexicon.ts`):
   - Added `Recommendation` interface
   - Updated `AppCollectiveSocialListitem.Record` to include optional `recommendations` array

3. **API Route** (`src/routes/collections.ts`):
   - Accept `recommendedBy` in POST request body
   - Handle single or multiple DIDs/handles
   - Resolve handles to DIDs using AT Protocol
   - Include recommendations in GET responses

### Frontend Changes

1. **CollectionDetailsPage** (`src/pages/CollectionDetailsPage.tsx`):
   - Added `Recommendation` interface
   - Updated `ListItem` interface to include `recommendations`
   - Added `recommendedBy` field to review data state
   - Display recommendations with clickable PDSLS links
   - Added input field for entering recommender DID/handle

## Privacy Considerations

- Recommendations are stored in the user's Personal Data Server (PDS)
- Public collections may expose recommendation data to anyone viewing the collection
- DIDs are displayed but can be linked to PDSLS for profile information
- Consider collection visibility settings when adding sensitive recommendations

## Future Enhancements

Potential improvements to the recommendation system:
- Fetch and display recommender names/handles from their profiles
- Add notes or context to recommendations ("said it's similar to X")
- Track recommendation success rate (did you enjoy items from this person?)
- Enable notifications when recommended items are marked complete
- Support removing individual recommenders from an item
- Add bulk operations for managing recommendations
