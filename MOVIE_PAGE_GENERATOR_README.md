# Movie HTML Page Generator Implementation

## Overview
This implementation automatically generates, updates, and deletes HTML pages for movies when they are added, edited, or deleted in the admin panel. The system uses a helper file to manage page generation and integrates directly with the movie administration workflow.

## Files Created/Modified

### 1. **movie_page_helper.php** (NEW)
Location: `c:\xampp\htdocs\php\movie_page_helper.php`

This helper file contains all the functions needed for HTML page generation:

#### Functions:
- **movie_slugify($text)** - Converts movie titles to URL-friendly slugs
- **movie_ensure_dir($dir)** - Creates directories if they don't exist
- **movie_tmdb_image_url($path, $size)** - Generates TMDB image URLs
- **movie_get_filename($title, $tmdb_id)** - Generates HTML filename based on title and TMDB ID
- **movie_get_downloads_from_db($pdo, $tmdb_id)** - Fetches movie versions from database
- **movie_render_downloads_html($downloads)** - Renders download links HTML
- **movie_fetch_from_db($pdo, $movie_id, $tmdb_id)** - Fetches complete movie data
- **movie_generate_page($pdo, $movie_id, $tmdb_id)** - Generates a new HTML page
- **movie_delete_page($title, $tmdb_id)** - Deletes existing HTML page(s)
- **movie_update_page($pdo, $oldTitle, $newTitle, $movie_id, $tmdb_id)** - Updates page (handles title changes)

### 2. **movies.php** (MODIFIED)
Location: `c:\xampp\htdocs\php\movies.php`

#### Changes Made:

1. **Added helper include** (Line ~91):
   ```php
   require_once 'movie_page_helper.php';
   ```

2. **Modified add_movie action**:
   - Movies are created with `status = 'draft'` by default
   - Pages are NOT generated for draft movies
   - Pages will be generated when status is toggled to 'published'

3. **Modified update_movie action**:
   - Now fetches the old movie title and status
   - If the movie is published, regenerates the HTML page
   - Handles title changes by deleting old filename and creating new one
   - Updates all page content with new movie details

4. **Modified delete_movie action**:
   - Now fetches the movie title along with TMDB ID
   - Calls `movie_delete_page()` to remove HTML files
   - Handles both direct deletion and cleanup of old filename variations

5. **Modified toggle_status action**:
   - When toggling to **'published'**: Generates the HTML page
   - When toggling to **'draft'**: Deletes the HTML page
   - Updates session messages to reflect these actions

## Workflow

### Adding a Movie
1. Admin adds a new movie with details
2. Movie is inserted into database with `status = 'draft'`
3. No HTML page is generated yet
4. Admin can view/edit the movie in the admin panel

### Publishing a Movie
1. Admin clicks the status button to toggle from "Draft" to "Published"
2. `toggle_status` action is triggered
3. HTML page is automatically generated
4. Page is saved to: `c:\xampp\htdocs\movies\{slug}-{tmdb_id}.html`
5. Page becomes publicly accessible

### Editing a Movie
1. Admin edits movie details (title, description, sources, etc.)
2. If the movie is **published**, the HTML page is automatically regenerated
3. If the title changed, the old HTML file is deleted and a new one created
4. Changes are immediately reflected on the public page

### Deleting a Movie
1. Admin deletes a movie
2. Movie is removed from database
3. Associated HTML page is automatically deleted
4. All cleanup is handled automatically

### Reverting to Draft
1. Admin clicks status button to toggle from "Published" to "Draft"
2. HTML page is automatically deleted
3. Movie data remains in the database
4. Movie becomes inaccessible to the public

## File Naming Convention

HTML files are named using the format: `{slug}-{tmdb_id}.html`

Examples:
- `3idiots-20453.html`
- `dune-438631.html`
- `amaran-927342.html`

The slug is derived from the movie title with:
- Special characters removed or converted to hyphens
- Converted to lowercase
- Multiple spaces/hyphens collapsed to single hyphens
- Trimmed of leading/trailing hyphens

## Page Content

Generated HTML pages include:
- Movie title and metadata
- TMDB poster and backdrop images
- Release date, genres, rating
- Audio languages (from admin notes)
- Source information (Netflix, Amazon, etc.)
- Cast information
- Admin notes (if provided)
- Download links grouped by source
- Proper SEO tags (meta description, og:image, canonical URL, etc.)
- Responsive styling using existing CSS

## Database Relationships

The system uses:
- **movies** table: Main movie records with `status` field
  - `id` - Primary key
  - `tmdb_id` - TMDB identifier
  - `title` - Movie title
  - `status` - 'draft' or 'published'
  - `audio_languages` - Available audio options
  - `source` - Streaming source info
  - `note` - Admin notes

- **movie_downloads** table: TMDb movie metadata
  - Synced with TMDB API on movie addition/update

- **movie_versions** table: Download links and versions
  - Multiple versions per movie (720p, 1080p, 2160p, etc.)
  - Each version has source information

## Error Handling

The helper functions include:
- Try-catch blocks for all database operations
- File operation checks
- Logging via `error_log()` for debugging
- Graceful fallbacks for missing data
- HTML escaping for security

## Security Features

- Output is properly HTML-escaped to prevent XSS
- Database uses prepared statements (mysqli in movies.php)
- PDO parameterized queries in helper functions
- File operations with proper permission checks

## Future Enhancements

Possible improvements:
1. Batch regeneration of all published pages
2. Page caching/invalidation
3. Automatic sitemap updates when pages are published/deleted
4. Webhook notifications when pages are published
5. Preview functionality before publishing
6. Version history for HTML pages

## Testing the Implementation

1. **Add a Movie**:
   - Click "Add New Movie" button
   - Fill in details
   - Movie should appear in database with "Draft" status
   - No HTML file should exist in `/movies/` folder

2. **Publish a Movie**:
   - Click the status button on a draft movie
   - Status should change to "Published"
   - HTML file should appear in `/movies/` folder
   - File name should follow the `{slug}-{tmdb_id}.html` format

3. **Edit a Movie**:
   - Edit a published movie's title or details
   - HTML file should be regenerated with new content
   - If title changed, check that old file is deleted

4. **Delete a Movie**:
   - Delete a published movie
   - HTML file should be removed from `/movies/` folder
   - Movie should be gone from admin panel

5. **Check Pages**:
   - Visit `https://yoursite.com/movies/{slug}-{tmdb_id}.html`
   - Page should display correctly with all content

## Notes

- The implementation prioritizes data integrity over performance
- All page generations are synchronous (blocking)
- For large-scale deployments, consider implementing queue-based generation
- The helper uses the PDO connection from `db.php` automatically
- Movies created before this implementation will have `status = 'draft'` by default
- Existing HTML pages are not affected; only new operations create/modify pages
