# TODO: Task Breakdown

This document breaks down PLAN.md into granular, testable tasks with clear acceptance criteria.

---

## Part A: Docker Compose Setup

### A1: Docker Infrastructure Setup

#### A1.1: Create Docker Directory Structure
- [ ] Create `docker/` directory
- [ ] Verify directory exists with `ls`

**Test**: `test -d docker && echo "PASS" || echo "FAIL"`

---

#### A1.2: Create Dockerfile.mkdocs
- [ ] Create `docker/Dockerfile.mkdocs`
- [ ] Base image: `python:3.11-slim`
- [ ] Install git package
- [ ] Install MkDocs and plugins:
  - mike
  - mkdocs-material
  - mkdocs-macros-plugin
  - mkdocs-monorepo-plugin
  - mkdocs-site-urls
  - mkdocs-caption
  - markdown-tables-extended
  - mkdocs-minify-plugin
- [ ] Set WORKDIR to `/docs`
- [ ] Copy entrypoint script
- [ ] Set entrypoint and default CMD

**Test**: `docker build -f docker/Dockerfile.mkdocs -t mkdocs-test docker/`

**Acceptance Criteria**:
- Image builds without errors
- All packages installed correctly
- Image size reasonable (< 500MB)

---

#### A1.3: Create Entrypoint Script
- [ ] Create `docker/entrypoint.sh`
- [ ] Make script executable (`chmod +x`)
- [ ] Check if git submodules initialized
- [ ] Initialize submodules if needed
- [ ] Set default environment variables (CURRENT_YEAR, BUILD_DATE, GIT_INFO)
- [ ] Handle `build` command: run `mkdocs build --clean --site-dir /docs/site`
- [ ] Handle `serve` command: run `mkdocs serve --dev-addr=0.0.0.0:8000`
- [ ] Add logging/echo statements for visibility
- [ ] Handle errors with `set -e`

**Test (syntax)**: `bash -n docker/entrypoint.sh`

**Acceptance Criteria**:
- Script has no syntax errors
- Handles both build and serve modes
- Provides clear log output

---

#### A1.4: Create Nginx Configuration
- [ ] Create `docker/nginx.conf`
- [ ] Configure server block listening on port 80
- [ ] Set root to `/usr/share/nginx/html`
- [ ] Enable gzip compression (level 6)
- [ ] Configure gzip for text/css/js/json/xml/svg
- [ ] Add cache headers for static assets (1 year)
- [ ] Add security headers (X-Frame-Options, X-Content-Type-Options, X-XSS-Protection)
- [ ] Configure `/health` endpoint returning 200
- [ ] Set up `try_files` directive
- [ ] Configure error pages (404, 50x)
- [ ] Enable `sendfile`, `tcp_nopush`, `tcp_nodelay`

**Test (syntax)**: `docker run --rm -v $(pwd)/docker/nginx.conf:/etc/nginx/nginx.conf:ro nginx nginx -t`

**Acceptance Criteria**:
- Nginx config syntax is valid
- All required directives present
- Health check endpoint defined

---

#### A1.5: Create Dockerfile.nginx
- [ ] Create `docker/Dockerfile.nginx`
- [ ] Base image: `nginx:alpine`
- [ ] Remove default nginx config
- [ ] Copy custom `nginx.conf`
- [ ] Create `/usr/share/nginx/html` directory
- [ ] Expose port 80
- [ ] Add HEALTHCHECK instruction
- [ ] Set CMD to run nginx in foreground

**Test**: `docker build -f docker/Dockerfile.nginx -t nginx-test docker/`

**Acceptance Criteria**:
- Image builds without errors
- Image is small (alpine-based)
- Healthcheck is configured

---

#### A1.6: Create Docker Compose Configuration
- [ ] Create `docker-compose.yml` in project root
- [ ] Define `mkdocs-build` service:
  - Build from `docker/Dockerfile.mkdocs`
  - Mount `./documentation:/docs/documentation`
  - Mount named volume `site-data:/docs/site`
  - Environment variables from .env
  - Command: `build`
- [ ] Define `mkdocs-serve` service:
  - Build from `docker/Dockerfile.mkdocs`
  - Mount `./documentation:/docs/documentation`
  - Port mapping: `8000:8000`
  - Environment variables from .env
  - Command: `serve`
  - Profile: `dev`
- [ ] Define `nginx` service:
  - Build from `docker/Dockerfile.nginx`
  - Mount `site-data:/usr/share/nginx/html:ro`
  - Port mapping: `80:80`
  - Depends on `mkdocs-build` (condition: `service_completed_successfully`)
  - Healthcheck configuration
- [ ] Define `site-data` named volume
- [ ] Define `docs-network` bridge network
- [ ] Add all services to network

**Test (syntax)**: `docker compose config`

**Acceptance Criteria**:
- Docker Compose file is valid YAML
- All services defined correctly
- Volume and network configured
- Dependencies set up properly

---

#### A1.7: Create Environment Variables Template
- [ ] Create `.env.example`
- [ ] Add CURRENT_YEAR with default value
- [ ] Add BUILD_DATE with default value
- [ ] Add GIT_INFO with default value
- [ ] Add usage instructions in comments

**Test**: `test -f .env.example && echo "PASS" || echo "FAIL"`

**Acceptance Criteria**:
- File exists and is readable
- All required variables documented
- Includes helpful comments

