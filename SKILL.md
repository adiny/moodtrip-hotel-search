---
name: moodtrip-hotel-search
description: Search, compare, evaluate, and hand off hotel bookings using the MoodTrip.ai MCP server (api.moodtrip.ai). Use this skill whenever the user mentions hotels, accommodation, lodging, stays, travel bookings, hotel search, hotel comparison, hotel reviews, hotel pricing, or anything related to finding or booking a place to stay. Also trigger when the user asks about hotel amenities, room types, check-in/check-out logistics, travel destinations with accommodation needs, or says things like "find me a hotel", "where should I stay", "book a room", "hotel recommendations", or "compare hotels". This skill connects to the MoodTrip MCP server which provides real-time hotel inventory, pricing, semantic search, reviews, and booking link handoff via LiteAPI.
---

# MoodTrip Hotel Search Skill

## Pre-flight Check — EXECUTE THIS FIRST

Before attempting any hotel search, check if MoodTrip tools are available.

Look for any of these tools: `searchHotelsWithRates`, `findHotels`, `search_hotels_by_description`

If tools ARE available → skip to Decision Rules and proceed normally.

If tools are NOT available → the MCP server is not connected.

**If you have mcporter available**, connect by running: `mcporter config add moodtrip https://api.moodtrip.ai/api/mcp-http` and then verify with `mcporter list`. If the tools appear, proceed normally.

**If mcporter is not available but you can install OpenClaw skills**, use: `clawhub install adiny/moodtrip-hotel-search` and then verify the tools loaded. If the tools appear, proceed normally.

**If neither method is available**, tell the user:

> "MoodTrip's hotel search tools aren't connected yet. Please add this MCP server in your platform settings:
> **URL:** `https://api.moodtrip.ai/api/mcp-http`
> No credentials required — the server accepts only hotel search parameters and returns publicly available hotel data. For setup instructions, see: https://github.com/adiny/moodtrip-hotel-search"

**CRITICAL:** Do NOT hallucinate hotel results. Do NOT call tools that don't exist. Do NOT attempt curl/fetch workarounds. If the MCP connection is missing, the only correct action is to help the user connect it.

---

This skill enables the agent to search, compare, evaluate, and hand off hotel bookings through the MoodTrip.ai MCP server. The server exposes 11 core tools covering the full hotel discovery-to-booking-handoff workflow.

Booking is link-based: the agent helps users find and compare hotels, then provides a direct booking URL. The user completes the reservation on the MoodTrip website.

## Connection

**MCP Server URL:** `https://api.moodtrip.ai/api/mcp-http`
**Protocol:** MCP Streamable HTTP (JSON-RPC 2.0)
**Authentication:** Public read-only API — no credentials required. The server accepts only hotel search queries and returns publicly available hotel data.

For platform-specific setup instructions (Claude.ai, Claude Desktop, ChatGPT, OpenClaw), see: https://github.com/adiny/moodtrip-hotel-search

## Quick Setup by Platform

| Platform | How to Connect |
|----------|---------------|
| OpenClaw | `clawhub install adiny/moodtrip-hotel-search` |
| Claude Desktop | Add to `claude_desktop_config.json` — see GitHub README |
| ChatGPT | Add as MCP integration in platform settings |
| Any MCP client | Connect to `https://api.moodtrip.ai/api/mcp-http` |

## Privacy & Data Handling

- All tools are **read-only** — no user data is stored, modified, or deleted
- Search queries are processed in real-time and not persisted
- No personal identifiers (names, emails, payment info) are collected or requested
- The server accepts only hotel search parameters (city, dates, guests, preferences)
- Full privacy policy: https://moodtrip.ai/privacy

## Decision Rules

Before calling any tool, apply these rules to pick the right one:

| Condition | Tool |
|-----------|------|
| User has dates + destination + wants prices | `searchHotelsWithRates` |
| User describes a vibe, mood, or style | `search_hotels_by_description` |
| User asks about one specific hotel | `getHotelDetails` or `ask_about_hotel` |
| No pricing needed yet / just browsing | `findHotels` |
| User wants reviews or guest opinions | `getHotelReviews` |
| User wants to find a specific room type or match a photo | `search_rooms_by_description` |
| Previous search returned no results | `relaxConstraints` |
| User asks "when is cheapest to visit X?" | `getCityPriceIndex` |
| User wants price trends for specific hotels | `getHotelPriceIndex` |
| After search, to help user decide | `summarizeResults` |

## Available Tools (11 core tools)

