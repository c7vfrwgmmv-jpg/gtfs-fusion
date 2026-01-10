# GTFS Fusion - Code Documentation

This document contains detailed documentation about the code architecture, design decisions, and implementation details extracted from the source code.

## Data Processing Pipeline Overview

The GTFS data flows through several processing stages:

### 1. PARSE (CSV → Raw Objects)
- ZIP extraction & CSV parsing
- Worker-based parallel parsing for performance
- Produces raw JavaScript objects

### 2. NORMALIZE (Raw → Standardized)
- normalizeKey: CSV headers → GTFS standard field names
- normalizeRecord: Clean values (trim, remove quotes, handle nulls)
- Output: Clean GTFS-compliant records

### 3. INDEX (Records → Lookup Tables)
- Build indexes: stopTimesIndex, stopsIndex, shapesIndex, etc.
- Group related data for fast lookup
- Output: Queryable data structures

### 4. ENRICH (Add Computed Fields)
- enrichTripsWithDirectionId: Generate missing direction_id
- Shape simplification: Reduce coordinate density
- Route pattern analysis: Build canonical trip patterns
- Output: Enhanced domain model

### 5. CACHE (Persist Processed Data)
- cacheShapes: Store simplified shapes in localStorage
- Compression: Delta encoding + base36
- TTL: 30 days
- Output: Faster subsequent loads

### 6. RENDER (Display in UI)
- Map visualization (Leaflet)
- Timetable generation
- escapeHtml: Sanitize for safe HTML rendering
- Output: Interactive web interface

## Function Purity

- **@pure** = No side effects, same input → same output (can be memoized)
- **@impure** = Has side effects (DOM updates, localStorage, mutations)
- **@cached** = Results are memoized internally
- **@presentation** = UI-specific logic (not domain logic)

## Module Organization

Functions are organized by concern:

- **3.1**: Data Normalization (CSV/GTFS cleaning)
- **3.2**: Time/Date (GTFS temporal logic)
- **3.3**: Geometry/Geography (WGS84 calculations)
- **3.4**: Cache/Serialization (localStorage persistence)
- **3.5**: Data Quality (Heuristics for feed quality)
- **3.6**: Direction Enrichment (direction_id generation)

---

## 3.1. Data Normalization Module

**Pipeline**: CSV/GTFS → normalized keys/values → domain model

### Questions Addressed:
- normalizeKey is deterministic and globally cacheable (keyCache)
- normalizeRecord handles CSV-level normalization (quotes, whitespace)
- escapeHtml is UI-layer sanitization (separate concern)
- KEY_ALIAS provides GTFS field name standardization
- Future: Could support per-feed alias injection via config

---

## 3.2. Time and Date Module (GTFS-specific)

**Internal representation**: Time is ALWAYS stored as minutes since midnight. This allows for GTFS times >24h (e.g., "25:30:00" for late-night service)

### Questions Addressed:
- Internal time format: minutes since midnight (0-based, can exceed 1440)
- HH:MM:SS >24h is domain logic (GTFS spec allows it for service after midnight)
- formatTime handles presentation layer (wraps at 24h for display)
- GTFS dates are always in local timezone (YYYYMMDD format, no TZ info)
- Date parsing/formatting is bidirectional (GTFS ↔ Date object)

---

## 3.3. Geometry and Geography Module

**Coordinate system**: WGS84 (latitude/longitude in decimal degrees). All geographic calculations assume WGS84 datum (implicit assumption)

### Questions Addressed:
- Coordinate system: WGS84 everywhere (GTFS standard)
- Distance tolerance: 100m hardcoded for stop-shape matching (heuristic)
- Shape simplification: Part of parsing/caching, not rendering
- Simplification tolerance: 0.0001° (~11m) is configurable via parameter
- Coverage calculation: Heuristic for quality assessment, not strict validation
- These functions are GTFS-agnostic and could be reused for other geo apps

---

## 3.4. Cache and Serialization Module

**Cache backend**: localStorage (browser-based persistent storage)  
**Cache strategy**: Content-addressable via hash, with TTL and versioning

### Questions Addressed:
- Cache backend: localStorage (suitable for client-side caching)
- Hash collision resistance: Simple hash (good enough for cache keys, not cryptographic)
- TTL: 30 days (arbitrary but reasonable for transit data staleness)
- Versioning: Explicit version field (v2 = compressed format)
- Cache scope: Currently shapes only, but design is generic (could extend to routeProfiles)
- Compression: Delta encoding + base36 (stable, lossy to 6 decimal places ≈ 0.11m precision)
- Cleanup: Best-effort on QuotaExceededError (not deterministic)

---

## 3.5. Data Quality Heuristics Module

**Quality scoring and validation for GTFS feed data**

### Questions Addressed:
- These heuristics are GTFS-specific (rely on GTFS naming conventions)
- "Garbage label" is domain logic (determines if route name is usable)
- short_name vs long_name: implicit scoring via looksLikeGarbageLabel
- Heuristics are currently hardcoded but could be made configurable
- These rules encode domain knowledge about transit data quality

---