---

### A2: Docker Build and Test

#### A2.1: Test MkDocs Build Image
- [ ] Build mkdocs image: `docker compose build mkdocs-build`
- [ ] Check for build errors
- [ ] Verify image exists: `docker images | grep mkdocs-build`
- [ ] Check image size is reasonable

**Test**: `docker compose build mkdocs-build`

**Acceptance Criteria**:
- Build completes without errors
- Image is created
- Image size < 500MB

---

#### A2.2: Test Submodule Initialization
- [ ] Run build container: `docker compose run --rm mkdocs-build build`
- [ ] Check logs for "Initializing git submodules"
- [ ] Verify submodule directory created
- [ ] Check for errors in submodule init

**Test**: `docker compose run --rm mkdocs-build build 2>&1 | grep -i submodule`

**Acceptance Criteria**:
- Submodules initialize without errors
- documentation-assets directory populated
- Build continues after init

---

#### A2.3: Test MkDocs Build Process
- [ ] Run full build: `docker compose run --rm mkdocs-build build`
- [ ] Check for MkDocs build success message
- [ ] Verify no plugin errors or warnings
- [ ] Check that build completes in reasonable time (< 5 min)

**Test**: `docker compose run --rm mkdocs-build build`

**Acceptance Criteria**:
- Build completes successfully
- No error messages in output
- Environment variables properly set
- Build time < 5 minutes

---

#### A2.4: Verify Site Output
- [ ] Check that `site/` directory exists in volume
- [ ] Count files in site directory (should be ~4,203)
- [ ] Verify index.html exists
- [ ] Check for CSS files in assets
- [ ] Check for JS files
- [ ] Verify images directory exists

**Test**:
```bash
docker compose run --rm mkdocs-build sh -c "ls -la /docs/site && find /docs/site -type f | wc -l"
```

**Acceptance Criteria**:
- site/ directory exists
- ~4,203 files present
- index.html exists
- Assets (CSS/JS/images) present

---

#### A2.5: Test Nginx Build
- [ ] Build nginx image: `docker compose build nginx`
- [ ] Check for build errors
- [ ] Verify image exists: `docker images | grep nginx`
- [ ] Check nginx config is copied

**Test**: `docker compose build nginx`

**Acceptance Criteria**:
- Build completes without errors
- Image is small (alpine-based)
- Custom config included

---

#### A2.6: Test Full Stack (Build + Serve)
- [ ] Start services: `docker compose up`
- [ ] Wait for mkdocs-build to complete
- [ ] Verify nginx starts
- [ ] Check nginx health: `curl http://localhost/health`
- [ ] Verify response is "healthy"

**Test**:
```bash
docker compose up -d
sleep 30
curl http://localhost/health
```

**Acceptance Criteria**:
- mkdocs-build completes successfully
- nginx starts and stays running
- Health check returns 200 OK
- Response body is "healthy"

---

#### A2.7: Test Homepage Loading
- [ ] Access homepage: `curl http://localhost/`
- [ ] Verify HTTP 200 response
- [ ] Check HTML content returned
- [ ] Verify page title exists
- [ ] Check for DOCTYPE declaration

**Test**:
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost/
curl -s http://localhost/ | grep -i "<!DOCTYPE"
```

**Acceptance Criteria**:
- Returns HTTP 200
- Valid HTML content
- Contains DOCTYPE
- Page title present

---

#### A2.8: Test Asset Loading
- [ ] Extract CSS links from homepage
- [ ] Test loading a CSS file: `curl http://localhost/assets/stylesheets/...`
- [ ] Verify CSS file returns 200
- [ ] Extract JS links from homepage
- [ ] Test loading a JS file
- [ ] Verify JS file returns 200
- [ ] Test loading an image
- [ ] Verify image returns 200

**Test**:
```bash
# Get homepage and extract first CSS link
CSS_URL=$(curl -s http://localhost/ | grep -oP 'href="([^"]*\.css)"' | head -1 | cut -d'"' -f2)
curl -s -o /dev/null -w "%{http_code}" "http://localhost${CSS_URL}"
```

**Acceptance Criteria**:
- CSS files load (200 OK)
- JS files load (200 OK)
- Images load (200 OK)
- Correct MIME types returned

---

#### A2.9: Test Navigation Between Pages
- [ ] Extract internal links from homepage
- [ ] Test loading 3-5 different pages
- [ ] Verify all return 200
- [ ] Check that sub-site pages load
- [ ] Verify navigation across monorepo structure works

**Test**:
```bash
# Extract and test a few internal links
curl -s http://localhost/ | grep -oP 'href="(/[^"]*)"' | head -5 | while read link; do
  echo "Testing: $link"
  curl -s -o /dev/null -w "%{http_code}\n" "http://localhost$(echo $link | cut -d'"' -f2)"
done
```

**Acceptance Criteria**:
- Multiple pages return 200
- Sub-site pages accessible
- No 404 errors for valid links

---

#### A2.10: Test Search Functionality
- [ ] Load page with search
- [ ] Check for search index files (search.json or similar)
- [ ] Verify search assets load
- [ ] Check search JavaScript present

**Test**:
```bash
curl -s http://localhost/search/search_index.json -o /dev/null -w "%{http_code}"
```

**Acceptance Criteria**:
- Search index file exists
- Returns 200 OK
- Valid JSON format

