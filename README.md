# saurabh-karn.github.io

A minimal, no-JS personal site and blog powered by GitHub Pages (Jekyll).

## Structure
- `_config.yml` — site configuration
- `_layouts/` — templates for pages and posts
- `_includes/` — header and footer
- `_posts/` — blog posts (`YYYY-MM-DD-title.md`)
- `assets/css/styles.css` — global styles
- `blog/` — blog index
- `resume.md`, `work.md`, `tech.md` — primary pages

## Writing a post
1. Create a file in `_posts/` named `YYYY-MM-DD-title.md`
2. Add front matter:
   ```yaml
   ---
   title: "Your Post Title"
   description: "One-line summary"
   tags: [tech]
   ---
   ```
3. Write in Markdown below the front matter.

Posts with the `tech` tag will appear on `/tech/`. All posts list on `/blog/`. The RSS feed is available at `/feed.xml`.

## Development
This repo relies on GitHub Pages’ built-in Jekyll build. No JavaScript or build steps required. Push to `main` and GitHub Pages will publish the site.
