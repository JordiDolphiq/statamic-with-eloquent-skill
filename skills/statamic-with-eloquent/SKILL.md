---
name: statamic-with-eloquent
description: "Develops with Statamic CMS (statamic/cms) and the Eloquent Driver (statamic/eloquent-driver). Activates when working with Statamic entries, collections, blueprints, fieldsets, globals, navigations, taxonomies, assets, views, templates, View Composers, Bard/Replicator fields, content blocks, or cp utilities; or when the user mentions Statamic, CMS, blueprint, collection, entry, fieldset, global set, content blocks, please commands, or Statamic routing."
license: MIT
compatible_agents:
  - Claude Code
  - Cursor
tags:
  - eloquent
  - database
  - laravel
  - statamic
  - blade
  - cms
  - php
keywords: ['eloquent', 'statamic', 'laravel', 'database', 'blade', 'cms', 'php']
metadata:
  triggers: Statamic, statamic/cms, statamic/eloquent-driver, Blade CMS, blueprint, collection entries, Statamic entries
  author: Dolphiq
---

# Statamic CMS Development

## When to Apply

Activate this skill when:

- Creating or editing Statamic collections, blueprints, fieldsets, or globals
- Working with Statamic entries, taxonomies, or navigations
- Building page templates or content block partials
- Creating or modifying View Composers for Statamic views
- Working with Bard, Replicator, or other Statamic fieldtypes
- Querying content via Statamic facades
- Running `php please` commands

## Documentation

Use `search-docs` with `packages: ["statamic/cms"]` for version-specific Statamic documentation.

## Blade-Only Templating

This project uses **Blade exclusively** — no Antlers. All templates are `.blade.php` files. Statamic variables are accessed as regular PHP variables in Blade.

```blade
{{-- Correct: Blade syntax --}}
<h1>{{ $title }}</h1>

{{-- WRONG: Antlers syntax — never use this --}}
{{ title }}
```

## No Inline PHP in Blade (`@php`)

**Never use `@php ... @endphp` (or `<?php ?>`) in Blade templates.** Templates render data; they don't compute it.

### Order of Preference for Getting Data Into a Template

Work down this list and stop at the first option that works:

**1. Use what Statamic already provides (always try this first).** On Statamic-routed pages, the data cascade injects everything into the view. Don't fetch, compute, or pass what's already there:

