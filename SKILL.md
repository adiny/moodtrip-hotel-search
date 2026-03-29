---
name: moodtrip-hotel-search
description: Search, compare, evaluate, and hand off hotel bookings using the MoodTrip.ai MCP server (api.moodtrip.ai). Use this skill whenever the user mentions hotels, accommodation, lodging, stays, travel bookings, hotel search, hotel comparison, hotel reviews, hotel pricing, or anything related to finding or booking a place to stay. Also trigger when the user asks about hotel amenities, room types, check-in/check-out logistics, travel destinations with accommodation needs, or says things like "find me a hotel", "where should I stay", "book a room", "hotel recommendations", or "compare hotels". This skill connects to the MoodTrip MCP server which provides real-time hotel inventory, pricing, semantic search, reviews, and booking link handoff via LiteAPI.
---

# MoodTrip Hotel Search Skill

## Pre-flight Check — EXECUTE THIS FIRST

Before attempting any hotel search, verify MoodTrip tools are accessible.

**Step 1: Check for tools directly.**
Look for any of these tools in your available tool list: `searchHotelsWithRates`, `findHotels`, `search_hotels_by_description`, `search_hotels`, `get_hotel_details`

If ANY of these tools are found → skip to Decision Rules and proceed normally.

**Step 2: Check if MCP server is already configured.**
If tools are not in your direct tool list, check whether `api.moodtrip.ai` or `moodtrip` is already configured as an MCP server anywhere in your platform config — including under any of these paths:
- `mcp.servers.moodtrip`
- `plugins.entries.acpx.config.mcpServers.moodtrip`
- `skills.entries.moodtrip-hotel-search`
- `skills/moodtrip-hotel-search/_meta.json` → `mcpServers.moodtrip`
- `skills/moodtrip-hotel-search/.mcp.json`

This skill ships with `.mcp.json` and `_meta.json` files that declare the MCP server configuration. If your platform reads these files from skill folders, the server should be auto-configured on install.

If these files are missing from the skill folder, this is an older version. Suggest the user update: `clawhub update adiny/moodtrip-hotel-search` or `openclaw skills update`

If the server IS configured but tools are not showing, this is likely a **tool loading/exposure issue in the current session**, not a missing connection. Restart the session or gateway if appropriate — some platforms only load MCP tools at session start.

**Step 3: Auto-connect on OpenClaw when possible.**
If the server is NOT configured and you are running inside OpenClaw with config-edit capability:

- Prefer adding the MCP through OpenClaw-managed config:
  - `mcp.servers.moodtrip.url = "https://api.moodtrip.ai/api/mcp-http"`
- If you have an OpenClaw MCP management command/tool available, use that first.
- If you have OpenClaw config tools available, patch config so the MCP server is added automatically.
- After adding it, restart/reload the gateway or start a fresh session if needed.
- Then re-check whether the MoodTrip tools are exposed.

OpenClaw-specific rule:
- If the user asked to connect/install/fix MoodTrip MCP, and you can safely edit OpenClaw config, **do it automatically** instead of just telling the user to do it manually.
- Prefer `mcp.servers` over custom plugin-specific hacks.
- Do not invent tool availability; verify after restart/session refresh.

**Step 4: Fallback if auto-connect is not possible.**
Only if auto-connect is not possible:
- **mcporter available?** Run: `mcporter config add moodtrip https://api.moodtrip.ai/api/mcp-http`
- **clawhub available?** Run: `clawhub install adiny/moodtrip-hotel-search`
- **Neither available?** Tell the user:
  > "MoodTrip's hotel search tools aren't connected yet. Please add this MCP server in your platform settings:
  > **URL:** `https://api.moodtrip.ai/api/mcp-http`
  > No credentials required. For setup: https://github.com/adiny/moodtrip-hotel-search"

**CRITICAL:** Do NOT hallucinate hotel results. Do NOT call tools that don't exist. Do NOT attempt curl/fetch workarounds. If tools cannot be verified after all steps, help the user connect or restart the session.

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
| OpenClaw | Prefer automatic config via `mcp.servers`; otherwise `clawhub install adiny/moodtrip-hotel-search` |
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
