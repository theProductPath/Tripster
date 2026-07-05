# Tripster

A multi-agent travel itinerary planner built on Claude sub-agents. A coordinator orchestrates specialist agents that each research one category of places — calibrated to *your* taste profile — then assembles their findings into a neighborhood-aware, day-by-day itinerary with navigation links and backup options.

Tripster is preference-driven: you describe what you love (coffee, food, shops, books, parks, markets, and the qualities that matter to you), and the agents hunt for places in any city that share that DNA.

## How It Works

1. The coordinator loads **your** preference profile from `preferences/<your-name>.json`
2. You provide a city, dates, and pace
3. Specialist agents research venues one at a time (sequentially, to stay within context limits), calibrated to your taste profile
4. A geo optimizer clusters results by neighborhood and walkability
5. The coordinator assembles a day-by-day itinerary with navigation links and "Also Consider" alternates

## Quickstart

1. **Create your profile.** Copy the template and fill it in:

   ```
   cp preferences/_template.json preferences/<your-name>.json
   ```

   Edit the new file with your tastes — the more reference places (shops/cafes/restaurants you already love) you add, the better the agents calibrate. See `preferences/README.md` for a field-by-field guide.

2. **Plan a trip.** From this project directory, give the coordinator (`prompts/coordinator.md`) a request:

   > Plan a 2-day trip to Austin, TX. Medium pace.

   If multiple profiles exist, name whose trip it is:

   > Plan a trip to London for Jane

3. The coordinator dispatches the specialist agents, runs the geo optimizer, and produces a walkable, day-by-day itinerary. Each venue includes a **Website** link and a **Maps & Photos** link (Google Maps listing) for on-the-ground navigation.

## Your Data Stays Local

Your preference profile and any generated trips are **personal** and are **not** committed to this repository. `.gitignore` excludes:

- `preferences/*.json` — every profile **except** `_template.json`
- `runs/` — generated itineraries and per-agent research (may contain your tastes and personal Notion links)
- local settings and OS cruft

Only the template profile is committed. Keep your own `preferences/<your-name>.json` locally and it will keep working — it's ignored by git, so your tastes and any private Notion IDs never get pushed.

## Project Structure

```
Tripster/
├── .claude/agents/            # Specialist sub-agent prompts
│   ├── coffee-finder.md       # Finds specialty coffee shops
│   ├── restaurant-finder.md   # Finds local restaurants by cuisine
│   ├── vintage-finder.md      # Finds curated vintage and thrift shops
│   ├── menswear-finder.md     # Finds heritage / craft menswear shops
│   ├── bookstore-finder.md    # Finds used, rare, and indie bookstores
│   ├── lifestyle-finder.md    # Finds artisan and craft lifestyle shops
│   ├── markets-finder.md      # Finds markets, flea markets, street events
│   ├── parks-finder.md        # Finds parks, gardens, public squares
│   └── geo-optimizer.md       # Clusters venues by neighborhood
├── schemas/                   # JSON schemas for data contracts
│   ├── context-packet.json    # Input schema for all agents
│   ├── venue-result.json      # Output schema for specialist agents
│   └── itinerary.json         # Output schema for the final itinerary
├── prompts/
│   └── coordinator.md         # Coordinator orchestration prompt
├── preferences/               # Preference profiles
│   ├── _template.json         # Template for new users (the only profile committed)
│   └── README.md              # How to create and update your profile
├── examples/                  # Example itinerary outputs
└── PLAN.md                    # Full implementation plan / design notes
```

`preferences/<your-name>.json` and `runs/` are created as you use the tool and stay local (gitignored).

## Agents

| Agent | Job | Returns |
|-------|-----|---------|
| Coffee Finder | Specialty coffee shops | 8-15 candidates |
| Restaurant Finder | Local restaurants by cuisine preference | 6-10 candidates |
| Vintage & Thrift Finder | Curated vintage, thrift, consignment | 6-10 candidates |
| New Menswear Finder | Japanese denim, heritage workwear, craft menswear | 4-8 candidates |
| Bookstore Finder | Used, rare, indie bookstores and magazine shops | 4-8 candidates |
| Lifestyle & Craft Finder | Chocolate, tea, ceramics, stationery, apothecaries | 6-10 candidates |
| Markets & Events Finder | Outdoor markets, flea markets, street festivals | 4-8 candidates |
| Parks & Outdoors Finder | Parks, botanical gardens, public squares | 4-8 candidates |
| Geo Optimizer | Neighborhood clustering and walkability | Walkable daily plans |

## Preference Profiles

Each profile contains:

- **Tags & notes** — quick labels and detailed descriptions per category
- **Positive/negative signals** — what to look for and what to avoid
- **Reference places** — known-good examples the agents use to calibrate their search
- **Context signals** — universal heuristics across all categories

The richer your profile, the better the results. See `preferences/README.md` for details.

## Optional: Notion Integration

Tripster works entirely with local files, but if you use Notion you can wire in three optional capabilities by adding IDs to your (local) profile:

- **`personal_places_database`** — a Notion database of places you've already bookmarked. The coordinator checks it for the destination city and surfaces your saved spots as high-priority picks.
- **`tripster_research_database`** — a Notion database that caches every agent's research so future trips to the same city can reuse it (saving time and tokens).
- **`notion_parent_page_id`** — where generated itineraries are saved as Notion pages for easy mobile reference.

All three are optional and live only in your local profile — none of these IDs are committed to the repo.