---

#### A2.11: Test Gzip Compression
- [ ] Request CSS file with Accept-Encoding: gzip
- [ ] Verify Content-Encoding: gzip header present
- [ ] Test with JS file
- [ ] Test with HTML file
- [ ] Verify images NOT gzipped (already compressed)

**Test**:
```bash
curl -H "Accept-Encoding: gzip" -I http://localhost/assets/stylesheets/main.css | grep -i "content-encoding"
```

**Acceptance Criteria**:
- Text files (HTML/CSS/JS) are gzipped
- Gzip header present in response
- Binary files (images) not gzipped

---

#### A2.12: Test Cache Headers
- [ ] Request CSS file
- [ ] Check for Cache-Control header
- [ ] Verify cache duration (1 year for assets)
- [ ] Check Expires header
- [ ] Test with different asset types

**Test**:
```bash
curl -I http://localhost/assets/stylesheets/main.css | grep -i "cache-control"
```

**Acceptance Criteria**:
- Cache-Control header present
- Long cache duration for assets
- Appropriate cache policy

---

#### A2.13: Test Security Headers
- [ ] Request any page
- [ ] Check for X-Frame-Options header
- [ ] Check for X-Content-Type-Options header
- [ ] Check for X-XSS-Protection header
- [ ] Verify header values are correct

**Test**:
```bash
curl -I http://localhost/ | grep -i "x-frame-options"
curl -I http://localhost/ | grep -i "x-content-type"
curl -I http://localhost/ | grep -i "x-xss-protection"
```

**Acceptance Criteria**:
- X-Frame-Options: SAMEORIGIN
- X-Content-Type-Options: nosniff
- X-XSS-Protection: 1; mode=block

---

#### A2.14: Test Development Mode
- [ ] Start dev server: `docker compose --profile dev up mkdocs-serve`
- [ ] Wait for server to start
- [ ] Access site at `http://localhost:8000/`
- [ ] Verify page loads
- [ ] Make a small change to a doc file
- [ ] Verify live reload works (check logs)

**Test**:
```bash
docker compose --profile dev up -d mkdocs-serve
sleep 10
curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/
```

**Acceptance Criteria**:
- Dev server starts on port 8000
- Site accessible
- Live reload enabled
- Changes trigger rebuild

---

#### A2.15: Browser Console Check
- [ ] Open `http://localhost/` in browser
- [ ] Open DevTools console
- [ ] Check for 404 errors (assets)
- [ ] Check for JavaScript errors
- [ ] Check for broken images
- [ ] Verify no console warnings

**Manual Test**: Open browser, check console

**Acceptance Criteria**:
- No 404 errors in console
- No JavaScript errors
- No broken images
- No console warnings

---

#### A2.16: Test Cleanup and Rebuild
- [ ] Stop all containers: `docker compose down`
- [ ] Remove volumes: `docker compose down -v`
- [ ] Rebuild images: `docker compose build --no-cache`
- [ ] Start fresh: `docker compose up`
- [ ] Verify everything works again

**Test**:
```bash
docker compose down -v
docker compose build --no-cache
docker compose up -d
sleep 60
curl http://localhost/health
```

**Acceptance Criteria**:
- Clean rebuild succeeds
- Site works after fresh build
- No cached issues
- Reproducible setup

---

## Part B: Link Checker Tool

### B1: Project Setup

#### B1.1: Create Link Checker Directory Structure
- [ ] Create `link_check/` directory
- [ ] Create `link_check/__init__.py`
- [ ] Create `link_check/tests/` directory
- [ ] Create `link_check/tests/__init__.py`
- [ ] Create `link_check/tests/fixtures/` directory

**Test**: `test -d link_check && test -d link_check/tests && echo "PASS" || echo "FAIL"`

**Acceptance Criteria**:
- All directories exist
- Python package structure correct
- Test directory set up

---

#### B1.2: Create pyproject.toml
- [ ] Create `link_check/pyproject.toml`
- [ ] Add [project] section with name, version, python requirement
- [ ] Add dependencies:
  - httpx >= 0.27.0
  - beautifulsoup4 >= 4.12.0
  - pyyaml >= 6.0.0
  - click >= 8.1.0
  - ruamel.yaml >= 0.18.0
  - rich >= 13.0.0
- [ ] Add dev dependencies:
  - pytest >= 8.0.0
  - pytest-asyncio >= 0.23.0
  - pytest-cov >= 4.1.0
  - pytest-mock >= 3.12.0
  - black >= 24.0.0
  - ruff >= 0.3.0
- [ ] Add [project.scripts] entry point
- [ ] Add [tool.black] configuration
- [ ] Add [tool.pytest.ini_options]

**Test**: `cd link_check && uv sync --dry-run`

**Acceptance Criteria**:
- Valid TOML syntax
- All dependencies specified
- Can be parsed by uv
- Entry point configured

---

#### B1.3: Create Ruff Configuration
- [ ] Create `link_check/ruff.toml`
- [ ] Set line-length = 88
- [ ] Set target-version = "py311"
- [ ] Configure [lint] section
- [ ] Select rules: E, W, F, I, N, UP, B, A, C4, PT
- [ ] Ignore E501 (line too long)
- [ ] Add per-file-ignores for tests

**Test**: `cd link_check && uv run ruff check --config ruff.toml .`