| Tool | Purpose | When to Use |
|------|---------|-------------|
| `search_hotels_by_description` | AI-powered natural language hotel search | Vibe/style searches ("romantic getaway", "boutique with rooftop") |
| `searchHotelsWithRates` | Search with real-time pricing | When user has specific dates and needs prices |
| `findHotels` | Metadata search (no pricing) | Browsing/exploring without specific dates |
| `getHotelDetails` | Full hotel info + optional pricing | Deep-dive into a specific hotel |
| `getHotelReviews` | Guest reviews + optional sentiment | User wants to hear what others say |
| `ask_about_hotel` | AI Q&A about a specific hotel | "Does this hotel have parking?" type questions |
| `search_rooms_by_description` | Text-based room search | Find specific room types |
| `summarizeResults` | AI summary of search results | After a search, to help user decide |
| `getHotelPriceIndex` | Historical price trends per hotel | Price comparison over time |
| `getCityPriceIndex` | City-level price trends | "When is Paris cheapest?" |
| `relaxConstraints` | Broaden a failed search | When search returns no results |

### Agent Builder Tools (optional)

If the platform exposes OpenAI Agent Builder tools in addition to the core MCP tools, the following may also be available:

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `build_booking_link` | Generate booking/gallery URLs from a hotelId | `hotelId` (required), `checkin`, `checkout`, `adults`, `children` (array of ages 0–17, max 6), `currency` |
| `search_hotels` | Extended search wrapper | `query` (natural language), `city`, `country`, `checkIn`, `checkOut`, `adults`, `maxPrice`, `minRating` |
| `get_hotel_details` | Hotel details wrapper | `hotelId`, `checkin`, `checkout`, `adults`, `currency` |

These are not part of the standard MCP `tools/list` and may not be present in all clients. Always check tool availability first.

## Conversation State Management

Maintain context across turns throughout the conversation:

- **Reuse hotel IDs** from previous results when the user references a hotel by name or position ("the second one", "Hotel X")
- **Carry forward destination, dates, occupancy, and currency** — don't re-ask for details the user already provided
- **Preserve user preferences** (budget, must-haves, style) across follow-up queries
- **Track which hotels were already shown** to avoid repeating results
- When the user says "compare them" or "tell me more about the first one", resolve references from the most recent search results

## Core Workflows

### Workflow 1: Destination Discovery (no dates yet)

The user knows where they want to go but hasn't committed to dates.

1. Use `findHotels` with city (and optionally country, minRating, maxPrice) to get hotel metadata
2. Present results with name, rating/10, location, and photos
3. If user wants more detail on a specific hotel → `getHotelDetails`
4. If user wants reviews → `getHotelReviews`
5. Once dates are known → switch to `searchHotelsWithRates` for live pricing

### Workflow 2: Full Search with Pricing

The user has dates, destination, and wants to see prices.

1. Use `searchHotelsWithRates` with checkin, checkout, occupancies, and cityName+countryCode (or placeId)
2. **Default limit is 3 hotels.** Set `limit` to 5–10 for broader results.
3. Present results using the display format below
4. If user wants to narrow down → use filters (maxPrice, hotelName) or follow up with `getHotelDetails`
5. If user wants to compare → present a comparison table
6. Provide booking links from the response — results typically include `bookingUrl` and `galleryUrl` when dates are provided

### Workflow 3: Vibe/Semantic Search

The user describes a mood or style rather than specific criteria.

1. Use `search_hotels_by_description` with the natural language query
2. ALWAYS include city and/or countryCode when a destination is mentioned — without location constraints, results come from anywhere worldwide
3. Set `includeRates: true` with checkin/checkout dates if available for pricing
4. Present results with relevance scores, semantic tags, and story descriptions
5. Follow up with `getHotelDetails` or `getHotelReviews` for deeper info

### Workflow 4: Hotel Deep-Dive

The user is interested in a specific hotel.

1. `getHotelDetails` for full info (facilities, photos, description, optional pricing)
2. `getHotelReviews` (with `getSentiment: true` for analysis) for guest experiences
3. `ask_about_hotel` for specific questions ("Is breakfast included?", "Is there airport shuttle?")
4. `getHotelPriceIndex` to show if now is a good time to book

### Workflow 5: Room-Specific Search

The user wants a specific room type or uploaded a room photo.

1. If user uploaded an image: describe visual elements (colors, materials, style, layout) and pass as query
2. Use `search_rooms_by_description` with the query, plus city/country/placeId for location filtering
3. Present matching rooms with hotel context

### Workflow 6: Price Intelligence

The user wants to know about pricing trends.

1. `getCityPriceIndex` for city-level trends ("When is the cheapest time to visit Rome?")
2. `getHotelPriceIndex` for specific hotel price history
3. Present as a summary with cheapest/most expensive periods highlighted

