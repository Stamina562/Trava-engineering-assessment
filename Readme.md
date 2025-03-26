# Trava Engineering Assessment – URL Shortening for Filtered Table Views

Hi Trava team — thanks for the great prompt. I appreciated how open-ended this challenge was and had fun thinking through it. Below is a practical solution that I believe balances usability, technical trade-offs, and long-term maintainability. This is how I'd approach solving the problem in a real product environment.

---

## The Request

> “The table URLs are super useful but also super long. Can we shorten them?”

Totally makes sense. The table URL encodes a lot of filtering and sorting logic, which is great for deep-linking, but as more filters are added, the URL gets pretty unwieldy.

We want to:

- Preserve the current functionality (filters, sorts, search, deep-linking)
- Make the URL significantly shorter and cleaner
- Avoid large infrastructure changes or over-engineering

---

## Goals

- Preserve full filter/sort/search state
- Allow users to share a link that reliably restores the same table view
- Make URLs shorter and easier to share
- Keep the solution lightweight and easy to maintain

---

## High-Level Approach

There are a couple of valid ways to tackle this:

### Option 1: Client-Side Compression

Compress the entire table state into a single string using a URL-safe compression library like [`lz-string`](https://www.npmjs.com/package/lz-string). Instead of this: /users?first_name=John&first_name_op=eq&age=30&age_op=gte&sort=age&sort_dir=asc

We’d get: /users?state=N4IgzgTgpg...


The compressed string contains all the filter/sort logic, just packed into one query parameter.

### Option 2: Backend-Persisted Views

Store the full state server-side and return a short ID (e.g., `abc123`). The frontend URL becomes: /users/view/abc123


This is ideal for saved views or sharing across teams, but requires API/storage support.

---

## My Recommended Approach: Hybrid

Start with client-side compression (fast, no infra), and support backend-stored views as an optional fallback for large states or saved views.

This covers 90% of use cases without requiring a new backend service, but still allows us to scale up the feature later.

---

## Pseudocode – Frontend State Compression

```js
import {
  compressToEncodedURIComponent,
  decompressFromEncodedURIComponent
} from 'lz-string';

function saveStateToURL(state) {
  const compressed = compressToEncodedURIComponent(JSON.stringify(state));
  const url = new URL(window.location);
  url.searchParams.set('state', compressed);
  history.replaceState(null, '', url.toString());
}

function readStateFromURL() {
  const params = new URLSearchParams(window.location.search);
  const compressed = params.get('state');
  if (!compressed) return null;
  const decompressed = decompressFromEncodedURIComponent(compressed);
  return JSON.parse(decompressed);
}

Backend Option – API View Storage (Optional)
If/when we need to support persistent saved views or large filter states:

API Example
POST /api/view/save
Returns: { "id": "abc123" }

GET /api/view/:id
Returns the full filter/sort state

Stored View Format

{
  "id": "abc123",
  "user_id": "optional",
  "created_at": "2025-03-25T10:00:00Z",
  "state": {
    "filters": [...],
    "sort": {...},
    "search": "..."
  }
}

## Pros and Cons

| Approach               | Pros                                               | Cons                                                   |
|------------------------|----------------------------------------------------|--------------------------------------------------------|
| Client-side compression | Lightweight, fast, no backend changes              | Still exposes full state (compressed) in the URL       |
| Backend storage        | Cleanest URLs, great for saved views               | Requires backend API and persistence layer             |
| Hybrid (recommended)   | Flexible, scalable, minimal infrastructure needed  | Slightly more complexity in implementation             |


Testing Strategy
Unit Tests
Validate compression/decompression

Handle edge cases (empty filters, malformed state, max length)

Integration Tests
Sharing a compressed URL reproduces the correct view

Legacy query param support still works

Monitoring Ideas
Log errors when parsing ?state=

Track size distribution of states for future backend cutoff

Track usage of shared/saved links

Development Plan
Day 1
Respond to CEO:

“Yes — we can compress the state into one query parameter using a safe encoding format. This should make the URLs much shorter without losing functionality. I’ll have a working version ready shortly.”

Day 2–3
Implement compression + URL update

Parse legacy URLs for backward compatibility

Add metrics + error logging

Day 4+
(Optional) Add backend endpoint for saved views

Support named views (e.g., “All Trial Users Over 30”)

Add a "Copy short link" UI element

Future Iterations
Shareable named views across the org

Expiring saved states

History of recent views

Support for exporting/importing view configs

Final Thoughts
This kind of polish makes a big UX difference, and we can deliver most of the value with minimal overhead. I’d start with the compressed URL approach and layer in backend support if usage or needs grow.

Happy to dive deeper or prototype anything discussed here.

– Ryan


---