## 3.6. Direction_ID Enrichment Module

**Pipeline for generating direction_id when missing from GTFS feed**  
**Decision pipeline**: Group by route → Find patterns → Score sequences → Assign directions

### Questions Addressed:
- direction_id is treated as binary classification (0/1) but handles edge cases
- Routes with >2 directions: Uses heuristics (longest trip as reference pattern)
- Circular routes: Detected via first==last stop, handled specially
- Subsequence matching: Ignores branch-specific stops, focuses on common trunk
- Bearing as tiebreaker: Used when sequence scores are identical
- Incomplete stopsIndex: Gracefully degrades (trips without stops get direction_id=0)
- Determinism: Stable within a feed, but may vary between feeds with different data
- This is a decision pipeline, not a single function (multi-stage enrichment)

---

## 4. Route Profile Building System

**Purpose**: Build canonical route patterns to handle route variations and branches

The route profile building system analyzes all trips for a route/direction to identify:
- **Core stops**: Stops served by ≥10% of trips (stable trunk of the route)
- **Passenger branches**: Low-frequency stops that serve passengers
- **Depot/tail stops**: Non-passenger stops (pickup_type=1 or drop_off_type=1)

### Algorithm Overview:

#### Step 1: Station-Level Edge Frequency Analysis
- Groups individual stops into stations (parent_station or logical groups)
- Counts station-to-station edges across all trips
- Identifies core stations based on frequency threshold (10%)

**Why stations instead of stops?**
- Handles multiple platforms at the same station
- Avoids treating platform 1 and platform 2 as separate route segments
- Prevents circular routes within a station from breaking the analysis

#### Step 2: Core Boundary Detection
- For each non-core stop, computes maximum frequency of edges connecting to core
- This helps identify which stops are close to the main route trunk

#### Step 3: Recursive Tail Classification
**Three-tier classification system:**

