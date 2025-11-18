# Quick Setup Guide - Cloudflare Cache Purge

## 5-Minute Setup

### Step 1: Get Your Cloudflare Credentials (2 minutes)

1. Go to https://dash.cloudflare.com/
2. Select your domain (xdmovies.site or your domain)
3. **Get Zone ID:**
   - Click "Overview" in the left menu
   - Scroll right side to find "Zone ID"
   - Copy it (looks like: `d1234567890abcdef1234567890abcdef`)

4. **Create API Token:**
   - Click your profile icon (bottom left corner)
   - Select "My Profile"
   - Click "API Tokens" tab
   - Click "Create Token"
   - Find "Cache Purge" in the template list
   - Click "Use Template"
   - Keep default permissions (Zone - Cache Purge - Purge)
   - Select your domain
   - Click "Continue to summary"
   - Click "Create Token"
   - Copy the token (save it somewhere safe!)

### Step 2: Configure movies.php (2 minutes)

1. Open: `c:\xampp\htdocs\php\movies.php`
2. Find this section (around line 72):
   ```php
   // Cloudflare API configuration for cache purging
   $cloudflare_zone_id = 'YOUR_ZONE_ID_HERE';
   $cloudflare_api_token = 'YOUR_API_TOKEN_HERE';
   $enable_cloudflare_auto_purge = false;
   ```

3. Replace with your credentials:
   ```php
   // Cloudflare API configuration for cache purging
   $cloudflare_zone_id = 'your-actual-zone-id-here';
   $cloudflare_api_token = 'your-actual-api-token-here';
   $enable_cloudflare_auto_purge = true;  // Enable auto-purge
   ```

4. Save the file

### Step 3: Test It (1 minute)

1. Go to admin panel: http://localhost/php/movies.php
2. Click "Purge Cloudflare Cache" button
3. You should see the cache purge modal
4. That's it! You're ready to use it

## Using the Cache Purge Feature

### Manual Purge (Single URL)
1. Click "Purge Cloudflare Cache" button
2. Enter URL: `https://yoursite.com/movies/movie-name-123.html`
3. Click "Purge Cache"
4. See success message ✓

### Manual Purge (Multiple URLs)
1. Click "Purge Cloudflare Cache" button
2. Click "Multiple URLs" tab
3. Paste URLs (one per line):
   ```
   https://yoursite.com/movies/movie1-123.html
   https://yoursite.com/movies/movie2-456.html
   ```
4. Click "Purge Cache"

### Automatic Purge
Just publish/update/delete movies and cache is purged automatically!

## Verification

### Check if it's working:
1. Open browser developer tools (F12)
2. Go to admin panel
3. Publish a movie
4. Check browser Console tab for network requests
5. You should see request to `cache_purge_api.php`

### Check logs:
1. Open: `c:\xampp\php\logs\php_error_log`
2. Look for entries like:
   ```
   Auto-purged movie cache: https://...
   ```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Invalid credentials" error | Check Zone ID and API Token in movies.php |
| "Unauthorized" error | API Token may have wrong permissions |
| Cache not purging | Set `$enable_cloudflare_auto_purge = true` |
| Button not showing | Verify files are in correct location |

## File Locations

- Config: `c:\xampp\htdocs\php\movies.php` (lines ~72)
- Handler: `c:\xampp\htdocs\php\cloudflare_cache.php`
- API Endpoint: `c:\xampp\htdocs\php\cache_purge_api.php`
- Documentation: `c:\xampp\htdocs\CLOUDFLARE_CACHE_PURGE_README.md`

## What Gets Purged?

**Automatic purging handles:**
- ✓ Movie pages when published/updated/deleted
- ✓ Homepage (/) 
- ✓ Search page (/search.html)
- ✓ Old filenames if movie title changes

**You can manually purge:**
- Any single URL
- Multiple URLs at once
- Entire zone cache (if needed)

## Important Notes

⚠️ **Never share your API Token**
- Treat it like a password
- Don't put in version control
- Regenerate if you suspect it's compromised

⚠️ **Zone vs URL purge**
- "Purge All" clears everything - use carefully
- Single URL purge only affects that page
- Start with single URLs when testing

✓ **Best Practices**
- Use auto-purge for automation
- Use manual purge for emergency cache clearing
- Check logs to verify operations
- Keep API Token secure
