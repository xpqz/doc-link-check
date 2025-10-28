winget install --id=astral-sh.uv -e
uv python install 3.12
winget install --id GitHub.cli (or choco install gh)
gh auth login


Here is our job for today: creating a PLAN.md. No implementation yet, just a plan. My goal is to (a) write a docker compose setup that renders the mkdocs site and serves it. I want to serve it in Docker, using nginx for speed. (b), create a tool link_check/link_check.py which should spider the site as served in Docker, and verify every link, reporting back on any 404s. The report should be in yaml, and map the page to any broken links. Also verify the nav itself: if any links presented by the nav are broken, report those, too. First, carefully construct a detailed plan for (a) and (b), considering how we can iterate and especially test each step. Consider performance for (b). As we're always running against a local Docker container, we're likely compute bound, not IO. We want tests using pytest, formatting with black, and linting with ruff.