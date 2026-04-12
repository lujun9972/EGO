# Incremental Tag Generation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make tag page generation incremental by caching tag metadata in `.ego.db`, so only changed files and affected tag pages are regenerated on each publish.

**Architecture:** Extend the existing `.ego.db` file format from cons cells `(path . uri)` to lists `(path uri :title ... :tags ...)`. On publish, read the cache, compute affected tags from changed/deleted files, and only re-render those tag pages. Fall back to full rebuild when cache is missing or in old format.

**Tech Stack:** Emacs Lisp, existing EGO functions (`ego--get-org-file-options`, `ego--update-summary`, mustache templates)

---

### Task 1: Extend `.ego.db` read/write to support new list format

**Files:**
- Modify: `ego-config.el:539-585` (all four `ego-*-org-html-mapping` functions)

**Context:** The current db stores `(org-path . html-uri)` cons cells. We change it to `(org-path html-uri :title "..." :date "..." :mod-date "..." :tags ("t1" "t2") :category "...")` lists. The read function must detect old format and return `nil` (triggering full rebuild). The write function always writes new format.

- [ ] **Step 1: Modify `ego-get-org-html-mapping` to validate format**

Replace `ego-config.el:539-545` with:

```elisp
(defun ego-get-org-html-mapping ()
  "Return org-html-mapping-alist stored in ego-db-file.
Each entry is a list: (org-path html-uri :title ... :tags ...).
Returns nil if file is unreadable, empty, or contains old cons-cell format."
  (let* ((ego-db-file (ego-get-db-file))
         (org-html-mapping-lines (when (file-readable-p ego-db-file)
                                   (delete "" (split-string (ego--file-to-string ego-db-file) "[\r\n]+")))))
    (when org-html-mapping-lines
      (condition-case nil
          (let ((entries (mapcar #'read org-html-mapping-lines)))
            ;; If any entry is a cons cell (old format), return nil to trigger full rebuild
            (if (cl-some #'consp (mapcar #'cdr entries))
                (progn
                  (message "EGO: Old .ego.db format detected, will rebuild")
                  nil)
              entries))
        (error
         (message "EGO: Failed to read .ego.db, will rebuild")
         nil)))))
```

- [ ] **Step 2: Modify `ego-update-org-html-mapping` to accept and write metadata**

Replace `ego-config.el:554-569` with:

```elisp
(defun ego-update-org-html-mapping (org-path html-uri &optional del metadata)
  "Update org-path and html-uri mapping in ego-db-file.
METADATA is an optional plist with :title, :date, :mod-date, :tags, :category.
If DEL is non-nil, delete origin-html-uri as well.
Return the origin html uri of ORG-PATH."
  (let* ((org-html-mapping-alist (ego-get-org-html-mapping))
         (existing (assoc org-path org-html-mapping-alist))
         (origin-html-uri (when existing (cadr existing))))
    ;; If db was unreadable or old-format, start fresh
    (unless org-html-mapping-alist
      (setq org-html-mapping-alist nil))
    (if existing
        (setcdr existing (if metadata
                             (cons html-uri metadata)
                           (cdr existing)))
      (push (if metadata
                (cons org-path (cons html-uri metadata))
              (list org-path html-uri))
            org-html-mapping-alist))
    (ego-save-org-html-mapping org-html-mapping-alist)
    (when (and del
               origin-html-uri
               (not (string= html-uri origin-html-uri)))
      (delete-file origin-html-uri))
    origin-html-uri))
```

- [ ] **Step 3: Modify `ego-delete-org-html-mapping` for list format**

Replace `ego-config.el:571-585` with:

```elisp
(defun ego-delete-org-html-mapping (org-path &optional del)
  "Delete org-path entry from ego-db-file.
If DEL is non-nil, delete origin-html-uri as well.
Return the origin html uri of ORG-PATH."
  (message "EGO DEBUG:ego-delete-org-html-mapping(%s %s)" org-path del)
  (let* ((org-html-mapping-alist (ego-get-org-html-mapping))
         (existing (assoc org-path org-html-mapping-alist))
         (origin-html-uri (when existing (cadr existing))))
    (setq org-html-mapping-alist
          (delq existing org-html-mapping-alist))
    (ego-save-org-html-mapping org-html-mapping-alist)
    (when (and del origin-html-uri)
      (message "EGO DEBUG:delete-file %s" origin-html-uri)
      (delete-file origin-html-uri))
    origin-html-uri))
```