- **Current entry fields** as top-level variables (`$title`, `$hero_title`) and via `$page` (`$page->title`)
- **Global sets** as one variable per handle (`$settings->site_name`, `$footer->copyright`)
- **Relationship fields** (`entries`, `terms`, `users`) as query builders. Property access resolves them; method access lets you chain: `$page->related_posts()->where('featured', true)->limit(3)->get()`
- **System/context variables** (see [All Variables](https://statamic.dev/variables/all-variables)): `$site`, `$sites`, `$homepage`, `$current_url`, `$current_uri`, `$segment_1`…`$segment_x`, `$last_segment`, `$is_homepage`, `$logged_in`, `$current_user`, `$environment`, `$live_preview`, `$now`, `$get`, `$post`, `$old`, `$config`, `$csrf_token`, `$csrf_field`, `$response_code`, `$current_template`, `$current_layout`
- **Entry/asset variables** on each item: `$url`, `$permalink`, `$edit_url`, `$slug`, `$date`, `$last_modified`, `$is_entry`, `$is_term`, `$collection`, `$taxonomy`; for assets also `$is_image`, `$width`, `$height`, `$focus_css`, `$size`, `$extension`, `$filename`
- **Blade's `$loop`** instead of Antlers loop vars: `$loop->first`, `$loop->last`, `$loop->iteration`, `$loop->index`, `$loop->count`

```blade
{{-- No composer, no @php: everything comes from the cascade --}}
<title>{{ $title }} | {{ $settings->site_name }}</title>
@if (! $is_homepage) <a href="{{ $homepage }}">Home</a> @endif
@foreach ($page->related_posts as $post)
    <a href="{{ $post->url }}" @class(['first' => $loop->first])>{{ $post->title }}</a>
@endforeach
```

**2. Use Statamic tags and modifiers in Blade.** For data outside the current entry, use tags rather than writing queries:

- Tag components: `<s:collection:news limit="3" sort="date:desc">`, `<s:nav:main>`, `<s:taxonomy:tags>`, `<s:glide :src="$image" width="800">`
- Fluent tags: `@foreach (Statamic::tag('collection:news')->limit(3) as $item)` (camelCase multi-word params; use `->param('title:contains', 'x')` for colon params)
- `@tags(['posts' => ['collection:blog' => ['limit' => 5]]])` to expose tag results as variables
- Modifiers: `{{ Statamic::modify($content)->stripTags()->safeTruncate([160, '…']) }}`

**3. Make the cascade available where it isn't.** In Blade components or views rendered from custom routes/controllers, `$page` and cascade data are missing. Use `@cascade(['page' => null, 'settings'])` (works like `@props`), or `@aware(['page'])` with context passed down from the layout, instead of re-querying.

**4. Only then, write PHP outside the template:**

- **Data preparation / unwrapping / complex queries** → a View Composer
- **Reusable formatting** → a helper function, a custom modifier, or a custom tag
- **Repeated markup with its own logic** → a Blade component (class-based if it needs PHP)
- **Simple expressions** → inline in the directive itself (`@foreach (unwrap($usps ?? []) as $item)`)

```blade
{{-- WRONG --}}
@php $items = unwrap($usps ?? []) @endphp
@foreach ($items as $item) ... @endforeach

{{-- WRONG: re-fetching what the cascade already provides --}}
@php $footer = \Statamic\Facades\GlobalSet::findByHandle('footer')->inCurrentSite() @endphp

{{-- Correct: cascade variable --}}
{{ $footer->copyright }}

{{-- Correct: inline expression --}}
@foreach (unwrap($usps ?? []) as $item) ... @endforeach
```

Before adding a View Composer, verify the data isn't already in the view. `@dump(array_keys($__data))` in the template (temporary, remove afterwards) shows what the cascade provides.

### When `@php` Seems Necessary

Only use `@php` if there is genuinely no reasonable alternative. In that case, **stop and ask the user for approval first**, explaining why options 1–4 don't work. Don't add `@php` blocks on your own initiative, and don't treat existing `@php` usage as precedent. That includes `@php` in Statamic's own docs examples (`use function Statamic\View\Blade\{value}`, `@php($tag = Statamic::tag(...)->fetch())` for pagination).

## Eloquent Driver

This project uses the Statamic Eloquent Driver with a **hybrid storage strategy**. Check `config/statamic/eloquent-driver.php` to see which resources use `eloquent` (database) vs `file` (disk).

- **Entries, terms, globals variables, nav trees, revisions** → stored in database
- **Blueprints, fieldsets, collections, forms** → stored as YAML files on disk
- All Eloquent tables use a configurable prefix (see `STATAMIC_ELOQUENT_PREFIX` env var)

**Always query content via Statamic facades** — they abstract the storage driver.

### Driver Commands

```bash
php please install:eloquent-driver     # Install driver, publish config, choose repositories to move to DB
php please eloquent:import-entries     # Import flat-file content into DB (one command per repository)
php please eloquent:export-entries     # Move a repository back to flat files
```

Export commands exist per repository: `export-addon-settings`, `export-assets`, `export-blueprints` (incl. fieldsets), `export-collections`, `export-entries`, `export-forms`, `export-globals`, `export-navs`, `export-revisions`, `export-taxonomies`, `export-sites`. Changing a repository's driver is a data migration — confirm with the user and back up the database first.

## Querying Content

In templates, first use cascade variables and tags (see "Order of Preference" above). In PHP code (composers, controllers, commands, listeners), use Statamic facades and never raw DB queries:

```php
use Statamic\Facades\Entry;
use Statamic\Facades\GlobalSet;
use Statamic\Facades\Collection;
use Statamic\Facades\Term;

// Query entries from a collection
$entries = Entry::query()
    ->where('collection', 'pages')
    ->where('status', 'published')
    ->get();

// Find a single entry
$entry = Entry::find($id);

// Get a global set's values
$globalData = GlobalSet::findByHandle('footer')?->inCurrentSite()?->values();

// Collection tree (hierarchical content)
$tree = Collection::findByHandle('knowledge_base')?->structure()?->in('default');
$page = $tree->find($entryId);
$parent = $page?->parent();
```

## Field Access in Blade Templates (Critical)

Statamic injects entry fields into Blade templates. This is the **primary way data reaches templates** and how you should access it. There are two patterns:

### 1. Top-Level Variables (Most Common)

Statamic cascades all entry fields as individual Blade variables. Blueprint field handles become variable names directly:

```blade
{{-- Blueprint fields are available as top-level $variables --}}
{{ $title }}
{{ $hero_title ?? '' }}
{{ $hero_subtitle ?? '' }}

{{-- Always null-coalesce optional fields --}}
@if ($hero_intro ?? false)
    {!! $hero_intro !!}
@endif

{{-- Replicator/Bard fields are iterable --}}
@foreach ($page_content as $set)
    @include('partials.sets.' . $set->type, [...$set])
@endforeach

{{-- Pre-unwrapped blocks (from View Composers) --}}
@foreach ($blocks ?? [] as $block)
    @include("partials.sets.{$block['type']}", $block)
@endforeach
```

### 2. Via the `$page` Entry Object

The current entry is also available as `$page`. Use this for accessing the entry directly:

```blade
{{-- Dynamic property access on the entry --}}
<h1>{{ $page->title }}</h1>
<h2>{{ $page->subtitle }}</h2>

{{-- Conditionals on entry fields --}}
@if ($page->hero_title && $page->show_hero)
    @include('partials.hero')
@endif

{{-- URL and metadata --}}
<a href="{{ $page->url() }}">{{ $page->title }}</a>
```

### Which to Use

- **Top-level `$variables`** — Use for fields that may need null-coalescing or when passing to components. This is what Statamic injects by default.
- **`$page->field`** — Use when you need to access the entry object directly, check multiple fields, or when the field name might conflict with a Blade variable.
- **`$entry->get('field')`** — Use in PHP code (View Composers, helpers) to avoid PHPStan ignores.

### Important: Fields Are Value Objects

All field variables in templates are `Statamic\Fields\Value` objects, not raw PHP values. Blade's `{{ }}` syntax calls `__toString()` automatically for simple fields, but for complex fields (arrays, relationships) you must handle them explicitly:

```blade
{{-- Simple fields: Value auto-casts to string in {{ }} --}}
{{ $title }}

{{-- Complex fields: use unwrap() inline (never in a @php block) --}}
@foreach (unwrap($usps ?? []) as $item)
    <li>{{ $item['text'] }}</li>
@endforeach
```

## Value Object Handling in PHP

In PHP code (View Composers, helpers), Statamic wraps field data in `Statamic\Fields\Value` and `Statamic\Fields\Values` objects. These must be unwrapped explicitly.

### Checking and Unwrapping Values

```php
use Statamic\Fields\Value;

// Check if a field is a Value with actual content
if ($field instanceof Value && $field->value()) {
    $raw = $field->value();  // Get the underlying data
}

// For relationship fields (entries fieldtype)
if ($entries instanceof Value && $entries->value()) {
    foreach ($entries as $entry) {
        // Each $entry is a Statamic Entry object
    }
}
```

### Field Access in PHP Code

```php
// Preferred in PHP — no PHPStan ignores needed
$value = $entry->get('field_name');

// Dynamic property access — works but needs @phpstan-ignore
$title = $entry->title;  // @phpstan-ignore property.notFound
```

## Blueprints

Blueprints are YAML files in `resources/blueprints/`. They define the field structure for collections, globals, navigations, and asset containers.

### Common Fieldtypes

| Fieldtype | Description |
|-----------|-------------|
| `text` | Single-line text |
| `textarea` | Multi-line text |
| `bard` | Rich text editor with sets |
| `replicator` | Repeatable content blocks |
| `entries` | Relationship to other entries |
| `assets` | File/image upload |
| `toggle` | Boolean |
| `select` | Dropdown |
| `color` | Color picker |
| `link` | URL or entry reference |
| `icon` | Icon picker |
| `taxonomy` | Taxonomy term relationship |
| `grid` | Table-like repeating rows |

### Blueprint Conventions

- Use `import` for shared fieldsets (e.g., `hero`, `seo`)
- Group fields into tabs: `main`, `seo`, `sidebar`
- Use `required: true` on essential fields
- Handles use `snake_case`
- Never use a reserved word as a field handle (see below)

### Reserved Field Handles

These collide with Statamic tags, contextual variables, or CP Vue components — they *may* appear to work but can silently override data. Also avoid names of Statamic Tags and contextual variables.

- **General:** `blueprint`, `content_type`, `count`, `elseif`, `endif`, `endunless`, `id`, `if`, `length`, `path`, `private`, `publish`, `published`, `reference`, `resource`, `route`, `save`, `site`, `status`, `unless`, `uri`, `url`
- **Entries:** `order`, `origin`, `parent`
- **Forms:** `date`, `message`, `messages`
- **Taxonomy handles:** `register`
- **Route wildcards:** `entry`, `taxonomy`
- **Special cases:** `type` inside a Replicator/Bard set (it's the set handle); `global` when a global field with the same name exists; `content` when also using a Markdown file's content area

Prefix instead: `hero_title`, `item_count`, `page_status`, `link_url`.

```yaml
tabs:
  main:
    sections:
      - fields:
          - handle: title
            field:
              type: text
              required: true
          - import: page_blocks
  seo:
    sections:
      - import: seo
```

## Content Blocks (Replicator Pattern)

Content blocks are Replicator sets rendered as Blade partials. Each block type has a corresponding partial:

```blade
@foreach ($page_content as $set)
    @include('partials.sets.' . $set->type, [...$set])
@endforeach
```

When creating a new block:
1. Add the set definition to the Replicator field in the blueprint
2. Create the Blade partial (e.g., `block_<name>.blade.php`)

## View Composers

Use View Composers to enrich Statamic template data with additional queries or transformations, **only when** the cascade (entry fields, globals, system variables) and Statamic tags/modifiers can't provide it. Never use a composer to re-expose data the cascade already injects.

```php
use Illuminate\Contracts\View\View;
use Illuminate\View\Composer as ViewComposer;

class PageComposer implements ViewComposer
{
    public function compose(View $view): void
    {
        $data = $view->getData();

        // Transform or add data
        $view->with('extraData', $this->fetchRelated($data));
    }
}
```

Register in a ServiceProvider:

```php
View::composer('pages.show', PageComposer::class);
```

## Asset Handling

Statamic asset fields can return different types. Always handle defensively:

```php
// Asset field might be a Value, an Asset object, an array, or a string path
$asset = $entry->get('logo');

// Safe resolution patterns:
if ($asset instanceof Value) {
    $asset = $asset->value();
}

// If it's an array of assets, take the first
if (is_array($asset) && isset($asset[0])) {
    $asset = $asset[0];
}

// Get the URL
$url = $asset?->url();
```

## Navigation

Navigations are hierarchical structures for building site menus. They support entry references, hardcoded URLs, and text-only nodes — all managed via drag-and-drop in the CP.

### Storage

- **Nav definitions** → `content/navigation/<handle>.yaml` (file driver)
- **Tree data** → stored via Eloquent (`navigation_trees` driver)
- **Blueprints** → `resources/blueprints/navigation/<handle>.yaml` (file driver)

### Rendering Navs in Blade (Preferred)

**Always prefer the `<statamic:nav:handle>` Blade tag** (short alias: `<s:nav:handle>`; bare `<s:nav>` defaults to the `pages` collection structure). It handles entry resolution, URL generation, and variable scoping automatically — no manual `Nav::find()` or `resolveNavEntry()` needed for standard menus.

```blade
<ul>
    <statamic:nav:footer_main>
        <li>
            <a href="{{ $url }}" @if ($is_current || $is_parent) class="active" @endif>
                {{ $title }}
            </a>
        </li>
    </statamic:nav:footer_main>
</ul>
```

#### Available Variables Inside the Tag

| Variable | Type | Description |
|----------|------|-------------|
| `$url` | string | Resolved URL (entry URL or hardcoded) |
| `$title` | string | Item title (nav override or entry title) |
| `$is_current` | bool | Whether this is the exact current page |
| `$is_parent` | bool | Whether this is a parent of the current page |
| `$is_entry` | bool | Whether the item references an entry |
| `$is_external` | bool | Whether the URL is external |
| `$children` | array | Child nav items |
| `$depth` | int | Current nesting level |
| Custom fields | mixed | Any fields from the nav blueprint |

#### Nested Children with `@recursive_children`

For multi-level navs, use `@recursive_children` to repeat the tag contents for child items:

```blade
<ul>
    <statamic:nav:main_navigation>
        <li>
            <a href="{{ $url }}">{{ $title }}</a>
            @if (count($children))
                <ul>
                    @recursive_children
                </ul>
            @endif
        </li>
    </statamic:nav:main_navigation>
</ul>
```

Branch on `$depth` for different markup per level (e.g. footer columns), and use `@class` for per-level styling — no `@php` needed:

```blade
<div class="flex">
    <s:nav:footer>
        @if ($depth == 1)
            <div>
                <h3>{{ $title }}</h3>
                @if (count($children) > 0)
                    <ul>@recursive_children</ul>
                @endif
            </div>
        @elseif ($depth == 2)
            <li @class(['font-bold' => $is_current])><a href="{{ $url }}">{{ $title }}</a></li>
        @endif
    </s:nav:footer>
</div>
```

#### Tag Parameters

Control output with parameters on the tag:

```blade
<statamic:nav:main_navigation
    max_depth="2"
    include_home="true"
    select="title|url"
>
    ...
</statamic:nav:main_navigation>
```

- `max_depth` — Limit nesting depth (performance optimization)
- `include_home` — Include the home page in the tree
- `show_unpublished` — Include unpublished entries
- `select` — Limit fields retrieved (pipe-separated, improves performance)

### When to Use `Nav::find()` Instead

The `<statamic:nav>` tag covers most cases. Only fall back to the PHP facade when you need to **pre-process nav data before rendering** — e.g., serializing to JSON for Alpine.js, pre-rendering icons server-side, or building complex dropdown structures:

```php
use Statamic\Facades\Nav;

$tree = Nav::find('main_navigation')?->in('default')?->tree() ?? [];
```

Each item in the tree is an array with keys: `title`, `url`, `entry` (UUID if linked), `children`, plus any custom blueprint fields. When using this approach, resolve items with the project's `resolveNavEntry()` helper:

```php
$resolved = resolveNavEntry($item);
// Returns: ['url' => '/some-page', 'title' => 'Page Title']
```

The helper checks `entry`, `reference`, `data.reference`, and `data.entry` for an entry UUID, resolves it via `Entry::find()`, and falls back to the item's `url` field.

**Note:** Custom blueprint fields may appear at the top level or under `data.*`. Defensively check both:

```php
$icon = $item['icon'] ?? ($item['data']['icon'] ?? null);
```

### Nav Config Options

In the nav definition YAML (`content/navigation/<handle>.yaml`):

```yaml
title: 'Main Navigation'
collections:          # Which collections appear in the CP entry selector
  - pages
  - integrations
max_depth: 2          # Maximum nesting depth
```

In multi-site, the `tree` is localized per site; a nav only exists in sites that have a tree for it. With the Eloquent `navigation_trees` driver, each site's tree is a separate DB row — read via `Nav::find('handle')->in($site)`.

### Best Practices

- **Use `<statamic:nav>` by default** — It resolves entries, builds URLs, and provides `$is_current`/`$is_parent` automatically. Only use `Nav::find()` when you need PHP pre-processing.
- **Restrict collections** — Configure `collections` in the nav YAML to limit which entries editors can link to.
- **Set `max_depth`** — Match the frontend's rendering capability (e.g., `2` for a mega nav with one level of children).
- **Filter empty items** — After manual resolution, filter out items where title is empty (entry may have been deleted).
- **Pre-process for Alpine/JS** — When passing nav data to JavaScript, resolve entries and render server-side components (like icons) in PHP before encoding to JSON.

## Routing

Statamic handles routing via collection URLs defined in each collection's YAML config. Custom middleware can control which requests Statamic handles.

Key config: `config/statamic/routes.php`

### Trailing Slashes

To generate URLs with trailing slashes (e.g. keeping SEO URLs after a migration), call `URL::enforceTrailingSlashes()` (`Statamic\Facades\URL`) in `AppServiceProvider::boot()`. This only changes generated URLs; redirecting non-slash requests is a web-server rule (308), not app code.

### Custom 404/403/401 Handling

Statamic's `NotFoundHttpException`, `ForbiddenHttpException`, and `UnauthorizedHttpException` (in `Statamic\Exceptions`) render themselves, so Laravel's `$exceptions->render()` never fires for them. Use `renderUsing()` in a service provider's `boot()`. `$this` is the exception; return a response to take over, or `null` to fall back. Bail early for CP/API routes:

```php
use Illuminate\Http\Request;
use Statamic\Exceptions\NotFoundHttpException;
use Statamic\Statamic;

NotFoundHttpException::renderUsing(function (Request $request) {
    if (Statamic::isCpRoute() || Statamic::isApiRoute()) {
        return null;
    }

    // e.g. redirect legacy URLs, fall back to search, log the 404...
});
```

## Multi-Site & Localization

- Convert single → multi-site with `php please multisite`. Sites live in `resources/sites.yaml` (or CP `/cp/sites`); default handle is `default`.
- Collections, globals, and navs each declare which `sites` they're available in. Without it the CP hides the site selector.
- An entry only exists in a site once it's **explicitly localized**. `propagate: true` on the collection auto-creates localizations in all sites on first save.
- Localized entries/global sets have an `origin` and inherit any value they don't define. Every localization has its own ID — don't assume one ID across sites.
- Mark fields localizable in the blueprint (`localizable: true`); non-localizable fields are read-only on localizations.
- In code, always scope to a site: `GlobalSet::findByHandle('x')->in($site)` / `->inCurrentSite()`, `Entry::query()->where('site', $site)`, `$entry->in($site)` for a specific localization.

## Dates & Timezones (Statamic 6+)

- Keep `config/app.php` timezone at `UTC`; dates are stored in UTC. Set the display timezone via `display_timezone` in `config/statamic/system.php`.
- In templates, dates are UTC Carbon instances. Casting to string converts to the display timezone, but modifiers/tags receive UTC. Either set `'localize_dates_in_modifiers' => true` in `config/statamic/system.php`, or convert explicitly:

```blade
{{ $date->tz(config('statamic.system.display_timezone'))->format('j F Y') }}
```

- Custom routes passing Carbon instances to views must pass UTC and use the `Localize` middleware (`Route::statamic()` and the `statamic.web` group include it already).
- Migrating an existing non-UTC site: `php please migrate-dates-to-utc <Old/Timezone>` — destructive data migration; back up and confirm with the user first. With the Eloquent driver it must run against each environment's database after deploy.

## Creating Content Programmatically

Use the repository/facade, never insert rows into the Eloquent tables directly (bypasses events, URI/structure updates, search indexing):

```php
use Illuminate\Support\Str;
use Statamic\Facades\Entry;

Entry::make()
    ->collection('team')
    ->locale('default')
    ->slug(Str::slug($name))
    ->data(['title' => $name, 'role' => $role])
    ->save();
```

For one-off imports wrap this in an Artisan command; for CSV/XML imports via the CP, the first-party `statamic/importer` addon exists. Same pattern for `Term::make()`, `User::make()`, etc.

## Security

- Blade `{{ }}` escapes output; `{!! !!}` does not. Only use `{!! !!}` for trusted, augmented HTML (Bard, Markdown). For user-supplied content use `{!! Statamic::modify($value)->sanitize() !!}` or plain `{{ }}`.
- Escaping doesn't stop `javascript:` URLs in user-provided link fields. Validate URL schemes and/or add a Content Security Policy.
- Apply CSP middleware via the `web` group only (`$middleware->web(append: [...])`). The Control Panel needs inline JS and must not receive the frontend CSP.
- Maintenance mode: keep the CP reachable with `$middleware->preventRequestsDuringMaintenance(except: ['/cp*'])` in `bootstrap/app.php`.
- GDPR: prefer storing users in the database (not git-tracked YAML) and consider disabling form submission storage when email-only is enough.

## Performance & Deployment

- **Stache watcher:** `STATAMIC_STACHE_WATCHER=auto` (on locally, off in production). With the Eloquent driver, DB-stored content isn't watched anyway.
- **Asset caches:** give `asset_meta` and `asset_container_contents` their own cache stores in `config/cache.php` (Redis on multi-server setups) so `cache:clear` doesn't wipe them. Clear with `php please assets:clear-cache`.
- **Asset meta / presets:** warm with `php please assets:meta` and `php please assets:generate-presets` (deploy step or queue); set `generate_presets_on_upload => false` for big libraries. Large libraries → move assets to the Eloquent driver.
- **Existence checks:** use `$asset->exists()` (cached), not `$asset->disk()->exists(...)` / `File::exists()` (hits S3 every time).
- **Zero-downtime deploys** (Forge/Envoyer/Ploi/Deployer): never share the Stache cache between releases — it stores absolute paths. Give the Stache its own store (`'cache_store' => 'stache'` in `config/statamic/stache.php`) pointing at a per-release path or a Redis prefix like `basename(base_path()).'_stache_'`.
- **CP default listing columns:** define team-wide defaults in `resources/preferences.yaml` (same shape as a user's `preferences` key, e.g. `collections.articles.columns`).

## Artisan / Please Commands

Use `php please` for Statamic-specific operations:

```bash
php please make:collection      # Create a new collection
php please make:fieldset        # Create a reusable fieldset
php please make:nav             # Create a navigation
php please make:widget          # Create a CP dashboard widget
php please stache:refresh       # Rebuild the content cache
php please static:clear         # Clear static cache
php please support:details      # Debug Statamic setup info
php please multisite            # Convert single-site to multi-site
php please assets:meta          # Pre-generate asset metadata
php please assets:generate-presets  # Pre-generate Glide presets
php please make:user            # Create a CP user (preferred over hand-written user files)
```

## PHPStan Considerations

Statamic's dynamic properties trigger PHPStan errors. Common patterns:

```php
$entry->title           // @phpstan-ignore property.notFound
$entry->url()           // @phpstan-ignore method.notFound
```

Prefer `$entry->get('field_name')` in PHP code to avoid ignores. Dynamic property access is fine in Blade templates where PHPStan doesn't analyze.

## Common Pitfalls

1. **Forgetting to unwrap Values** — Statamic field data is wrapped in Value objects. Always unwrap before using in conditions or passing to components.
2. **Asset fields returning arrays** — An `assets` field may return an array even for `max_files: 1`. Handle both single and array returns.
3. **Link fields with `entry::` prefix** — Link fieldtype stores entry references as `entry::<uuid>`. Resolve via `Entry::find()`.
4. **Global sets need site context** — Always chain `->inCurrentSite()` when reading global set values for multi-site support.
5. **Tree/structure queries** — Use `Collection::findByHandle()->structure()->in('default')` for hierarchical collections, not flat `Entry::query()`.
6. **Bard content is augmented** — Bard fields return augmented HTML when accessed as Values. Use `->value()` for raw ProseMirror JSON.
7. **Component name conflicts** — When using Flux UI alongside Statamic Blade components, watch for name collisions (e.g., `header`, `footer`). Register explicit Blade aliases if needed.
8. **Reserved field handles** — Handles like `status`, `url`, `id`, `site`, `date` (forms), `parent` (entries), or `type` (in sets) silently break things. Check the reserved list before adding fields.
9. **Laravel exception handlers ignoring 404s** — Statamic's HTTP exceptions render themselves; use `NotFoundHttpException::renderUsing()`.
10. **Timezone shifts in formatted dates** — Modifiers/`->format()` get UTC Carbon; convert to `display_timezone` first.
11. **Logic creeping into templates** — No `@php` blocks. Use cascade variables and Statamic tags first, then a View Composer, component, or helper. Ask the user before adding `@php` if truly unavoidable.
