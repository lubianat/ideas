# Fruit Watching App - Development Guidelines

## Project Overview

A SvelteKit web application for tracking fruits found in markets, inspired by nature observation apps like eBird, iNaturalist, and WikiAves. The app focuses exclusively on "fruit watching" without media support, providing a simple and focused experience.

### Core Scope

**ONLY these features (nothing else):**
1. Login via Wikimedia OAuth (only authentication method)
2. Market selector via OpenStreetMap (store id, coordinates, name)
3. Fruit list via Wikidata query
4. Observation lists (track which fruits were found at which market on which day)

**Explicitly out of scope:**
- Analytics
- Media/photo uploads
- Social features beyond basic list sharing
- Multiple authentication providers
- Mobile apps (web-first approach)

---

## Technical Architecture

### Tech Stack

- **Frontend Framework:** SvelteKit (latest stable)
- **Language:** TypeScript
- **Database:** SQLite or PostgreSQL (start with SQLite for simplicity)
- **ORM:** Drizzle ORM or Prisma
- **Authentication:** OAuth 2.0 with Wikimedia
- **Styling:** Tailwind CSS (optional but recommended for rapid development)
- **Deployment:** Vercel, Netlify, or self-hosted

### External APIs

1. **Wikimedia OAuth** - User authentication
2. **OpenStreetMap Nominatim API** - Market location search
3. **Wikidata Query Service (SPARQL)** - Fruit data retrieval

---

## Work Breakdown Structure

### Phase 1: Project Setup & Foundation

#### Task 1.1: Initialize SvelteKit Project
**Estimated Time:** 2-4 hours

**Steps:**
```bash
npm create svelte@latest fruit-watching-app
cd fruit-watching-app
npm install
```

**Configuration:**
- Enable TypeScript
- Add ESLint and Prettier
- Configure SvelteKit adapter (adapter-node or adapter-vercel)
- Set up environment variables structure

**Dependencies to install:**
```json
{
  "@sveltejs/adapter-auto": "latest",
  "typescript": "latest",
  "vite": "latest",
  "drizzle-orm": "latest",
  "better-sqlite3": "latest"
}
```

**Files to create:**
- `.env.example` - Template for environment variables
- `src/lib/db/` - Database configuration
- `src/lib/types/` - TypeScript type definitions

#### Task 1.2: Database Schema Design
**Estimated Time:** 2-3 hours

**Tables needed:**

```typescript
// users table
interface User {
  id: string; // UUID or Wikimedia user ID
  wikimedia_username: string;
  wikimedia_id: string;
  created_at: Date;
  updated_at: Date;
}

// markets table
interface Market {
  id: string; // UUID
  osm_id: string; // OpenStreetMap ID
  name: string;
  latitude: number;
  longitude: number;
  address?: string;
  created_at: Date;
  created_by: string; // user_id
}

// fruits table (cached from Wikidata)
interface Fruit {
  id: string; // UUID
  wikidata_id: string; // e.g., Q8086
  name: string;
  scientific_name?: string;
  cached_at: Date;
}

// observations table (the core feature)
interface Observation {
  id: string; // UUID
  user_id: string;
  market_id: string;
  observation_date: Date;
  notes?: string;
  created_at: Date;
  updated_at: Date;
}

// observation_fruits (many-to-many)
interface ObservationFruit {
  id: string;
  observation_id: string;
  fruit_id: string;
  quantity?: string; // "abundant", "common", "few", "rare"
  notes?: string;
}
```

**Files to create:**
- `src/lib/db/schema.ts` - Drizzle schema definitions
- `src/lib/db/migrations/` - SQL migration files
- `drizzle.config.ts` - Drizzle configuration

---

### Phase 2: Authentication (Wikimedia OAuth)

#### Task 2.1: Set Up OAuth Configuration
**Estimated Time:** 3-4 hours

**Steps:**
1. Register app at Wikimedia: https://meta.wikimedia.org/wiki/Special:OAuthConsumerRegistration
2. Get OAuth credentials (Client ID, Client Secret)
3. Configure callback URL

**Environment Variables:**
```env
WIKIMEDIA_CLIENT_ID=your_client_id
WIKIMEDIA_CLIENT_SECRET=your_client_secret
WIKIMEDIA_CALLBACK_URL=http://localhost:5173/auth/callback
```

