# System Architecture & Data Flow

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    XDMovies Admin Panel                      │
│                   (movies.php - Web Interface)               │
└────┬──────────────────────────────────────┬─────────────────┘
     │                                      │
     ▼                                      ▼
┌──────────────────────┐          ┌────────────────────┐
│  Movie Management    │          │  Cache Purge Modal │
│  - Add              │          │  - Single URL       │
│  - Edit             │          │  - Multiple URLs    │
│  - Delete           │          │  - Purge All        │
│  - Publish/Draft    │          └────────┬────────────┘
└────┬─────────────────┘                  │
     │                                    │
     ▼                                    ▼
┌──────────────────────────┐    ┌──────────────────────┐
│ Movie Page Generator     │    │ Cache Purge API      │
│ (movie_page_helper.php)  │    │ (cache_purge_api.php)│
│                          │    │                      │
│ - Generate HTML pages    │    │ - Validate input    │
│ - Update pages           │    │ - Call Cloudflare   │
│ - Delete pages           │    │ - Return JSON       │
└────┬─────────────────────┘    └──────┬───────────────┘
     │                                 │
     ▼                                 ▼
┌──────────────────────┐    ┌────────────────────────┐
│  /movies/ Folder     │    │ Cloudflare Cache API   │
│  HTML Files          │    │ (cloudflare_cache.php) │
│  - Published pages   │    │                        │
│  - Searchable        │    │ - purgeUrl()          │
│  - SEO optimized     │    │ - purgeUrls()         │
└──────────────────────┘    │ - purgeEverything()   │
                            │ - purgeTags()         │
                            └──────┬──────────────────┘
                                   │
                                   ▼
                            ┌────────────────────┐
                            │  Cloudflare API    │
                            │  (Internet)        │
                            │                    │
                            │ Zone: xdmovies...  │
                            │ Auth: Bearer Token  │
                            └────────────────────┘
```

---

## Data Flow: Publishing a Movie

```
Admin clicks Publish Button
    │
    ▼
Movies.php receives toggle_status action
    │
    ├─ Get movie data (title, tmdb_id, status)
    │
    ├─ Update database (status → published)
    │
    ▼
Movie Page Generator
    │
    ├─ Fetch movie details from database
    │  ├─ From movies table: title, audio, source, note
    │  ├─ From movie_downloads: description, genres, rating
    │  └─ From movie_versions: download links
    │
    ├─ Generate HTML content
    │  ├─ Add metadata (title, description, images)
    │  ├─ Add download sections (grouped by source)
    │  └─ Add SEO tags (og:image, canonical, etc.)
    │
    ├─ Create filename: slug-tmdbid.html
    │
    └─ Write to /movies/ folder
            │
            ▼ (if auto-purge enabled)
    
    Cloudflare Cache Purger
    │
    ├─ Generate movie URL: https://site.com/movies/slug-tmdbid.html
    │
    ├─ Call cloudflare_cache.php
    │  │
    │  └─ Make API request to Cloudflare
    │     ├─ Headers: Authorization: Bearer TOKEN
    │     ├─ Endpoint: /zones/{zone_id}/purge_cache
    │     └─ Body: {"files": ["url"]}
    │
    ├─ Purge related pages (/,/search.html)
    │
    └─ Log result
            │
            ▼
    
    Admin sees success message
    ✓ Movie published
    ✓ Page generated
    ✓ Cache purged
```

---

## Data Flow: Manual Cache Purge

```
Admin clicks "Purge Cloudflare Cache" button
    │
    ▼
Cache Purge Modal opens with 3 tabs
    │
    ├─ Single URL tab: Admin enters one URL
    ├─ Multiple URLs tab: Admin enters multiple URLs (one per line)
    └─ Purge All tab: Admin confirms with checkbox
        │
        ▼
    Admin clicks "Purge Cache" button
        │
        ▼
    JavaScript handler
        │
        ├─ Validate input
        │  ├─ Check URL format (must be valid URL)
        │  └─ Check confirmation (for purge all)
        │
        ├─ Show loading spinner
        │
        ├─ Send POST request to cache_purge_api.php
        │  ├─ Include action (purge_url/purge_urls/purge_everything)
        │  └─ Include data (URLs or confirmation)
        │
        ▼
    cache_purge_api.php
        │
        ├─ Verify admin authentication
        │
        ├─ Load cloudflare_cache.php
        │
        ├─ Get Zone ID and API Token
        │
        ├─ Call appropriate CloudflareCache method
        │  ├─ purgeUrl($url) for single
        │  ├─ purgeUrls($urls) for multiple
        │  └─ purgeEverything() for all
        │
        ├─ Make HTTPS request to Cloudflare API
        │
        └─ Return JSON response
            │
            ▼
    JavaScript handler receives response
        │
        ├─ Hide loading spinner
        │
        ├─ Display result
        │  ├─ Success: Green alert with message
        │  └─ Error: Red alert with error details
        │
        └─ Clear form inputs (on success)
            │
            ▼
    Admin sees confirmation message
    ✓ Cache purged successfully
    OR
    ✗ Error message with details
