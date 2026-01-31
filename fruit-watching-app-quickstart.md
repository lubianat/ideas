# Fruit Watching App - Quick Start Guide

> **TL;DR:** Build a SvelteKit app for tracking fruits found at markets. Think eBird, but for fruit watching.

## The Concept

Users log in with Wikimedia, search for markets via OpenStreetMap, select fruits from Wikidata, and create observations (lists) of which fruits they found at which market on which day. That's it!

## Core Features (Nothing Else!)

1. **Wikimedia Login** - Only authentication method
2. **Market Search** - Via OpenStreetMap Nominatim API
3. **Fruit List** - From Wikidata SPARQL queries  
4. **Observations** - Track fruits found at markets on specific dates

## 3-Minute Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     SvelteKit App                        │
├─────────────────────────────────────────────────────────┤
│ Frontend (Svelte Components)                            │
│  ├─ Auth: Login/Logout                                  │
│  ├─ Markets: Search & Select                            │
│  ├─ Fruits: Browse & Select                             │
│  └─ Observations: Create, View, Share                   │
├─────────────────────────────────────────────────────────┤
│ Backend (SvelteKit API Routes)                          │
│  ├─ OAuth with Wikimedia                                │
│  ├─ Database operations (SQLite/PostgreSQL)             │
│  └─ API integrations                                    │
├─────────────────────────────────────────────────────────┤
│ External APIs                                            │
│  ├─ Wikimedia OAuth (auth)                              │
│  ├─ OpenStreetMap Nominatim (markets)                   │
│  └─ Wikidata Query Service (fruits)                     │
└─────────────────────────────────────────────────────────┘
```

## Database Schema (5 Tables)

```typescript
users              // Wikimedia authenticated users
markets            // Saved market locations (OSM data)
fruits             // Cached fruit data (Wikidata)
observations       // User's fruit watching sessions
observation_fruits // Many-to-many: which fruits in which observations
```

## Development Phases

### Phase 1: Setup (4-7 hours)
- Initialize SvelteKit with TypeScript
- Set up database with Drizzle ORM
- Configure environment variables

### Phase 2: Authentication (5-7 hours)
- Wikimedia OAuth integration
- Session management
- Protected routes

### Phase 3: Markets (6-9 hours)
- OpenStreetMap search
- Save/manage markets
- Optional: Map display

### Phase 4: Fruits (6-8 hours)
- Wikidata SPARQL queries
- Cache fruits in database
- Searchable fruit selector

### Phase 5: Observations (12-16 hours) ⭐ Core Feature
- Create observations (select market, date, fruits)
- View/edit user observations
- Share observations publicly

### Phase 6: Polish (5-7 hours)
- Dashboard
- Navigation
- Final testing

**Total:** 40-60 hours for a complete v1.0

## Quick Start Commands

```bash
# 1. Initialize project
npm create svelte@latest fruit-watching-app
cd fruit-watching-app

# 2. Install core dependencies
npm install drizzle-orm better-sqlite3
npm install -D drizzle-kit @types/better-sqlite3

# 3. Set up environment
cp .env.example .env
# Edit .env with your API credentials

# 4. Run migrations & seed data
npm run db:push
npm run seed

# 5. Start development
npm run dev
```

## Required API Credentials

### 1. Wikimedia OAuth
Register at: https://meta.wikimedia.org/wiki/Special:OAuthConsumerRegistration
- Get: Client ID, Client Secret
- Set callback URL to: `http://localhost:5173/auth/callback`

### 2. OpenStreetMap
No API key required! Just respect rate limits (1 req/sec)
- Endpoint: https://nominatim.openstreetmap.org/search

### 3. Wikidata
No API key required!
- SPARQL endpoint: https://query.wikidata.org/sparql

## Sample SPARQL Query (Fruits)

```sparql
SELECT ?fruit ?fruitLabel ?scientificName WHERE {
  ?fruit wdt:P31/wdt:P279* wd:Q3314483.
  OPTIONAL { ?fruit wdt:P225 ?scientificName. }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
LIMIT 1000
```

## Key Routes

```
/                          → Dashboard (requires auth)
/auth/login               → Initiate Wikimedia OAuth
/auth/callback            → OAuth callback
/markets                  → Search & manage markets
/fruits                   → Browse fruits
/observations             → List observations
/observations/new         → Create new observation
/observations/[id]        → View observation (private)
/observations/[id]/share  → View observation (public)
```

## Testing Checklist

Before calling it done, verify:

- [ ] Can log in with Wikimedia
- [ ] Can search for a market via OpenStreetMap
- [ ] Can see a list of fruits from Wikidata
- [ ] Can create an observation with market + date + fruits
- [ ] Can view my past observations
- [ ] Can share an observation publicly
- [ ] Can log out

## Common Pitfalls

1. **OAuth Callback Mismatch** - Make sure callback URL in Wikimedia settings matches your env variable exactly
2. **Nominatim Rate Limit** - Add debouncing (500ms) to search inputs
3. **Wikidata Timeouts** - Cache fruits locally, don't query on every page load
4. **Session Security** - Use HTTP-only, secure cookies with a strong secret

## Need More Details?

See the complete guide: [fruit-watching-app-guidelines.md](./fruit-watching-app-guidelines.md)

- Full work breakdown with time estimates
- Complete database schema with TypeScript types
- All API endpoints documented
- Component structure
- Security considerations
- Deployment guide

## Philosophy

> "Do one thing well. Track fruits at markets. Nothing else."

Resist the temptation to add:
- Photo uploads
- Social features
- Analytics dashboards
- Multiple auth providers
- Gamification

These can come later. v1.0 is about a clean, focused fruit tracking experience.

---

**Ready to build?** Start with Phase 1 in the full guidelines document! 🍎🍊🍌
