# WordPress Elementor: Direct DB Management and Theme Builder Migration

A practical guide from building and managing a WordPress/Elementor website via direct database manipulation, including migrating from per-page nav/footer injection to Elementor Pro Theme Builder templates. Based on the Peregrine Technology Systems website build (March 2026).

## Why Direct DB Management?

When building Elementor pages programmatically (e.g., from JSON templates managed in git), the Elementor editor UI is too slow and error-prone for rapid iteration. Direct database manipulation lets you:

- Store canonical page templates as JSON in git
- Push changes via script in seconds
- Maintain version history and rollback capability
- Work alongside AI coding assistants that can generate Elementor JSON

## Architecture Overview

```
Local (git repo)                       Staging Server (WordPress)
+---------------------------+          +-----------------------------------+
| theme/elementor-*.json    |   SCP    | ~/peregrinetechsys.net/           |
| push-page.php             |--------->|   page-content.json               |
| brand-book/               |          |   push-page.php                   |
+---------------------------+          |   backups/page_*_timestamp.json   |
                                       +-----------------------------------+
                                              |
                                       +------+------+
                                       | wpig_postmeta                    |
                                       |   _elementor_data (JSON)         |
                                       |   _elementor_element_cache       |
                                       |   _elementor_css                 |
                                       +----------------------------------+
```

## Key Database Tables

| Table | Purpose |
|-------|---------|
| `wpig_posts` | Pages, templates, menu items (post_type: page, elementor_library, nav_menu_item) |
| `wpig_postmeta` | Elementor data, page settings, template type, display conditions |
| `wpig_options` | Active plugins, theme mods, Elementor global settings, transient cache |
| `wpig_terms` / `wpig_term_taxonomy` | Menu definitions |
| `wpig_term_relationships` | Menu item → menu associations |

## Gotchas

### 1. Always clear Elementor element cache after DB changes
Elementor caches rendered output in `_elementor_element_cache` postmeta. If you update `_elementor_data` without deleting the cache, the old content renders.

```sql
DELETE FROM wpig_postmeta WHERE post_id=<PAGE_ID> AND meta_key='_elementor_element_cache';
DELETE FROM wpig_postmeta WHERE post_id=<PAGE_ID> AND meta_key='_elementor_css';
DELETE FROM wpig_options WHERE option_name LIKE '%_transient_%';
```

### 2. json_decode/json_encode round-trips are lossy for Elementor data
When you decode Elementor JSON, modify it in PHP, and re-encode, `json_encode` subtly changes Unicode escaping, forward slash escaping, or whitespace. This can cause:
- `affected_rows = 0` on UPDATE (MySQL sees identical value)
- Elementor failing to parse the re-encoded data
- Widget settings not rendering correctly

**Fix:** For targeted text changes, use `str_replace()` on the raw DB string without decoding. Only use `json_decode`/`json_encode` for structural changes (adding/removing sections).

### 3. LiteSpeed Cache fights with everything
The staging server runs LiteSpeed Cache, which caches aggressively at multiple levels:
- `wp-content/litespeed/cssjs/` — CSS/JS cache
- `wp-content/litespeed/htmlfinalize/` — HTML cache
- `wp-content/cache/` — General cache

Even after clearing Elementor caches in the DB, LiteSpeed serves stale pages. You must also:
```bash
rm -rf wp-content/litespeed/* wp-content/cache/* wp-content/uploads/elementor/css/*
```

### 4. `elementor_canvas` vs `default` page template
- `elementor_canvas` = full-screen, NO theme header/footer (used when nav/footer baked into page data)
- `default` = theme template, renders Theme Builder header/footer via `elementor_theme_do_location()`
- `elementor_header_footer` = used on Theme Builder templates themselves

When migrating to Theme Builder, switch pages from `elementor_canvas` to `default`.

### 5. Theme Builder templates must be created via UI first
Creating Theme Builder templates (header/footer) via raw DB INSERT misses critical metadata initialization:
- `_elementor_page_settings` needs to be a properly serialized array
- `_elementor_page_assets` needs widget registration
- Display conditions need proper serialization in both the template's `_elementor_conditions` meta AND the global `elementor_pro_theme_builder_conditions` option

**Fix:** Create blank templates via the Elementor UI (Templates → Theme Builder → Add New), then populate content via DB. The UI properly initializes all internal metadata.

### 6. Nav Menu widget settings don't apply via DB
The Elementor Pro `nav-menu` widget has specific setting keys that differ from documentation. Setting `color_menu_item`, `typography_menu_*` etc. via DB doesn't reliably apply.

