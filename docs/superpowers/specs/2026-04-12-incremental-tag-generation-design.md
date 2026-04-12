# Incremental Tag Generation Design

## Problem

Currently `ego--update-summary` regenerates ALL tag pages every time ANY file changes.
It calls `ego--get-org-file-options` for EVERY org file to rebuild the full tag→posts mapping,
even when only one post changed. This is slow for blogs with many posts.

## Solution

Extend the existing `.ego.db` file to cache tag metadata per post, then incrementally update
only the affected tag pages on subsequent publishes.

## Data Structure Change

**Old `.ego.db` format (cons cell):**
```elisp
("posts/blog1.org" . "/blog/blog1.html")
```

**New format (list with plist metadata):**
```elisp
("posts/blog1.org" "/blog/blog1.html"
 :title "文章标题" :date "2024-01-01" :mod-date "2024-03-15"
 :tags ("emacs" "org-mode") :category "blog")
```

Each entry is a list: first element is the org file path, second is the html-uri,
followed by a plist of metadata including tags.

## Fallback Strategy

Any anomaly with `.ego.db` triggers a full rebuild:

- File does not exist
- File is unreadable or corrupted (read fails)
- File contains old format entries (cons cells)
- Any single entry cannot be parsed
- Any unexpected error during incremental path

Full rebuild regenerates all tag pages AND rewrites `.ego.db` in the new format.

The incremental path is only taken when `.ego.db` exists, is readable, and ALL entries
are in the new list format.

## Core Flow

```
ego--publish-changes (with force-all flag)
  |
  |- If force-all is non-nil:
  |    -> Full path: existing ego--update-summary
  |
  |- Read .ego.db
  |    -> Any anomaly? -> Full path
  |
  |- For changed/deleted files only, call ego--get-org-file-options
  |    to extract new metadata (tags, title, date, etc.)
  |
  |- Compare old vs new tag data to compute affected-tags:
  |    - Tags added to a file
  |    - Tags removed from a file
  |    - Tags on a file whose title/date changed
  |
  |- Update .ego.db entries for changed files
  |
  |- ego--update-summary-incremental:
  |    Input: affected-tags + full tag cache (rebuilt from .ego.db)
  |    Only re-render tag pages for affected-tags + tag index page
  |
  |- If no files changed tags/title/date -> skip summary update entirely
```

## Function Changes

### New functions (ego-export.el)

- **`ego--get-tag-cache-from-db`** — Read `.ego.db` and build a tag→posts alist.
  Returns `nil` if any entry is in old format or unreadable.

- **`ego--compute-affected-tags`** — Compare old and new tag data for changed files.
  Returns list of tag names whose pages need re-rendering.

- **`ego--update-summary-incremental`** — Like `ego--update-summary` but only renders
  pages for specified tags. Takes the full tag cache as input instead of file-attr-list.

### Modified functions

- **`ego--publish-changes`** — Add incremental path: read cache, compute affected tags,
  call incremental summary update. Skip when force-all or cache unavailable.
  Pass `force-all` through to control this behavior.

- **`ego-update-org-html-mapping`** — Extend to write new list format with metadata plist.

- **`ego-delete-org-html-mapping`** — Adapt to new list format.

- **`ego-get-org-html-mapping`** — Read compatibly: return alist where each value is
  the full entry (list with metadata). Callers that only need html-uri extract the second element.

### Unchanged functions

- **`ego--update-summary`** — Kept as-is for full rebuild fallback.
- **`ego--get-org-file-options`** — Not modified. Incremental path only calls it for changed files.

## force-all Control

The existing `force-all` parameter from `ego-do-publication` (ego.el:64) naturally controls
this: when user selects "Full publish?", the full rebuild path is used, regenerating all
tag pages and rewriting `.ego.db`.

## Tag Index Page

The tag index page (`/tags/index.html`) is re-rendered whenever any tag is added or removed,
since it lists all tags with counts. This is low cost as there are typically only dozens of tags.
