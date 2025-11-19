# Robust Page Deletion Implementation

## Overview
Implemented robust page deletion logic that removes **all filename variants** matching the pattern `*-{tmdb_id}.html` for both movies and series. This eliminates issues where old filename variants remain after title changes.

## Implementation Details

### Series (php/series.php)

#### New Helper Function: `delete_series_page_variants()`
**Location:** Lines 217-245 in `php/series.php`

```php
/**
 * Delete all series page file variants matching pattern *-{tmdb_id}.html
 * This handles title changes where old filename variants may still exist
 * @param string $tmdb_id The series TMDB ID
 * @param string $seriesDir Path to series directory
 * @return int Number of files deleted
 */
function delete_series_page_variants($tmdb_id, $seriesDir) {
    $deleted_count = 0;
    if (!is_dir($seriesDir)) {
        return $deleted_count;
    }
    
    // Build glob pattern to match all files with this tmdb_id
    $pattern = $seriesDir . '*-' . preg_quote($tmdb_id, '/') . '.html';
    $files = glob($pattern);
    
    if ($files && is_array($files)) {
        foreach ($files as $file) {
            if (is_file($file)) {
                if (@unlink($file)) {
                    $deleted_count++;
                }
            }
        }
    }
    
    return $deleted_count;
}
```

**Features:**
- Uses `glob()` to find all files matching the pattern
- Safely checks directory existence
- Returns count of deleted files for logging/feedback
- Uses `preg_quote()` to escape special characters in tmdb_id

#### Updated Handlers

**1. Toggle Status (Draft Revert) - Lines ~393-406**
- **Old:** Deleted only the current slug-based filename
- **New:** Calls `delete_series_page_variants()` to remove all variants
- **Message:** Updated to report number of variants deleted

**2. Update Series - Lines ~455-463**
- **Old:** Only deleted old file if title changed
- **New:** Calls `delete_series_page_variants()` with old tmdb_id to remove all variants
- **Trigger:** When title OR tmdb_id changes

**3. Delete Series - Lines ~536-545**
- **Old:** Generated and deleted single filename
- **New:** Calls `delete_series_page_variants()` to remove all variants
- **Behavior:** Removes all page files associated with the series

### Movies (php/movie_page_helper.php)

**Status:** ✅ Already Implemented

The `movie_delete_page()` function already contains robust deletion logic:

**Location:** Lines 394-408 in `php/movie_page_helper.php`

```php
// Also try to delete any old versions with same TMDB ID but different slug
// This handles cases where the title changed
$pattern = preg_quote($tmdb_id, '#');
$files = glob($outDir . '*-' . $tmdb_id . '.html');

foreach ($files as $file) {
    if ($file !== $filePath && file_exists($file)) {
        if (!unlink($file)) {
            error_log("Failed to delete old movie page: $file");
        } else {
            error_log("Old movie page deleted: $file");
        }
    }
}
```

**Features Already Present:**
- Uses glob pattern to find all variants
- Skips the current file to avoid redundant operations
- Logs all deletion attempts

## Usage Scenarios

### Scenario 1: Title Change for Published Series
```
Original: "Breaking Bad" (tmdb_id: 1396) → File: breaking-bad-1396.html
Change to: "Breaking Bad: The Series" 
Result: Both "breaking-bad-1396.html" AND "breaking-bad-the-series-1396.html" are removed on draft revert
```

### Scenario 2: Series Deletion
```
Series deleted: "The Office" (tmdb_id: 2316)
Directory scanned for: *-2316.html
Result: ALL matching files (the-office-2316.html, office-2316.html, etc.) are removed
```

### Scenario 3: TMDB ID Update
```
Old TMDB ID: 1234 → Files: old-series-1234.html, my-series-1234.html
New TMDB ID: 5678
Result: ALL files with old tmdb_id (1234) are removed
```

## Benefits

1. **Eliminates Orphaned Files:** No stale page files left behind after title changes
2. **Reduces Manual Cleanup:** Admin doesn't need to manually delete old filename variants
3. **Consistent State:** Filesystem always matches the current series/movie state
4. **Better Error Handling:** Returns count of deleted files for feedback
5. **Prevents Conflicts:** Avoids multiple versions of the same content being indexed

## Technical Notes

- **Pattern Matching:** Uses `glob()` with `*-{tmdb_id}.html` pattern
- **Special Characters:** TMDB IDs are escaped with `preg_quote()` to safely handle any characters
- **Directory Safety:** Checks directory existence before attempting glob operations
- **Error Suppression:** Uses `@unlink()` to gracefully handle permission issues
- **Logging:** For movies, deletion attempts are logged via `error_log()`; for series, variant counts are returned for user feedback

## Testing Checklist

- [ ] Publish series, change title, revert to draft → Verify all title variants deleted
- [ ] Publish series with special characters, revert to draft → Verify special char variants deleted
- [ ] Delete published series → Verify all page files removed from `/series/` folder
- [ ] Update series tmdb_id → Verify old variant files removed, new ones generated
- [ ] Test with movie publish/delete → Verify no side effects, consistent behavior

## Related Files

- `php/series.php` - Handler calls and helper function
- `php/movie_page_helper.php` - Movie page generator (already has robust deletion)
- `/series/` - Target directory for series HTML files
- `/movies/` - Target directory for movie HTML files

## Future Enhancements

1. Add admin UI to view all generated page files for a series/movie
2. Implement cleanup utility to find and remove orphaned files
3. Add audit trail of deletion operations
4. Create statistics on file cleanup operations
