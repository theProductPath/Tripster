# Tripster — Implementation Plan

## Overview
Build a multi-agent travel itinerary planner using Claude Code sub-agents. A **Coordinator (Trip Planner)** orchestrates specialist sub-agents that each research one category of places, then assembles their findings into a neighborhood-aware, day-by-day itinerary.

> **Current state (July 2026):** The system ships with **8 specialist agents** (coffee, restaurant, vintage, menswear, bookstore, lifestyle, markets, parks) plus the Geo Optimizer, dispatched **sequentially** (see "Dispatch Strategy" below). Navigation uses **Google Maps "Maps & Photos" links** (name+address based) rather than embedded venue photos, and `lat`/`lng` are best-effort hints only. Phase 1 below documents the original 3-agent foundation for history; Phases 2–3a reflect what was actually shipped.

## Architecture (original design, from Notion doc)

```
User Request (city, dates, pace)
        │
        ▼
┌─────────────────────┐
│  Preference Loader   │  ← Reads from Notion pages
│  (Context Packet)    │
└────────┬────────────┘
         │ context packet
         ▼
┌─────────────────────┐
│    Coordinator       │  ← The Trip Planner
│  (Trip Planner)      │
└──┬──────┬──────┬────┘
   │      │      │        dispatch in parallel
   ▼      ▼      ▼
┌─────┐┌─────┐┌─────┐
│Coffee││Shops││ Geo │    ← Specialist sub-agents
│Agent ││Agent││Agent│
└──┬──┘└──┬──┘└──┬──┘
   │      │      │        structured JSON results
   └──────┼──────┘
          ▼
┌─────────────────────┐
│    Coordinator       │  ← Merges, clusters, sequences
│  assembles itinerary │
└─────────────────────┘
          │
          ▼
   Final Itinerary
   (day-by-day, by neighborhood)
```

## Tech Stack
- **Runtime**: Claude Code sub-agents (`.claude/agents/*.md`)
- **Data sources**: Claude web search + structured APIs (combination approach)
- **Preferences**: Pulled from Notion pages at runtime via MCP
- **Output**: Structured JSON → formatted itinerary (Notion page or local file)

## Phase 1 — Foundation (what we build now)

### Step 1: Project scaffolding
Create the project structure:
```
Tripster/
├── .claude/
│   └── agents/
│       ├── coffee-finder.md
│       ├── shops-finder.md
│       └── geo-optimizer.md
├── schemas/
│   ├── context-packet.json       # JSON Schema for preference packet
│   ├── venue-result.json         # Shared schema all specialists return
│   └── itinerary.json            # Schema for final itinerary output
├── prompts/
│   └── coordinator.md            # Trip Planner coordinator prompt
├── preferences/
│   └── README.md                 # Documents which Notion pages hold preferences
├── examples/
│   └── portland-3day.md          # Example output for reference
├── PLAN.md                       # This file
└── README.md                     # Project overview
```

### Step 2: Define the Context Packet schema
The Context Packet is the single structured object passed to every specialist.
It contains:
- **Trip details**: city, dates, number of days, accommodation location
- **Pace**: light / medium / maximal
- **Preferences**: pulled from Notion
  - Coffee: independent, espresso quality, good seating, walkable
  - Shopping: thrift menswear, vintage, design-y shops
  - Books: used + rare bookstores
  - Food: likes/dislikes, budget, reservation tolerance
- **Constraints**: budget, must-dos, weather, mobility
- **Output format**: what the coordinator expects back

### Step 3: Build the three starter specialists

#### Coffee Finder Agent (`coffee-finder.md`)
- **Input**: Context Packet (city, preferences.coffee)
- **Research method**: Web search for specialty/third-wave coffee shops
- **Output contract** (JSON):
  - 8-15 candidates
  - Each: name, address, neighborhood, hours, vibe tags, why it matches, best time to go, one nearby pairing
- **Matching logic**: Filter by preference tags (independent, espresso-focused, good seating)