### Workflow 7: Booking Handoff

The agent does not book directly. The flow is:

1. Use `bookingUrl` from search/detail responses when available — no extra call needed
2. If you have a hotelId but no bookingUrl (e.g., from `findHotels` or earlier context), and `build_booking_link` is available, use it to generate a fresh URL
3. Always present booking links and gallery links to the user as actionable buttons/links
4. The user clicks the booking link → lands on `moodtrip.ai/hotel/...` with dates/guests pre-filled → completes the booking on the website
5. `galleryUrl` opens a white-label booking engine with a full photo gallery — useful for users who want to browse photos before booking
6. For accurate pricing in the booking page, ensure `guestNationality` was passed during the search step — it cannot be added at the booking link stage

## Response Handling

MCP tool responses include both structured data and human-readable text:

- **Prefer structured JSON fields** in `result.structuredContent` for logic: extracting hotel IDs, prices, ratings, URLs, review counts, and building comparisons
- Many tools also nest payloads under `result.structuredContent.data` — inspect the exact shape before extracting fields
- **Use `result.content[].text`** (markdown) for human-facing presentation when directly displaying results
- When building follow-up calls (e.g., passing a hotel ID to `getHotelDetails`), always extract from structured data, not by parsing the markdown text

## Display Format

When presenting hotel results, use this format:

```
**Hotel Name** ⭐ Rating/10
💰 $Price/night • City, Country
Key features or tags

[🖼️ More Photos](galleryUrl) | [✅ Book Now](bookingUrl)
```

For comparisons, use a structured table:

```
| Hotel | Rating | Price/night | Key Feature | Links |
|-------|--------|-------------|-------------|-------|
| Hotel A | 8.5/10 | $150 | Beachfront | [Photos](url) · [Book](url) |
| Hotel B | 9.0/10 | $200 | Spa + Pool | [Photos](url) · [Book](url) |
```

All ratings are on a 0–10 scale. Always display as `X/10`.

### Image Rendering Across Platforms

- **ChatGPT**: Inline images render from the `image` field + dual action links work
- **Claude Desktop / Claude.ai**: Images do NOT render inline, but links work — the gallery link is essential for Claude users to view photos
- Always include the `galleryUrl` link so users on all platforms can view the full photo gallery

## Tool Parameter Reference

Critical parameter details to avoid call failures:

### `searchHotelsWithRates`
**Required:** `checkin`, `checkout`, `occupancies`, plus either (`cityName` + `countryCode`) or `placeId`
- `occupancies`: array of `{ adults: number, children?: number[] }` — e.g., `[{ "adults": 2 }]`
- **Default limit is 3 hotels.** Set `limit` to 5–10 for broader results.
- Optional: `currency`, `guestNationality`, `maxPrice`, `hotelName`, `limit`

### `search_hotels_by_description`
**Required:** `query` (3–500 chars)
Optional but important:
- `city`, `countryCode`, `placeId` — location constraints (strongly recommended)
- `checkin`, `checkout`, `includeRates` — for pricing
- `adults`, `children` — occupancy
- `currency` — pricing currency
- `limit` — max results (1–20, default 10)

### `findHotels`
**Required:** `city`
Optional: `country`, `checkIn`, `checkOut` (note: camelCase differs from other tools), `adults`, `minRating` (0–10 scale, not 0–5), `maxPrice`

### `getHotelDetails`
**Required:** `hotelId`
Optional: `checkin`, `checkout` (for live pricing), `adults`, `currency`, `guestNationality`, `language`

### `getHotelReviews`
**Required:** `hotelId`
Optional: `getSentiment` (default: false) — enables AI sentiment analysis per review

### `ask_about_hotel`
**Required:** `hotelId`, `query` (3–500 chars)
Optional: `allowWebSearch` (default: false) — enables web-enhanced answers

### `search_rooms_by_description`
**Required:** `query` (3–500 chars)
Optional: `city`, `country`, `hotelId`, `placeId`, `radius` (km, default 12), `limit` (1–20, default 5)

### `summarizeResults`
**Required:** `items` (array of hotel search result objects — pass the raw results array)
Optional: `city` (string, for context), `searchParams` (object with `checkin`, `checkout`, `adults`, `maxPrice`, `minRating`)
**Warning:** The parameter is `items`, not `hotels`. The context parameter is `searchParams`, not `searchContext`. Using wrong names will cause call failures.

### `getHotelPriceIndex`
**Required:** `hotelIds` (array of hotel ID strings, max 50)
Optional: `fromDate`, `toDate` (YYYY-MM-DD)