**OAuth Flow:**
1. User clicks "Login with Wikimedia"
2. Redirect to Wikimedia OAuth authorization
3. Wikimedia redirects back with authorization code
4. Exchange code for access token
5. Fetch user profile from Wikimedia API
6. Create/update user in database
7. Set session cookie

**Files to create:**
- `src/routes/auth/login/+server.ts` - Initiate OAuth flow
- `src/routes/auth/callback/+server.ts` - Handle OAuth callback
- `src/routes/auth/logout/+server.ts` - Handle logout
- `src/lib/server/auth.ts` - Auth utilities and session management
- `src/hooks.server.ts` - Session validation middleware

**Libraries:**
- Consider using `@auth/sveltekit` (formerly SvelteKit Auth) with custom Wikimedia provider
- Or implement OAuth manually using `node-fetch`

#### Task 2.2: Protected Routes & Session Management
**Estimated Time:** 2-3 hours

**Implementation:**
- Use SvelteKit hooks to validate sessions
- Store session in secure HTTP-only cookies
- Redirect unauthenticated users to login page

**Files to update:**
- `src/hooks.server.ts` - Add session validation
- `src/routes/+layout.server.ts` - Pass user data to all pages

---

### Phase 3: Market Selector (OpenStreetMap Integration)

#### Task 3.1: Market Search Interface
**Estimated Time:** 4-6 hours

**Features:**
- Search box for market names/locations
- Autocomplete using Nominatim API
- Display results with name, address, coordinates
- Map preview using Leaflet or Mapbox (optional but recommended)
- Save selected market to database

