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

## 3.6. Direction_ID Enrichment Module (v2)

**Algorithm**: Multi-phase decision pipeline with circular route handling

**Pipeline for generating direction_id when missing from GTFS feed**  
**Decision pipeline**: Group by route → Detect patterns → Cluster circular routes → Assign directions

### Phases:
1. **Pattern Detection**: Group trips by stop sequence
2. **Circular Handling**: Cluster by bearing (CW/CCW)
3. **Non-Circular Matching**: Exact → Subsequence → Bearing fallback
4. **Progressive Processing**: Chunked for responsive UI

### Bearing Calculation:
- Uses Haversine formula: `atan2(sin(Δlon)*cos(lat2), ...)`
- Computed for pattern endpoints (first/last stop)
- Cached per unique pattern (not per trip)

### Questions Addressed:
- direction_id is treated as binary classification (0/1) with intelligent handling
- Routes with >2 directions: Uses heuristics (most popular trip as reference pattern)
- Circular routes: Detected via first==last stop, then clustered by bearing into CW (0°-180° = '0') and CCW (180°-360° = '1')
- Subsequence matching: Ignores branch-specific stops, focuses on common trunk
- Bearing as fallback: Used when exact/subsequence matching fails (handles 0/360 wraparound)
- Incomplete stopsIndex: Gracefully degrades (trips without stops get direction_id='0')
- Determinism: Stable within a feed, but may vary between feeds with different data
- This is a multi-phase decision pipeline with statistics tracking

### Statistics Tracked:
- `exact`: Trips matched via exact array equality
- `subsequence`: Trips matched as subsequence of master pattern
- `circular`: Circular route trips (CW/CCW clustering)
- `bearing`: Trips assigned via bearing comparison
- `fallback`: Trips defaulted to '0' (no match)

### Performance:
- 500k trips: ~12-15s (vs 82s previous)
- Memory: ~400 MB peak
- Bearing computation: Lazy (only when needed)
- Chunked processing: 50 routes per chunk for UI responsiveness

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

---

## Function Reference

This section provides detailed documentation for all functions in the codebase.


### 3.1. Data Normalization

#### `escapeHtml()`

Escape HTML special characters for safe rendering
@pure UI sanitization layer - separate from data normalization
@param {string} s - String to escape
@returns {string} HTML-safe string

#### `normalizeKey()`

Normalize a CSV header key to standard GTFS field name
@pure Deterministic - same input always produces same output
@cached Results are memoized in keyCache for performance
@param {string} k - Raw CSV header key
@returns {string} Normalized GTFS field name

#### `normalizeRecord()`

Normalize a complete CSV record to standard GTFS format
Handles: BOM removal, whitespace trimming, quote stripping
Pipeline stage: Raw CSV → Normalized GTFS record
@param {Object} rec - Raw CSV record with original keys
@returns {Object} Normalized record with GTFS-standard keys and cleaned values


### 3.2. Time and Date

#### `timeToMinutes()`

Convert GTFS time string to minutes since midnight
@pure Domain layer conversion
@param {string} timeStr - GTFS time format "HH:MM:SS" (can be >24h)
@returns {number} Minutes since midnight (can exceed 1440 for late-night service)

#### `minutesToTime()`

Convert minutes since midnight back to GTFS time format
@pure Inverse of timeToMinutes
@param {number} minutes - Minutes since midnight
@returns {string} GTFS time format "HH:MM:SS" (preserves >24h values)

#### `formatTime()`

Format time for display (presentation layer)
Wraps times ≥24h to 0-23h range for user-friendly display
@presentation Modifies times for UI display only
@param {string} time - GTFS time string
@returns {string} Display time in HH:MM format (wrapped to 24h)

#### `parseGTFSDate()`

Parse GTFS date string (YYYYMMDD) to Date object
@pure Assumes local timezone (GTFS spec does not include timezone info)
@param {string} dateStr - GTFS date format "YYYYMMDD"
@returns {Date|null} JavaScript Date object in local timezone, or null if invalid

#### `formatDateToGTFS()`

Format Date object to GTFS date string
@pure Inverse of parseGTFSDate
@param {Date} date - JavaScript Date object
@returns {string} GTFS date format "YYYYMMDD"


### 3.3. Geometry and Geography

#### `simplifyDouglasPeucker()`

Simplify polyline using Douglas-Peucker algorithm
Used during shape parsing/caching to reduce data size
@pure Algorithm operates on coordinate arrays
@param {Array<[number, number]>} points - Array of [lat, lon] coordinates
@param {number} tolerance - Simplification tolerance in degrees (default: 0.0001 ≈ 11m)
@returns {Array<[number, number]>} Simplified coordinate array