**Acceptance Criteria**:
- Valid TOML syntax
- Ruff can parse config
- Appropriate rules selected

---

#### B1.4: Initialize uv Project
- [ ] Run `cd link_check && uv sync`
- [ ] Verify .venv created
- [ ] Verify all dependencies installed
- [ ] Check that lock file created (uv.lock)
- [ ] Verify entry point installed

**Test**: `cd link_check && uv sync && uv run python -c "import httpx; import click; import pytest"`

**Acceptance Criteria**:
- Virtual environment created
- All deps installed successfully
- Can import all dependencies
- No version conflicts

---

### B2: Configuration Module

#### B2.1: Create config.py
- [ ] Create `link_check/config.py`
- [ ] Define DEFAULT_CONCURRENCY = 20
- [ ] Define DEFAULT_TIMEOUT = 10
- [ ] Define DEFAULT_USER_AGENT
- [ ] Define DEFAULT_OUTPUT_FILE = "link_check_report.yaml"
- [ ] Define MAX_RETRIES = 3
- [ ] Define EXTERNAL_LINK_TIMEOUT = 5
- [ ] Add docstrings

**Test**: `cd link_check && uv run python -c "from link_check.config import *; print(DEFAULT_CONCURRENCY)"`

**Acceptance Criteria**:
- Module imports successfully
- All constants defined
- Reasonable default values
- Properly documented

---

### B3: Spider Module (Crawler)

#### B3.1: Create Spider Class Structure
- [ ] Create `link_check/spider.py`
- [ ] Import required modules (httpx, asyncio, BeautifulSoup, urllib.parse)
- [ ] Define `Spider` class
- [ ] Add `__init__` method with base_url, concurrency params
- [ ] Create instance variables: visited, to_visit, link_map
- [ ] Add docstrings

**Test**: `cd link_check && uv run python -c "from link_check.spider import Spider; s = Spider('http://localhost')"`

**Acceptance Criteria**:
- Module imports successfully
- Class instantiates without errors
- Instance variables initialized
- Proper type hints

---

#### B3.2: Implement URL Normalization
- [ ] Add `normalize_url(url, base_url)` method
- [ ] Handle relative URLs
- [ ] Remove fragments for deduplication
- [ ] Ensure trailing slash consistency
- [ ] Handle query parameters
- [ ] Add tests

**Test**: Run pytest on test_spider.py::test_normalize_url

**Acceptance Criteria**:
- Correctly converts relative to absolute
- Handles edge cases (empty, malformed)
- Consistent output format
- Tests pass

---

#### B3.3: Implement Same-Origin Checking
- [ ] Add `is_same_origin(url, base_url)` method
- [ ] Parse URLs to extract scheme + host
- [ ] Compare origins
- [ ] Handle edge cases
- [ ] Add tests

**Test**: Run pytest on test_spider.py::test_same_origin

**Acceptance Criteria**:
- Correctly identifies same-origin URLs
- Handles different schemes (http vs https)
- Handles ports correctly
- Tests pass

---

#### B3.4: Implement Link Extraction
- [ ] Add `extract_links(html, page_url)` method
- [ ] Parse HTML with BeautifulSoup
- [ ] Extract `<a href>` links
- [ ] Extract `<link href>` (CSS)
- [ ] Extract `<script src>` (JS)
- [ ] Extract `<img src>` (images)
- [ ] Return set of URLs
- [ ] Add tests with sample HTML

**Test**: Run pytest on test_spider.py::test_extract_links

**Acceptance Criteria**:
- Extracts all link types
- Handles malformed HTML
- Returns deduplicated set
- Preserves fragments for anchors
- Tests pass

---

#### B3.5: Implement Page Fetching
- [ ] Add `async def fetch_page(url, client)` method
- [ ] Use httpx.AsyncClient
- [ ] Handle 200 responses (return content)
- [ ] Handle redirects (follow)
- [ ] Handle errors (return None)
- [ ] Add timeout handling
- [ ] Log errors appropriately
- [ ] Add tests with mocked responses

**Test**: Run pytest on test_spider.py::test_fetch_page

**Acceptance Criteria**:
- Successfully fetches valid pages
- Handles HTTP errors gracefully
- Follows redirects
- Respects timeout
- Tests pass

---

#### B3.6: Implement Main Crawl Method
- [ ] Add `async def crawl(start_url)` method
- [ ] Initialize httpx.AsyncClient with connection pool
- [ ] Create asyncio.Semaphore for concurrency control
- [ ] Use queue for BFS traversal
- [ ] For each page:
  - Fetch HTML
  - Extract links
  - Add to link_map
  - Queue unvisited same-origin links
- [ ] Return link_map: dict[str, set[str]]
- [ ] Add progress logging
- [ ] Add tests with mock HTTP server

**Test**: Run pytest on test_spider.py::test_crawl

**Acceptance Criteria**:
- Crawls entire site
- Respects concurrency limit
- Builds correct link map
- Handles errors gracefully
- No infinite loops
- Tests pass

---

### B4: Checker Module (Link Validator)

#### B4.1: Create BrokenLink Dataclass
- [ ] Create `link_check/checker.py`
- [ ] Import dataclasses
- [ ] Define `@dataclass BrokenLink`:
  - url: str
  - status_code: int | None
  - error_message: str
  - link_text: str | None
  - link_type: str
- [ ] Add docstring

