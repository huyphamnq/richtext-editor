# Rich Text Editor

Rich Text Editor is a Vue 3 + Tiptap application focused on practical writing and document workflows, including structured formatting, media handling, table editing, export, and drawn electronic signatures.

## English

### Features

- Rich text formatting
  - Bold, italic, underline, strikethrough
  - Subscript and superscript
  - Headings, paragraph styles, blockquote, horizontal rule
  - Font family, font size, text color, background color
- Layout and alignment
  - Left, center, right, justify for text blocks
  - Left, center, right alignment for images and signature blocks
- Lists and structure
  - Bullet and numbered lists
  - Grouped context-menu actions with second-level submenus
- Media and embeds
  - Upload image and video
  - Insert image/video URL
  - Insert hyperlink and generic embed URL
- Table editing
  - Insert table
  - Right-click row/column controls
  - Cell operations: merge, split, background color
- Electronic signature
  - Draw signature on canvas
  - Insert trimmed signature image with signer name
- Export and clipboard
  - Preview and download HTML, Markdown, JSON
  - Copy export output
- AI assist
  - Gemini-powered create/edit actions
  - API key stored locally in browser localStorage

### Tech Stack

- Vue 3
- Tiptap 2
- Vite 5
- Tailwind CSS + scoped CSS
- DOMPurify, Turndown, Prettier

### Getting Started

Prerequisites:

- Node.js 18+
- npm 9+

Install dependencies:

```bash
npm install
```

Run development server:

```bash
npm run dev
```

Build production bundle:

```bash
npm run build
```

Preview production bundle:

```bash
npm run preview
```

### Project Structure

```text
.
├── index.html
├── package.json
├── postcss.config.js
├── tailwind.config.js
├── vite.config.js
└── src
    ├── App.vue
    ├── main.js
    └── style.css
```

### Documentation

- Contribution guide: [CONTRIBUTING.md](CONTRIBUTING.md)
- Code of conduct: [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)
- Changelog: [CHANGELOG.md](CHANGELOG.md)
- Release notes v1.0.0: [RELEASE_NOTES_v1.0.0.md](RELEASE_NOTES_v1.0.0.md)
- License: [LICENSE](LICENSE)

## Tiếng Việt

### Tính năng

- Soạn thảo rich text
  - In đậm, in nghiêng, gạch chân, gạch ngang
  - Chỉ số trên và chỉ số dưới
  - Heading, paragraph, blockquote, horizontal rule
  - Font chữ, cỡ chữ, màu chữ, màu nền chữ
- Bố cục và căn lề
  - Căn trái, giữa, phải, justify cho văn bản
  - Căn trái, giữa, phải cho ảnh và khối chữ ký
- Danh sách và cấu trúc
  - Bullet list, numbered list
  - Context menu nhóm chức năng với dropbar cấp 2
- Media và embed
  - Upload ảnh và video
  - Chèn URL ảnh/video
  - Chèn hyperlink và embed URL
- Bảng
  - Chèn bảng
  - Chuột phải để thao tác hàng/cột
  - Gộp ô, tách ô, tô màu nền ô
- Chữ ký điện tử
  - Vẽ chữ ký trên canvas
  - Chèn ảnh chữ ký đã cắt viền dư và hiển thị họ tên người ký
- Export và clipboard
  - Preview và tải HTML, Markdown, JSON
  - Sao chép nội dung export
- Trợ lý AI
  - Sinh/chỉnh sửa nội dung bằng Gemini
  - API key lưu local trong localStorage

### Bắt đầu nhanh

Yêu cầu:

- Node.js 18+
- npm 9+

Cài đặt:

```bash
npm install
```

Chạy môi trường phát triển:

```bash
npm run dev
```

Build production:

```bash
npm run build
```

Preview bản build:

```bash
npm run preview
```

### Ghi chú

- Build có thể cảnh báo chunk size lớn, nhưng không làm fail build.
- Nếu cần tối ưu sâu hơn, có thể tách bundle bằng dynamic import hoặc cấu hình manualChunks.

## Versioning

This project follows Semantic Versioning.

- Current documented release: 1.0.0
- Detailed history: [CHANGELOG.md](CHANGELOG.md)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for details.
