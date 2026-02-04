# IMLGS - Index to Marine and Lacustrine Geological Samples

A read-only web application providing interactive access to marine and lacustrine geological sample data from NOAA's archived Index to Marine and Lacustrine Geological Samples (IMLGS) database.

This project was created to preserve access to the IMLGS data after the original NOAA service was decommissioned on 2025-05-05.

## Overview

The application allows users to:
- Browse and search geological sample records
- Filter by platform, device, repository, cruise, and sample identifiers (IMLGS ID or IGSN)
- View sample locations on an interactive map with "Zoom to Sample" button
- Auto-display sample details when searching by identifier (works in both Map and Table views)
- Access detailed sample metadata including stratigraphic intervals
- Link to external IGSN resolver for registered samples

## Architecture

```
imlgs_ro/
├── ui/                     # Observable Framework web application
│   ├── src/
│   │   ├── index.md        # Main application page
│   │   ├── lib/
│   │   │   └── common.js   # Core data access class (IMLGSData)
│   │   └── data/
│   │       └── imlgs_full.parquet  # Bundled dataset (~17MB)
│   ├── observablehq.config.js
│   └── package.json
├── cli/                    # Python CLI utilities
│   └── imlgs/
│       └── __main__.py     # CSV to Parquet conversion tool
├── data/                   # Source data files (original copy)
│   └── imlgs_full.parquet
└── .github/workflows/
    └── deploy.yml          # GitHub Pages deployment
```

### Technology Stack

