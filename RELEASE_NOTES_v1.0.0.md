# Release v1.0.0

Release date: 2026-04-01

## Overview

v1.0.0 is the first stable release of Rich Text Editor.
This version delivers a complete writing workflow with structured formatting,
media insertion, table editing, export pipelines, AI assist, and drawn
electronic signature support.

## What is Included

### Editor and Formatting

- Full rich text toolbar for inline and block formatting
- Heading and paragraph controls
- Color, highlight, font family, and font size controls
- Alignment controls for text blocks

### Context Menu and UX

- Grouped right-click menu with second-level submenus
- Improved hover behavior to prevent submenu loss
- Cleaner interaction flow for table and text actions

### Media and Tables

- Upload image and video from local files
- Insert media and links from URLs
- Insert and edit tables with row/column operations
- Cell merge, split, and background color support

### Signature and Export

- Draw electronic signatures on canvas
- Auto-trim signature image before insert
- Dedicated signature block node (no paragraph wrappers)
- Export to HTML, Markdown, and JSON with preview support

### AI Assist

- Gemini-based create/edit writing actions
- Local API key storage in browser localStorage

## Notable Fixes in This Release

- Fixed image upload regression caused by command override
- Fixed second-level submenu hover gaps in context menu
- Fixed alignment behavior for images and signature blocks
- Fixed export dropdown close behavior and positioning issues

## Upgrade Notes

- No migration steps required for local use.
- For public deployment, review security policy for client-side API key handling.

## Checks

- Production build command completed successfully:

```bash
npm run build
```

## Full Changelog

See [CHANGELOG.md](CHANGELOG.md).
