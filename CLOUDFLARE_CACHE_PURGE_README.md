# Cloudflare Cache Purge Implementation

## Overview
This implementation adds Cloudflare cache purging capabilities to the XDMovies admin panel. It includes:
- Manual cache purge interface for admins
- Automatic cache purging when movies are published/updated/deleted
- Support for single URL, multiple URLs, tags, and full zone purge
- Real-time feedback with success/error alerts

## Files Created

### 1. **cloudflare_cache.php**
Location: `c:\xampp\htdocs\php\cloudflare_cache.php`

Main helper class and functions for Cloudflare API integration.

#### CloudflareCache Class Methods:
- `__construct($zoneId, $apiToken)` - Initialize with Cloudflare credentials
- `purgeUrl($url)` - Purge a single URL
- `purgeUrls($urls)` - Purge multiple URLs
- `purgeEverything()` - Purge entire zone cache (WARNING: clears all cache)
- `purgeTags($tags)` - Purge by cache tags
- `testConnection()` - Test Cloudflare API connectivity

#### Helper Functions:
- `auto_purge_movie_cache($movieTitle, $tmdbId, $zoneId, $apiToken)` - Auto-purge a movie page
- `auto_purge_related_pages($urlPaths, $zoneId, $apiToken)` - Auto-purge index/search pages

### 2. **cache_purge_api.php**
Location: `c:\xampp\htdocs\php\cache_purge_api.php`

AJAX endpoint for handling cache purge requests from the admin panel.

#### Supported Actions:
- `purge_url` - Purge single URL
- `purge_urls` - Purge multiple URLs
- `purge_everything` - Purge entire zone (requires confirmation)
- `purge_tags` - Purge by tags
- `test_connection` - Test API connection

### 3. **movies.php** (MODIFIED)
Added:
- Cloudflare configuration variables
- Cache purge UI button in admin panel
- Cache purge modal with three tabs
- JavaScript handlers for cache purge functionality
- Auto-purge integration in movie actions

## Configuration

### Step 1: Get Cloudflare Credentials

1. Log in to your Cloudflare dashboard
2. Go to your domain's settings
3. Get your **Zone ID** from the right sidebar (Overview page)
4. Create an **API Token** with cache purge permissions:
   - Click on your profile icon (bottom-left)
   - Go to "My Profile" → "API Tokens"
   - Create a new token with these permissions:
     - Zone - Cache Purge - Purge
     - Zone - Zone - Read (for connection test)

### Step 2: Add Credentials to movies.php

Edit `c:\xampp\htdocs\php\movies.php` (around line 72):

```php
// Cloudflare API configuration for cache purging
$cloudflare_zone_id = 'YOUR_ZONE_ID_HERE';  // Replace with your Zone ID
$cloudflare_api_token = 'YOUR_API_TOKEN_HERE';  // Replace with your API Token
$enable_cloudflare_auto_purge = true;  // Set to true to enable auto-purge
```

**Best Practices for Production:**
- Store credentials in environment variables instead of hardcoding
- Never commit API tokens to version control
- Use restricted API tokens with minimal permissions

Example using environment variables:
```php
$cloudflare_zone_id = getenv('CLOUDFLARE_ZONE_ID') ?: '';
$cloudflare_api_token = getenv('CLOUDFLARE_API_TOKEN') ?: '';
```

## Usage

### Manual Cache Purge

1. **Access the Cache Purge Tool:**
   - In the admin panel, click the "Purge Cloudflare Cache" button
   - A modal will open with three tabs

2. **Purge Single URL:**
   - Go to "Single URL" tab
   - Enter the full URL (e.g., `https://xdmovies.site/movies/sample-123.html`)
   - Click "Purge Cache"

3. **Purge Multiple URLs:**
   - Go to "Multiple URLs" tab
   - Enter one URL per line
   - Click "Purge Cache"

4. **Purge Everything:**
   - Go to "Purge All" tab
   - Check the confirmation checkbox
   - Click "Purge Cache"
   - **Warning:** This clears all cache for your entire zone

### Automatic Cache Purge

When `$enable_cloudflare_auto_purge` is set to `true`:

**Publishing a Movie:**
- Movie status changed from Draft → Published
- HTML page is generated
- Movie page URL is purged from Cloudflare cache
- Related pages (/, /search.html) are purged

**Updating a Movie:**
- If the movie is already published
- Movie page URL is purged from Cloudflare cache
- Related pages (/, /search.html) are purged
- New content is cached by Cloudflare

**Deleting a Movie:**
- Movie is removed from database
- Movie HTML page is deleted
- Movie page URL is purged from Cloudflare cache
- Related pages (/, /search.html) are purged

**Reverting to Draft:**
- Movie status changed from Published → Draft
- HTML page is deleted
- Movie page URL is purged from Cloudflare cache
- Related pages (/, /search.html) are purged

## URL Naming Convention

The system automatically generates URLs based on the movie's title and TMDB ID:

**Pattern:** `https://[domain]/movies/{slug}-{tmdb_id}.html`