#### Shops Finder Agent (`shops-finder.md`)
- **Input**: Context Packet (city, preferences.shopping)
- **Research method**: Web search for thrift, vintage, and curated shops
- **Output contract** (JSON):
  - 8-15 candidates
  - Each: name, address, neighborhood, hours, tags (thrift/vintage/menswear/design), curation level, estimated browse time
- **Matching logic**: Prioritize menswear thrift, vintage, and well-curated shops

#### Geo Optimizer Agent (`geo-optimizer.md`)
- **Input**: Context Packet + array of all venue results from other agents
- **Job**: Cluster venues by neighborhood proximity, estimate walking times, identify which neighborhoods to pair together per day
- **Output contract** (JSON):
  - Neighborhood clusters with venues grouped
  - Walking time estimates between venues in same cluster
  - Transit friction notes (how hard to get between clusters)
  - Suggested daily neighborhood pairings
  - Operating hours conflicts flagged

### Step 4: Build the Coordinator prompt
The coordinator (`prompts/coordinator.md`) is not a sub-agent itself — it's the system prompt / instructions for the main Claude Code session that orchestrates everything.

It will:
1. Accept: city, dates, pace, constraints
2. Load preferences from Notion (or local cache)
3. Build the Context Packet
4. Dispatch Coffee Finder and Shops Finder in parallel
5. Pass combined results to Geo Optimizer
6. Assemble a day-by-day itinerary:
   - Morning: coffee + neighborhood wander
   - Midday: shops/browsing
   - Afternoon: flexible (park, museum, bookstore)
   - Evening: dinner (if applicable)
7. Output: structured itinerary + alternates bench by neighborhood

### Step 5: Test with a real city
Run a complete flow for a 2-3 day trip to a test city (e.g., Portland, OR or Austin, TX).
Validate:
- Each specialist returns well-structured, real results
- Geo optimizer correctly clusters by neighborhood
- Coordinator produces a coherent, walkable itinerary
- Preferences actually influence the results

## Phase 2 — Expanded Specialists (COMPLETE)

All 8 specialist agents are now implemented and tested:

| Agent | File | Returns |
|-------|------|---------|
| Coffee Finder | `.claude/agents/coffee-finder.md` | 8-15 candidates |
| Restaurant Finder | `.claude/agents/restaurant-finder.md` | 6-10 candidates |
| Vintage & Thrift Finder | `.claude/agents/vintage-finder.md` | 6-10 candidates |
| New Menswear Finder | `.claude/agents/menswear-finder.md` | 4-8 candidates |
| Bookstore Finder | `.claude/agents/bookstore-finder.md` | 4-8 candidates |
| Lifestyle & Craft Finder | `.claude/agents/lifestyle-finder.md` | 6-10 candidates |
| Markets & Events Finder | `.claude/agents/markets-finder.md` | 4-8 candidates |
| Parks & Outdoors Finder | `.claude/agents/parks-finder.md` | 4-8 candidates |

### Notion Integration (COMPLETE)
- Optional personal "places" database (e.g., your own Notion bookmarks) queried at runtime via MCP; configure its IDs in your profile under `personal_places_database`
- Bookmarked places injected as `must_dos` in the context packet
- Any bookmarked venues in the destination city are surfaced as high-priority references

### Preference Profiles (COMPLETE)
- `preferences/<your-name>.json` — your populated profile (preference categories + reference places), kept local via `.gitignore`
- `preferences/_template.json` — template for new users (the only profile committed to the repo)

### Venue Research Cache (COMPLETE)
- **Tripster Research** Notion database stores all venue candidates from every agent run
- Configure the database and data-source IDs in your profile under `tripster_research_database` (kept local, not committed)
- Properties: Name, City, Neighborhood, Address, Category, Match Score, Match Reason, Hours, Best Time, Agent, Trip, Itinerary Status (Selected/Alternate/Dropped), Status (New/Reviewed/Visited/Skip), Tags
- Coordinator checks this cache before dispatching agents (Step 2c) — if substantial research exists for a city, agents can be skipped
- After every itinerary assembly, all venue candidates are auto-saved (Step 7)
- User reviews venues in Notion, marking them Reviewed/Visited/Skip
- Currently cached: Brooklyn (70 venues), Chicago (21 venues)

