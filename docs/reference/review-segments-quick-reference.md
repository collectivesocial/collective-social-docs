# Review Segments - Quick Reference

## When Review Segments Show Up

Review segments **only** display when:
1. ✅ Item status is `in-progress`
2. ✅ User is the owner of the item
3. ✅ Item has either:
   - A length value (pages, episodes, minutes, etc.), OR
   - At least one existing review segment

## Component Usage

### For Editing (Owner View)
```tsx
import { ReviewSegments } from '../components/ReviewSegments';

<ReviewSegments
  listItemUri={item.uri}
  mediaItemId={item.mediaItemId}
  mediaType={item.mediaType}
  itemLength={item.mediaItem?.length || null}
  apiUrl={apiUrl}
/>
```

### For Viewing (Public View)
```tsx
import { ReviewSegmentsViewer } from '../components/ReviewSegmentsViewer';

<ReviewSegmentsViewer
  userDid={userDid}
  mediaItemId={mediaItemId}
  mediaType={mediaType}
  itemLength={itemLength}
  apiUrl={apiUrl}
/>
```

## API Endpoints Quick Reference

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/reviewsegments` | Create new segment |
| PUT | `/reviewsegments/:uri` | Update segment |
| DELETE | `/reviewsegments/:uri` | Delete segment |
| GET | `/reviewsegments/media/:mediaItemId` | Get user's segments for item |
| GET | `/reviewsegments/list/:listItemUri` | Get user's segments for list item |
| GET | `/reviewsegments/user/:did?mediaItemId=X` | Get any user's segments |

## Length Units by Media Type

| Media Type | Length Unit |
|------------|------------|
| book | pages |
| movie | minutes |
| tv | episodes |
| podcast | minutes |
| course | modules |
| other | units |

## Required vs Optional Fields

### Required
- `text` - The actual review segment content
- `percentage` - Number between 0-100

### Optional
- `title` - Short title/label for the segment
- `mediaItemId` - Database ID of the media item
- `mediaType` - Type of media (book, movie, etc.)
- `listItem` - URI of the list item this belongs to

## Progress Calculation Examples

### Percentage Input
User enters: `45%`
Stored as: `45`

### Length-based Input (Book with 300 pages)
User enters: `150` pages
Calculated as: `Math.round((150 / 300) * 100)` = `50%`
Stored as: `50`

### Length-based Input (TV show with 24 episodes)
User enters: `12` episodes
Calculated as: `Math.round((12 / 24) * 100)` = `50%`
Stored as: `50`

## UI States

### Empty State (No Segments)
```
Progress
[+]

No review segments yet. Click the + button to add your first one!
```

### With Segments
```
Progress                   [+]

Overall Progress                              35%
████████████░░░░░░░░░░░░░░░░░░░░░░░░
~105 of 300 pages

┌─────────────────────────────────────┐
│ Chapter 5 thoughts            [✏️] [🗑️] │
│ At 35% (~105 pages)                  │
│ The plot is really picking up here!  │
│ Jan 15, 2024                         │
└─────────────────────────────────────┘
```

### Add/Edit Form
```
Title (optional)
[Chapter 7 reflections          ]

Your thoughts
[The character development...    ]
[                                ]

Progress
[By pages] [By Percentage]

[150] of 300 pages (~50%)

[Cancel] [Add Segment]
```

## Common Patterns

### Check if segments should show
```typescript
if (item.status === 'in-progress' && isOwner) {
  // Show ReviewSegments component
}
```

### Get highest progress
```typescript
const highestPercentage = segments.length > 0
  ? Math.max(...segments.map(s => s.value.percentage))
  : 0;
```

### Calculate progress from length
```typescript
const percentage = itemLength
  ? Math.round((currentProgress / itemLength) * 100)
  : currentProgress;
```

### Sort segments chronologically
```typescript
segments.sort((a, b) => a.value.percentage - b.value.percentage);
```

## Error Handling

### Validation Errors
- Empty text → "Text is required"
- Missing percentage → "Percentage is required"
- Percentage < 0 or > 100 → "Percentage must be between 0 and 100"

### Authorization Errors
- Not authenticated → 401 "Not authenticated"
- Not owner → 403 "You can only update your own review segments"

### Not Found Errors
- Invalid URI → 404 "Review segment not found"

## Testing Commands

```bash
# Create a segment
curl -X POST http://localhost:3000/reviewsegments \
  -H "Content-Type: application/json" \
  --cookie "session=..." \
  -d '{
    "text": "Great chapter!",
    "percentage": 25,
    "title": "Chapter 5",
    "mediaItemId": 123,
    "mediaType": "book",
    "listItem": "at://did:plc:xxx/..."
  }'

# Get segments for a list item
curl http://localhost:3000/reviewsegments/list/at%3A%2F%2F... \
  --cookie "session=..."

# Update a segment
curl -X PUT http://localhost:3000/reviewsegments/at%3A%2F%2F... \
  -H "Content-Type: application/json" \
  --cookie "session=..." \
  -d '{"text": "Updated thoughts!", "percentage": 30}'

# Delete a segment
curl -X DELETE http://localhost:3000/reviewsegments/at%3A%2F%2F... \
  --cookie "session=..."
```

## Keyboard Shortcuts (Future)

Currently no keyboard shortcuts, but could add:
- `n` - New segment
- `e` - Edit focused segment
- `Delete` - Delete focused segment
- `Esc` - Cancel add/edit mode
- `Enter` (in text field) - Save segment

## Accessibility

- ✅ Progress bar has proper ARIA attributes
- ✅ Buttons have aria-labels
- ✅ Forms use proper Field components with labels
- ✅ Edit/delete actions use icon buttons with labels
- ✅ All interactive elements keyboard accessible

## Browser Compatibility

Works in all modern browsers:
- Chrome/Edge 90+
- Firefox 88+
- Safari 14+

Uses standard Web APIs:
- Fetch API for HTTP requests
- localStorage for any caching (future)
- No experimental features

## Performance Notes

- Segments loaded once on component mount
- No automatic polling/refetching
- Manual refresh by unmounting/remounting component
- Consider adding refresh button for long-lived pages
- API responses are small (segments are text-only)

## Security

- All endpoints require authentication
- Ownership verified before update/delete
- No XSS vulnerabilities (React escapes by default)
- CORS properly configured on backend
- Session cookies are httpOnly and secure

## Troubleshooting

### Segments not showing up
- Check item status is `in-progress`
- Verify you are the owner (`isOwner` prop)
- Check browser console for API errors
- Verify authentication (session cookie present)

### Progress bar at 0%
- No segments created yet
- All segments have percentage: 0
- Check API response has segments array

### Can't save segment
- Check text field is not empty
- Verify percentage is 0-100
- Check network tab for API errors
- Verify you're authenticated

### Length-based input not working
- Verify `itemLength` prop is not null
- Check media type has valid length unit
- Ensure length value is positive number