```

---

## Database Schema Integration

```
┌─────────────────────┐
│     movies          │
├─────────────────────┤
│ id (PK)             │
│ tmdb_id             │ ◄── Used to generate URL
│ title               │ ◄── Used in page title & slug
│ status              │ ◄── draft/published
│ audio_languages     │ ◄── Shown in HTML page
│ source              │ ◄── Shown in HTML page
│ note                │ ◄── Admin note in page
│ created_at          │
│ updated_at          │
│ last_updated        │
└─────────────────────┘
         │
         │ FK: tmdb_id
         ▼
┌──────────────────────┐
│   movie_downloads    │
├──────────────────────┤
│ id (PK)              │
│ tmdb_id (FK)         │
│ overview             │ ◄── In HTML page description
│ poster_path          │ ◄── In HTML <img> tags
│ backdrop_path        │ ◄── Page background
│ release_date         │ ◄── In HTML page
│ genres               │ ◄── In HTML page
│ rating               │ ◄── In HTML page
│ cast                 │ ◄── In HTML page
└──────────────────────┘
         │
         │ FK: movie_id
         ▼
┌──────────────────────┐
│  movie_versions      │
├──────────────────────┤
│ id (PK)              │
│ movie_id (FK)        │
│ resolution           │ ◄── 720p, 1080p, 2160p
│ custom_title         │ ◄── Download button text
│ size                 │ ◄── Download link
│ download_link        │ ◄── In HTML buttons
│ source               │ ◄── Groups downloads
└──────────────────────┘
```

---

## File Dependency Tree

```
php/movies.php (Main Admin Panel)
    ├─ Requires: movie_page_helper.php
    │  └─ Functions:
    │     ├─ movie_generate_page()
    │     ├─ movie_delete_page()
    │     ├─ movie_update_page()
    │     └─ Helper functions (slugify, ensure_dir, etc.)
    │
    ├─ Requires: cloudflare_cache.php
    │  └─ Class: CloudflareCache
    │     ├─ purgeUrl()
    │     ├─ purgeUrls()
    │     ├─ purgeEverything()
    │     └─ Helper functions
    │
    ├─ Requires: tmdb_api.php (existing)
    │
    ├─ Requires: db.php (creates $pdo)
    │
    └─ Handles Actions:
       ├─ add_movie
       ├─ update_movie
       ├─ delete_movie
       ├─ toggle_status ◄── Integrates both features
       ├─ add_version
       ├─ update_version
       └─ delete_version

cache_purge_api.php (AJAX Endpoint)
    ├─ Requires: cloudflare_cache.php
    │
    └─ Handles Actions:
       ├─ purge_url
       ├─ purge_urls
       ├─ purge_everything
       ├─ purge_tags
       └─ test_connection

movie_page_helper.php (Page Generation)
    ├─ Uses: PDO connection ($pdo from db.php)
    │
    └─ Generates:
       └─ /movies/{slug}-{tmdb_id}.html files

cloudflare_cache.php (Cache Management)
    ├─ Uses: cURL for HTTPS requests
    │
    └─ Communicates:
       └─ With Cloudflare API v4
```

---

## Request/Response Flow - Cache Purge

```
┌─ Browser (JavaScript) ─┐
│ fetch('cache_purge_api.php')
│ FormData:
│   - action: 'purge_url'
│   - url: 'https://...'
└─────┬──────────────────┘
      │ POST
      ▼
┌─ cache_purge_api.php ─┐
│ 1. Check authentication
│ 2. Validate POST data
│ 3. Create CloudflareCache object
│ 4. Call appropriate method
│ 5. Return JSON response
└─────┬──────────────────┘
      │ JSON: {success: true/false, message: '...'}
      ▼
┌─ CloudflareCache ─┐
│ 1. Validate URL
│ 2. Build request
│ 3. Call makeRequest()
│ 4. Return result array
└─────┬──────────────┘
      │
      ▼
┌─ cURL ─┐
│ POST https://api.cloudflare.com/client/v4/zones/.../purge_cache
│ Headers:
│   Authorization: Bearer TOKEN
│   Content-Type: application/json
│ Body: {"files": ["https://..."]}
└─────┬─────┘
      │ HTTPS
      ▼
