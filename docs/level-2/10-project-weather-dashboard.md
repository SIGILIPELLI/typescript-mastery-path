# 10 · Project — Typed Weather Dashboard

The capstone for Level 2: a command-line weather dashboard that looks up
a city, fetches its current conditions and a multi-day forecast, and
prints a typed report — combining generics
([Module 2](02-generics.md)), utility types
([Module 5](05-utility-types.md)), typed async/await
([Module 6](06-async-await-types.md)), JSON validation
([Module 8](08-working-with-json-apis.md)), and Jest tests
([Module 7](07-testing-jest.md)) into one real project.

## The API: Open-Meteo (free, no key required)

This project uses [Open-Meteo](https://open-meteo.com/), a free weather
API that needs **no API key and no signup** — ideal for a learning
project you can run immediately. It has two endpoints we need:

- **Geocoding** — turn a city name into coordinates:
  `https://geocoding-api.open-meteo.com/v1/search?name=London&count=1`
- **Forecast** — turn coordinates into current + daily weather:
  `https://api.open-meteo.com/v1/forecast?latitude=...&longitude=...&current=...&daily=...&timezone=auto`

## What you'll build

A CLI that:

- Looks up a city name via the geocoding API
- Fetches current conditions and a multi-day forecast for that location
- Validates both API responses at runtime before trusting them
- Builds a typed `WeatherSummary` by combining the two responses
- Prints a readable report, and caches repeated city lookups
- Has a Jest test suite covering the pure data-shaping logic, with zero
  network calls in the tests themselves

## Project layout

```text
weather_dashboard/
    src/
        types.ts         -- shared interfaces for API responses & the dashboard's own types
        httpClient.ts     -- generic fetch-and-validate helper
        weatherCodes.ts   -- WMO weather code -> human-readable text
        geocode.ts        -- city name -> coordinates, with caching
        forecast.ts       -- coordinates -> current + daily forecast
        summary.ts        -- pure function combining location + forecast into a WeatherSummary
        display.ts        -- formatting the summary for the terminal
        index.ts          -- CLI entry point
    __tests__/
        weatherCodes.test.ts
        summary.test.ts
    tsconfig.json
    package.json
    jest.config.js
```

## package.json

```json
{
  "name": "weather-dashboard",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "start": "ts-node src/index.ts",
    "test": "jest"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "ts-node": "^10.9.0",
    "jest": "^29.7.0",
    "ts-jest": "^29.2.0",
    "@types/jest": "^29.5.0",
    "@types/node": "^22.0.0"
  }
}
```

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020", "DOM"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist"
  }
}
```

`"lib": [..., "DOM"]` is required here specifically for `fetch`'s type
declarations — Node itself has supported `fetch` natively since version
18, but its global type comes from `lib.dom.d.ts`, not from Node's own
types.

## src/types.ts — shapes for both API responses and the dashboard

```typescript
// Raw shapes returned by the Open-Meteo APIs -- only the fields we use.

export interface GeocodingResult {
  id: number;
  name: string;
  latitude: number;
  longitude: number;
  country: string;
  admin1?: string;
}

export interface GeocodingResponse {
  results?: GeocodingResult[];
}

export interface CurrentWeather {
  time: string;
  temperature_2m: number;
  relative_humidity_2m: number;
  wind_speed_10m: number;
  weather_code: number;
}

export interface DailyForecast {
  time: string[];
  temperature_2m_max: number[];
  temperature_2m_min: number[];
  weather_code: number[];
}

export interface ForecastResponse {
  timezone: string;
  current: CurrentWeather;
  daily: DailyForecast;
}

// The shape our dashboard actually displays.
export interface DayForecast {
  date: string;
  maxTempC: number;
  minTempC: number;
  conditions: string;
}

