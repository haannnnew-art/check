# XDMovies (local dev)

A lightweight PHP-based site for listing movie and TV series download pages and metadata, with server-side page generation and client-side interactivity. This README provides an overview and developer instructions for local setup and common tasks.

> NOTE: This README intentionally omits the **contents** of the `movies/` and `series/` folders (these contain many generated HTML pages). It still lists these folders as top-level locations.

---

## Quick Start (local)

Prerequisites:
- Windows with XAMPP (Apache + PHP + MySQL) or equivalent environment
- MySQL database (schema is used from `php/db.php`)
- PHP 7.4+ recommended (matching your XAMPP installation)

1. Place the repository inside your web directory (e.g., `C:\xampp\htdocs\`).
2. Create a `.env` file at the project root (this repo includes a `.env` in dev):

   ```ini
   AUTH_TOKEN=your-secret-token-here
   SITE_URL=http://localhost
   ```

3. Ensure `.env` is not committed (.gitignore contains `.env`).
4. Import or configure your database using `php/db.php` settings.
5. Regenerate all static pages (optional) via browser or command line:
   - Browser: `http://localhost/regenerate_pages.php?type=all&limit=500` (adjust for your use)
   - Or navigate to `regenerate_pages.php` and specify `?type=movie|series|all`.

6. Open `http://localhost` in your browser and verify pages.

---

## Project structure (top-level)

- `index.php` — Site homepage, includes server-side rendered `render_cards.php` and JS.
- `category.php` — Category (OTT/language) listing page.
- `search.html` — Static search HTML page backed by `php/search_api.php`.
- `movies/` — Generated HTML movie pages (many thousands) — NOT listed here.
- `series/` — Generated HTML series pages (many thousands) — NOT listed here.
- `php/` — Server-side scripts and helpers:
  - `env.php` — loads `.env` into `getenv()`, `$_ENV`, and `$_SERVER`.
  - `auth_check.php` — verifies `X-Auth-Token` for AJAX endpoints (uses `AUTH_TOKEN`).
  - `get_token.php` — a JSON endpoint to return `AUTH_TOKEN` for static pages to call (fallback; not required for pages where the token is injected into `window.AUTH_TOKEN`).
  - `search_api.php` — Search endpoint (AJAX) used by front-end (requires X-Auth-Token header or valid cookie).
  - `movie_page_helper.php` — Shared logic to generate movie filenames, SEO text, and download renders.
  - `page_generator_movie.php` / `page_generator_series.php` — generate static HTML pages for a record.
  - `regenerate_pages.php` — triggers generation across the DB.
  - `movie-insert.php`, `series-insert.php` — insert/update scripts for content (used by admin flows).
  - `render_cards.php` — renders cards used on the index / category pages from DB.
  - Other utilities for sitemap, tmdb sync, caching, Cloudflare cache purge, etc.
- `js/` — client-side JS code
  - `main.js`, `search.js` and other helpers for client UI; they use `window.AUTH_TOKEN` (injected by PHP) or fallback to `/php/get_token.php`.
- `css/` — styles including `styles.css` and `movie-grid.css`.
- `assets/` — images (logo, etc.).
- `.htaccess` — rewrite rules that remap `/movies/slug` to `/movies/slug.html` so front-end links do not need the `.html` extension.

---

## Key features & developer notes

- Token handling and security:
  - The site previously used a hardcoded token in JS; the token is now managed in `.env` and read via `php/env.php`.
  - `php/auth_check.php` validates `X-Auth-Token` for protected endpoints. Several endpoints (search, fetch data) use this for request validation.
  - For pages generated via PHP at runtime (like `index.php`, `category.php` and generated `movies/` or `series/` pages), the token is injected into `window.AUTH_TOKEN` for use by client side JS.
  - A fallback endpoint `php/get_token.php` returns the token in JSON for cases where the page is static and cannot embed window.AUTH_TOKEN.
  - Important security note: If `AUTH_TOKEN` is confidential server-side-only, avoid injecting it into the front-end; instead use server-proxied requests (not all calls require client-visible tokens). If you need a more secure approach, remove client-facing tokens entirely and implement server-side forwarding for all operations that need protection.

- Clean URLs & rewrite rules:
  - `.htaccess` rewrites `/movies/slug` to `/movies/slug.html` and similarly for `/series/` — this allows links without `.html` while still serving static pages.
  - Server-side generators used `movie_get_filename()` to produce canonical names like `title-quality-audio-download-tmdbid.html`. Links generated programmatically now strip the `.html` to keep URLs clean.

- SEO block markup:
  - The movie pages use a details/summary block structure for an interactive SEO block (`details.seo-text` with `summary.seo-summary`) and the CSS in `styles.css` provides consistent styling.
  - Series pages: previously used `<div class="seo-text">` — updated to the same `details` markup for interactive behavior.
  - `css/styles.css` now applies consistent styling across `details.seo-text` and `.seo-text`.

- Regenerating pages & sitemap:
  - Use `php/regenerate_pages.php` to re-generate static movie and series pages.
  - After a regeneration, be sure to update the sitemap via `php/update_sitemap.php` or `php/sitemap_trigger.php` if you have a sitemap workflow. The repo contains helper scripts for this.

- JavaScript & fetch headers:
  - `js/main.js` and `js/search.js` now use `window.AUTH_TOKEN` as the `X-Auth-Token`. If `window.AUTH_TOKEN` is not set, the JS will attempt to `fetch('/php/get_token.php')`.
  - Avoid exposing `AUTH_TOKEN` to the front-end if this token needs to remain secret — prefer server-side proxies instead.

---

## How to change the token or rotate it

1. Update the `.env` file with the new `AUTH_TOKEN`.
2. If you are generating static pages, re-run the page generation script (see `regenerate_pages.php`). These pages contain `window.AUTH_TOKEN` in their HTML head (for static content) after generating.
3. If you use proxies or have cloud-caching, purge caches where appropriate (Cloudflare purge helpers are in `php/cloudflare_cache.php` and Cloudflare related helpers).

---

## Development tasks commonly performed

- Update layout or styles: edit `css/styles.css` or `css/movie-grid.css`.
- Add or adjust JavaScript for client UI: edit `js/main.js`, `js/search.js`, etc.
- Modify server API or DB helpers: edit `php/search_api.php`, `php/movie_page_helper.php`, and `php/*` where needed.
- Regenerate static pages: `http://localhost/regenerate_pages.php?type=all&limit=500`.
- Add new content to the database: use `php/movie-insert.php` / `php/series-insert.php` or via admin panel if present.

---

## Suggestions & Best Practices

- Do not commit `.env` or any secret tokens to version control; they are ignored by `.gitignore`.
- Use server-side fetch flows to avoid embedding server secrets in client JS files.
- When making changes that impact caching (e.g., update `index.php`, or generated pages), remember to purge Cloudflare or other CDN caches.
- Keep the `movies/` and `series/` directories as generated output — do not manually edit the generated HTML; change the generator templates under `php/page_generator_*.php` instead.

---

## Where to look for help

- Data & DB functions: `php/db.php`, `php/movie_page_helper.php`.
- Page generation: `php/page_generator_movie.php`, `php/page_generator_series.php`, `php/regenerate_pages.php`.
- Search and API: `php/search_api.php`, `php/series_get.php`.
- Frontend UI: `js/main.js`, `js/search.js`, `js/menu.js`.
- Styling: `css/styles.css`, `css/movie-grid.css`.

---

If you want, I can:
- Add a section with developer commands to run `regenerate_pages.php` from CLI (wrapping with `curl`),
- Add a script to automate regeneration and cache purge,
- Regenerate all series pages to ensure `details.seo-text` is present on all existing HTML series pages.

Tell me if you'd like any of these and I can add them to the README or implement them automatically.
