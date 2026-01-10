# GTFS Fusion

A single-page web application for visualizing and analyzing GTFS (General Transit Feed Specification) data.

## Quick Start

1. Open `gtfs-fusion.html` in a modern web browser
2. Upload a GTFS `.zip` file
3. Explore routes, stops, and timetables with an interactive interface

## Features

- 📊 **Route Timetables** - Visual timetable with trip variants
- 🗺️ **Interactive Maps** - Route visualization with shapes and stops
- 🚏 **Stop Details** - Departure boards for individual stops
- 🔍 **Smart Grouping** - Logical route grouping by service type
- ⚡ **Performance** - Web Worker-based parsing, localStorage caching
- 🎯 **Route Analysis** - Automatic direction detection and route profiling

## Documentation

The project documentation is organized into separate files:

### 📘 [DOCUMENTATION.md](./DOCUMENTATION.md)
**Technical documentation and architecture guide**

Contains detailed information about:
- Data processing pipeline (Parse → Normalize → Index → Enrich → Cache → Render)
- Function purity annotations and module organization
- Route profile building algorithm
- Timetable rendering with canonical master lists
- Performance optimizations and caching strategies
- Security considerations
- And much more...

**Read this if you want to:**
- Understand how the code works
- Contribute to the project
- Learn about the algorithms used
- Understand the design decisions

### 📝 [REFACTORING.md](./REFACTORING.md)
**Exploratory refactoring notes and architectural analysis**

Contains Q&A-style analysis of the codebase:
- Normalization pipeline questions and answers
- Time/date handling decisions
- Geometry and geography module design
- Cache and serialization strategies
- Data quality heuristics
- Direction ID enrichment logic
- Module boundaries and responsibilities

**Read this if you want to:**
- Understand the reasoning behind design decisions
- See potential future improvements
- Learn about the exploratory refactoring process

### 💻 [gtfs-fusion.html](./gtfs-fusion.html)
**The application itself**

A single-file HTML application with embedded CSS and JavaScript.
- No build process required
- Just open in a browser
- All documentation moved to separate files for cleaner code

## Technology Stack

- **Vanilla JavaScript** (ES6+)
- **Leaflet.js** - Map rendering
- **fflate** - ZIP decompression
- **Tailwind CSS** - Styling
- **Web Workers** - Background data processing

## Browser Requirements

- Modern browser with ES6+ support
- Web Workers support
- localStorage support
- Recommended: Chrome, Firefox, Edge, Safari (latest versions)

## File Structure

```
gtfs-fusion/
├── gtfs-fusion.html    # Main application (single-page app)
├── DOCUMENTATION.md    # Technical documentation
├── REFACTORING.md      # Architectural analysis
└── README.md           # This file
```

## Usage Tips

1. **Upload GTFS Data**: Click the upload button and select a GTFS `.zip` file
2. **Browse Routes**: View routes grouped by service type (metro, tram, bus, etc.)
3. **View Timetables**: Click a route to see its timetable and map
4. **Explore Stops**: Switch to stops view to find departures from specific stops
5. **Filter by Date**: Use the date selector to see service for specific days

## Performance Notes

- First load: Parses and indexes the GTFS data
- Subsequent loads: Uses cached shapes (30-day TTL)
- Large feeds: Uses Web Workers for non-blocking parsing
- Map rendering: Automatically uses shapes.txt if available

## Contributing

When contributing, please:
1. Read [DOCUMENTATION.md](./DOCUMENTATION.md) to understand the architecture
2. Follow existing code style and patterns
3. Keep the single-file nature of gtfs-fusion.html
4. Update documentation when adding new features
5. Test with various GTFS feeds

## License

[Add license information here]

## Acknowledgments

- Built for transit enthusiasts and developers
- Uses the GTFS specification from Google
- Inspired by the need for better GTFS visualization tools
