Project: gokuls-blog (Zola static site)
Last verified: 2025-08-13

Scope
- This document captures project-specific build/configuration details, testing guidance, and development notes for contributors familiar with Zola and static site workflows.

1. Build and configuration
- Generator: Zola (https://www.getzola.org/)
  - Zola handles Markdown rendering, templates (Tera), Sass compilation, and site building.
  - config.toml declares compile_sass = true and theme = "simple-dev-blog".
- Install Zola (Windows):
  - Chocolatey: choco install zola
  - Cargo: cargo install zola
  - Scoop: scoop install zola
- Base URL
  - config.toml sets base_url = "https://gokulprathin8.github.io/gokuls-blog" for GitHub Pages.
  - zola serve ignores base_url for local dev; no need to change it to preview locally.
- Build/serve commands
  - Local dev server with live reload: zola serve --open
  - Production build to ./public: zola build
  - Include drafts in local builds: zola build --drafts or zola serve --drafts
- Theme
  - theme = "simple-dev-blog"; templates in ./templates override theme templates as needed.
  - Custom site variables live under [extra] in config.toml (e.g., nav, latest_text, accent colors, images).
- Sass/CSS
  - compile_sass = true; Zola compiles ./sass/*.scss into CSS during serve/build.
  - Any new stylesheet partials must be referenced from an entry SCSS file (e.g., main.scss) to be included.
- Taxonomies
  - taxonomies = [{ name = "tags" }] enabled. Ensure posts define tags in front matter to populate /tags pages.
- Search and feeds
  - build_search_index = false in config; enable if needed.
  - RSS/Atom feed generation is currently commented (generate_feed); enable if publishing feeds is desired.
- Images
  - Static images are under ./content (page bundles) or ./static (copied as-is). Processed images are generated under ./static/processed_images and surfaced under ./public/processed_images after build.

2. Testing: configuration, running, and adding tests
A. Quick smoke tests
- Build smoke test: zola build should complete without errors and produce public/index.html.
- Serve smoke test: zola serve should render the site locally with no template errors in the console.

B. Structural self-check (PowerShell) — example test
- Purpose: Validate that the repo contains the expected structure and that a prior build has produced essential artifacts. This does not require Zola to be installed and is useful in CI or constrained environments.
- How to run:
  1) Save the script below as scripts/selfcheck.ps1.
  2) Run: powershell -NoProfile -ExecutionPolicy Bypass -File scripts\selfcheck.ps1
  3) Expected: All self-checks passed.
- Script contents:
  - BEGIN scripts/selfcheck.ps1 -
    param()
    $ErrorActionPreference = 'Stop'
    Write-Host 'Zola blog repo self-check started...'
    if (-not (Test-Path -Path 'config.toml')) { throw 'Missing config.toml at repo root.' }
    $config = Get-Content 'config.toml' -Raw
    $requiredKeys = @('compile_sass = true','theme = "simple-dev-blog"','taxonomies','[markdown]')
    foreach ($k in $requiredKeys) { if ($config -notmatch [Regex]::Escape($k)) { throw "config.toml missing expected entry: $k" } }
    if (-not (Test-Path -Path 'content')) { throw 'Missing content directory.' }
    $posts = Get-ChildItem -Path 'content' -Recurse -Include *.md -File
    if ($posts.Count -lt 1) { throw 'No markdown content found under /content.' }
    $tpls = @('templates/index.html','templates/base.html','templates/blog.html')
    foreach ($t in $tpls) { if (-not (Test-Path -Path $t)) { throw "Missing template: $t" } }
    if ($config -match 'compile_sass\s*=\s*true') {
      if (-not (Test-Path -Path 'sass')) { throw 'compile_sass=true but /sass directory is missing.' }
      $scss = Get-ChildItem -Path 'sass' -Recurse -Include *.scss -File
      if ($scss.Count -lt 1) { throw 'No .scss files found under /sass.' }
    }
    if (-not (Test-Path -Path 'public/index.html')) { throw 'public/index.html was not found. Run `zola build` to generate it.' }
    Write-Host 'All self-checks passed.'
    exit 0
  - END scripts/selfcheck.ps1 -
- Notes:
  - This project already passed the above self-check on 2025-08-13.
  - For CI, you can run the self-check before and/or after zola build to report clearer failures.

C. Adding and executing new tests
- Repository-structure tests: Add more assertions in PowerShell under scripts/test-*.ps1 (e.g., check that every post has required front matter like title and date, or that no template includes unresolved Tera variables).
- Content-lint tests: Write a small script to parse front matter and enforce conventions (e.g., tags required, maximum title length). PowerShell or a tiny Node/Python script is sufficient; keep runtime dependencies minimal or pinned.
- Build tests: In CI or locally with Zola installed, run zola build and validate:
  - Exit code is zero.
  - public/index.html exists.
  - Optional: grep the HTML for key markers (e.g., nav entries in templates/base.html) using Select-String.
- Draft posts: To ensure drafts don’t leak into production, build without --drafts in CI and check that any draft slugs are absent in public/.

D. Example: add-and-build test (manual)
- To verify taxonomies and listing pages render correctly:
  1) Create a temporary post content/blog/tmp-test.md with front matter:
     +++
     title = "Temporary Test"
     date = 2025-08-13
     draft = true
     tags = ["beginner"]
     +++
     Body test.
  2) Run: zola build --drafts
  3) Verify public/blog/index.html and the tag page public/tags/beginner/index.html include the test title.
  4) Delete content/blog/tmp-test.md after validation.
- Important: Do not commit temporary content or generated artifacts when only testing locally.

3. Development notes and conventions
- Content organization
  - Blog posts live under content/blog/*.md. Use TOML front matter with at least: title, date (YYYY-MM-DD), tags (optional but recommended), and draft when appropriate.
  - About/landing pages live under content/*.md.
- Tera templates
  - Main templates in ./templates; base.html defines the layout; index.html, blog.html, blog-post.html, etc. extend/compose it.
  - Use built-in Zola variables and those defined under [extra] in config.toml (e.g., extra.nav, colors, images). Avoid hard-coding duplicate values in templates.
- Styling
  - Put SCSS in ./sass and import partials into an entry file referenced by templates (e.g., main.scss). Zola emits compiled CSS; ensure <link> tags point to the output CSS file name that Zola generates.
- Configuration hygiene
  - Keep base_url pointed at the canonical GitHub Pages URL. For forks, contributors should not change base_url; they can still run zola serve locally.
  - Consider enabling generate_feed when ready to publish feeds.
  - build_search_index can be enabled if client-side search is needed by the theme.
- Images and static assets
  - Use ./static for assets that should be copied verbatim; reference them with site-relative URLs (e.g., /icon.png).
  - Processed images under ./static/processed_images are generated; do not hand-edit these.
- Accessibility/SEO
  - Ensure alt attributes on images in content and templates.
  - Keep titles and descriptions concise; config.toml description feeds meta tags in templates.
- Debugging tips
  - Use zola serve and check terminal output for template errors (undefined variables, bad filters).
  - If CSS changes aren’t reflected, confirm the SCSS is imported by an entry file and that templates link to the compiled CSS.
  - For link issues on GitHub Pages, confirm base_url correctness and that relative links in content/templates don’t start with file:// or use mixed slashes.

4. Verified test run in this repository (2025-08-13)
- Ran structural self-check (PowerShell) described above; it passed with:
  - config.toml present and contains expected keys
  - content/*.md present
  - templates/* present (index.html, base.html, blog.html)
  - sass/*.scss present
  - public/index.html present (indicates a successful prior build)
- Cleaned up any temporary test scripts afterward as required.

Cleanup policy for tests in this repo
- Do not leave temporary test files, scripts, or content in the repository after demonstration. The only persistent file added by this effort is .junie/guidelines.md.
