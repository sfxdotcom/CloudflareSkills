---
name: wordpress-on-cloudflare
description: Optimise WordPress sites and plugins on Cloudflare. Covers Cloudflare R2 for the uploads/media library, Cloudflare Image Resizing (/cdn-cgi/image/) for on-the-fly thumbnails and format conversion, Cache Rules for HTML and static assets, APO/page rules for full-page caching, and Workers for edge-side personalisation. Use when asked to make a WordPress site faster on Cloudflare, configure R2 offloading, set up cdn-cgi/image transforms, design cache rules for wp-admin / logged-in users, or audit a WP plugin's Cloudflare integration. Biases towards retrieval from current Cloudflare and WordPress docs.
---

# WordPress on Cloudflare

End-to-end patterns for running a fast, cheap WordPress site behind Cloudflare. This skill ties together the lower-level skills (`cloudflare/`, `web-perf/`, `workers-best-practices/`) with WordPress-specific knobs (`wp-config.php`, plugin hooks, `wp_get_attachment_*`).

Your knowledge of Cloudflare APIs, prices, and limits may be outdated. **Prefer retrieval over pre-training** when citing numbers or feature flags.

## Retrieval Sources

| Source | How to retrieve | Use for |
|--------|----------------|---------|
| Cloudflare docs | `https://developers.cloudflare.com/` | Limits, pricing, Image Resizing options, cache directives |
| Cloudflare changelog | `https://developers.cloudflare.com/changelog/` | Recent feature additions, deprecations |
| WordPress code reference | `https://developer.wordpress.org/reference/` | Hook signatures, function behaviour |
| WP plugin handbook | `https://developer.wordpress.org/plugins/` | Settings APIs, REST endpoints, security |

## Decision tree

```
WordPress + Cloudflare goal?
├─ Faster image delivery       → "Image Resizing" + R2  (sections 1, 2)
├─ Offload /wp-content/uploads → R2 with S3-compatible SDK or plugin (section 2)
├─ Cache full HTML pages       → Cache Rules (or APO if WP.com hosted) (section 3)
├─ Bypass cache for logged-in  → Cache Rules + cookie match (section 3)
├─ Edge personalisation        → Workers between Cloudflare and WP origin (section 4)
├─ Faster admin REST API       → Argo + cache only safe GETs (section 5)
└─ Plugin authorship           → Use cdn-cgi/image URLs in srcset (section 6)
```

## 1. Cloudflare Image Resizing (URL-based)

URL pattern:

```
https://example.com/cdn-cgi/image/<options>/<source-url-or-path>
```

`<options>` is a comma-separated key=value list. Common keys:

| Key       | Values                                                | Notes |
|-----------|-------------------------------------------------------|-------|
| `width`   | int (px)                                              | Required for srcset |
| `height`  | int (px)                                              |       |
| `fit`     | `cover` `contain` `scale-down` `crop` `pad`           | Default `cover` |
| `format`  | `auto` `webp` `avif` `json`                           | `auto` returns the best for the requesting browser |
| `quality` | 1–100                                                 | 85 is a good default |
| `dpr`     | 1, 2, 3                                               | Device pixel ratio |
| `sharpen` | 0–10                                                  | Use sparingly |
| `metadata`| `keep` `copyright` `none`                             | `none` strips EXIF for privacy |
| `gravity` | `auto` `face` `left` `right` `top` `bottom` `<x>x<y>` | For crops |