1. **Test 1 - Position Test**: Does the stop appear BETWEEN core start and core end?
   - If YES → classify as `passenger` (it's a branch that serves passengers)
   - If NO → continue to next test

2. **Test 2 - Pickup/Dropoff Flags**:
   - If ANY occurrence has `pickup_type=1` OR `drop_off_type=1` → classify as `tail` (depot/non-passenger)
   - These are GTFS flags indicating the stop is not for regular passenger service

3. **Test 3 - Recursive Propagation**:
   - If a stop has normal flags (0/0) but appears on trips that have other tail stops → classify as `tail`
   - This recursively propagates tail classification to depot approach/exit segments

**Why recursive?**
- Depot approach routes may have multiple stops before reaching the depot
- Without recursion, only the final depot stop would be marked as tail
- Recursive propagation ensures entire depot segments are classified correctly

### Stop Merging at Station Level

**Station Merge Policy:**
- Stops are merged into stations when they share the same `parent_station`
- Merging is only allowed for stations that appear at most once per trip
- This prevents circular routes from collapsing into a single node

**Circular Route Handling:**
- If a station appears multiple times in a trip (e.g., loop routes), merging is disabled
- Each occurrence is kept separate to preserve the route topology

---

## 5. Logical Route Grouping

**Purpose**: Group related GTFS routes into logical lines for better UX

### Grouping Strategy:
Routes are grouped by:
1. **Agency ID** - Routes from the same operator
2. **Canonical Route Type** - E.g., all metros together, all buses together
3. **Route Short Name** - Routes with the same number/identifier

### Canonical Route Type Mapping:
- Maps GTFS `route_type` (including extended types 100-1300) to canonical categories
- Categories include: metro, tram, bus, rail, ferry, etc.
- Prevents mixing different transport modes with the same route number
  - Example: S1 S-Bahn (rail) vs S1 bus are kept separate

### Sorting:
1. First by canonical type order (metro → tram → bus → rail → ...)
2. Then by route short name (numeric-aware: "2" < "10" < "100")

---

## 6. Timetable Rendering with Canonical Master Lists

**Problem**: Timetable rows should remain stable when user changes filters (date, showAllTrips)

### Solution: Canonical Master Lists
- Built once per route/direction from ALL trips (no date filter)
- Stores the definitive row order based on route topology
- Cached in `state.canonicalMasterLists[key][direction]`

### Two-Axis Rendering:
1. **Rows (Y-axis)**: Use canonical master list (stable, never reordered)
2. **Columns (X-axis)**: Sort trips using adjacent-swaps algorithm (changes with filters)

### Row Visibility Filtering:
- Master list rows are FILTERED but never REORDERED
- A row is visible if ANY trip on the current date stops there
- Hidden rows are skipped during rendering (collapsed out)

### Column Ordering (Adjacent-Swaps Algorithm):
- Minimizes visual distance between similar trip patterns
- Orders trips to create smooth visual flow in the timetable
- Separates service trips (with core passenger stops) from depot trips
- Cached per (key, direction, date, showAllTrips) combination

**Why separate service/depot trips?**
- Service trips: Ordered by pattern similarity (user-facing timetable)
- Depot trips: Ordered by first core departure time (operations-focused)
- Prevents depot movements from disrupting passenger timetable readability

---

## 7. Map Rendering Hierarchy

**Route visualization on the map follows a hierarchy for geometry sources:**

### Hierarchy (best to worst):
1. **shapes.txt from GTFS** (highest quality)
   - Official route geometry from the transit agency
   - Accurately follows roads and track alignments
   
2. **Fallback: Straight lines between stops**
   - Generated when shapes.txt is missing
   - Simple but always available

### Why this hierarchy?
- shapes.txt provides the most accurate representation
- Straight lines are always available but less precise
- System gracefully degrades based on data availability

---

## 8. View State Management

The application has 6 distinct views, controlled by state inspection:

### View Precedence (checked in order):
1. **LOADING** - if `state.loading === true`
2. **ERROR** - if `state.error` is set
3. **UPLOAD** - if no GTFS data loaded
4. **STOP_DETAIL** - if showing stops view and a stop is selected
5. **STOPS_LIST** - if showing stops view (no route selected)
6. **ROUTES_LIST** - if no route selected (default view)
7. **ROUTE_TIMETABLE** - if a route is selected

### State Transitions:
- Back button: Navigates up the hierarchy (STOP_DETAIL → STOPS_LIST → ROUTES_LIST)
- Route selection: ROUTES_LIST → ROUTE_TIMETABLE
- Stop selection: STOPS_LIST → STOP_DETAIL
- View toggle: Switches between ROUTES_LIST ↔ STOPS_LIST

### Map State Cleanup:
- Map instances are reset when changing views to prevent memory leaks
- Layer state is tracked separately for each view (stop detail vs route timetable)

---

## 9. Performance Optimizations

### 9.1. Worker-Based Parsing
- Large CSV files (stop_times.txt, shapes.txt) are parsed in Web Workers
- Prevents UI blocking during data processing
- Progress updates are throttled to max 5 FPS to avoid excessive DOM updates

### 9.2. Caching Strategies

**Shape Cache (localStorage):**
- Simplified shapes compressed with delta encoding + base36
- 30-day TTL with version field for cache invalidation
- Reduces parsing time on subsequent loads

**Canonical Master List Cache:**
- Route topology cached in memory per route/direction
- Rebuilt only when route or direction changes
- Enables instant timetable updates when changing date

**Column Order Cache:**
- Trip ordering cached per (route, direction, date, showAllTrips)
- Adjacent-swaps algorithm is expensive - caching prevents recomputation
- Cache key includes all variables that affect trip filtering

### 9.3. Virtual Scrolling (Stops List)
- Only renders visible stop groups in the viewport
- Reduces DOM nodes for feeds with thousands of stops
- Dynamically measures item height for accurate scrolling

### 9.4. Memoization
- `normalizeKey` results cached in Map (called millions of times during parsing)
- Station name-to-ID cache for stop merging lookups
- Route type metadata cached to avoid repeated canonical type lookups

---

## 10. Data Quality Heuristics

### 10.1. Garbage Label Detection
**Purpose**: Identify and filter out uninformative route names

**Rules:**
- Empty or very short labels (< 2 chars) → garbage
- Numeric-only labels that duplicate the short_name → garbage
- Used to prefer `route_long_name` over `route_short_name` when more informative

### 10.2. Stop Edge Frequency Tiers
**Purpose**: Classify stops by how essential they are to the route

**Tier 1 (< 6% frequency):**
- Very low frequency stops
- Likely depot approaches or special event service
- Strongest candidates for tail classification

**Tier 2 (6-10% frequency):**
- Low frequency stops
- May be passenger branches or seasonal service
- Moderate candidates for tail classification

**Threshold (≥ 10%):**
- Core stops that define the main route trunk
- Always shown in timetable
- Used as reference points for other stop classification

---

## 11. Security Considerations

### HTML Escaping
- All user-generated content is escaped via `escapeHtml()` before rendering
- Prevents XSS attacks from malicious GTFS data
- Applied to: stop names, route names, headsigns, agency names, descriptions

### File Upload Safety
- Only accepts `.zip` files
- Validates file structure before processing
- Limits exposure to zip bomb attacks through streaming decompression

---

## 12. Internationalization Support

### Language
- UI text is currently in Polish (może być zmienione)
- Date/time formatting follows Polish conventions
- Route type labels use Polish names (Metro, Tramwaj, Autobus, etc.)

### Numeric Formatting
- Uses locale-aware sorting for route numbers (e.g., "2" < "10")
- Handles both numeric and alphanumeric route identifiers

---

## 13. Browser Compatibility

### Required Features:
- **Web Workers** - For background CSV parsing
- **localStorage** - For shape caching
- **Leaflet.js** - For map rendering (CDN loaded)
- **ES6+ features** - Arrow functions, template literals, Map/Set, async/await

### External Dependencies:
- **fflate** (0.8.1) - ZIP file decompression
- **Leaflet** (1.9.4) - Map rendering
- **Leaflet.markercluster** (1.5.3) - Stop clustering on map
- **Tailwind CSS** (3.4.17) - Styling via CDN