export interface WeatherSummary {
  city: string;
  country: string;
  timezone: string;
  currentTempC: number;
  humidityPercent: number;
  windKph: number;
  conditions: string;
  forecast: DayForecast[];
}
```

## src/httpClient.ts — a generic fetch-and-validate helper

```typescript
// A generic, reusable "fetch + validate" helper (see Level 2 Module 8).
// `T` is inferred from whichever type-guard function is passed in, so
// every call site gets a concretely-typed result with no `any` anywhere.
export async function fetchJson<T>(
  url: string,
  validate: (data: unknown) => data is T
): Promise<T> {
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`Request failed: ${response.status} ${response.statusText}`);
  }

  const data: unknown = await response.json();

  if (!validate(data)) {
    throw new Error(`Response from ${url} did not match the expected shape`);
  }

  return data;
}
```

## src/weatherCodes.ts — a typed lookup table

```typescript
// WMO weather interpretation codes used by Open-Meteo.
// https://open-meteo.com/en/docs -- "WMO Weather interpretation codes"
const WEATHER_CODES: Record<number, string> = {
  0: "Clear sky",
  1: "Mainly clear",
  2: "Partly cloudy",
  3: "Overcast",
  45: "Fog",
  48: "Depositing rime fog",
  51: "Light drizzle",
  53: "Moderate drizzle",
  55: "Dense drizzle",
  61: "Slight rain",
  63: "Moderate rain",
  65: "Heavy rain",
  71: "Slight snow",
  73: "Moderate snow",
  75: "Heavy snow",
  80: "Slight rain showers",
  81: "Moderate rain showers",
  82: "Violent rain showers",
  95: "Thunderstorm",
};

export function describeWeatherCode(code: number): string {
  return WEATHER_CODES[code] ?? `Unknown conditions (code ${code})`;
}
```

`Record<number, string>` gives every lookup a typed result, and the `??`
fallback means an undocumented WMO code (Open-Meteo's list is longer than
what's mapped here) degrades gracefully instead of printing `undefined`.

## src/geocode.ts — city name to coordinates, with a typed cache

```typescript
import { fetchJson } from "./httpClient";
import { GeocodingResponse, GeocodingResult } from "./types";

// `Location` reuses `GeocodingResult` via `Omit` instead of duplicating
// three of its four fields by hand -- drop the ones we don't need.
export type Location = Omit<GeocodingResult, "id" | "admin1">;

function isGeocodingResponse(value: unknown): value is GeocodingResponse {
  if (typeof value !== "object" || value === null) {
    return false;
  }
  const candidate = value as Record<string, unknown>;
  return candidate.results === undefined || Array.isArray(candidate.results);
}

// A small in-memory cache. `Partial<Record<string, Location>>` means
// "a dictionary keyed by city name where any entry might be missing" --
// exactly the guarantee a cache needs, and different from a plain
// `Record<string, Location>`, which would claim every possible string
// key already has a `Location`.
const cache: Partial<Record<string, Location>> = {};

export async function geocodeCity(city: string): Promise<Location> {
  const cached = cache[city];
  if (cached) {
    return cached;
  }

  const url = `https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(city)}&count=1`;
  const data = await fetchJson(url, isGeocodingResponse);

  const first: GeocodingResult | undefined = data.results?.[0];
  if (!first) {
    throw new Error(`No location found for "${city}"`);
  }

  const location: Location = {
    name: first.name,
    country: first.country,
    latitude: first.latitude,
    longitude: first.longitude,
  };

  cache[city] = location;
  return location;
}
```

## src/forecast.ts — coordinates to weather data

```typescript
import { fetchJson } from "./httpClient";
import { ForecastResponse } from "./types";

function isForecastResponse(value: unknown): value is ForecastResponse {
  if (typeof value !== "object" || value === null) {
    return false;
  }
  const candidate = value as Record<string, unknown>;
  return (
    typeof candidate.timezone === "string" &&
    typeof candidate.current === "object" &&
    candidate.current !== null &&
    typeof candidate.daily === "object" &&
    candidate.daily !== null
  );
}

export async function getForecast(latitude: number, longitude: number): Promise<ForecastResponse> {
  const params = new URLSearchParams({
    latitude: String(latitude),
    longitude: String(longitude),
    current: "temperature_2m,relative_humidity_2m,wind_speed_10m,weather_code",
    daily: "temperature_2m_max,temperature_2m_min,weather_code",
    timezone: "auto",
  });

  const url = `https://api.open-meteo.com/v1/forecast?${params.toString()}`;
  return fetchJson(url, isForecastResponse);
}
```

## src/summary.ts — the pure logic (this is what gets tested)

```typescript
import { Location } from "./geocode";
import { DayForecast, ForecastResponse, WeatherSummary } from "./types";
import { describeWeatherCode } from "./weatherCodes";