**API Integration:**
```typescript
// Nominatim search API
const searchMarkets = async (query: string) => {
  const url = `https://nominatim.openstreetmap.org/search?q=${query}&format=json&addressdetails=1&limit=10`;
  // Filter for marketplaces, supermarkets, etc.
  const params = {
    amenity: 'marketplace',
  };
};
```

**Files to create:**
- `src/routes/markets/+page.svelte` - Market search UI
- `src/routes/markets/+page.server.ts` - Server-side market operations
- `src/routes/api/markets/search/+server.ts` - Market search API endpoint
- `src/lib/components/MarketSearch.svelte` - Reusable market search component
- `src/lib/components/MarketMap.svelte` - Map display component (optional)
- `src/lib/services/osm.ts` - OpenStreetMap API utilities

**UX Considerations:**
- Debounce search input (300-500ms)
- Show loading states
- Handle API errors gracefully
- Cache recent searches

#### Task 3.2: Market Management
**Estimated Time:** 2-3 hours

**Features:**
- List user's saved markets
- View market details
- Edit market information (if needed)
- Set default market

**Files to create:**
- `src/routes/markets/[id]/+page.svelte` - Market detail page
- `src/routes/markets/my-markets/+page.svelte` - User's saved markets

---

### Phase 4: Fruit List (Wikidata Integration)

#### Task 4.1: Wikidata Query Setup
**Estimated Time:** 3-4 hours

**SPARQL Query Example:**
```sparql
SELECT ?fruit ?fruitLabel ?scientificName WHERE {
  ?fruit wdt:P31/wdt:P279* wd:Q3314483.  # Instance of/subclass of fruit
  OPTIONAL { ?fruit wdt:P225 ?scientificName. }  # Scientific name
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
LIMIT 1000
```

**Implementation:**
- Query Wikidata SPARQL endpoint on app startup or scheduled job
- Cache results in local database
- Update cache periodically (daily/weekly)
- Provide manual refresh option

**Files to create:**
- `src/lib/services/wikidata.ts` - Wikidata SPARQL query utilities
- `src/routes/api/fruits/sync/+server.ts` - Fruit data sync endpoint
- `src/lib/jobs/sync-fruits.ts` - Scheduled sync job (optional)

#### Task 4.2: Fruit Selection Interface
**Estimated Time:** 3-4 hours

**Features:**
- Searchable/filterable fruit list
- Alphabetical organization
- Quick selection (checkboxes or similar)
- Display scientific names
- Show Wikidata links for reference

**Files to create:**
- `src/lib/components/FruitSelector.svelte` - Fruit selection component
- `src/routes/api/fruits/+server.ts` - Fruit list API

**UX Considerations:**
- Support bulk selection
- Common fruits at the top
- Search by common name or scientific name
- Keyboard navigation support

---

### Phase 5: Observation Lists (Core Feature)

#### Task 5.1: Create Observation
**Estimated Time:** 5-7 hours

**Features:**
- Select market (from saved markets)
- Select observation date (default: today)
- Select multiple fruits found
- For each fruit: optional quantity indicator (abundant/common/few/rare)
- Add general notes about the observation
- Save observation

**Workflow:**
1. User navigates to "New Observation"
2. Selects market (or searches for new one)
3. Sets date
4. Selects fruits from the list
5. Optionally adds quantity/notes per fruit
6. Adds general observation notes
7. Saves observation

**Files to create:**
- `src/routes/observations/new/+page.svelte` - New observation form
- `src/routes/observations/new/+page.server.ts` - Handle observation creation
- `src/lib/components/ObservationForm.svelte` - Reusable observation form
- `src/routes/api/observations/+server.ts` - Observation CRUD API

#### Task 5.2: View & Manage Observations
**Estimated Time:** 4-5 hours

**Features:**
- List user's observations
- Filter by date range, market, fruit
- Sort by date (newest first)
- View observation details
- Edit/delete own observations

**Files to create:**
- `src/routes/observations/+page.svelte` - Observations list
- `src/routes/observations/+page.server.ts` - Load observations
- `src/routes/observations/[id]/+page.svelte` - Observation detail view
- `src/routes/observations/[id]/edit/+page.svelte` - Edit observation
- `src/lib/components/ObservationCard.svelte` - Observation display component

#### Task 5.3: Sharing & Public Lists
**Estimated Time:** 3-4 hours

**Features:**
- Generate shareable link for observation
- Public view (no login required)
- Show observer name, date, market, fruits found
- Basic privacy controls (public/private toggle)

**Files to create:**
- `src/routes/observations/[id]/share/+page.svelte` - Public observation view
- Add `is_public` field to observations table

---

### Phase 6: Dashboard & Navigation

#### Task 6.1: Home Dashboard
**Estimated Time:** 3-4 hours

**Features:**
- Welcome message with user name
- Quick stats (total observations, markets visited, fruits found)
- Recent observations
- Quick action buttons (New Observation, Search Markets, Browse Fruits)

**Files to create:**
- `src/routes/+page.svelte` - Home dashboard
- `src/routes/+page.server.ts` - Load dashboard data

#### Task 6.2: Navigation & Layout
**Estimated Time:** 2-3 hours

**Components:**
- Top navigation bar
- User menu (profile, logout)
- Mobile-responsive design

**Files to create:**
- `src/routes/+layout.svelte` - App layout with navigation
- `src/lib/components/Navigation.svelte` - Navigation component
- `src/lib/components/UserMenu.svelte` - User menu component

---

## Data Flow Diagrams

### Authentication Flow
```
User → Login Button → Wikimedia OAuth → Callback → 
Session Created → Redirect to Dashboard
```

### Observation Creation Flow
```
User → New Observation → Select Market → Select Date → 
Select Fruits → Add Notes → Save → 
Database (observations + observation_fruits) → 
Confirmation → Redirect to Observation View
```

### Market Search Flow
```
User → Search Input → Debounced Query → 
Nominatim API → Results List → 
Select Market → Save to DB → Use in Observation
```

---

## API Endpoints

### Authentication
- `GET /auth/login` - Initiate Wikimedia OAuth
- `GET /auth/callback` - OAuth callback handler
- `POST /auth/logout` - Logout user

### Markets
- `GET /api/markets/search?q={query}` - Search markets via OSM
- `GET /api/markets` - Get user's saved markets
- `POST /api/markets` - Save a new market
- `GET /api/markets/[id]` - Get market details
- `PATCH /api/markets/[id]` - Update market
- `DELETE /api/markets/[id]` - Delete market

### Fruits
- `GET /api/fruits` - Get all fruits (cached from Wikidata)
- `POST /api/fruits/sync` - Trigger Wikidata sync
- `GET /api/fruits/[id]` - Get fruit details

### Observations
- `GET /api/observations` - Get user's observations (with filters)
- `POST /api/observations` - Create new observation
- `GET /api/observations/[id]` - Get observation details
- `PATCH /api/observations/[id]` - Update observation
- `DELETE /api/observations/[id]` - Delete observation
- `GET /api/observations/[id]/share` - Get public observation view

---

## Frontend Components Structure

```
src/lib/components/
├── auth/
│   ├── LoginButton.svelte
│   └── UserMenu.svelte
├── markets/
│   ├── MarketSearch.svelte
│   ├── MarketCard.svelte
│   ├── MarketMap.svelte (optional)
│   └── MarketSelector.svelte
├── fruits/
│   ├── FruitSelector.svelte
│   ├── FruitList.svelte
│   └── FruitCard.svelte
├── observations/
│   ├── ObservationForm.svelte
│   ├── ObservationCard.svelte
│   ├── ObservationList.svelte
│   └── ObservationDetail.svelte
└── common/
    ├── Navigation.svelte
    ├── Footer.svelte
    ├── Loading.svelte
    └── ErrorMessage.svelte
```

---

## Development Setup

### Prerequisites
- Node.js 18+ 
- npm or pnpm
- Git

### Initial Setup
```bash
# Clone repository
git clone <repository-url>
cd fruit-watching-app

# Install dependencies
npm install

# Copy environment template
cp .env.example .env

# Edit .env with your credentials
# - WIKIMEDIA_CLIENT_ID
# - WIKIMEDIA_CLIENT_SECRET
# - DATABASE_URL (if using PostgreSQL)

# Run database migrations
npm run db:push

# Seed initial fruit data
npm run seed

# Start development server
npm run dev
```

### Environment Variables
```env
# Wikimedia OAuth
WIKIMEDIA_CLIENT_ID=
WIKIMEDIA_CLIENT_SECRET=
WIKIMEDIA_CALLBACK_URL=http://localhost:5173/auth/callback

# Database
DATABASE_URL=file:./data/app.db

# Session
SESSION_SECRET=generate-a-random-secret

# Optional: External API keys
MAPBOX_TOKEN= # If using Mapbox for maps
```

### Development Commands
```bash
npm run dev          # Start dev server
npm run build        # Build for production
npm run preview      # Preview production build
npm run check        # Type-check
npm run lint         # Lint code
npm run format       # Format code
npm run db:push      # Push schema changes to DB
npm run db:studio    # Open Drizzle Studio
npm run seed         # Seed database with fruits
```

---

## Testing Strategy

### Unit Tests
- Authentication utilities
- Wikidata query formatting
- Data validation functions

### Integration Tests
- API endpoints
- Database operations
- OAuth flow (mocked)

### E2E Tests (Optional but Recommended)
- Complete observation creation flow
- Market search and selection
- Authentication flow

**Testing Tools:**
- Vitest for unit tests
- Playwright for E2E tests

---

## Deployment Checklist

### Pre-deployment
- [ ] Set production environment variables
- [ ] Run database migrations on production DB
- [ ] Seed production database with fruits
- [ ] Test OAuth callback with production URL
- [ ] Enable HTTPS
- [ ] Configure CORS if needed
- [ ] Set up error logging (Sentry, etc.)

### Production Environment Variables
```env
WIKIMEDIA_CLIENT_ID=<production-client-id>
WIKIMEDIA_CLIENT_SECRET=<production-client-secret>
WIKIMEDIA_CALLBACK_URL=https://yourdomain.com/auth/callback
DATABASE_URL=<production-database-url>
SESSION_SECRET=<strong-random-secret>
```

### Deployment Platforms
- **Vercel:** Easiest for SvelteKit, automatic deployments
- **Netlify:** Similar to Vercel, good SvelteKit support
- **Railway/Render:** Good for apps needing persistent database
- **Self-hosted:** VPS with Node.js, nginx, and PM2

---

## Performance Considerations

### Optimization Strategies
1. **Database Indexing**
   - Index `user_id`, `market_id`, `observation_date`
   - Index `wikidata_id` for fruit lookups

2. **Caching**
   - Cache Wikidata fruit list (update daily)
   - Cache OSM market searches (short TTL)
   - Use SvelteKit's load cache

3. **API Rate Limiting**
   - Respect Nominatim usage policy (1 req/sec)
   - Respect Wikidata query service limits
   - Implement client-side debouncing

4. **Frontend Optimization**
   - Lazy load large fruit lists
   - Virtualize long lists (if >100 items)
   - Optimize images (if added later)
   - Use SvelteKit's code splitting

---

## Security Considerations

### Authentication
- Use HTTP-only secure cookies
- Implement CSRF protection
- Validate OAuth state parameter
- Secure session secrets

### Data Validation
- Validate all user inputs
- Sanitize data before database insertion
- Use parameterized queries (ORM handles this)

### API Security
- Rate limiting on all endpoints
- Validate user owns resources before modifications
- Implement proper authorization checks

### Privacy
- Allow users to make observations private
- Don't expose user emails or sensitive data
- GDPR compliance (if EU users)

---

## Future Enhancements (Out of Current Scope)

These are explicitly NOT part of the initial scope but could be added later:

1. **Analytics**
   - Fruit seasonality trends
   - Market fruit diversity scores
   - Geographic distribution maps

2. **Social Features**
   - Follow other observers
   - Comment on observations
   - Fruit identification help requests

3. **Media Support**
   - Photo uploads
   - Image galleries

4. **Advanced Features**
   - Export data (CSV, JSON)
   - API for third-party integrations
   - Mobile app (React Native, Flutter)
   - Offline support (PWA)

---

## Time Estimates

### Total Development Time: 40-60 hours

| Phase | Tasks | Estimated Time |
|-------|-------|----------------|
| Phase 1: Setup | Project init, database schema | 4-7 hours |
| Phase 2: Auth | OAuth implementation | 5-7 hours |
| Phase 3: Markets | OSM integration, UI | 6-9 hours |
| Phase 4: Fruits | Wikidata integration, UI | 6-8 hours |
| Phase 5: Observations | Core feature, CRUD, sharing | 12-16 hours |
| Phase 6: Dashboard | Home page, navigation | 5-7 hours |
| Testing & Polish | Bug fixes, refinements | 4-8 hours |

---

## Success Criteria

The app is complete when a user can:

1. ✅ Log in using their Wikimedia account
2. ✅ Search for and save a market using OpenStreetMap
3. ✅ Browse a list of fruits from Wikidata
4. ✅ Create an observation by selecting:
   - A market
   - A date
   - Multiple fruits found
   - Optional notes
5. ✅ View their past observations
6. ✅ Share an observation via a public link
7. ✅ Log out

**Nothing else is required for v1.0.**

---

## Getting Help

### Documentation Resources
- [SvelteKit Docs](https://kit.svelte.dev/docs)
- [Wikimedia OAuth Guide](https://www.mediawiki.org/wiki/OAuth/For_Developers)
- [OpenStreetMap Nominatim API](https://nominatim.org/release-docs/latest/api/Overview/)
- [Wikidata Query Service](https://www.wikidata.org/wiki/Wikidata:SPARQL_query_service)
- [Drizzle ORM Docs](https://orm.drizzle.team/docs/overview)

### API Playgrounds
- [Wikidata Query Service UI](https://query.wikidata.org/)
- [Nominatim Search](https://nominatim.openstreetmap.org/)

---

## Development Workflow

### Git Workflow
1. Create feature branches: `feature/market-search`, `feature/oauth`
2. Make small, focused commits
3. Write descriptive commit messages
4. Create pull requests for review
5. Merge to main after testing

### Code Style
- Use TypeScript strict mode
- Follow Svelte best practices
- Use Prettier for formatting
- Use ESLint for linting
- Write JSDoc comments for complex functions

### Documentation
- Update README.md with setup instructions
- Document environment variables
- Add inline comments for complex logic
- Create API documentation (optional)

---

## Conclusion

This guide provides a complete roadmap for building a focused, eBird-inspired fruit watching application using SvelteKit. The emphasis is on simplicity and a single, well-executed feature set: tracking fruits found at markets.

Start with Phase 1 (setup) and work sequentially through each phase. The modular structure allows for parallel development if multiple developers are involved.

Remember: **Resist scope creep.** The app should do one thing well: help users track fruits at markets. Analytics, social features, and media support can come later.

Happy coding! 🍎🍊🍌
