# Implementation Plan: MkDocs Docker Deployment + Link Checker

## Project Overview

This project has two main goals:

**(a) Docker Compose Setup**: Create a containerized environment that builds the Dyalog documentation MkDocs site and serves it via nginx for optimal performance.

**(b) Link Checker Tool**: Develop a Python-based tool (`link_check/link_check.py`) that crawls the Docker-served site, validates all links, checks navigation integrity, and generates a YAML report of any broken links.

## Repository Context

**Repository**: `documentation/` (cloned from https://github.com/dyalog/documentation)

**Key Characteristics**:
- **MkDocs monorepo** structure with 12 sub-sites
- Uses **Material for MkDocs** theme
- Requires **Python 3.11**
- Has **git submodule** (`documentation-assets/`) that must be initialized
- Uses plugins: mike, mkdocs-material, mkdocs-macros-plugin, mkdocs-monorepo-plugin, mkdocs-site-urls, mkdocs-caption, markdown-tables-extended, mkdocs-minify-plugin
- Environment variables: `CURRENT_YEAR`, `BUILD_DATE`, `GIT_INFO`
- Build output: `site/` directory (~4,203 files)

## Part A: Docker Compose Setup

### Objectives
1. Build MkDocs static site in a Docker container
2. Serve the built site with nginx for production-like performance
3. Support both development (live reload) and production (static serving) modes
4. Handle git submodules initialization automatically
5. Make the setup reproducible and easy to use

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Compose                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────┐      ┌─────────────────────────┐    │
│  │  mkdocs-build    │      │  mkdocs-serve (dev)     │    │
│  │                  │      │                         │    │
│  │  • Build static  │      │  • Live reload          │    │
│  │  • Init submod   │      │  • Port 8000            │    │
│  │  • Output: site/ │      │  • Profile: dev         │    │
│  └────────┬─────────┘      └─────────────────────────┘    │
│           │                                                 │
│           │ site/ volume                                    │
│           ▼                                                 │
│  ┌──────────────────┐                                      │
│  │      nginx       │                                      │
│  │                  │                                      │
│  │  • Port 80       │                                      │
│  │  • Gzip enabled  │                                      │
│  │  • Cache headers │                                      │
│  └──────────────────┘                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Files to Create

#### 1. `docker/Dockerfile.mkdocs`
**Purpose**: Build and serve MkDocs site

**Contents**:
- Base image: `python:3.11-slim`
- Install git (for submodules)
- Install MkDocs + all required plugins
- Copy entrypoint script
- Set working directory to `/docs`
- Entrypoint handles both build and serve modes

**Key Points**:
- Must support both `build` and `serve` commands
- Needs git for submodule initialization
- All Python dependencies must match repository requirements

#### 2. `docker/entrypoint.sh`
**Purpose**: Handle initialization and build process

**Responsibilities**:
1. Initialize git submodules if not already done
2. Set default environment variables (CURRENT_YEAR, BUILD_DATE, GIT_INFO)
3. Execute either `mkdocs build` or `mkdocs serve` based on command
4. Output built site to `/docs/site` directory

**Key Points**:
- Must check if submodules are initialized before attempting init
- Should provide clear logging of what it's doing
- Needs to handle both build and serve modes

#### 3. `docker/nginx.conf`
**Purpose**: Configure nginx for optimal static site serving

**Features**:
- Gzip compression for text/css/js files
- Cache headers for static assets (1 year for immutable files)
- Security headers (X-Frame-Options, X-Content-Type-Options, X-XSS-Protection)
- Health check endpoint at `/health`
- Proper MIME types
- Error page handling

**Performance Considerations**:
- `sendfile on` for efficient file serving
- `tcp_nopush` and `tcp_nodelay` for connection optimization
- Gzip compression level 6 (balance of speed vs compression)

#### 4. `docker/Dockerfile.nginx`
**Purpose**: Nginx container for serving static site

**Contents**:
- Base image: `nginx:alpine` (lightweight)
- Copy custom nginx.conf
- Create html directory
- Expose port 80
- Health check configuration

#### 5. `docker-compose.yml`
**Purpose**: Orchestrate all services

**Services**:

1. **mkdocs-build**:
   - Builds static site
   - Runs once and exits
   - Outputs to shared volume `site-data`
   - Mounts `documentation/` directory
   - Environment variables from .env file

2. **mkdocs-serve** (profile: dev):
   - Development server with live reload
   - Port 8000
   - Only runs when `--profile dev` specified
   - Mounts `documentation/` for live changes

3. **nginx**:
   - Production web server
   - Port 80
   - Depends on `mkdocs-build` completing successfully
   - Reads from `site-data` volume (read-only)
   - Health check configured

**Volumes**:
- `site-data`: Shared volume for built site (persists between runs)

**Networks**:
- `docs-network`: Bridge network for service communication

#### 6. `.env.example`
**Purpose**: Template for environment variables

**Variables**:
- `CURRENT_YEAR`: Year for copyright notices (default: 2025)
- `BUILD_DATE`: Build timestamp (default: current date)
- `GIT_INFO`: Branch and commit info (default: main:local)

### Testing Strategy for Part A

#### Test 1: Build Process
```bash
docker compose build mkdocs-build
docker compose run --rm mkdocs-build build
```

**Expected Results**:
- ✅ Image builds without errors
- ✅ Submodules are initialized
- ✅ MkDocs build completes successfully
- ✅ `site/` directory is created with ~4,203 files
- ✅ No plugin errors or warnings

#### Test 2: Nginx Serving
```bash
docker compose up mkdocs-build nginx
```

**Expected Results**:
- ✅ Build service completes
- ✅ Nginx starts and serves on port 80
- ✅ Health check passes: `curl http://localhost/health`
- ✅ Homepage loads: `curl http://localhost/`
- ✅ CSS/JS/images load correctly
- ✅ Navigation between sub-sites works
- ✅ Search functionality works

#### Test 3: Development Mode
```bash
docker compose --profile dev up mkdocs-serve
```

**Expected Results**:
- ✅ Development server starts on port 8000
- ✅ Live reload works when files are edited
- ✅ Site accessible at `http://localhost:8000`

#### Test 4: Browser Console Check
Open `http://localhost/` in browser and check DevTools console

**Expected Results**:
- ✅ No 404 errors for assets
- ✅ No JavaScript errors
- ✅ No broken images

### Usage Commands

**Production mode** (build + serve with nginx):
```bash
docker compose up
```

**Development mode** (live reload):
```bash
docker compose --profile dev up mkdocs-serve
```

**Rebuild everything**:
```bash
docker compose down -v  # Remove volumes
docker compose build --no-cache
docker compose up
```

**Access the site**:
- Production: http://localhost/
- Development: http://localhost:8000/
- Health check: http://localhost/health

---

## Part B: Link Checker Tool

### Objectives
1. Crawl the entire Docker-served site
2. Validate every link (internal and external)
3. Check navigation structure integrity
4. Generate YAML report mapping pages to broken links
5. Optimize for performance (compute-bound, not I/O bound)
6. Include comprehensive tests using pytest
7. Format code with black, lint with ruff

### Architecture

```
┌──────────────────────────────────────────────────────┐
│                 link_check CLI                       │
│          (link_check/link_check.py)                 │
└──────────────────┬───────────────────────────────────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
   ┌────────┐ ┌──────────┐ ┌───────────────┐
   │ Spider │ │ Checker  │ │ Nav Validator │
   │        │ │          │ │               │
   │ Crawl  │ │ Validate │ │ Check mkdocs  │
   │ site   │ │ links    │ │ navigation    │
   └───┬────┘ └────┬─────┘ └───────┬───────┘
       │           │                │
       └───────────┴────────────────┘
                   │
                   ▼
            ┌──────────────┐
            │   Reporter   │
            │              │
            │ Generate     │
            │ YAML report  │
            └──────────────┘
```

### Module Design

#### 1. `link_check/config.py`
**Purpose**: Configuration constants and settings

**Contents**:
- Default concurrency level (20 workers)
- Timeout settings
- User agent string
- URL patterns to exclude (if any)
- Output file path

#### 2. `link_check/spider.py`
**Purpose**: Crawl the site and discover all links

**Key Classes/Functions**:
- `Spider` class
  - `async def crawl(start_url: str) -> dict[str, set[str]]`
  - Returns: Map of source page URL → set of linked URLs

**Algorithm**:
1. Start from home page
2. Use BFS traversal
3. Extract all links from each page:
   - `<a href="...">`
   - `<link href="...">` (CSS)
   - `<script src="...">`
   - `<img src="...">`
   - Check `<nav>` elements
4. Deduplicate URLs
5. Stay within same origin (Docker container)
6. Use `asyncio` + `httpx` for concurrency

**Performance Optimization**:
- Semaphore to limit concurrent requests (20)
- Connection pooling via httpx.AsyncClient
- LRU cache for visited URLs
- Early termination on repeated errors

**Data Structures**:
```python
{
    "http://localhost/page1.html": {
        "http://localhost/page2.html",
        "http://localhost/assets/style.css",
        "https://external.com/link"
    },
    "http://localhost/page2.html": {
        "http://localhost/page3.html"
    }
}
```

#### 3. `link_check/checker.py`
**Purpose**: Validate discovered links

**Key Classes/Functions**:
- `LinkChecker` class
  - `async def check_links(link_map: dict) -> dict[str, list[BrokenLink]]`
  - Returns: Map of source page → list of broken links

**Check Types**:
1. **HTTP Status Check**: Send HEAD request (or GET if HEAD fails)
   - 2xx: Valid
   - 404: Broken
   - 5xx: Server error
   - Timeout: Connection error

2. **Anchor Check**: For URLs with fragments (#anchor)
   - Fetch page content
   - Parse HTML
   - Verify element with `id="anchor"` or `name="anchor"` exists

3. **Asset Check**: Verify CSS/JS/images load
   - Same as HTTP status check

**Data Structure**:
```python
@dataclass
class BrokenLink:
    url: str
    status_code: int | None
    error_message: str
    link_text: str | None
    link_type: str  # "internal", "external", "asset", "anchor"
```

**Result**:
```python
{
    "http://localhost/page1.html": [
        BrokenLink(
            url="http://localhost/missing.html",
            status_code=404,
            error_message="Not Found",
            link_text="Click here",
            link_type="internal"
        )
    ]
}
```

**Performance Optimization**:
- Concurrent checking with asyncio
- Reuse HTTP client connections
- Cache results for duplicate URLs
- External link checking with timeout (5s)

#### 4. `link_check/nav_validator.py`
**Purpose**: Validate MkDocs navigation structure

**Key Classes/Functions**:
- `NavValidator` class
  - `def validate_nav(mkdocs_yml_path: str, site_url: str) -> list[NavError]`
  - Returns: List of broken navigation entries

**Algorithm**:
1. Parse `documentation/mkdocs.yml` using `ruamel.yaml`
2. Extract `nav:` section
3. Handle `!include` directives (monorepo plugin):
   - Recursively parse included mkdocs.yml files
   - Build complete navigation tree
4. For each nav entry, extract URL
5. Check if URL exists on the served site
6. Report any 404s

**Challenges**:
- Monorepo structure with 12 sub-sites
- `!include` directives need special handling
- Relative vs absolute URLs
- Index pages may map to directory URLs

**Data Structure**:
```python
@dataclass
class NavError:
    nav_path: str  # e.g., "Core Reference → Language Guide"
    url: str
    error_message: str
```

#### 5. `link_check/reporter.py`
**Purpose**: Generate YAML report

**Key Classes/Functions**:
- `Reporter` class
  - `def generate_report(broken_links: dict, nav_errors: list) -> str`
  - Returns: YAML string

**Report Format**:
```yaml
summary:
  total_pages_crawled: 150
  total_links_checked: 5000
  broken_links_count: 12
  pages_with_errors: 8
  navigation_errors: 2
  timestamp: "2025-10-28T14:30:00Z"

broken_links:
  "http://localhost/guide/installation.html":
    - url: "http://localhost/missing-page.html"
      status: 404
      error: "Not Found"
      link_text: "See documentation"
      link_type: "internal"
    - url: "http://localhost/guide/overview.html#missing-anchor"
      status: 200
      error: "Anchor not found"
      link_text: "Jump to section"
      link_type: "anchor"

navigation_errors:
  - nav_path: "User Guide → Installation → Windows"
    url: "http://localhost/install/windows"
    error: "404 Not Found"
  - nav_path: "Core Reference → API Reference"
    url: "http://localhost/api/reference.html"
    error: "404 Not Found"

external_links_checked:
  total: 45
  failed: 2
  failures:
    - url: "https://example.com/broken"
      error: "Connection timeout"
      found_on:
        - "http://localhost/page1.html"
        - "http://localhost/page2.html"
```

**Functions**:
- Save report to `link_check_report.yaml`
- Optional: Print summary to console
- Optional: Return exit code based on errors (0 if no errors, 1 if errors)

#### 6. `link_check/link_check.py`
**Purpose**: CLI interface

**Implementation**: Use `click` library for CLI

**Command Structure**:
```python
import click

@click.command()
@click.option('--url', default='http://localhost', help='Base URL to check')
@click.option('--output', default='link_check_report.yaml', help='Output file')
@click.option('--concurrency', default=20, help='Number of concurrent workers')
@click.option('--timeout', default=10, help='Request timeout in seconds')
@click.option('--skip-external', is_flag=True, help='Skip external link checking')
@click.option('--verbose', is_flag=True, help='Verbose output')
def main(url, output, concurrency, timeout, skip_external, verbose):
    """Check all links in MkDocs site."""
    # Implementation
```

**Workflow**:
1. Initialize Spider and crawl site
2. Initialize LinkChecker and validate links
3. Initialize NavValidator and check navigation
4. Generate report with Reporter
5. Print summary
6. Save report to file
7. Exit with appropriate code

**Progress Reporting**:
- Show progress bar for crawling
- Show progress bar for link checking
- Print summary at end

**Example Output**:
```
🕷️  Crawling site at http://localhost...
Found 150 pages (5000 links)

🔗 Checking links...
[████████████████████████████████] 5000/5000 (100%)

📊 Navigation validation...
Checking 45 nav items...

✅ Link Check Complete!

Summary:
  Pages crawled: 150
  Links checked: 5000
  Broken links: 12
  Navigation errors: 2

Report saved to: link_check_report.yaml
```

### Project Structure

```
link_check/
├── __init__.py
├── link_check.py          # CLI entry point
├── config.py              # Configuration
├── spider.py              # Site crawler
├── checker.py             # Link validator
├── nav_validator.py       # Navigation checker
├── reporter.py            # YAML report generator
├── pyproject.toml         # Project config (uv)
├── ruff.toml             # Ruff linting config
├── tests/
│   ├── __init__.py
│   ├── test_spider.py
│   ├── test_checker.py
│   ├── test_nav_validator.py
│   ├── test_reporter.py
│   ├── test_integration.py
│   └── fixtures/
│       ├── sample_site/
│       └── mkdocs_configs/
└── README.md
```

### Dependencies (`pyproject.toml`)

```toml
[project]
name = "link-check"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27.0",           # Async HTTP client
    "beautifulsoup4>=4.12.0",   # HTML parsing
    "pyyaml>=6.0.0",            # YAML output
    "click>=8.1.0",             # CLI framework
    "ruamel.yaml>=0.18.0",      # YAML parsing (for mkdocs.yml)
    "rich>=13.0.0",             # Terminal formatting and progress bars
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.1.0",
    "pytest-mock>=3.12.0",
    "black>=24.0.0",
    "ruff>=0.3.0",
]

[project.scripts]
link-check = "link_check.link_check:main"

[tool.black]
line-length = 88
target-version = ["py311"]

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"
python_functions = "test_*"
addopts = "--cov=link_check --cov-report=html --cov-report=term"
asyncio_mode = "auto"
```

### Ruff Configuration (`ruff.toml`)

```toml
line-length = 88
target-version = "py311"

[lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "N",   # pep8-naming
    "UP",  # pyupgrade
    "B",   # flake8-bugbear
    "A",   # flake8-builtins
    "C4",  # flake8-comprehensions
    "PT",  # flake8-pytest-style
]

ignore = [
    "E501",  # Line too long (handled by black)
]

[lint.per-file-ignores]
"tests/**/*.py" = ["PT", "S"]  # Allow assert statements in tests
```

### Testing Strategy for Part B

#### Unit Tests

**`tests/test_spider.py`**:
- Test URL extraction from HTML
- Test deduplication
- Test same-origin filtering
- Test handling of relative vs absolute URLs
- Test malformed HTML handling
- Mock HTTP responses

**`tests/test_checker.py`**:
- Test 404 detection
- Test anchor validation
- Test external link checking
- Test timeout handling
- Test connection error handling
- Mock HTTP responses

**`tests/test_nav_validator.py`**:
- Test YAML parsing
- Test `!include` handling
- Test nav structure extraction
- Test URL validation
- Mock file system

**`tests/test_reporter.py`**:
- Test YAML output format
- Test summary calculation
- Test edge cases (no errors, all errors)
- Validate YAML structure

#### Integration Tests

**`tests/test_integration.py`**:
1. **Mock HTTP Server**:
   - Create simple HTTP server with known broken links
   - Test full crawl → check → report pipeline
   - Verify report accuracy

2. **Real Site Test** (optional):
   - Run against actual Docker container
   - Verify no false positives
   - Verify catches intentional broken links

#### Fixtures

**`tests/fixtures/sample_site/`**:
- `index.html` - Homepage with various link types
- `page1.html` - Working page
- `page2.html` - Page with broken links
- `page3.html` - Page with missing anchors
- `assets/` - CSS/JS files

**`tests/fixtures/mkdocs_configs/`**:
- `simple.yml` - Simple mkdocs config
- `monorepo.yml` - Config with !include directives
- `subsite.yml` - Included sub-site config

### Performance Optimization

**Goal**: Check ~150 pages with ~5000 links in under 5 minutes on local Docker

**Strategies**:
1. **Concurrency**: 20 workers (tunable)
2. **Connection Pooling**: Reuse HTTP connections
3. **Caching**: Cache checked URLs (avoid duplicate checks)
4. **HEAD Requests**: Use HEAD instead of GET when possible
5. **Batch Processing**: Check links in batches
6. **Early Termination**: Stop checking if too many errors
7. **Smart Queueing**: Prioritize internal links over external

**Benchmarking**:
- Measure time for crawling phase
- Measure time for checking phase
- Measure time for nav validation
- Measure total time
- Provide `--profile` flag to show timing breakdown

### Usage Examples

**Basic usage**:
```bash
# Start Docker containers
docker compose up -d

# Run link checker
cd link_check
uv run link-check
```

**Advanced usage**:
```bash
# Custom URL
uv run link-check --url http://localhost:8080

# Higher concurrency
uv run link-check --concurrency 50

# Skip external links
uv run link-check --skip-external

# Verbose output
uv run link-check --verbose

# Custom output file
uv run link-check --output custom_report.yaml
```

**Development workflow**:
```bash
# Install dependencies
cd link_check
uv sync

# Run tests
uv run pytest

# Run with coverage
uv run pytest --cov

# Format code
uv run black .

# Lint code
uv run ruff check .

# Auto-fix linting issues
uv run ruff check --fix .
```

---

## Implementation Phases

### Phase 1: Docker Setup (Part A)
**Goal**: Get MkDocs site building and serving with nginx

**Tasks**:
1. Create Docker directory and Dockerfiles
2. Create entrypoint script
3. Create nginx config
4. Create docker-compose.yml
5. Test build process
6. Test nginx serving
7. Verify site loads correctly

**Success Criteria**:
- Site builds without errors
- Nginx serves site on port 80
- All assets load correctly
- Navigation works
- No console errors

**Estimated Time**: 2-3 hours

### Phase 2: Link Checker Foundation (Part B)
**Goal**: Basic crawling and link checking working

**Tasks**:
1. Set up project structure
2. Create pyproject.toml with dependencies
3. Implement `config.py`
4. Implement `spider.py` (basic crawling)
5. Write tests for spider
6. Implement `checker.py` (basic checking)
7. Write tests for checker

**Success Criteria**:
- Can crawl local Docker site
- Can detect 404 errors
- Tests pass
- Code formatted with black
- Linting passes with ruff

**Estimated Time**: 3-4 hours

### Phase 3: Navigation Validation (Part B)
**Goal**: Validate MkDocs navigation structure

**Tasks**:
1. Implement `nav_validator.py`
2. Handle monorepo `!include` directives
3. Write tests for nav validator
4. Integrate with main checker

**Success Criteria**:
- Can parse mkdocs.yml files
- Detects broken nav links
- Handles sub-sites correctly
- Tests pass

**Estimated Time**: 2-3 hours

### Phase 4: Reporting & CLI (Part B)
**Goal**: Complete CLI tool with YAML reporting

**Tasks**:
1. Implement `reporter.py`
2. Implement `link_check.py` CLI
3. Add progress bars and output formatting
4. Write integration tests
5. Write README documentation

**Success Criteria**:
- Generates valid YAML report
- CLI is easy to use
- Progress feedback is clear
- Integration tests pass
- Documentation is complete

**Estimated Time**: 2-3 hours

### Phase 5: Optimization & Polish
**Goal**: Performance tuning and final testing

**Tasks**:
1. Benchmark performance
2. Tune concurrency settings
3. Add caching optimizations
4. Test against full site
5. Fix any issues found
6. Add any missing edge case handling

**Success Criteria**:
- Completes in under 5 minutes
- No false positives
- Catches all broken links
- Clean code (black + ruff)
- Good test coverage (>90%)

**Estimated Time**: 2-3 hours

---

## Total Estimated Time

**Part A (Docker)**: 2-3 hours
**Part B (Link Checker)**: 9-12 hours
**Total**: 11-15 hours

---

## Risks & Mitigation

### Risk 1: Git Submodules Not Initializing in Docker
**Mitigation**: Test submodule init early; ensure git is installed in container; provide clear error messages

### Risk 2: MkDocs Plugins Missing or Wrong Versions
**Mitigation**: Use exact version pins from repository; test build early; check GitHub Actions workflow for reference

### Risk 3: Link Checker False Positives
**Mitigation**: Thorough testing with known broken links; handle edge cases (redirects, anchor links); make configurable

### Risk 4: Performance Too Slow
**Mitigation**: Benchmark early; tune concurrency; add caching; consider async optimizations

### Risk 5: Monorepo Navigation Parsing Complex
**Mitigation**: Start with simple cases; incrementally handle `!include` directives; test with actual config files

### Risk 6: Docker Volume Permissions Issues (Windows)
**Mitigation**: Use named volumes; test on Windows early; document any platform-specific setup

---

## Success Metrics

**Part A - Docker Setup**:
- ✅ Site builds in < 5 minutes
- ✅ Nginx serves site without errors
- ✅ All 12 sub-sites accessible
- ✅ Assets load correctly (no 404s)
- ✅ Search works
- ✅ Can rebuild from scratch reliably

**Part B - Link Checker**:
- ✅ Discovers all pages (compare to expected count)
- ✅ Detects 100% of intentionally broken links
- ✅ Zero false positives on working links
- ✅ Completes in < 5 minutes for full site
- ✅ YAML report is valid and informative
- ✅ All tests pass
- ✅ Code passes black + ruff checks
- ✅ Test coverage > 90%
- ✅ Documentation is clear and complete

---

## Next Steps

1. **Review this plan** - Confirm approach and make any adjustments
2. **Start Phase 1** - Create Docker setup
3. **Test Phase 1** - Verify Docker works end-to-end
4. **Start Phase 2** - Begin link checker implementation
5. **Iterate** - Test each component as it's built
6. **Integrate** - Run link checker against Docker site
7. **Polish** - Optimize and document
8. **Done!** - Deliverable complete

---

## Open Questions

1. Should external links be checked by default, or skip them?
   - **Recommendation**: Check by default, but make it configurable with `--skip-external`

2. What timeout should be used for external links?
   - **Recommendation**: 10 seconds (configurable)

3. Should the link checker run inside Docker or as a separate tool?
   - **Recommendation**: Separate tool (easier to develop and test)

4. How to handle redirects (301/302)?
   - **Recommendation**: Follow redirects, but note them in verbose output

5. Should we check for dead external links or just validate they respond?
   - **Recommendation**: Just validate they respond (2xx/3xx status)

6. Git submodule in Docker: Should we commit the submodule or always fetch fresh?
   - **Recommendation**: Initialize fresh each build (more reliable)

7. How to handle rate limiting for external links?
   - **Recommendation**: Add delay between external requests (configurable)
