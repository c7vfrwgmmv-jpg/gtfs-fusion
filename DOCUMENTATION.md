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