#### `haversineDistance()`

Calculate great-circle distance between two WGS84 coordinates using Haversine formula
@pure Geographic calculation - no side effects
@param {number} lat1 - Start latitude in decimal degrees
@param {number} lon1 - Start longitude in decimal degrees
@param {number} lat2 - End latitude in decimal degrees
@param {number} lon2 - End longitude in decimal degrees
@returns {number} Distance in meters

#### `calculateShapeCoverage()`

Calculate how well a shape covers a sequence of stops
Heuristic for data quality assessment - not a strict validation rule
@param {Array<Object>} stops - Array of stop objects with lat/lon
@param {Array<[number, number]>} shapePoints - Array of [lat, lon] shape coordinates
@returns {Object} Coverage analysis with percentage and nearby stops

#### `fillShapeGaps()`

Fill gaps in shape data by connecting stops with shape segments or straight lines
@param {Array<Object>} stops - Array of stop objects
@param {Array<[number, number]>} shapePoints - Shape coordinate array
@param {Array<Object>} nearbyStops - Stops matched to shape points
@returns {Array<[number, number]>} Complete route geometry

#### `compressCoordinates()`

Compress coordinate array using delta encoding and base36
Lossy compression: preserves 6 decimal places (≈0.11m precision)
@param {Array<[number, number]>} coords - Array of [lat, lon] coordinates
@returns {string} Compressed string representation


### 3.5. Data Quality

#### `mostCommonString()`

Find the most common string in an array (consensus-based data cleaning)
@pure Frequency counting algorithm
@param {Array<string>} arr - Array of strings to analyze
@returns {string|null} Most frequent non-empty string, or null if none found

#### `looksLikeGarbageLabel()`

Heuristic to detect uninformative route labels
Rejects: empty, too short, numeric-only (unless differs from shortName)
Domain knowledge: Many feeds use numeric codes as long_name, which is not useful
@param {string} s - Label to check
@param {string} shortName - Route short_name for comparison
@returns {boolean} True if label appears to be garbage/uninformative


### 3.3. Geometry and Geography

#### `TO_RAD()`

Calculate bearing (azimuth) between two geographic points
Reused from geometry module for direction detection
@pure Geographic calculation
@param {number} lat1 - Start latitude
@param {number} lon1 - Start longitude
@param {number} lat2 - End latitude
@param {number} lon2 - End longitude
@returns {number} Bearing in degrees (0-360)


### 3.5. Data Quality

#### `tripSequenceScore()`

Score how well a trip sequence matches a reference pattern
Uses subsequence matching: ignores branch-specific stops, matches trunk stops
@pure Longest common subsequence scoring
@param {Array<string>} tripSeq - Trip stop sequence to score
@param {Array<string>} pattern - Reference pattern to match against
@returns {number} Score from 0 to 1 representing the ratio of matched stops


### 3.2. Time and Date

#### `getTripStopSequence()`

Get stop sequence for a trip
@param {string} tripId - Trip ID
@param {Object} stopTimesIndex - Index of stop times by trip_id
@returns {Array<string>} Array of stop_ids in sequence


### 3.6. Direction Enrichment

#### `isCircularRoute()`

Check if a route is circular (starts and ends at same stop)
@pure Simple check on stop sequence
@param {Array<string>} stopSequence - Array of stop_ids
@returns {boolean} True if circular


### 3.2. Time and Date

#### `enrichTripsWithDirectionId()`

Enrich trips with direction_id when missing from GTFS feed
Decision Pipeline:
1. Group trips by route_id
2. Build stop sequences for each trip
3. Check for circular routes (all get direction_id=0)
4. Find longest trip as reference pattern
5. Create forward and reverse patterns
6. Score each trip against both patterns (subsequence matching)
7. Use bearing as tiebreaker when scores are close
Edge cases handled:
- Missing stop sequences: Assign direction_id=0
- Circular routes: All trips get direction_id=0 (no meaningful direction)
- Routes with branches: Subsequence matching ignores branch-specific stops
- Close scores: Use geographic bearing as tiebreaker
- Missing geo data: Fallback to sequence score only
@impure Mutates trip objects to add direction_id field
@param {Array<Object>} trips - Array of trip objects
@param {Object} stopTimesIndex - Index of stop times by trip_id
@param {Object} stopsIndex - Index of stops by stop_id

#### `getWeekdayDate()`