**Test**: `cd link_check && uv run python -c "from link_check.checker import BrokenLink; b = BrokenLink('url', 404, 'err', 'text', 'internal')"`

**Acceptance Criteria**:
- Dataclass defined correctly
- All fields present
- Type hints correct
- Can instantiate

---

#### B4.2: Create LinkChecker Class Structure
- [ ] Define `LinkChecker` class
- [ ] Add `__init__` with timeout, concurrency params
- [ ] Create instance variables
- [ ] Initialize cache for checked URLs
- [ ] Add docstrings

**Test**: `cd link_check && uv run python -c "from link_check.checker import LinkChecker; c = LinkChecker()"`

**Acceptance Criteria**:
- Class instantiates successfully
- Parameters configurable
- Cache initialized
- Proper type hints

---

#### B4.3: Implement HTTP Status Checking
- [ ] Add `async def check_http_status(url, client)` method
- [ ] Try HEAD request first
- [ ] Fall back to GET if HEAD fails
- [ ] Return status code
- [ ] Handle timeouts
- [ ] Handle connection errors
- [ ] Add tests with mocked responses

**Test**: Run pytest on test_checker.py::test_check_http_status

**Acceptance Criteria**:
- Returns correct status codes
- Handles 2xx, 4xx, 5xx
- Tries HEAD before GET
- Handles errors gracefully
- Tests pass

---

#### B4.4: Implement Anchor Validation
- [ ] Add `async def check_anchor(url, client)` method
- [ ] Parse URL to extract fragment
- [ ] Fetch page content
- [ ] Parse HTML with BeautifulSoup
- [ ] Search for element with id or name matching anchor
- [ ] Return True if found, False otherwise
- [ ] Add tests

**Test**: Run pytest on test_checker.py::test_check_anchor

**Acceptance Criteria**:
- Correctly finds valid anchors
- Detects missing anchors
- Handles pages without anchors
- Tests pass

---

#### B4.5: Implement Link Type Detection
- [ ] Add `classify_link(url, base_url)` method
- [ ] Determine if internal or external
- [ ] Check if asset (CSS/JS/image by extension)
- [ ] Check if has anchor
- [ ] Return link type string
- [ ] Add tests

**Test**: Run pytest on test_checker.py::test_classify_link

**Acceptance Criteria**:
- Correctly classifies internal links
- Correctly classifies external links
- Identifies assets
- Identifies anchors
- Tests pass

---

#### B4.6: Implement Main Check Method
- [ ] Add `async def check_links(link_map)` method
- [ ] Initialize httpx.AsyncClient
- [ ] Create semaphore for concurrency
- [ ] For each unique URL in link_map:
  - Check HTTP status
  - If has anchor, validate anchor
  - If 404 or anchor missing, create BrokenLink
- [ ] Build result dict: source_page → list[BrokenLink]
- [ ] Return results
- [ ] Add tests

**Test**: Run pytest on test_checker.py::test_check_links

**Acceptance Criteria**:
- Checks all links
- Detects 404s
- Detects missing anchors
- Returns correct format
- Tests pass

---

### B5: Navigation Validator Module

#### B5.1: Create NavError Dataclass
- [ ] Create `link_check/nav_validator.py`
- [ ] Import dataclasses
- [ ] Define `@dataclass NavError`:
  - nav_path: str
  - url: str
  - error_message: str
- [ ] Add docstring

**Test**: `cd link_check && uv run python -c "from link_check.nav_validator import NavError; n = NavError('path', 'url', 'err')"`

**Acceptance Criteria**:
- Dataclass defined
- All fields present
- Can instantiate

---

#### B5.2: Create NavValidator Class Structure
- [ ] Define `NavValidator` class
- [ ] Add `__init__` with mkdocs_root, base_url params
- [ ] Add instance variables
- [ ] Add docstrings

**Test**: `cd link_check && uv run python -c "from link_check.nav_validator import NavValidator; v = NavValidator('path', 'url')"`

**Acceptance Criteria**:
- Class instantiates
- Parameters stored
- Proper type hints

---

#### B5.3: Implement YAML Parsing
- [ ] Add `parse_mkdocs_config(config_path)` method
- [ ] Use ruamel.yaml to load file
- [ ] Extract nav section
- [ ] Handle missing nav gracefully
- [ ] Return parsed nav structure
- [ ] Add tests with sample config

**Test**: Run pytest on test_nav_validator.py::test_parse_config

**Acceptance Criteria**:
- Parses valid YAML
- Extracts nav section
- Handles missing files
- Handles malformed YAML
- Tests pass

---

#### B5.4: Implement Include Directive Handling
- [ ] Add `resolve_includes(nav, base_dir)` method
- [ ] Check for `!include` directives
- [ ] Recursively parse included files
- [ ] Merge nav structures
- [ ] Handle circular includes
- [ ] Add tests

**Test**: Run pytest on test_nav_validator.py::test_resolve_includes

**Acceptance Criteria**:
- Finds !include directives
- Loads included files
- Builds complete nav tree
- Prevents infinite recursion
- Tests pass

---

#### B5.5: Implement Nav URL Extraction
- [ ] Add `extract_nav_urls(nav_structure)` method
- [ ] Recursively walk nav tree
- [ ] Extract URLs from leaf nodes
- [ ] Track nav path for each URL
- [ ] Return list of (nav_path, url) tuples
- [ ] Add tests