## Phase 3a — Navigation & Discovery (COMPLETE)

Three features that transform the itinerary from a static document into an on-the-ground mobile navigation tool.

### 1. Google Maps Links ("Maps & Photos")
Every venue in the itinerary includes a clickable Google Maps link:
- **Venue links**: `https://www.google.com/maps/search/?api=1&query={name}+{address}` — shows the business listing with reviews, hours, and photos. This link doubles as the venue's **photo source**, so no images are embedded in the itinerary.
- **Inter-cluster transit links**: `https://www.google.com/maps/dir/?api=1&origin={name}+{address}&destination={name}+{address}&travelmode=transit` — directions between neighborhood clusters, built from **name+address text** (Google geocodes it) rather than agent-supplied coordinates.

### 2. Website URLs
Every venue includes its official website URL (from agent `source_url`), allowing quick preview of hours, menus, and what to expect before walking over.

### 3. "Also Consider" Lists
Each neighborhood cluster includes up to 5 backup venues, sorted by match_score. Each backup shows:
- What it offers and what it could swap for
- Walking time from the nearest planned venue
- Website URL and Google Maps (Maps & Photos) link

> **Note (July 2026):** Embedded venue photos (`image_url`) were removed. Agents couldn't reliably produce verified direct image URLs, so most rendered broken. The Google Maps "Maps & Photos" link provides storefront recognition instead. `lat`/`lng` are now optional best-effort hints used only for clustering, never for navigation links.

### Implementation
- All 8 agents REQUIRE `source_url` for every candidate; `lat`/`lng` optional (best-effort, clustering hint only); `image_url` not used
- Geo Optimizer preserves all venue metadata through clustering and builds per-cluster `also_consider` arrays
- Coordinator generates Google Maps URLs at format time and formats the itinerary with embedded links (no images)
- Primary output is a **Notion page** (mobile-friendly with clickable links); markdown file kept as backup

### Schema Changes
- `schemas/venue-result.json` — `source_url` required; `lat`, `lng`, `image_url` optional; `google_maps_url` added
- `schemas/itinerary.json` — blocks have `address`, `neighborhood`, `website_url`, `google_maps_url`; days have `transit_directions` and `also_consider` arrays

## Phase 3 — Future

### Learning Layer:
- **Trip Debrief Agent**: After a trip, capture what you loved/liked/disliked
- **Preference Curator**: Propose updates to preferences based on debrief
- Approval workflow before updating source of truth

### Production migration:
- Convert to Claude Agent SDK for standalone CLI tool
- Add real API integrations (Google Places, Yelp)
- Package as `tripster` CLI command

## Design Decisions

### Why Claude Code sub-agents first?
1. Zero infrastructure — create markdown files, start testing immediately
2. Natural language coordination fits the "conductor + orchestra" pattern
3. Web search built in — specialists can research real venues today
4. Easy to iterate on prompts and schemas
5. Transferable — prompts and schemas move to Agent SDK later

### Why combination data sources?
1. Web search gives broad, current results (new shops, updated hours)
2. Structured APIs (Phase 2) give reliable, structured data
3. LLM reasoning filters and scores against preferences
4. Best of both: discovery breadth + data reliability

### Why Notion for preferences?
1. Already your system of record
2. Single source of truth — update once, all agents see it
3. MCP integration means agents can read directly
4. Learning layer can propose updates back to Notion

## Dispatch Strategy: Sequential vs. Parallel

### Problem: Context Window Exhaustion
The initial Brooklyn test (Feb 2026) launched all 8 specialist agents simultaneously via parallel Task tool calls. This caused:
1. Agents timed out while researching (web searches take time, 8 concurrent agents is heavy)
2. When results started returning, the combined output from 8 agents exceeded the context window
3. Coordinator could not process results — every subsequent action returned "Prompt is too long"
4. No results were saved to disk — total loss of work

### Solution: Sequential Dispatch with Incremental Persistence
The coordinator prompt says "Launch all 8 sub-agents simultaneously" (Step 3). **This should be treated as the ideal, not a requirement.** The recommended dispatch strategy is:

