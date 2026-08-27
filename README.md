# HoeWarmIsHetInDelft (Weather Scraping Service)

*"Hoe warm is het in Delft?"* — Dutch for "How warm is it in Delft?"

A small service that scrapes the live temperature for Delft from
[weerindelft.nl](http://www.weerindelft.nl/) using a headless browser, since
the site renders the temperature client-side inside an iframe rather than
serving it in the raw HTML.

## Why Selenium instead of requests + BeautifulSoup

The temperature value on the target page isn't present in the initial HTML
response — it's loaded into an `<iframe>` and populated by JavaScript after
the page loads (element id `ajaxtemp`). A plain HTTP GET (`requests`) would
only see an empty shell. This project uses **Selenium with headless Chrome**
so it gets a fully rendered DOM, switches into the iframe's context, and
waits for the element to actually populate before reading it.

## Architecture

```
┌────────────────────┐
│ HoeWarmIsHetInDelft │  entrypoint (main.py)
│        .py          │
└─────────┬───────────┘
          │ instantiates
          ▼
┌─────────────────────┐        ┌──────────────────────┐
│   WeatherService     │───────▶│  TemperatureScraper   │
│   (app/weather.py)   │  uses  │   (app/scraper.py)    │
│                      │        │                       │
│ - rounds the reading │        │ - drives headless     │
│ - wraps scraper      │        │   Chrome via Selenium │
│   failures as         │        │ - switches into the   │
│   ValueError           │        │   page's iframe       │
└─────────────────────┘        │ - waits up to 20s for  │
                                │   #ajaxtemp to render  │
                                │ - parses "15,5°C" →    │
                                │   15.5 (locale-aware)  │
                                │ - always quits the     │
                                │   driver (finally)     │
                                └──────────┬────────────┘
                                           │ HTTP + JS render
                                           ▼
                                ┌──────────────────────┐
                                │  weerindelft.nl        │
                                │  (temperature is       │
                                │   inside an iframe,    │
                                │   populated by JS)     │
                                └──────────────────────┘
```

### Flow, step by step

1. `main()` builds a `TemperatureScraper` pointed at the target URL and
   wraps it in a `WeatherService`.
2. `WeatherService.get_temperature()` calls down into the scraper, catches
   *any* exception it raises, and re-raises as a `ValueError` with context —
   so callers only ever have to handle one exception type, regardless of
   whether the failure was a timeout, a missing element, or a driver crash.
3. Inside `TemperatureScraper.fetch_temperature()`:
   - Headless Chrome launches via `webdriver-manager` (auto-downloads the
     matching ChromeDriver binary — no manual driver version management).
   - Chrome flags disable extensions, GPU, sandboxing, and image loading —
     minimizes container overhead since only text content is needed.
   - The driver navigates to the page, looks for an `<iframe>`, and
     switches context into it if one is found (with a try/except fallback,
     since not every page load may have the iframe present).
   - `WebDriverWait` polls for up to 20 seconds for `#ajaxtemp` to become
     visible — necessary because the value is populated asynchronously,
     not present at initial page load.
   - The raw text (e.g. `"15,5°C"`) is cleaned: strip `°C`, replace the
     Dutch decimal comma with a period, cast to `float`.
   - The driver is always `.quit()`-ed in a `finally` block, so a failed
     scrape never leaks a headless Chrome process.
4. `WeatherService` rounds the float to the nearest whole degree and
   returns it; `main()` prints it.

## Tests

```bash
pytest tests/
```

Both the scraper and the service are tested with Selenium fully mocked
(`unittest.mock`) — no real browser or network call is needed to run the
suite. Coverage includes:

- **Happy path** — iframe found, temperature parsed correctly
- **No-iframe fallback** — scraper still succeeds when the iframe lookup
  fails, instead of crashing
- **Timeout / element-not-found** — verifies the wrapped exception message
  surfaces correctly
- **`WeatherService` rounding and error translation** — confirms any
  scraper exception becomes a `ValueError` with a clear message

## Running it

### Locally

```bash
pip install -r requirements.txt
python HoeWarmIsHetInDelft.py
```

Requires a local Chrome/Chromium install — `webdriver-manager` handles
fetching the matching ChromeDriver binary automatically.

### Docker

```bash
docker build -t hoewarmishetindelft .
docker run --rm hoewarmishetindelft
```

The `Dockerfile` installs Chrome and ChromeDriver into a `python:3.10-slim`
base image (Selenium's headless Chrome requirement is the reason the image
isn't a minimal one) and runs the script directly as its `CMD`.

## CI/CD (GitLab CI)

Three-stage pipeline, all on `main`:

1. **build** — builds the Docker image using Docker-in-Docker (`docker:dind`)
2. **test** — installs dependencies and runs `pytest tests/` in a clean
   `python:3.10-slim` container, independent of the Docker build
3. **deploy** — runs the built image once (`docker run --rm ...`) — in its
   current form this executes the scrape once per pipeline run rather than
   deploying a long-running service

## What I'd add with more time

- Schedule the deploy stage (GitLab CI `schedule` trigger, or a cron-based
  runner) instead of a single one-off run per pipeline
- Persist readings over time (a small CSV/DB append) instead of printing
  to stdout — currently every run is stateless
- Retry logic around the Selenium wait, since a single slow page load
  currently means a single failed run
- Swap the manual iframe-lookup try/except for an explicit check
  (`find_elements` returns `[]` rather than raising) — slightly cleaner
  control flow than relying on an exception for an expected "no iframe"
  case