**Test**: Run pytest on test_nav_validator.py::test_extract_urls

**Acceptance Criteria**:
- Extracts all URLs from nav
- Preserves nav path hierarchy
- Handles nested structure
- Tests pass

---

#### B5.6: Implement Main Validation Method
- [ ] Add `async def validate_nav()` method
- [ ] Parse main mkdocs.yml
- [ ] Resolve includes
- [ ] Extract all nav URLs
- [ ] For each URL:
  - Convert to full URL (handle relative paths)
  - Check if URL is accessible (HTTP request)
  - If 404, create NavError
- [ ] Return list of NavErrors
- [ ] Add tests

**Test**: Run pytest on test_nav_validator.py::test_validate_nav

**Acceptance Criteria**:
- Validates entire nav structure
- Detects broken nav links
- Handles monorepo structure
- Tests pass

---

### B6: Reporter Module

#### B6.1: Create Reporter Class Structure
- [ ] Create `link_check/reporter.py`
- [ ] Import yaml (PyYAML)
- [ ] Define `Reporter` class
- [ ] Add `__init__` method
- [ ] Add docstrings

**Test**: `cd link_check && uv run python -c "from link_check.reporter import Reporter; r = Reporter()"`

**Acceptance Criteria**:
- Module imports
- Class instantiates
- Proper structure

---

#### B6.2: Implement Summary Calculation
- [ ] Add `calculate_summary(link_map, broken_links, nav_errors)` method
- [ ] Count total pages crawled
- [ ] Count total links checked
- [ ] Count broken links
- [ ] Count pages with errors
- [ ] Count navigation errors
- [ ] Add timestamp
- [ ] Return summary dict
- [ ] Add tests

**Test**: Run pytest on test_reporter.py::test_calculate_summary

**Acceptance Criteria**:
- Correct counts
- Includes timestamp
- Returns dict with all fields
- Tests pass

---

#### B6.3: Implement Broken Links Formatting
- [ ] Add `format_broken_links(broken_links)` method
- [ ] Convert BrokenLink objects to dicts
- [ ] Group by source page
- [ ] Format for YAML output
- [ ] Return formatted dict
- [ ] Add tests

**Test**: Run pytest on test_reporter.py::test_format_broken_links

**Acceptance Criteria**:
- Correct format for YAML
- All fields included
- Properly grouped
- Tests pass

---

#### B6.4: Implement Nav Errors Formatting
- [ ] Add `format_nav_errors(nav_errors)` method
- [ ] Convert NavError objects to dicts
- [ ] Format as list for YAML
- [ ] Return formatted list
- [ ] Add tests

**Test**: Run pytest on test_reporter.py::test_format_nav_errors

**Acceptance Criteria**:
- Correct format
- All fields included
- Tests pass

---

#### B6.5: Implement Report Generation
- [ ] Add `generate_report(link_map, broken_links, nav_errors)` method
- [ ] Calculate summary
- [ ] Format broken links
- [ ] Format nav errors
- [ ] Combine into single dict structure
- [ ] Convert to YAML string
- [ ] Return YAML string
- [ ] Add tests

**Test**: Run pytest on test_reporter.py::test_generate_report

**Acceptance Criteria**:
- Valid YAML output
- Complete report structure
- Can be parsed by YAML parser
- Tests pass

---

#### B6.6: Implement File Writing
- [ ] Add `save_report(report_yaml, output_path)` method
- [ ] Write YAML to file
- [ ] Handle file errors gracefully
- [ ] Add tests

**Test**: Run pytest on test_reporter.py::test_save_report

**Acceptance Criteria**:
- File written successfully
- Valid YAML in file
- Error handling works
- Tests pass

---

### B7: CLI Module

#### B7.1: Create CLI Structure
- [ ] Create `link_check/link_check.py`
- [ ] Import click, asyncio, and all modules
- [ ] Define `@click.command()` main function
- [ ] Add docstring
- [ ] Add placeholder implementation

**Test**: `cd link_check && uv run link-check --help`

**Acceptance Criteria**:
- Help text displays
- No import errors
- Command registered

---