┌─ Cloudflare API ─┐
│ 1. Authenticate
│ 2. Validate zone
│ 3. Purge cache
│ 4. Return status
└─────┬────────────┘
      │ JSON response
      ▼
┌─ Browser ─┐
│ Display result to admin
│ - Green alert: Success
│ - Red alert: Error
└────────────┘
```

---

## Status Workflow

```
Movie Lifecycle:

┌─────────┐
│  Draft  │ ◄─── Default when added
└────┬────┘
     │
     │ Admin clicks Publish
     │ ├─ movie_generate_page() ◄── Creates HTML
     │ └─ auto_purge_movie_cache() ◄── Purges Cloudflare
     ▼
┌──────────┐
│Published │ ◄─── Accessible to public
└────┬─────┘
     │
     │ Admin updates movie
     │ ├─ Fetches fresh TMDb data
     │ ├─ movie_update_page() ◄── Updates HTML
     │ └─ auto_purge_movie_cache() ◄── Purges Cloudflare
     │
     ▼ (same state)
┌──────────┐
│Published │ ◄─── Updated & cached
└────┬─────┘
     │
     │ Admin deletes movie
     │ ├─ movie_delete_page() ◄── Deletes HTML
     │ ├─ auto_purge_movie_cache() ◄── Purges cache
     │ └─ Database record deleted
     ▼
┌─────────────┐
│   DELETED   │ ◄─── No longer accessible
└─────────────┘

OR

┌──────────┐
│Published │
└────┬─────┘
     │
     │ Admin clicks Revert to Draft
     │ ├─ movie_delete_page() ◄── Deletes HTML
     │ ├─ auto_purge_movie_cache() ◄── Purges cache
     │ └─ Status = draft
     ▼
┌─────────┐
│  Draft  │ ◄─── No longer public
└─────────┘
```

---

## Configuration Chain

```
1. Admin puts credentials in movies.php:
   $cloudflare_zone_id = 'abc123'
   $cloudflare_api_token = 'token_xyz'
   $enable_cloudflare_auto_purge = true

   │
   ▼

2. When auto-purge needed:
   - movies.php checks: if ($enable_cloudflare_auto_purge && credentials set)
   
   │
   ▼

3. Calls helper function:
   auto_purge_movie_cache($title, $tmdbId, $zoneId, $token)
   
   │
   ▼

4. Creates CloudflareCache object:
   $cf = new CloudflareCache($zoneId, $token)
   
   │
   ▼

5. Calls purge method:
   $cf->purgeUrl('https://...')
   
   │
   ▼

6. Makes API request to Cloudflare
```

---

## Error Handling Flow

```
Error Occurs
    │
    ├─ Validation Error
    │  └─ Return to user with message (no API call)
    │
    ├─ API Error
    │  ├─ Catch exception
    │  ├─ Log to error_log
    │  └─ Return error JSON to user
    │
    ├─ Network Error
    │  ├─ cURL error caught
    │  ├─ Log to error_log
    │  └─ Return error message
    │
    └─ Database Error
       ├─ Query fails
       ├─ Log to error_log
       └─ Return error to user
```

---

## Key Relationships

```
Admin Action → Database Update → Page Generation → Cache Purge → User Sees Result

Add Movie:
  POST form → INSERT query → Page not generated (draft) → No purge → Not public

Publish:
  Click publish → UPDATE status → Generate page → Purge cache → Public & fresh

Update:
  POST form → UPDATE query → Update page → Purge cache → Updated content

Delete:
  Click delete → DELETE query → Delete page → Purge cache → No longer public
```

---

## Integration Points

```
movies.php
    │
    ├─ add_movie action
    │  └─ No page generation (draft status)
    │
    ├─ update_movie action
    │  ├─ movie_update_page() if published
    │  └─ auto_purge_movie_cache() if published
    │
    ├─ delete_movie action
    │  ├─ movie_delete_page()
    │  └─ auto_purge_movie_cache()
    │
    ├─ toggle_status action ◄─── MAIN INTEGRATION POINT
    │  ├─ If → published: Generate page + purge cache
    │  └─ If → draft: Delete page + purge cache
    │
    ├─ Manual cache purge button
    │  └─ Opens modal → cache_purge_api.php
    │
    └─ UI Elements
       ├─ "Purge Cloudflare Cache" button
       ├─ Cache purge modal
       └─ Status success/error alerts
```

---

This architecture ensures:
✓ Separation of concerns
✓ Easy to maintain
✓ Easy to disable features
✓ Proper error handling
✓ Secure API access
✓ Scalable design
