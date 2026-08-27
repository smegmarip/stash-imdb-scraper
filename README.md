# stash-imdb-scraper

A Python [Stash](https://github.com/stashapp/stash) scraper for IMDB, replacing
the [CommunityScrapers](https://github.com/stashapp/CommunityScrapers) YAML
scraper of the same name.

## Why a Python rewrite?

IMDB fronts its site with an AWS WAF bot challenge: non-browser clients receive
an HTTP 202 with an empty body and an `x-amzn-waf-action: challenge` header.
Stash's built-in `scrapeXPath` action cannot execute the JavaScript challenge,
so every scrape fails with `EOF`. No User-Agent or header spoofing gets past
it — the WAF fingerprints the TLS client itself.

This scraper solves the challenge through
[FlareSolverr](https://github.com/FlareSolverr/FlareSolverr) (a real headless
browser), then caches the resulting `aws-waf-token` cookie via `py_common`'s
proxy layer so subsequent requests are plain, fast HTTP until the token
expires.

It also reads IMDB's embedded structured data (`__NEXT_DATA__` and JSON-LD)
in preference to the visible DOM, which yields more fields and survives
markup changes; the original XPath selectors remain as fallbacks.

## Supported operations and fields

| Operation | Fields |
| --- | --- |
| Performer by URL | name, birthdate, death date, gender¹, country², bio, aliases (nicknames + AKAs), height, full-resolution image, official/social links |
| Performer by name | search results from IMDB's name search |
| Scene by URL | title, code (`tt` id), date, details, director, studio, genre/interest tags, cast, image, containing group |
| Group (movie) by URL | name, original-title alias, date, synopsis, director, studio, tags, rating, duration, front image |

¹ Inferred from the actress/actor credit category — the only gender signal
IMDB exposes.
² Derived from the birthplace, mapped to an ISO 3166-1 alpha-2 code where
recognized.

## Requirements

- Stash v0.30.x (tested against v0.30.1)
- Python 3.10+ reachable by Stash, with `lxml`, `requests`, and
  `cloudscraper` installed (`py_common` installs missing packages on first
  run)
- The CommunityScrapers `py_common` package installed as a sibling directory
  of this scraper
- A running [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr)
  instance reachable from the Stash process

## Installation

1. Copy this directory into your Stash scrapers path alongside `py_common`,
   e.g.:

   ```
   scrapers/
   ├── community/
   │   ├── py_common/
   │   └── IMDB/
   │       ├── IMDB.py
   │       └── IMDB.yml
   ```

2. Set the `FLARESOLVERR_URL` environment variable on the Stash process
   (default: `http://localhost:8191/v1`). For a dockerized Stash with
   FlareSolverr published on the host:

   ```yaml
   environment:
     - FLARESOLVERR_URL=http://host.docker.internal:8191/v1
   ```

3. Reload scrapers (Settings → Metadata Providers → Scrapers → Reload
   scrapers).

An `HTTP_PROXY`/`HTTPS_PROXY` environment variable, if present, is honored
for plain requests and passed through to FlareSolverr where supported.

## Testing from the command line

The scraper reads a JSON fragment on stdin and takes the operation as its
argument:

```sh
echo '{"url": "https://www.imdb.com/name/nm0683467/"}' | python3 IMDB.py performer-by-url
echo '{"name": "Alison Pill"}'                         | python3 IMDB.py performer-by-name
echo '{"url": "https://www.imdb.com/title/tt0110912/"}' | python3 IMDB.py scene-by-url
echo '{"url": "https://www.imdb.com/title/tt0110912/"}' | python3 IMDB.py group-by-url
```

## Caveats

- If the CommunityScrapers package manager previously installed the YAML
  IMDB scraper, exclude it from package updates: an update would restore the
  broken YAML version and delete `IMDB.py`.
- When the cached WAF token expires, the next scrape transparently re-solves
  through FlareSolverr; if FlareSolverr is unreachable at that moment, the
  scrape fails until it is available again.