Requirements: zone on Pro+ with "Image Resizing" enabled, **or** Images Transformations enabled. The source URL must be reachable (same zone for fastest path, or in the zone's "transformable" allowlist).

### Pattern: srcset over cdn-cgi/image

```html
<img
  src="/cdn-cgi/image/width=800,format=auto/wp-content/uploads/2026/05/sunset.jpg"
  srcset="
    /cdn-cgi/image/width=320,format=auto/wp-content/uploads/2026/05/sunset.jpg 320w,
    /cdn-cgi/image/width=640,format=auto/wp-content/uploads/2026/05/sunset.jpg 640w,
    /cdn-cgi/image/width=1280,format=auto/wp-content/uploads/2026/05/sunset.jpg 1280w"
  sizes="(max-width: 600px) 100vw, 50vw"
  width="1280" height="800"
  loading="lazy" decoding="async" alt="…">
```

For a WordPress plugin, hook `wp_get_attachment_image_src` / `wp_calculate_image_srcset` and rewrite the URLs before they're rendered:

```php
add_filter( 'wp_calculate_image_srcset', function ( $sources ) {
    foreach ( $sources as $w => &$source ) {
        $source['url'] = 'https://example.com/cdn-cgi/image/width=' . $w . ',format=auto,quality=85/' . ltrim( wp_make_link_relative( $source['url'] ), '/' );
    }
    return $sources;
} );
```

### Common mistakes

- Putting `cdn-cgi/image/...` on a URL that isn't on the same zone — Cloudflare won't fetch arbitrary off-zone sources unless allowlisted.
- Wrapping already-transformed URLs (double-wrapping). Always work from the canonical origin URL.
- Forgetting `format=auto`. Without it you'll serve JPEGs to AVIF-capable browsers.
- Caching the transformed URL with `Cache-Control: no-cache` on the origin object — Cloudflare needs to be allowed to cache.

## 2. R2 for /wp-content/uploads

R2 is S3-compatible and egress-free, which makes it the ideal store for WordPress media.

Configuration steps:

1. Create an R2 bucket and an API token with `Object Read & Write` on the bucket.
2. Set a custom domain on the bucket (e.g., `assets.example.com`) so URLs are clean.
3. Have the plugin (or a tool like `WP Offload Media`) upload to R2 on `wp_handle_upload`.
4. Filter `wp_get_attachment_url` so WP returns the R2 URL instead of `/wp-content/uploads/...`.
5. Optionally `unlink()` the local file after a successful upload (and keep the DB row).

Required headers on PUT:

| Header | Value |
|--------|-------|
| `Content-Type` | actual mime type (don't default to octet-stream) |
| `Cache-Control` | `public, max-age=31536000, immutable` for hashed filenames; `max-age=86400` for mutable names |

R2 **does not support ACLs**. Public visibility is bucket-level via the Cloudflare dashboard. Don't send `x-amz-acl`.

### Pairing R2 with Image Resizing

The cleanest setup is:

- Origin upload → R2 (raw file, no thumbnails generated)
- Page renders → `https://example.com/cdn-cgi/image/width=…/https://assets.example.com/path/to/raw.jpg`
- All thumbnail sizes are generated on demand by Cloudflare, cached at the edge.

This eliminates the need to bake N thumbnail variants on upload, which saves CPU on the WP origin and storage on R2.

### Gotchas

- Off-zone source URLs (R2 custom domain on a *different* zone than the WP origin) must be allowlisted under "Image Transformations → Sources" or Image Resizing will return 415.
- WordPress's `wp_get_attachment_metadata()` still expects local file paths — keep at least the metadata row even if the file is offloaded.

## 3. Cache Rules for WordPress

WordPress is *mostly* cacheable, except for: `wp-admin/*`, `wp-login.php`, REST API write methods, and any page where the visitor has a `wordpress_logged_in_*` cookie.

### Minimal Cache Rules (Cloudflare → Rules → Cache Rules)

| Order | Match | Action |
|-------|-------|--------|
| 1 | `URI Path starts with "/wp-admin"` OR `URI Path equals "/wp-login.php"` | **Bypass cache** |
| 2 | `Cookie contains "wordpress_logged_in"` OR `Cookie contains "wp-postpass"` OR `Cookie contains "woocommerce_items_in_cart"` | **Bypass cache** |
| 3 | `URI Path starts with "/wp-json/"` AND `Request Method ne "GET"` | **Bypass cache** |
| 4 | (catch-all) | **Eligible for cache**, `Edge TTL: 4h`, `Browser TTL: respect origin`, ignore query strings except `?p`, `?page_id`, `?s` |

Tier-2 caches (Tiered Cache / Argo) amplify hit ratios for long-tail content.

### Stale-while-revalidate

Set the origin to `Cache-Control: public, max-age=60, s-maxage=3600, stale-while-revalidate=86400` so Cloudflare can serve a stale copy while it refetches.

## 4. Workers in front of WordPress

Use a Worker when you need:

- A/B testing without round-tripping to origin
- Geo-aware redirects or pricing
- Edge-side cookie stamping (e.g., a `country` cookie for logged-out users)
- Stitching a static React shell with WP REST data

Pattern: route Worker handles `/api/*` and `/`; everything else passes through to origin via `fetch( request )`.

Avoid using a Worker to serve WP HTML directly unless you're caching it heavily — every request that traverses a Worker incurs CPU billing.

## 5. WP REST API + Argo

The WP REST API is `GET`-cacheable for public endpoints (`/wp-json/wp/v2/posts`, etc.). Add a Cache Rule that:

1. Matches `URI Path starts with "/wp-json/wp/v2/"` AND `Request Method eq "GET"`.
2. Sets `Edge TTL: 5m`.
3. Ignores all query strings except `per_page`, `page`, `_fields`.

Combine with Argo Smart Routing for read-heavy applications hitting a single origin.

## 6. For WP plugin authors

If you ship a WordPress plugin and want it to be Cloudflare-aware:

- **Provide a settings panel** for `enabled`, `zone_url`, `default_format`, `default_quality`, `default_fit`.
- **Build a helper class** that wraps source URLs with `/cdn-cgi/image/<opts>/<src>`. Always sort option keys for stable URLs (cache-key friendly).
- **Filter `wp_calculate_image_srcset`** so every `<img srcset>` in the theme — not just your plugin's output — benefits.
- **Don't hardcode widths**. Use the same widths WordPress uses (`thumbnail`, `medium`, `medium_large`, `large`) plus a few high-DPR ones (320, 640, 960, 1280, 1920).
- **Emit `loading="lazy"`** on every `<img>` except the first one above the fold; that one gets `fetchpriority="high"`.
- **Decide format on the client**. Use `format=auto` so Cloudflare picks AVIF/WebP/JPEG based on `Accept:`.
- **Strip EXIF** with `metadata=none` unless your users explicitly want it preserved (privacy & bytes).
- **Set `Cache-Control: public, max-age=31536000, immutable`** when uploading to R2 with content-addressable keys.

### Example plugin helper

```php
final class CloudflareImages {
    public static function transform( string $url, array $opts = [] ): string {
        $opts = array_merge(
            [ 'format' => 'auto', 'quality' => 85, 'fit' => 'cover' ],
            array_filter( $opts )
        );
        ksort( $opts );
        $segment = implode( ',', array_map(
            fn( $k, $v ) => $k . '=' . rawurlencode( (string) $v ),
            array_keys( $opts ),
            $opts
        ) );
        $zone = get_option( 'my_plugin_zone_url' );
        $path = ltrim( wp_make_link_relative( $url ), '/' );
        return rtrim( $zone, '/' ) . "/cdn-cgi/image/{$segment}/{$path}";
    }
}
```

## Quick Reference

| Goal | Lever |
|------|-------|
| Smaller images, no thumb-baking | `cdn-cgi/image/width=…,format=auto` |
| Cheap media storage | R2 + custom domain |
| No DB calls on warm pageloads | Cache Rules: cache HTML for 4h, bypass on login cookie |
| Personalise without origin round-trip | Worker on the route |
| Faster wp-admin AJAX | Argo + Cache Rules: bypass `wp-admin/*` |
| Logged-out comments still working | Bypass on `wordpress_logged_in_` AND on `wp-postpass_` AND on `comment_author_` cookies |
