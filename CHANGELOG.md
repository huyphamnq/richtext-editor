# Changelog

All notable changes to this project will be documented in this file.

The format is based on Keep a Changelog,
and this project follows Semantic Versioning.

## [1.0.0] - 2026-04-01

### Added

- Rich text formatting controls:
  - Bold, italic, underline, strikethrough, subscript, superscript
  - Heading and paragraph tools
  - Font family, size, text color, and text background color
- Multi-level right-click context menu with grouped submenus
- Table editing features:
  - Insert table
  - Row/column controls via context menu
  - Cell merge/split and background color editing
- Media features:
  - Upload image and video
  - Insert image/video by URL
  - Insert generic embed and hyperlinks
- Electronic signature workflow:
  - Canvas drawing modal
  - Trimmed signature insertion as a dedicated block node
  - Signer name display under signature image
- Alignment behavior for:
  - Text blocks
  - Images
  - Signature blocks
- Export workflows:
  - HTML, Markdown, JSON preview and download
  - Copy export output to clipboard
- AI-assisted drafting/editing with Gemini API

### Changed

- Improved context-menu hover behavior for second-level dropbars
- Refined signature block spacing and trimming quality
- Improved image alignment command behavior and selection handling

### Fixed

- Restored image upload by preserving Tiptap base image commands
- Resolved submenu hover-loss issue caused by pointer gap
- Corrected export dropdown positioning and outside-click close behavior

[1.0.0]: https://github.com/your-org/rich-text-editor/releases/tag/v1.0.0