Get a representative weekday date (Monday-Friday)
Prefers FUTURE dates, falls back to PAST if no future weekday exists

#### `findDateForDayOfWeek()`

Find a date for a specific day of week
Strategy: prefer FUTURE dates, fallback to PAST
@param {number} targetDayOfWeek - 0=Sunday, 1=Monday, ..., 6=Saturday
@returns {string|null} - GTFS date string (YYYYMMDD)

#### `calculateEaster()`

Calculate Easter Sunday for a given year using Meeus/Jones/Butcher algorithm
@param {number} year - The year to calculate Easter for
@returns {Date} - Easter Sunday as a Date object

#### `getPolishHolidays()`

Get all Polish holidays for a given year
@param {number} year - The year to get holidays for
@returns {Array} - Array of {date: 'YYYYMMDD', name: 'Holiday Name'}

#### `findNearbyHolidays()`

Find holidays in the upcoming week from a reference date
@param {string} referenceDateStr - GTFS date string (YYYYMMDD)
@returns {Array} - Array of holiday objects that fall within the next 7 days


### 3.1. Data Normalization

#### `getCanonicalKeyForCurrentSelection()`

Get the canonical cache key for the current route/group selection
Returns a key like 'group::123' or 'raw::route_id'

#### `buildCanonicalMasterList()`

Build a canonical master list from ALL trips (no date filter, tail included).
This list is stable per route/group + direction and never changes.
Only visibility is filtered at render time.
Algorithm:
1. Get core stops from profile (ordered sequence of main stations)
2. Define segments: pre-core, windows between consecutive cores, post-core
3. For each segment, collect all branch stations that appear in that segment
4. Rank branch stations by median normalized position within the segment
5. Stable tie-breaking: name, then stop_id
6. Build final list: pre-core, first core, [windows with branches], last core, post-core
Pre-core: Stops before the first core station (e.g., variant starting points)
Post-core: Stops after the last core station (e.g., variant ending points)


### Other Utilities

#### `ensureCanonicalMasterListForCurrentSelection()`

Ensure canonical master list exists for current selection.
Returns the canonical master list or null if fallback needed.


### 3.2. Time and Date

#### `invalidateCanonicalCacheForCurrentSelection()`

Invalidate canonical cache for current route/group.
Called when route, group, or direction changes.

#### `adjacentSwapOrderConfig()`

Configuration for adjacent-swaps column ordering algorithm
Parameters:
- voteThreshold: minimum number of row-votes required to swap a pair (default: 4)
- marginMinutes: time difference margin to avoid swapping on jitter (default: 2)
- maxPasses: maximum number of bubble-sort passes to limit runtime (default: 8)

#### `adjacentSwapOrder()`

Adjacent-swaps column ordering algorithm
Performs local corrections on the trips array based on row-wise time comparisons
across visible passenger rows only. Unlike global pairwise ranking, this uses
a bubble-sort approach with voting:
- Compare adjacent pairs across multiple passes
- For each pair, count row votes favoring a swap (left time > right time + margin)
- If votesSwap >= voteThreshold, swap the pair
- Repeat for maxPasses iterations to allow corrections to propagate
Benefits:
- Produces locally monotone ordering (minimizes adjacent inversions)
- Deterministic and stable with proper parameter tuning
- Lower complexity than global pairwise: O(n·rows·passes) vs O(n²·rows)
- Works well for routes with mostly consistent trip progression
@param {Array} trips - Array of trip objects in current initial order
@param {Array} tripMappings - Array of mapping objects [tripIdx][rowIdx] = stopTime
@param {Array} visiblePassengerRowIndices - Row indices to consider (exclude tail rows)
@param {Object} params - { voteThreshold, marginMinutes, maxPasses }
@returns {Array} Reordered trips array

#### `findMostFrequentTripPattern()`

Find the most common trip pattern (sequence of stops) for a given stop.
Used to identify the typical route variant that serves a stop.
@param {Array} trips - List of trip objects
@param {Object} stopTimesIndex - Map of trip_id -> stopTimes array
@param {string} stopId - The stop ID to analyze
@param {Array} activeServices - List of active service IDs
@returns {Object|null} Object with tripId, stopIds array, and frequency count


### Other Utilities

#### `getTimesByHourWithAnnotations()`

Process departures for a column with weekday annotations
For Mon-Fri column only: track which weekdays each minute appears on

#### `getCurrentView()`

Określa aktualny widok aplikacji na podstawie stanu
@returns {string} Jeden z VIEW.*