**Fix:** Use a physical CSS file enqueued at high priority (9999) via `functions.php`:
```php
add_action('wp_enqueue_scripts', function() {
    wp_enqueue_style('overrides', get_template_directory_uri() . '/overrides.css', [], '1.0');
}, 9999);
```

### 7. WordPress custom CSS post doesn't always load
Adding CSS via the `custom_css` post type and linking it in `theme_mods` doesn't reliably apply. The Elementor kit and theme settings can override it.

**Fix:** Use a physical CSS file (gotcha #6) instead of the WordPress customizer CSS.

### 8. Body background color hides wrapper gaps
The `hello-elementor` theme wraps page content in `<main>`, `<article>`, `<div.entry-content>` elements that can have default margin/padding. These create visible gaps between full-width sections and the Theme Builder header/footer.

**Fix:** Set `body { background-color: #0f172a !important; }` (matching your dark sections) so gaps blend in, plus aggressive margin/padding removal on all wrapper elements.

### 9. Always backup before pushing
The push script should always backup existing DB content before overwriting:
```php
$backup_file = "$backup_dir/page_{$page_id}_" . date('Y-m-d_His') . ".json";
file_put_contents($backup_file, $existing[0]);
```
This saved us multiple times when pushes corrupted page data.

### 10. UTF-8 encoding: use utf8mb4, never str_replace on DB data
WordPress DB charset is often utf8mb3/utf8mb4. Direct `str_replace()` on DB data containing curly quotes, em dashes, or arrows can corrupt encoding. Use:
- `$db->set_charset('utf8mb4')` on connection
- Python with `json.load()`/`json.dump()` for safe JSON manipulation
- Avoid inline PHP with complex HTML/CSS — write to file and upload via SCP

### 11. Cache-busting image URLs in Elementor
When updating images (e.g., report screenshots), the browser and WordPress serve cached versions even after uploading new files. Add cache-busting query strings:
```
/wp-content/uploads/2026/03/report-cover.png?v=1774201927
```
Update the Elementor JSON to include `?v=TIMESTAMP` on image src URLs.

### 12. Maintenance mode via plugin activation
Toggle maintenance mode by adding/removing the maintenance plugin from the `active_plugins` option:
```php
$plugins = unserialize($db->query("SELECT option_value FROM wpig_options WHERE option_name='active_plugins'")->fetch_row()[0]);
$plugins[] = 'maintenance/maintenance.php'; // ON
// or: array_filter($plugins, fn($p) => $p !== 'maintenance/maintenance.php'); // OFF
$db->query("UPDATE wpig_options SET option_value='" . serialize($plugins) . "' WHERE option_name='active_plugins'");
```

## Workflow: Edit → Validate → Commit → Push

```bash
# 1. Edit local JSON (Python for structural, text editor for content)
python3 -c "import json; ..."

# 2. Validate
python3 -c "import json; json.load(open('theme/elementor-page.json')); print('Valid')"

# 3. Commit to git
git add theme/elementor-page.json && git commit -m "feat: description"

# 4. Push to staging
scp theme/elementor-page.json staging:~/site/page-content.json
ssh staging "cd ~/site && php push-page.php <page_id> page-content.json"
```

## Theme Builder Migration Checklist

- [ ] Create header and footer templates via Elementor UI (not DB)
- [ ] Set display conditions to "Entire Site" via UI
- [ ] Populate template content via DB (after UI creates proper metadata)
- [ ] Set up WordPress menu (Appearance → Menus, or via DB)
- [ ] Add physical CSS file for nav colors and gap fixes
- [ ] Enqueue CSS via `functions.php` at priority 9999
- [ ] Switch pages from `elementor_canvas` to `default` template
- [ ] Strip nav/footer sections from page `_elementor_data`
- [ ] Update push script to stop injecting nav/footer
- [ ] Clear ALL caches (Elementor DB, LiteSpeed files, browser)
- [ ] Test on fresh incognito/private browser window

## Files Referenced

| File | Purpose |
|------|---------|
| `push-page.php` | Pushes JSON content to WordPress DB with backup |
| `theme/elementor-home-complete.json` | Home page Elementor template |
| `theme/elementor-penetrator.json` | Penetrator product page template |
| `wp-content/themes/hello-elementor/peregrine-overrides.css` | Global CSS overrides |
| `wp-content/themes/hello-elementor/functions.php` | CSS enqueue hook |