```
SEQUENTIAL (recommended for most trips):
  For each agent in [coffee, restaurant, vintage, menswear, bookstore, lifestyle, markets, parks]:
    1. Dispatch agent with context packet
    2. Wait for results
    3. Save results to runs/<trip>/agent-results.json
    4. Continue to next agent

  Then:
    5. Load all 8 result files
    6. Dispatch geo-optimizer with combined summary
    7. Save optimizer results
    8. Assemble itinerary
```

### Why Sequential Works Better:
- **No context bloat**: Each agent runs in its own context, returns results, and the coordinator saves to disk before moving on
- **Incremental persistence**: If anything fails mid-run, completed results are on disk and can be resumed
- **Coordinator stays lean**: The coordinator's context holds the trip config + the current agent interaction, not all 8 agents' outputs simultaneously
- **Geo optimizer gets a compact summary**: Instead of raw agent outputs, feed it a condensed venue list (name, neighborhood, score, category)

### When Parallel Might Work:
- Short trips (1-2 days) in small cities with fewer venues
- If agent prompts are simplified to return fewer candidates
- If a future version uses file-based handoff instead of context-based

### Results Directory Convention:
```
runs/
└── brooklyn-april-2026/
    ├── context-packet.json
    ├── coffee-finder-results.json
    ├── restaurant-finder-results.json
    ├── vintage-finder-results.json
    ├── menswear-finder-results.json
    ├── bookstore-finder-results.json
    ├── lifestyle-finder-results.json
    ├── markets-finder-results.json
    ├── parks-finder-results.json
    ├── geo-optimizer-results.json
    └── itinerary.md
```

## Completed Test Runs

### Chicago 2-Day (Feb 14-15, 2026)
- File: `examples/chicago-2day.md`
- Maximal pace, Valentine's Day
- 3 agents (original architecture)

### Portland 3-Day (Mar 15-17, 2026)
- File: `examples/portland-3day.md`
- Medium pace
- 3 agents (original architecture)

### Brooklyn 1-Day (Apr 4, 2026)
- File: `runs/brooklyn-april-2026/itinerary.md`
- Maximal pace, Saturday, Greenpoint hotel start
- **8 agents, sequential dispatch** (new architecture)
- 69 candidates evaluated → 10 selected → 6/6 must-dos covered
- Route: Greenpoint → Williamsburg → NYC Ferry → DUMBO → Brooklyn Heights
- Key finding: Smorgasburg opens April 5 (one day after trip date)

## Output Schema (preview)

### Venue Result (what every specialist returns):
```json
{
  "agent": "coffee-finder",
  "city": "Portland, OR",
  "candidates": [
    {
      "name": "Heart Coffee Roasters",
      "address": "2211 E Burnside St",
      "neighborhood": "Buckman",
      "hours": "7am-6pm daily",
      "tags": ["third-wave", "espresso", "light-roast", "minimal"],
      "match_score": 0.92,
      "match_reason": "Independent, known for espresso, clean minimal space with good seating",
      "best_time": "morning",
      "nearby_pairing": "Powell's Books (0.8 mi)",
      "source_url": "https://...",
      "lat": 45.5231,
      "lng": -122.6427
    }
  ],
  "observations": [
    "Portland's coffee scene is heavily concentrated in SE and NE neighborhoods"
  ]
}
```

### Itinerary Day (what the coordinator produces):
```json
{
  "day": 1,
  "date": "2026-03-15",
  "neighborhoods": ["Buckman", "Hawthorne"],
  "theme": "Coffee + Vintage in SE Portland",
  "blocks": [
    {
      "time": "9:00am",
      "activity": "coffee",
      "venue": "Heart Coffee Roasters",
      "duration_min": 45,
      "notes": "Start with their espresso flight"
    },
    {
      "time": "10:00am",
      "activity": "shopping",
      "venue": "Red Light Clothing Exchange",
      "duration_min": 60,
      "walk_from_previous": "12 min"
    }
  ],
  "alternates": [
    {"venue": "Coava Coffee", "swaps_for": "Heart Coffee", "reason": "If you prefer pour-over focus"}
  ]
}
```