**Examples:**
- `https://xdmovies.site/movies/dune-438631.html`
- `https://xdmovies.site/movies/3idiots-20453.html`
- `https://xdmovies.site/movies/amaran-927342.html`

## Error Handling

The system includes comprehensive error handling:

- **Invalid URL Format:** Validates all URLs before purging
- **Connection Failures:** Catches API errors and logs them
- **Authentication Issues:** Returns clear error messages if credentials are wrong
- **Rate Limiting:** Cloudflare may rate-limit requests (usually 1,200 requests/5 minutes)

## Monitoring & Logging

All cache purge operations are logged to PHP error_log:

```
[date] Admin auto-purged movie cache: https://xdmovies.site/movies/sample-123.html
[date] Auto-purged related pages: https://xdmovies.site/, https://xdmovies.site/search.html
[date] Failed to auto-purge movie cache: [Error message]
```

Check your PHP error log to monitor purge operations:
- On XAMPP: `c:\xampp\php\logs\php_error_log`

## API Reference

### CloudflareCache::purgeUrl()

```php
$cf = new CloudflareCache($zoneId, $apiToken);
$result = $cf->purgeUrl('https://example.com/page.html');

// Returns:
[
    'success' => true/false,
    'message' => 'Cache purged successfully...'
]
```

### CloudflareCache::purgeUrls()

```php
$urls = [
    'https://example.com/page1.html',
    'https://example.com/page2.html'
];
$result = $cf->purgeUrls($urls);

// Returns:
[
    'success' => true/false,
    'message' => 'Cache purged successfully for 2 URL(s)',
    'count' => 2
]
```

### CloudflareCache::purgeEverything()

```php
// WARNING: This purges ALL cache!
$result = $cf->purgeEverything();

// Returns:
[
    'success' => true/false,
    'message' => 'All cache purged successfully!'
]
```

### CloudflareCache::testConnection()

```php
$result = $cf->testConnection();

// Returns:
[
    'success' => true/false,
    'message' => 'Successfully connected to Cloudflare zone: example.com'
]
```

## Troubleshooting

### "Unauthorized" Error
- **Cause:** API token has wrong permissions or is invalid
- **Solution:** Verify API token has "Cache Purge" permission

### "Invalid Zone ID" Error
- **Cause:** Zone ID is incorrect
- **Solution:** Double-check Zone ID from Cloudflare dashboard

### "Connection Failed" Error
- **Cause:** Network issue or Cloudflare API is down
- **Solution:** Check internet connection, try again later

### Cache Not Being Purged
- **Check if enabled:** Verify `$enable_cloudflare_auto_purge = true`
- **Check credentials:** Verify Zone ID and API Token are correct
- **Check logs:** Look at PHP error_log for error messages
- **Test connection:** Use the manual cache purge "Test Connection" feature

### "Purge Everything" Button Grayed Out
- **Cause:** Confirmation checkbox not checked
- **Solution:** Check the confirmation checkbox before clicking purge

## Cloudflare API Limits

- **Rate Limit:** 1,200 requests per 5 minutes
- **Files per Request:** Up to 30 URLs can be purged in one request
- **Tags per Request:** Up to 30 tags can be purged in one request

## Performance Impact

- Purging single URL: ~0.5-1 second
- Purging multiple URLs (batch): ~0.5-1 second
- Purging everything: ~1-2 seconds

Cache purge operations are asynchronous and don't block page generation.

## Security Considerations

1. **API Token Security:**
   - Never commit tokens to version control
   - Use restricted tokens with minimal permissions
   - Rotate tokens regularly

2. **Access Control:**
   - Only authenticated admins can purge cache
   - Manual purge modal is behind login
   - Auto-purge is admin-initiated only

3. **URL Validation:**
   - All URLs are validated before sending to Cloudflare
   - Invalid URLs are rejected with error message

## Disable Auto-Purge

If you need to temporarily disable auto-purge:

```php
$enable_cloudflare_auto_purge = false;  // Disables all automatic purging
```

Manual cache purge through the admin panel will still work.

## Testing the Setup

1. **Test Connection:**
   - Open Cache Purge modal
   - Single URL tab
   - Enter a test URL
   - Check browser console for response

2. **Manual Purge Test:**
   - Publish a movie
   - Open Cache Purge modal
   - Enter the movie URL
   - Click Purge Cache
   - Should see success message

3. **Auto-Purge Test:**
   - Add/publish a new movie
   - Check PHP error log for purge entries
   - Visit the movie page to verify it loads correctly

## Support Resources

- [Cloudflare API Documentation](https://developers.cloudflare.com/api/)
- [Cache Purge API Reference](https://developers.cloudflare.com/api/operations/zone-cache-purge-cache)
- [API Token Management](https://developers.cloudflare.com/api/tokens/create/)

## Future Enhancements

Possible improvements:
1. Queue-based purging for better performance
2. Purge by page type (movies, series, search, etc.)
3. Schedule automatic full purges
4. Purge history/audit log
5. Bulk movie operations with coordinated purging
6. Custom purge rules based on content type