#### B7.2: Add CLI Options
- [ ] Add `--url` option (default: http://localhost)
- [ ] Add `--output` option (default: link_check_report.yaml)
- [ ] Add `--concurrency` option (default: 20)
- [ ] Add `--timeout` option (default: 10)
- [ ] Add `--skip-external` flag
- [ ] Add `--verbose` flag
- [ ] Add `--mkdocs-root` option (default: ../documentation)

**Test**: `cd link_check && uv run link-check --help`

**Acceptance Criteria**:
- All options shown in help
- Defaults documented
- Type hints correct

---

#### B7.3: Implement Crawl Phase
- [ ] Initialize Spider with URL
- [ ] Call `await spider.crawl()`
- [ ] Store link_map
- [ ] Add progress indication with rich
- [ ] Add error handling
- [ ] Log results if verbose

**Test**: Manual test with running Docker site

**Acceptance Criteria**:
- Crawls successfully
- Progress displayed
- Errors handled
- Link map populated

---

#### B7.4: Implement Check Phase
- [ ] Initialize LinkChecker
- [ ] Call `await checker.check_links(link_map)`
- [ ] Store broken_links
- [ ] Add progress indication
- [ ] Add error handling
- [ ] Log results if verbose

**Test**: Manual test with running Docker site

**Acceptance Criteria**:
- Checks all links
- Progress displayed
- Broken links detected
- Errors handled

---

#### B7.5: Implement Nav Validation Phase
- [ ] Initialize NavValidator
- [ ] Call `await validator.validate_nav()`
- [ ] Store nav_errors
- [ ] Add progress indication
- [ ] Add error handling
- [ ] Log results if verbose

**Test**: Manual test with mkdocs config

**Acceptance Criteria**:
- Validates navigation
- Progress displayed
- Nav errors detected
- Errors handled

---

#### B7.6: Implement Report Generation Phase
- [ ] Initialize Reporter
- [ ] Call `reporter.generate_report()`
- [ ] Save report to file
- [ ] Print summary to console
- [ ] Add error handling

**Test**: Manual test, check output file

**Acceptance Criteria**:
- Report generated
- File saved
- Summary displayed
- Valid YAML

---

#### B7.7: Add Exit Code Logic
- [ ] Check if broken_links is empty
- [ ] Check if nav_errors is empty
- [ ] Return 0 if no errors
- [ ] Return 1 if errors found
- [ ] Add tests

**Test**: Run CLI and check exit code

**Acceptance Criteria**:
- Exit 0 on success
- Exit 1 on errors
- Consistent behavior

---

### B8: Testing

#### B8.1: Create Test Fixtures
- [ ] Create `link_check/tests/fixtures/sample_site/`
- [ ] Create `index.html` with various link types
- [ ] Create `page1.html` (working page)
- [ ] Create `page2.html` (with broken links)
- [ ] Create `page3.html` (with missing anchors)
- [ ] Create `assets/style.css`
- [ ] Create `assets/script.js`

**Test**: Files exist and have valid content

**Acceptance Criteria**:
- All fixture files created
- Valid HTML
- Represents real-world scenarios

---

#### B8.2: Create MkDocs Config Fixtures
- [ ] Create `link_check/tests/fixtures/mkdocs_configs/`
- [ ] Create `simple.yml` (basic config)
- [ ] Create `monorepo.yml` (with !include)
- [ ] Create `subsite.yml` (included config)

**Test**: Files parse as valid YAML

**Acceptance Criteria**:
- Valid YAML syntax
- Represents real mkdocs configs
- Includes test cases for features

---

#### B8.3: Write Spider Tests
- [ ] Create `link_check/tests/test_spider.py`
- [ ] Test URL normalization
- [ ] Test same-origin checking
- [ ] Test link extraction from HTML
- [ ] Test page fetching with mocks
- [ ] Test full crawl with mock HTTP server
- [ ] Test error handling
- [ ] Achieve > 90% coverage for spider.py

**Test**: `cd link_check && uv run pytest tests/test_spider.py -v --cov=link_check.spider`

**Acceptance Criteria**:
- All tests pass
- Coverage > 90%
- Edge cases covered

---

#### B8.4: Write Checker Tests
- [ ] Create `link_check/tests/test_checker.py`
- [ ] Test HTTP status checking
- [ ] Test anchor validation
- [ ] Test link type classification
- [ ] Test full check_links with mocks
- [ ] Test error handling
- [ ] Achieve > 90% coverage for checker.py

**Test**: `cd link_check && uv run pytest tests/test_checker.py -v --cov=link_check.checker`

**Acceptance Criteria**:
- All tests pass
- Coverage > 90%
- Edge cases covered

---

#### B8.5: Write Nav Validator Tests
- [ ] Create `link_check/tests/test_nav_validator.py`
- [ ] Test YAML parsing
- [ ] Test !include resolution
- [ ] Test URL extraction
- [ ] Test validation with mocks
- [ ] Test error handling
- [ ] Achieve > 90% coverage for nav_validator.py

**Test**: `cd link_check && uv run pytest tests/test_nav_validator.py -v --cov=link_check.nav_validator`

**Acceptance Criteria**:
- All tests pass
- Coverage > 90%
- Edge cases covered

---

#### B8.6: Write Reporter Tests
- [ ] Create `link_check/tests/test_reporter.py`
- [ ] Test summary calculation
- [ ] Test broken links formatting
- [ ] Test nav errors formatting
- [ ] Test report generation
- [ ] Test file writing
- [ ] Test YAML validity
- [ ] Achieve > 90% coverage for reporter.py

**Test**: `cd link_check && uv run pytest tests/test_reporter.py -v --cov=link_check.reporter`

**Acceptance Criteria**:
- All tests pass
- Coverage > 90%
- Output is valid YAML

---

#### B8.7: Write Integration Tests
- [ ] Create `link_check/tests/test_integration.py`
- [ ] Set up mock HTTP server with fixture site
- [ ] Test full pipeline: crawl → check → report
- [ ] Verify report accuracy
- [ ] Test with intentional broken links
- [ ] Test with valid site (no errors)

**Test**: `cd link_check && uv run pytest tests/test_integration.py -v`

**Acceptance Criteria**:
- Full pipeline works
- Report is accurate
- No false positives
- Catches all broken links

---

#### B8.8: Run Full Test Suite
- [ ] Run `pytest` with coverage
- [ ] Check overall coverage > 90%
- [ ] Check all tests pass
- [ ] Generate HTML coverage report
- [ ] Review coverage report for gaps

**Test**: `cd link_check && uv run pytest --cov --cov-report=html`

**Acceptance Criteria**:
- All tests pass
- Overall coverage > 90%
- No critical gaps in coverage

---

### B9: Code Quality

#### B9.1: Format Code with Black
- [ ] Run black on all .py files
- [ ] Verify formatting consistent
- [ ] Check no errors

**Test**: `cd link_check && uv run black . --check`

**Acceptance Criteria**:
- All files formatted
- Black check passes
- Consistent style

---

#### B9.2: Lint Code with Ruff
- [ ] Run ruff on all .py files
- [ ] Fix any errors
- [ ] Fix any warnings
- [ ] Verify clean output

**Test**: `cd link_check && uv run ruff check .`

**Acceptance Criteria**:
- No ruff errors
- No ruff warnings
- Code quality high

---

#### B9.3: Type Checking (Optional)
- [ ] Add type hints to all functions
- [ ] Run mypy or pyright (optional)
- [ ] Fix type issues

**Test**: `cd link_check && uv run mypy link_check/`

**Acceptance Criteria**:
- Type hints present
- No type errors
- Improved code quality

---

### B10: Documentation

#### B10.1: Write Module Docstrings
- [ ] Add docstrings to all modules
- [ ] Add docstrings to all classes
- [ ] Add docstrings to all public methods
- [ ] Follow Google or NumPy docstring style

**Test**: Review code for docstrings

**Acceptance Criteria**:
- All public APIs documented
- Consistent style
- Clear explanations

---

#### B10.2: Create link_check README
- [ ] Create `link_check/README.md`
- [ ] Add project description
- [ ] Add installation instructions
- [ ] Add usage examples
- [ ] Add development setup instructions
- [ ] Add testing instructions
- [ ] Add configuration options
- [ ] Add troubleshooting section

**Test**: Review README for completeness

**Acceptance Criteria**:
- Clear and complete
- All commands documented
- Easy to follow

---

## Part C: Integration & Final Testing

### C1: End-to-End Integration

#### C1.1: Test Full Workflow
- [ ] Start Docker containers: `docker compose up -d`
- [ ] Wait for site to be available
- [ ] Run link checker: `cd link_check && uv run link-check`
- [ ] Verify report generated
- [ ] Check report content
- [ ] Verify exit code

**Test**: Complete workflow from scratch

**Acceptance Criteria**:
- Docker builds and runs
- Site is accessible
- Link checker runs successfully
- Report is generated
- Report is accurate

---

#### C1.2: Introduce Intentional Broken Link
- [ ] Add broken link to a doc page
- [ ] Rebuild Docker site
- [ ] Run link checker
- [ ] Verify broken link detected in report
- [ ] Check exit code is 1

**Test**: Manual test with modified doc

**Acceptance Criteria**:
- Broken link detected
- Appears in report
- Exit code indicates error

---

#### C1.3: Test with Large Site
- [ ] Run against full documentation site
- [ ] Measure execution time
- [ ] Verify completes in < 5 minutes
- [ ] Check memory usage reasonable
- [ ] Verify no timeouts or crashes

**Test**: Run with time: `time uv run link-check`

**Acceptance Criteria**:
- Completes successfully
- Time < 5 minutes
- Memory usage reasonable
- No errors

---

### C2: Performance Testing

#### C2.1: Benchmark Crawl Phase
- [ ] Add timing to crawl phase
- [ ] Run multiple times
- [ ] Record average time
- [ ] Verify reasonable performance

**Test**: Add logging with timestamps

**Acceptance Criteria**:
- Crawl phase timed
- Performance acceptable
- Consistent results

---

#### C2.2: Benchmark Check Phase
- [ ] Add timing to check phase
- [ ] Run multiple times
- [ ] Record average time
- [ ] Verify reasonable performance

**Test**: Add logging with timestamps

**Acceptance Criteria**:
- Check phase timed
- Performance acceptable
- Consistent results

---

#### C2.3: Test Different Concurrency Levels
- [ ] Run with concurrency=10
- [ ] Run with concurrency=20
- [ ] Run with concurrency=50
- [ ] Compare times and resource usage
- [ ] Determine optimal value

**Test**: `uv run link-check --concurrency N`

**Acceptance Criteria**:
- All levels work
- Optimal value identified
- Document recommendation

---

### C3: Documentation

#### C3.1: Update Main README
- [ ] Update `README.md` with complete instructions
- [ ] Document both Docker and link_check usage
- [ ] Add examples
- [ ] Add troubleshooting

**Test**: Follow README from scratch

**Acceptance Criteria**:
- README is complete
- Instructions work
- Easy to follow

---

#### C3.2: Create Usage Examples
- [ ] Add examples to README
- [ ] Common use cases
- [ ] Advanced usage
- [ ] Troubleshooting scenarios

**Test**: Review examples

**Acceptance Criteria**:
- Examples are clear
- Cover common scenarios
- Easy to adapt

---

## Summary

**Total Tasks**: 150+
**Part A Tasks**: 16 (Docker setup and testing)
**Part B Tasks**: 120+ (Link checker implementation and testing)
**Part C Tasks**: 10+ (Integration and documentation)

Each task is designed to be:
1. **Testable** - Clear test command provided
2. **Independent** - Can be worked on in isolation
3. **Granular** - Small enough to complete and verify quickly
4. **Documented** - Acceptance criteria clearly defined
