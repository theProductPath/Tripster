# Preferences

User preference profiles that drive how Tripster researches and recommends venues.

## How It Works

Each user gets their own `.json` file in this directory (e.g., `jane.json`, `alex.json`). When the coordinator starts a trip planning session, it loads the appropriate profile and injects it into the Context Packet that every specialist agent receives.

> **Privacy:** Your personal profile stays local. `.gitignore` excludes `preferences/*.json` (everything except `_template.json`), so your tastes and any Notion IDs are never committed to the repo.

## Creating Your Own Profile

1. Copy `_template.json` to `<your-name>.json`
2. Fill in your preferences — be as detailed or minimal as you like
3. The more reference places you add, the better agents calibrate to your taste

### What to Include

- **Identity**: Your travel style, aesthetic preferences, and values
- **Coffee**: What you look for in coffee shops, plus examples of places you love
- **Shopping**: Types of shops you're drawn to, curation level, reference places
- **Food**: Cuisine rankings, dining style, budget, dietary notes
- **Books**: Bookstore preferences
- **Travel**: What draws you to or repels you from attractions
- **Context signals**: Cross-category heuristics that help agents evaluate any venue

### Key Concepts

- **Tags**: Quick labels agents use for broad filtering (e.g., `"independent"`, `"thrift"`, `"menswear"`)
- **Positive/Negative Signals**: Detailed qualities that boost or penalize venue scores during evaluation
- **Reference Places**: Known-good examples from your past travel. Agents research these to understand your taste DNA, then look for similar places in new cities
- **Context Signals**: Universal heuristics that apply across all categories (e.g., "owner-operated" is always a plus)

## Multi-User Usage

When planning a trip for a specific person:
```
"Plan a trip to Austin for Jane"    → loads preferences/jane.json
"Plan a trip to London for Alex"    → loads preferences/alex.json
```

If no name is specified and only one profile exists, it's used automatically.
If multiple profiles exist and no name is given, the coordinator will ask.

## Updating Preferences

Edit your `.json` file directly, or paste new information into the conversation and ask the coordinator to merge it into your profile. Sources might include:

- Descriptions of places you've loved
- Links to shops, restaurants, or cafes you want to remember
- General taste statements ("I've gotten really into natural wine lately")
- Lists from other apps (Google Maps saved places, etc.)

The coordinator will extract preference signals and update your file.

## Notion Integration (Optional)

If your profile includes a `personal_places_database` section with your Notion IDs, the coordinator can also query your Notion bookmarks database at runtime to check if you've already saved places in the destination city. These become high-priority recommendations. Similarly, a `tripster_research_database` section lets the coordinator cache and reuse venue research across trips, and a `notion_parent_page_id` tells it where to save generated itineraries. All of these are optional and stay in your local (gitignored) profile.
