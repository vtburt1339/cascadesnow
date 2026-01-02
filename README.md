# Cascade Snow Forecast Viewer

Vue 3 app that visualizes National Weather Service forecast zones for the Oregon Cascades, highlighting snow level bands on a Leaflet map and summarizing the next 8 forecast periods.

## Features
- Two forecast zones (ORZ127, ORZ128) with quick switching.
- Snow level extraction from forecast text and matching overlay polygons.
- Interactive graph for snow level and precipitation chance.
- Snow-themed favicon and styled UI optimized for desktop and mobile.

## Data Sources
- Forecast data: https://api.weather.gov/zones/forecast/{zoneId}/forecast
- Map tiles: https://toposm.ahlzen.com/tiles/hikemap_composite_20240618/{z}/{x}/{y}.png
- Threshold overlays: `public/thresholds/*.geojson`

## Requirements
- Node.js 16+ (or current LTS recommended)
- pnpm

## Development
```bash
pnpm install
pnpm run serve
```

## Build
```bash
pnpm run build
```

## Lint
```bash
pnpm run lint
```

## Notes
- The app requests the NWS API directly from the browser; if rate limits or CORS restrictions change, proxying may be needed.
- Overlay file names must match extracted snow levels (in feet) from the forecast text.