| Component | Technology |
|-----------|------------|
| Frontend Framework | [Observable Framework](https://observablehq.com/framework/) |
| In-Browser Database | [DuckDB WASM](https://duckdb.org/docs/api/wasm/overview.html) |
| Mapping | [OpenLayers](https://openlayers.org/) with WebGL rendering |
| Data Format | Apache Parquet |
| Deployment | GitHub Pages |

### How It Works

1. **Data Loading**: The application loads a bundled Parquet file containing all IMLGS records (~17MB). The file is served directly from GitHub Pages.

2. **In-Browser SQL**: DuckDB WASM runs entirely in the browser, enabling SQL queries against the Parquet data without a backend server. Extensions for spatial queries (`spatial`) and H3 hexagonal binning (`h3`) are loaded automatically.

3. **Reactive Filtering**: User inputs (dropdowns, text fields) generate SQL WHERE clauses that filter records in real-time. The `IMLGSData` class in `common.js` manages query construction and execution. When filtering by Sample ID, the first matching record is automatically selected and its details displayed.

4. **Map Visualization**: Sample locations are rendered as WebGL points on an OpenLayers map, color-coded by repository. The map supports hover tooltips, click-to-select, viewport-based counting, and a "Zoom to Sample" button that zooms to fit all filtered samples (with a minimum 20 km extent).

5. **URL Parameters**: Filters can be applied via URL query parameters (e.g., `?platform=JOIDES&cruise=ODP191&search=NAU`).

## Getting Started

### Prerequisites

- Node.js 18+
- npm or yarn

### Local Development

1. Clone the repository:
   ```bash
   git clone https://github.com/smrgeoinfo/imlgs_ro.git
   cd imlgs_ro
   ```

2. Install dependencies:
   ```bash
   cd ui
   npm install
   ```

3. (Optional) Apply DuckDB patch for newer features:
   ```bash
   patch node_modules/@observablehq/framework/dist/duckdb.js ddb_132.patch
   ```

4. Start the development server:
   ```bash
   npm run dev
   ```

5. Open http://localhost:3000

### Data Source

The parquet data file (~17MB) is bundled with the application in `ui/src/data/imlgs_full.parquet`. This avoids CORS issues and makes the application self-contained.

### Building for Production

```bash
cd ui
npm run build
```

The static site is generated in `ui/dist/`.

## URL Parameters

Filter the data using URL query parameters:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `platform` | Ship/platform name (regex) | `?platform=JOIDES` |
| `device` | Sampling device type (regex) | `?device=piston` |
| `repository` | Repository/facility code (regex) | `?repository=OSU` |
| `cruise` | Cruise identifier (exact match) | `?cruise=ODP191` |
| `search` | IMLGS ID or IGSN (regex) | `?search=NAU0001` |

Parameters can be combined: `?platform=Atlantis&device=core&search=WHO`

## Data Schema

Key columns in the Parquet file:

| Column | Type | Description |
|--------|------|-------------|
| `imlgs` | VARCHAR | Unique IMLGS identifier (primary key) |
| `igsn` | VARCHAR | International Geo Sample Number |
| `sample` | VARCHAR | Sample identifier |
| `platform` | VARCHAR | Ship/platform name |
| `device` | VARCHAR | Sampling device type |
| `facility` | STRUCT | Repository info (`facility_code`, `facility`, `other_link`) |
| `cruise` | STRUCT | Cruise info (`cruise`) |
| `lat`, `lon` | DOUBLE | Sample location coordinates |
| `water_depth` | DOUBLE | Water depth in meters |
| `begin_jd` | DOUBLE | Collection date as Julian Date |
| `intervals` | ARRAY | Stratigraphic interval data |

## CLI Tools

The `cli/` directory contains Python utilities for data processing:

```bash
cd cli
pip install -e .

# Convert CSV to Parquet (example usage)
python -m imlgs toparquet input.csv output.parquet
```

---

## Agent Instructions

This section provides context for AI agents working with this codebase.

### Project Purpose

This is a static web application that provides searchable access to archived NOAA geological sample data. The entire application runs in the browser with no backend server - data queries are executed using DuckDB WASM against a Parquet file.

### Key Files

| File | Purpose |
|------|---------|
| `ui/src/index.md` | Main application page - contains all UI components, map setup, filtering logic, and table configuration |
| `ui/src/lib/common.js` | Core `IMLGSData` class - handles database initialization, query building, and data access methods |
| `ui/observablehq.config.js` | Framework configuration including DuckDB extensions |

### Code Patterns

**Filter/Observer Pattern**: Each filter input uses an observer pattern:
```javascript
const [_input_element, input_observer] = await imlgs_data.newInputObserver(
    "column_name",     // Database column
    "Label",           // UI label
    "sql_template=?",  // SQL WHERE template with ? placeholder
    "url_param"        // URL query parameter name
);
```

**WHERE Clause Building**: The `getWhereClause()` method in `IMLGSData` combines all active filters:
- Templates can have multiple `?` placeholders (value is repeated for each)
- Clauses are joined with `AND`
- Empty values are excluded

**Data Access Methods**:
- `imlgs_data.getDisplayRecords(whereClause)` - Get filtered table records
- `imlgs_data.count(whereClause)` - Count matching records
- `imlgs_data.getRecord(imlgs_id)` - Get full record by ID
- `imlgs_data.distinct(column)` - Get unique values for dropdowns

### Adding New Filters

1. Create an observer in `index.md`:
   ```javascript
   const [_new_input, new_input] = await imlgs_data.newTextInputObserver(
       "column_name",
       "Display Label",
       "column_name=?",
       "url_param"
   );
   ```

2. Display the input: `display(_new_input);`

3. Add to inputs array: `ui_inputs.push(new_input);`

For multi-column search (OR logic), use multiple `?` in the template:
```javascript
"(regexp_matches(col1,?,'i') OR regexp_matches(col2,?,'i'))"
```

### Map Layer

The map uses OpenLayers with WebGL for performance:
- Points are loaded via `loadParquetLayer()`
- Features store `imlgs` as ID, `repository` for coloring
- `zoomToPid(id)` centers map on a sample
- `countSamplesInPolygon()` counts visible samples

### Common Tasks

**Modify table columns**: Edit `display_fields` array in `index.md`

**Add new record detail field**: Edit `getRecordHtml()` in `common.js`

**Change map styling**: Modify `makeColors()` and WebGL style in `index.md`

**Add DuckDB extension**: Update `duckdb.extensions` in `observablehq.config.js`

### Dependencies

The application relies on:
- `@observablehq/framework` - Build system and runtime
- `@duckdb/duckdb-wasm` - In-browser SQL database
- `ol` (OpenLayers) - Mapping library
- DuckDB extensions: `spatial`, `h3`

### Testing Changes

1. Run `npm run dev` in `ui/` directory
2. Open http://localhost:3000
3. Check browser console for errors
4. Test filter combinations and URL parameters

### Deployment

Pushes to `main` branch trigger GitHub Actions workflow that:
1. Installs dependencies
2. Applies DuckDB patch
3. Builds static site
4. Deploys to GitHub Pages