// A pure function: no fetch, no I/O -- just data in, data out. This is
// what makes it trivial to unit test without touching the network (see
// __tests__/summary.test.ts).
export function buildSummary(location: Location, forecast: ForecastResponse): WeatherSummary {
  const days: DayForecast[] = forecast.daily.time.map((date, i) => ({
    date,
    maxTempC: forecast.daily.temperature_2m_max[i],
    minTempC: forecast.daily.temperature_2m_min[i],
    conditions: describeWeatherCode(forecast.daily.weather_code[i]),
  }));

  return {
    city: location.name,
    country: location.country,
    timezone: forecast.timezone,
    currentTempC: forecast.current.temperature_2m,
    humidityPercent: forecast.current.relative_humidity_2m,
    windKph: forecast.current.wind_speed_10m,
    conditions: describeWeatherCode(forecast.current.weather_code),
    forecast: days,
  };
}
```

Splitting "fetch the data" (`geocode.ts`, `forecast.ts`) from "shape the
data" (`summary.ts`) is the single most useful structural decision in
this project: it's the difference between a dashboard you can only test
by hitting a real network, and one where the actual business logic is
testable in milliseconds, offline, deterministically.

## src/display.ts — formatting for the terminal

```typescript
import { WeatherSummary } from "./types";

// A compact view showing only headline fields, built with `Pick` instead
// of a hand-duplicated interface -- if `WeatherSummary` ever gains a new
// field, `HeadlineView` doesn't need to be touched at all.
export type HeadlineView = Pick<WeatherSummary, "city" | "currentTempC" | "conditions">;

export function formatHeadline(summary: HeadlineView): string {
  return `${summary.city}: ${summary.currentTempC}°C, ${summary.conditions}`;
}

export function formatFullReport(summary: WeatherSummary): string {
  const lines = [
    `Weather for ${summary.city}, ${summary.country} (${summary.timezone})`,
    `Now: ${summary.currentTempC}°C, ${summary.conditions}, humidity ${summary.humidityPercent}%, wind ${summary.windKph} km/h`,
    "",
    "Forecast:",
    ...summary.forecast.map(
      (day) => `  ${day.date}: ${day.minTempC}–${day.maxTempC}°C, ${day.conditions}`
    ),
  ];
  return lines.join("\n");
}
```

## src/index.ts — the CLI entry point

```typescript
import { geocodeCity } from "./geocode";
import { getForecast } from "./forecast";
import { buildSummary } from "./summary";
import { formatFullReport } from "./display";

async function main(): Promise<void> {
  const city = process.argv[2] ?? "London";

  try {
    const location = await geocodeCity(city);
    const forecast = await getForecast(location.latitude, location.longitude);
    const summary = buildSummary(location, forecast);
    console.log(formatFullReport(summary));
  } catch (err) {
    if (err instanceof Error) {
      console.error(`Failed to load weather for "${city}": ${err.message}`);
    } else {
      console.error("An unknown error occurred");
    }
    process.exitCode = 1;
  }
}

main();
```

Note the `catch` block narrows with `instanceof Error` before reading
`.message` — the same pattern from [Module 6](06-async-await-types.md),
applied to a real failure a user can actually trigger (typo a city name
and see for yourself).

## __tests__/weatherCodes.test.ts

```typescript
import { describeWeatherCode } from "../src/weatherCodes";

test("maps known WMO codes to readable descriptions", () => {
  expect(describeWeatherCode(0)).toBe("Clear sky");
  expect(describeWeatherCode(61)).toBe("Slight rain");
  expect(describeWeatherCode(95)).toBe("Thunderstorm");
});

test("falls back gracefully for an unknown code", () => {
  expect(describeWeatherCode(999)).toBe("Unknown conditions (code 999)");
});
```

## __tests__/summary.test.ts

```typescript
import { buildSummary } from "../src/summary";
import { Location } from "../src/geocode";
import { ForecastResponse } from "../src/types";
import { formatHeadline, formatFullReport } from "../src/display";