- [ ] **Step 4: Verify existing callers still work**

The call at `ego-export.el:85`:
```elisp
(ego-update-org-html-mapping org-file new-html-uri 'del)
```
This still works — METADATA is optional, defaults to nil. The entry will be `(org-path html-uri)` without metadata. That's fine for asset files (line 211). Blog posts will get metadata added in Task 3.

The call at `ego-export.el:66`:
```elisp
(ego-delete-org-html-mapping org-file 'del)
```
`cadr` on the existing entry returns the second element (html-uri) — works for both old cons cells (cdr of cons is the uri) and new lists (second element is uri).

Wait — `cadr` on a cons cell `(a . b)` returns `nil`, not `b`. Need to handle this. But since we return `nil` for old-format dbs, this won't happen. In new format, `(cadr '(path uri :tags ...))` = `uri`. Correct.

- [ ] **Step 5: Commit**

```bash
git add ego-config.el
git commit -m "Extend .ego.db format to list with metadata plist"
```

---

### Task 2: Add `ego--get-tag-cache-from-db`

**Files:**
- Modify: `ego-export.el` (add new function after `ego--generate-summary-uri`, before `ego--update-summary`)

**Context:** This function reads `.ego.db` and builds a `summary-alist` structure identical to what `ego--update-summary` builds from `file-attr-list`. This is the core data structure that enables incremental updates.

The `summary-alist` format is:
```elisp
(("emacs" (:uri "/blog/1.html" :title "Post 1" :date "2024-01-01" :mod-date "2024-01-01" :tags ("emacs"))
              (:uri "/blog/3.html" :title "Post 3" :date "2024-03-01" :mod-date "2024-03-01" :tags ("emacs")))
 ("org-mode" (:uri "/blog/1.html" :title "Post 1" :date "2024-01-01" :mod-date "2024-01-01" :tags ("emacs" "org-mode"))))