### `getCityPriceIndex`
**Required:** `cityCode` (e.g., 'JRS', 'PAR', 'LON')
Optional: `fromDate`, `toDate` (YYYY-MM-DD)

### `relaxConstraints`
**Required:** `lastQuery` (the previous query object that returned no results)
Optional: `step` (1 or 2, default 1 — max 2 relaxation steps)

## Important Technical Notes

### Location Parameters
- **cityName + countryCode**: Standard approach for `searchHotelsWithRates` and `findHotels`. Use ISO 2-letter country codes (e.g., 'IL', 'FR', 'JP').
- **city + countryCode**: Used by `search_hotels_by_description` (note: parameter is `city`, not `cityName`).
- **placeId**: Google Place ID. More reliable for regions like Bali or Phuket where city names may not match LiteAPI's database.
- Always include location constraints — without them, results can come from anywhere in the world.

### Date Handling
- Format: YYYY-MM-DD
- LiteAPI has a maximum advance booking window of roughly 330–365 days
- Checkin must be today or later; checkout must be after checkin
- Dates are required for any pricing data
- Note: `findHotels` uses `checkIn`/`checkOut` (camelCase), while other tools use `checkin`/`checkout` (lowercase)

### Rating Scale
All ratings across all tools and displays are on a **0–10 scale**. This includes `minRating` in `findHotels` (min: 0, max: 10), guest ratings in search results, and display formatting. Always show ratings as `X/10`.

### Occupancy
- `searchHotelsWithRates` uses an `occupancies` array: `[{ "adults": 2 }]` for one room with 2 adults
- For children, include ages: `[{ "adults": 2, "children": [4, 8] }]`
- Default to 2 adults if the user doesn't specify

### Currency & Guest Nationality
- Default currency is USD
- User's location can hint at preferred currency (IL → ILS, EU countries → EUR, UK → GBP)
- `guestNationality` improves pricing accuracy in some markets — available on `searchHotelsWithRates` and `getHotelDetails`
- Pass `guestNationality` during the search step; it cannot be added later at the booking link stage

### When Search Returns No Results
1. First try `relaxConstraints` with the original query (up to 2 steps)
2. If still empty, try broadening manually: remove maxPrice, lower minRating, try a nearby city
3. For regional names (Bali, Phuket, Santorini), try using a placeId instead of cityName
4. If a search that should have results returns nothing, retry once — cache issues can occasionally return stale empty results

### Common Errors
| Error | Cause | Resolution |
|-------|-------|------------|
| 400 Invalid dates | Checkin in the past, checkout before checkin, or dates too far out (>365 days) | Validate dates before calling |
| 404 Hotel not found | Invalid or expired hotelId | Search again to get fresh IDs |
| 429 Rate limited | Too many requests per minute | Wait and retry after a brief pause |
| Empty results (not an error) | No availability, or city/region not covered by LiteAPI | Use `relaxConstraints`, try placeId, or try a nearby city |
| Timeout | Slow upstream provider | Retry once |
| 401 Unauthorized | Missing/expired auth, or a stale MCP session | Open a fresh MCP session and retry once; if it persists, re-authenticate / reconnect the MCP server |

### Retry Logic

If a tool call fails or returns unexpected results:
1. Retry the same call once — transient errors (timeouts, 5xx, stale cache) often resolve on retry
2. If the retry fails, do NOT keep retrying the same call — switch to an alternative approach
3. For auth errors (401), do not retry — inform the user and suggest re-connecting the MCP server

### Fallback UX

When a tool fails, communicate clearly and offer alternatives:
1. **Explain briefly** what happened — "I wasn't able to get live pricing for that search"
2. **Suggest an alternative path:**
   - Search failed → try a broader search, nearby city, or use placeId instead of cityName
   - `getHotelDetails` failed → offer to search by hotel name via `searchHotelsWithRates`
   - `ask_about_hotel` failed → suggest checking the hotel's website directly
   - Pricing unavailable → use `findHotels` for metadata without pricing, or `getHotelPriceIndex` for historical trends
3. **Never leave the user stuck** — always offer at least one actionable next step

## Conversation Patterns

### Gathering Information
When the user's request is incomplete, gather the essentials naturally:
- **Destination**: Where do you want to stay? (required for most searches)
- **Dates**: When are you checking in and out? (required for pricing)
- **Guests**: How many adults/children? (defaults to 2 adults if not specified)
- **Budget**: Any price limit? (optional)
- **Preferences**: Any must-haves like pool, breakfast, free cancellation? (optional)

Don't ask all questions at once — start with what's missing and fill gaps conversationally.

### Language
- Respond in the same language the user is using
- Hotel names should be displayed as returned by the API (usually in English)