// Hand-built fixtures -- no network calls in this test file at all.
const fixtureLocation: Location = {
  name: "Testville",
  country: "Testland",
  latitude: 1,
  longitude: 2,
};

const fixtureForecast: ForecastResponse = {
  timezone: "UTC",
  current: {
    time: "2024-01-01T00:00",
    temperature_2m: 20,
    relative_humidity_2m: 50,
    wind_speed_10m: 10,
    weather_code: 0,
  },
  daily: {
    time: ["2024-01-01", "2024-01-02"],
    temperature_2m_max: [22, 25],
    temperature_2m_min: [15, 17],
    weather_code: [0, 61],
  },
};

describe("buildSummary", () => {
  it("combines location and forecast into a WeatherSummary", () => {
    const summary = buildSummary(fixtureLocation, fixtureForecast);

    expect(summary.city).toBe("Testville");
    expect(summary.country).toBe("Testland");
    expect(summary.currentTempC).toBe(20);
    expect(summary.conditions).toBe("Clear sky");
    expect(summary.forecast).toHaveLength(2);
    expect(summary.forecast[1]).toEqual({
      date: "2024-01-02",
      maxTempC: 25,
      minTempC: 17,
      conditions: "Slight rain",
    });
  });
});

describe("display formatting", () => {
  const summary = buildSummary(fixtureLocation, fixtureForecast);

  it("formats a one-line headline", () => {
    expect(formatHeadline(summary)).toBe("Testville: 20°C, Clear sky");
  });

  it("formats a full multi-line report including every forecast day", () => {
    const report = formatFullReport(summary);
    expect(report).toContain("Weather for Testville, Testland (UTC)");
    expect(report).toContain("2024-01-01: 15–22°C, Clear sky");
    expect(report).toContain("2024-01-02: 17–25°C, Slight rain");
  });
});
```

## Running it

```bash
npm install
npm run build
node dist/index.js London
```

```text
Weather for London, United Kingdom (Europe/London)
Now: 18.7°C, Mainly clear, humidity 47%, wind 14 km/h

Forecast:
  2026-08-02: 16.7–27.2°C, Overcast
  2026-08-03: 18.6–31°C, Overcast
  2026-08-04: 21.4–28.3°C, Overcast
  2026-08-05: 19.1–25.5°C, Light drizzle
  2026-08-06: 14.9–23.3°C, Mainly clear
  2026-08-07: 17.9–24.9°C, Overcast
  2026-08-08: 18.4–23.8°C, Overcast
```

(Exact numbers will differ when you run it — this is live weather data.)

Try a city that doesn't exist to see the error path:

```bash
node dist/index.js "ZzzNotACityXyz123"
# Failed to load weather for "ZzzNotACityXyz123": No location found for "ZzzNotACityXyz123"
```

Run the test suite (fully offline — no network calls, no API involved):

```bash
npm test

# PASS  __tests__/weatherCodes.test.ts
# PASS  __tests__/summary.test.ts
#
# Test Suites: 2 passed, 2 total
# Tests:       5 passed, 5 total
```

## Stretch goals

- Add a `units` option (`"metric" | "imperial"`) threaded through
  `getForecast` as a typed parameter, using Open-Meteo's
  `temperature_unit=fahrenheit` query parameter when imperial is chosen.
- Add a `--days <n>` CLI flag that slices `summary.forecast` down to the
  requested number of days, validating that `n` is a positive integer
  before using it.
- Write a type guard `isValidCityInput(value: unknown): value is string`
  that rejects empty strings and non-string `process.argv` values, and
  use it to give a clearer error message than "No location found" when
  the user runs the CLI with no arguments at all.
- Add a second geocoding result disambiguation step: when
  `data.results` has more than one match (e.g. searching "Springfield"),
  print all candidates with their country/region and let the user note
  which one they meant, instead of silently picking the first result.
- Persist the `Partial<Record<string, Location>>` cache to a JSON file
  between runs (reusing the `storage.ts` pattern from the
  [Level 1 to-do project](../level-1/10-project-todo-cli.md)), so
  repeated lookups of the same city don't re-hit the geocoding API at
  all.