```

- [ ] **Step 1: Add `ego--get-tag-cache-from-db`**

Insert before `ego--update-summary` in `ego-export.el`:

```elisp
(defun ego--get-tag-cache-from-db ()
  "Build tag->posts alist from .ego.db cache.
Returns nil if cache is unavailable or incomplete (missing :tags in any entry).
Each entry in the returned alist is (tag-name attr-plist1 attr-plist2 ...)."
  (let ((db-entries (ego-get-org-html-mapping))
        summary-alist)
    (when db-entries
      (catch 'invalid
        (mapc
         (lambda (entry)
           (let* ((org-path (car entry))
                  (html-uri (cadr entry))
                  (metadata (cddr entry))
                  (tags (plist-get metadata :tags)))
             (unless tags
               (message "EGO: Entry %s missing :tags in cache, will rebuild" org-path)
               (throw 'invalid nil))
             (let ((attr-plist (append (list :source-file org-path
                                             :uri html-uri)
                                       metadata)))
               (mapc
                (lambda (tag-name)
                  (let ((summary-list (assoc tag-name summary-alist)))
                    (unless summary-list
                      (add-to-list 'summary-alist (setq summary-list (list tag-name))))
                    (nconc summary-list (list attr-plist))))
                tags))))
         db-entries)
        summary-alist))))
```

- [ ] **Step 2: Commit**

```bash
git add ego-export.el
git commit -m "Add ego--get-tag-cache-from-db to build tag cache from db"
```

---

### Task 3: Add `ego--compute-affected-tags`

**Files:**
- Modify: `ego-export.el` (add after `ego--get-tag-cache-from-db`)

**Context:** Given the old db entries and new attr-plists for changed files, plus the list of deleted files, compute which tags need their pages regenerated.

- [ ] **Step 1: Add `ego--compute-affected-tags`**

```elisp
(defun ego--compute-affected-tags (db-entries changed-attr-plists deleted-files)
  "Compute tags whose pages need re-rendering.
DB-ENTRIES is the alist from ego-get-org-html-mapping.
CHANGED-ATTR-PLISTS is a list of new attr-plists for updated files.
DELETED-FILES is a list of org file paths that were deleted.
Returns a list of tag name strings, or t meaning all tags are affected."
  (let (affected-tags)
    ;; Process changed files: compare old vs new tags/title/date
    (mapc
     (lambda (new-plist)
       (let* ((org-path (plist-get new-plist :source-file))
              (old-entry (assoc org-path db-entries))
              (old-metadata (when old-entry (cddr old-entry)))
              (old-tags (when old-metadata (plist-get old-metadata :tags)))
              (new-tags (plist-get new-plist :tags))
              (old-title (when old-metadata (plist-get old-metadata :title)))
              (new-title (plist-get new-plist :title))
              (old-date (when old-metadata (plist-get old-metadata :date)))
              (new-date (plist-get new-plist :date))
              (old-mod-date (when old-metadata (plist-get old-metadata :mod-date)))
              (new-mod-date (plist-get new-plist :mod-date)))
         ;; Add all old and new tags as affected
         (setq affected-tags (append old-tags new-tags affected-tags))
         ;; If title or date changed, all tags on this file are affected
         ;; (already covered by adding all old+new tags)
         ))
     changed-attr-plists)
    ;; Process deleted files: their tags need page updates
    (mapc
     (lambda (org-path)
       (let* ((old-entry (assoc org-path db-entries))
              (old-metadata (when old-entry (cddr old-entry)))
              (old-tags (when old-metadata (plist-get old-metadata :tags))))
         (setq affected-tags (append old-tags affected-tags))))
     deleted-files)
    (delete-dups affected-tags)))
```

- [ ] **Step 2: Commit**

```bash
git add ego-export.el
git commit -m "Add ego--compute-affected-tags for incremental tag detection"
```

---

### Task 4: Add `ego--update-summary-incremental`

**Files:**
- Modify: `ego-export.el` (add after `ego--compute-affected-tags`)

**Context:** This function renders only the affected tag pages. It uses the same mustache templates as `ego--update-summary` but only for specified tags. It receives the full `summary-alist` (built from cache) and filters it.

- [ ] **Step 1: Add `ego--update-summary-incremental`**

```elisp
(defun ego--update-summary-incremental (summary-alist affected-tags pub-base-dir summary-name)
  "Re-render only affected tag pages.
SUMMARY-ALIST is the full tag->posts alist (built from cache + updates).
AFFECTED-TAGS is the list of tag names whose pages need regeneration.
PUB-BASE-DIR is the root publication directory.
SUMMARY-NAME is the summary type name (typically \"tags\")."
  (let* ((summary-base-dir (expand-file-name
                            (concat summary-name "/")
                            pub-base-dir))
         (summary-update-number (car (cddr (cdr (assoc summary-name (ego--get-config-option :summary))))))
         summary-dir)
    ;; Filter summary-alist to only affected tags
    (let ((filtered-alist (cl-remove-if-not
                           (lambda (entry) (member (car entry) affected-tags))
                           summary-alist)))
      ;; Re-render tag index page (shows all tags with counts, needs update when tags change)
      (ego--save-to-file
       (mustache-render
        (ego--get-cache-create
         :container-template
         (message "EGO: Read container.mustache from file")
         (ego--file-to-string (ego--get-template-file "container.mustache")))
        (ht ("header"
             (ego--render-header
              (ht ("page-title" (concat (capitalize summary-name)
                                        " Index - "
                                        (ego--get-config-option :site-main-title)))
                  ("author" (or user-full-name "Unknown Author")))))
            ("nav" (ego--render-navigation-bar))
            ("content"
             (ego--render-content
              "summary-index.mustache"
              (ht ("summary-name" (capitalize summary-name))
                  ("updates-p" (numberp summary-update-number))
                  ("updates"
                   (when (numberp summary-update-number)
                     (mapcar
                      (lambda (attr-plist)
                        (let ((tags-multi (mapcar
                                           (lambda (tag-name)
                                             (ht ("link" (ego--generate-summary-uri "tags" tag-name))
                                                 ("name" tag-name)))
                                           (plist-get attr-plist :tags))))
                          (ht ("post-uri" (plist-get attr-plist :uri))
                              ("post-title" (plist-get attr-plist :title))
                              ("post-date" (plist-get attr-plist :mod-date))
                              ("tag-links" (if (not tags-multi) "N/A"
                                             (mapconcat
                                              (lambda (tag)
                                                (mustache-render
                                                 "<a href=\"{{link}}\">{{name}}</a>" tag))
                                              tags-multi " : "))))))
                      (-uniq (-take
                              summary-update-number
                              (sort (cl-do ((k summary-alist (cdr k))
                                         (result-k nil (append (cdr (car k)) result-k)))
                                    ((equal k nil) result-k))
                                  (lambda (plist1 plist2)
                                    (< (ego--compare-standard-date
                                        (ego--fix-timestamp-string
                                         (plist-get plist1 :mod-date))
                                        (ego--fix-timestamp-string
                                         (plist-get plist2 :mod-date)))
                                       0))))))))
                  ("summary"
                   (mapcar
                    (lambda (summary-list)
                      (ht ("summary-item-name" (car summary-list))
                          ("summary-item-uri" (ego--generate-summary-uri summary-name (car summary-list)))
                          ("count" (number-to-string (length (cdr summary-list))))))
                    summary-alist)))))
            ("footer"
             (ego--render-footer
              (ht ("show-meta" nil)
                  ("show-comment" nil)
                  ("author" (or user-full-name "Unknown Author"))
                  ("google-analytics" (ego--get-config-option :personal-google-analytics-id))
                  ("google-analytics-id" (ego--get-config-option :personal-google-analytics-id))
                  ("creator-info" (ego--get-html-creator-string))
                  ("email" (ego--confound-email-address (or user-mail-address
                                                            "Unknown Email"))))))))
       (concat summary-base-dir "index.html"))
      ;; Re-render individual tag pages for affected tags only
      (mapc
       (lambda (summary-list)
         (setq summary-dir (file-name-as-directory
                            (concat summary-base-dir
                                    (ego--encode-string-to-url (car summary-list)))))
         (unless (file-directory-p summary-dir)
           (mkdir summary-dir t))
         (ego--save-to-file
          (mustache-render
           (ego--get-cache-create
            :container-template
            (message "EGO: Read container.mustache from file")
            (ego--file-to-string (ego--get-template-file "container.mustache")))
           (ht ("header"
                (ego--render-header
                 (ht ("page-title" (concat (capitalize summary-name) ": " (car summary-list)
                                           " - " (ego--get-config-option :site-main-title)))
                     ("author" "ego"))))
               ("nav" (ego--render-navigation-bar))
               ("content"
                (ego--render-content
                 "summary.mustache"
                 (ht ("summary-name" (capitalize summary-name))
                     ("summary-item-name" (car summary-list))
                     ("summary"
                      (mapcar
                       (lambda (sl)
                         (ht ("summary-item-name" (car sl))
                             ("summary-item-uri" (ego--generate-summary-uri summary-name (car sl)))
                             ("count" (number-to-string (length (cdr sl))))))
                       summary-alist))
                     ("posts"
                      (mapcar
                       (lambda (attr-plist)
                         (let ((tags-multi (mapcar
                                            (lambda (tag-name)
                                              (ht ("link" (ego--generate-summary-uri "tags" tag-name))
                                                  ("name" tag-name)))
                                            (plist-get attr-plist :tags))))
                           (ht ("post-uri" (plist-get attr-plist :uri))
                               ("post-title" (plist-get attr-plist :title))
                               ("post-date" (plist-get attr-plist :date))
                               ("tag-links" (if (not tags-multi) "N/A"
                                              (mapconcat
                                               (lambda (tag)
                                                 (mustache-render
                                                  "<a href=\"{{link}}\">{{name}}</a>" tag))
                                               tags-multi " : "))))))
                       (cdr summary-list))))))
               ("footer"
                (ego--render-footer
                 (ht ("show-meta" nil)
                     ("show-comment" nil)
                     ("author" (or user-full-name "Unknown Author"))
                     ("google-analytics" (ego--get-config-option :personal-google-analytics-id))
                     ("google-analytics-id" (ego--get-config-option :personal-google-analytics-id))
                     ("creator-info" (ego--get-html-creator-string))
                     ("email" (ego--confound-email-address (or user-mail-address
                                                               "Unknown Email"))))))))
          (concat summary-dir "index.html")))
       filtered-alist))))
```

Note: The tag index page uses the full `summary-alist` (all tags with counts), while individual tag pages are only rendered for `filtered-alist` entries.

- [ ] **Step 2: Commit**

```bash
git add ego-export.el
git commit -m "Add ego--update-summary-incremental for partial tag page rendering"
```

---

### Task 5: Modify `ego--publish-changes` to support incremental path

**Files:**
- Modify: `ego-export.el:44-104` (rewrite `ego--publish-changes`)
- Modify: `ego.el:126` (pass `force-all` to `ego--publish-changes`)

**Context:** This is the orchestration change. We add a `force-all` parameter, and branch between incremental and full paths. In the incremental path, we only call `ego--get-org-file-options` for changed files (not all files), and update the cache + render only affected tag pages.

- [ ] **Step 1: Modify `ego--publish-changes` signature and body**

Replace `ego-export.el:44-104` with:

```elisp
(defun ego--publish-changes (files-list addition-list change-plist pub-root-dir &optional force-all)
  "Publish changed org files and update tag pages.
When FORCE-ALL is non-nil, do full rebuild of all tag pages.
When FORCE-ALL is nil, attempt incremental tag update using .ego.db cache.
`files-list' and `addition-list' contain paths of org files, `change-plist'
contains two properties, one is :update for files to be updated, another is :delete
for files to be deleted. `pub-root-dir' is the root publication directory."
  (let* ((repo-dir (ego--get-repository-directory))
         (upd-list (delete-dups
                    (append (plist-get change-plist :update)
                            addition-list)))
         (del-list (plist-get change-plist :delete))
         (files-list (delete-dups (append files-list addition-list)))
         file-attr-list)
    (message "EGO DEBUG: del-list=[%s]" del-list)
    (when del-list
      (mapcar
       (lambda (org-file)
         (message "EGO DEBUG: org-file=[%s]" org-file)
         (ego--handle-deleted-file org-file)
         (ego-delete-org-html-mapping org-file 'del))
       del-list))
    (message "EGO DEBUG: upd-list=[%s]" upd-list)
    (when upd-list
      (cond
       ;; Incremental path
       ((not force-all)
        (let* ((db-entries (ego-get-org-html-mapping))
               (tag-cache (when db-entries (ego--get-tag-cache-from-db))))
          (if (not tag-cache)
              ;; Cache unavailable — fall through to full path
              (progn
                (message "EGO: No valid tag cache, using full summary generation")
                (setq file-attr-list
                      (reverse (mapcar
                                (lambda (org-file)
                                  (message "EGO DEBUG: org-file=[%s]" org-file)
                                  (let* ((need-upd-p (member org-file upd-list)))
                                    (let* ((attr-cell (ego--get-org-file-options
                                                       org-file
                                                       pub-root-dir
                                                       need-upd-p))
                                           (attr-plist (car attr-cell))
                                           (component-table (cdr attr-cell)))
                                      (when need-upd-p
                                        (run-hook-with-args 'ego-pre-publish-hooks attr-plist)
                                        (let ((new-html-uri (ego--publish-modified-file component-table
                                                                                            (plist-get attr-plist :pub-dir))))
                                          (ego-update-org-html-mapping org-file new-html-uri 'del
                                                                       (list :title (plist-get attr-plist :title)
                                                                             :date (plist-get attr-plist :date)
                                                                             :mod-date (plist-get attr-plist :mod-date)
                                                                             :tags (plist-get attr-plist :tags)
                                                                             :category (plist-get attr-plist :category))))
                                        (run-hook-with-args 'ego-post-publish-hooks attr-plist))
                                      attr-plist)))
                                files-list)))
                (unless (member
                         (expand-file-name "index.org" repo-dir)
                         files-list)
                  (ego--generate-default-index file-attr-list pub-root-dir))
                (when (and (ego--get-config-option :about)
                           (not (member
                                 (expand-file-name "about.org" repo-dir)
                                 files-list)))
                  (ego--generate-default-about pub-root-dir))
                (ego--update-category-index file-attr-list pub-root-dir)
                (when (ego--get-config-option :rss)
                  (ego--update-rss file-attr-list pub-root-dir))
                (mapc
                 (lambda (name)
                   (ego--update-summary file-attr-list pub-root-dir name))
                 (mapcar #'car (ego--get-config-option :summary))))

            ;; Incremental path: only process changed files
            (message "EGO: Using incremental summary generation")
            (let (changed-attr-plists)
              ;; Only call ego--get-org-file-options for files in upd-list
              (mapc
               (lambda (org-file)
                 (message "EGO DEBUG: incremental org-file=[%s]" org-file)
                 (let* ((attr-cell (ego--get-org-file-options
                                    org-file
                                    pub-root-dir
                                    t))
                        (attr-plist (car attr-cell))
                        (component-table (cdr attr-cell)))
                   (run-hook-with-args 'ego-pre-publish-hooks attr-plist)
                   (let ((new-html-uri (ego--publish-modified-file component-table
                                                                   (plist-get attr-plist :pub-dir))))
                     (ego-update-org-html-mapping org-file new-html-uri 'del
                                                  (list :title (plist-get attr-plist :title)
                                                        :date (plist-get attr-plist :date)
                                                        :mod-date (plist-get attr-plist :mod-date)
                                                        :tags (plist-get attr-plist :tags)
                                                        :category (plist-get attr-plist :category))))
                   (run-hook-with-args 'ego-post-publish-hooks attr-plist)
                   (push attr-plist changed-attr-plists)))
               upd-list)
              ;; Build file-attr-list from cache + changes for category/rss/index
              (let ((all-attr-plists (mapcar
                                      (lambda (entry)
                                        (append (list :source-file (car entry)
                                                      :uri (cadr entry))
                                                (cddr entry)))
                                      db-entries)))
                ;; Replace old entries with updated ones, or append new entries
                (mapc
                 (lambda (new-plist)
                   (let* ((src (plist-get new-plist :source-file))
                          (pos (cl-position src all-attr-plists
                                            :test (lambda (s p) (equal s (plist-get p :source-file))))))
                     (if pos
                         (setcar (nthcdr pos all-attr-plists) new-plist)
                       ;; New file not in cache — add it
                       (push new-plist all-attr-plists))))
                 changed-attr-plists)
                (setq file-attr-list all-attr-plists))
              ;; Update category index, RSS, default index using full attr list
              (unless (member
                       (expand-file-name "index.org" repo-dir)
                       files-list)
                (ego--generate-default-index file-attr-list pub-root-dir))
              (when (and (ego--get-config-option :about)
                         (not (member
                               (expand-file-name "about.org" repo-dir)
                               files-list)))
                (ego--generate-default-about pub-root-dir))
              (ego--update-category-index file-attr-list pub-root-dir)
              (when (ego--get-config-option :rss)
                (ego--update-rss file-attr-list pub-root-dir))
              ;; Incremental summary update
              (let ((affected-tags (ego--compute-affected-tags db-entries changed-attr-plists del-list)))
                (when affected-tags
                  ;; Rebuild summary-alist from updated cache
                  (let ((updated-cache (ego--get-tag-cache-from-db)))
                    (mapc
                     (lambda (name)
                       (let ((summary-name (if (equal name (caar (-filter (lambda (e) (equal :tags (cadr e)))
                                                                         (ego--get-config-option :summary))))
                                               "tags"
                                             name)))
                         ;; Sort each tag's post list by date
                         (mapc
                          (lambda (summary-list)
                            (setcdr
                             summary-list
                             (sort (cdr summary-list)
                                   (lambda (p1 p2)
                                     (<= (ego--compare-standard-date
                                          (ego--fix-timestamp-string (plist-get p1 :date))
                                          (ego--fix-timestamp-string (plist-get p2 :date)))
                                         0)))))
                          updated-cache)
                         ;; Sort tags alphabetically
                         (setq updated-cache
                               (sort updated-cache
                                     (lambda (a b) (string< (car a) (car b)))))
                         (ego--update-summary-incremental updated-cache affected-tags pub-root-dir summary-name)))
                     (mapcar #'car (ego--get-config-option :summary))))))))))

       ;; Full rebuild path (force-all)
       (t
        (setq file-attr-list
              (reverse (mapcar
                        (lambda (org-file)
                          (message "EGO DEBUG: org-file=[%s]" org-file)
                          (let* ((need-upd-p (member org-file upd-list)))
                            (let* ((attr-cell (ego--get-org-file-options
                                               org-file
                                               pub-root-dir
                                               need-upd-p))
                                   (attr-plist (car attr-cell))
                                   (component-table (cdr attr-cell)))
                              (when need-upd-p
                                (run-hook-with-args 'ego-pre-publish-hooks attr-plist)
                                (let ((new-html-uri (ego--publish-modified-file component-table
                                                                                  (plist-get attr-plist :pub-dir))))
                                  (ego-update-org-html-mapping org-file new-html-uri 'del
                                                               (list :title (plist-get attr-plist :title)
                                                                     :date (plist-get attr-plist :date)
                                                                     :mod-date (plist-get attr-plist :mod-date)
                                                                     :tags (plist-get attr-plist :tags)
                                                                     :category (plist-get attr-plist :category))))
                                (run-hook-with-args 'ego-post-publish-hooks attr-plist))
                              attr-plist)))
                        files-list)))
        (unless (member
                 (expand-file-name "index.org" repo-dir)
                 files-list)
          (ego--generate-default-index file-attr-list pub-root-dir))
        (when (and (ego--get-config-option :about)
                   (not (member
                         (expand-file-name "about.org" repo-dir)
                         files-list)))
          (ego--generate-default-about pub-root-dir))
        (ego--update-category-index file-attr-list pub-root-dir)
        (when (ego--get-config-option :rss)
          (ego--update-rss file-attr-list pub-root-dir))
        (mapc
         (lambda (name)
           (ego--update-summary file-attr-list pub-root-dir name))
         (mapcar #'car (ego--get-config-option :summary))))))))
```

- [ ] **Step 2: Pass `force-all` from `ego-do-publication` to `ego--publish-changes`**

Change `ego.el:126` from:
```elisp
(ego--publish-changes repo-files addition-files changed-files store-dir)
```
to:
```elisp
(ego--publish-changes repo-files addition-files changed-files store-dir force-all)
```

- [ ] **Step 3: Commit**

```bash
git add ego-export.el ego.el
git commit -m "Add incremental tag generation path in ego--publish-changes"
```

---

### Task 6: Handle `ego-link-type-process-html` — single-file caller of `ego--get-org-file-options`

**Files:**
- Modify: `ego-export.el:794-796`

**Context:** `ego-link-type-process-html` calls `(car (ego--get-org-file-options org-file default-directory nil))` which returns an attr-plist. This still works — `ego--get-org-file-options` is unchanged and still returns `(attr-plist . component-table)`.

No change needed. This is a verification step.

- [ ] **Step 1: Verify no breakage** — confirm `ego--get-org-file-options` signature and return value are unchanged.

No code changes needed.

---

### Task 7: Handle summary config `summary-name` normalization for tags

**Files:**
- Modify: `ego-export.el` (in the incremental path of Task 5)

**Context:** The config `:summary` can map any name to `:tags`. For example `("tags" :tags)` means summary-name is `"tags"`. But in `ego--update-summary`, the code at line 574-579 renames it to `"tags"` if the attribute is `:tags`. The incremental path needs the same normalization.

This is already handled in the Task 5 code above with the `(if (equal name ...) "tags" name)` logic. No additional changes needed.

- [ ] **Step 1: Verify normalization** — confirm the summary-name mapping in incremental path matches the full path logic.

No code changes needed.

---

### Task 8: End-to-end manual testing

**Files:** None (testing only)

**Context:** Since this is an Emacs Lisp project without an automated test suite, we verify by manual testing.

- [ ] **Step 1: Test full rebuild (force-all)**

1. Delete `.ego.db` if it exists in store-dir
2. Run `M-x ego-do-publication` and select "Full publish"
3. Verify all tag pages are generated correctly
4. Verify `.ego.db` is created with new list format entries

- [ ] **Step 2: Test incremental with cache**

1. Make a small change to one blog post (change title or add a tag)
2. Run `M-x ego-do-publication` without force-all
3. Verify only the affected tag pages are regenerated
4. Verify `.ego.db` is updated with new metadata

- [ ] **Step 3: Test fallback from old format**

1. Create a `.ego.db` with old cons-cell format
2. Run `M-x ego-do-publication` without force-all
3. Verify it falls back to full rebuild and regenerates `.ego.db` in new format

- [ ] **Step 4: Test deletion**

1. Delete a blog post file
2. Run `M-x ego-do-publication` without force-all
3. Verify the tag pages for that post's tags are updated
4. Verify the entry is removed from `.ego.db`
