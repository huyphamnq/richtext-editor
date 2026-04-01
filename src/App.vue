<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { EditorContent, useEditor } from '@tiptap/vue-3'
import { Extension, Node as TiptapNode } from '@tiptap/core'
import { NodeSelection } from '@tiptap/pm/state'
import StarterKit from '@tiptap/starter-kit'
import Underline from '@tiptap/extension-underline'
import TextStyle from '@tiptap/extension-text-style'
import Color from '@tiptap/extension-color'
import FontFamily from '@tiptap/extension-font-family'
import Highlight from '@tiptap/extension-highlight'
import TextAlign from '@tiptap/extension-text-align'
import Link from '@tiptap/extension-link'
import TiptapImage from '@tiptap/extension-image'
import Youtube from '@tiptap/extension-youtube'
import Subscript from '@tiptap/extension-subscript'
import Superscript from '@tiptap/extension-superscript'
import Table from '@tiptap/extension-table'
import TableRow from '@tiptap/extension-table-row'
import TableHeader from '@tiptap/extension-table-header'
import TableCell from '@tiptap/extension-table-cell'
import DOMPurify from 'dompurify'
import TurndownService from 'turndown'
import prettier from 'prettier/standalone'
import babelPlugin from 'prettier/plugins/babel'
import estreePlugin from 'prettier/plugins/estree'
import htmlPlugin from 'prettier/plugins/html'
import postcssPlugin from 'prettier/plugins/postcss'

const GEMINI_KEY_STORAGE = 'editorial-gemini-key'
const DARK_MODE_STORAGE = 'editorial-dark-mode'
const WIN98_MODE_STORAGE = 'editorial-win98-mode'
const AI_MAX_OUTPUT_TOKENS = 4096
const AI_EDITORIAL_FORMAT_RULES = [
  'Return ONLY a clean HTML fragment that can be inserted into a rich text editor.',
  'Never wrap with <html>, <head>, or <body>.',
  'Do not use markdown fences or plain markdown syntax.',
  'Prefer semantic tags: <p>, <h2>, <h3>, <ul>/<ol>/<li>, <blockquote>, <strong>, <em>, <a>.',
  'Keep output concise and directly usable with minimal manual cleanup.',
  'Preserve structure consistency and avoid repeating the same sentence.',
  'When editing selected text, keep the same intent, tone, and language unless explicitly requested otherwise.',
].join('\n')

const editorRef = ref(null)
const showAiMenu = ref(false)
const aiMenuX = ref(0)
const aiMenuY = ref(0)
const showAiKeyPanel = ref(false)
const showExportMenu = ref(false)
const showTableMenu = ref(false)
const tableMenuX = ref(0)
const tableMenuY = ref(0)
const showMediaMenu = ref(false)
const mediaMenuX = ref(0)
const mediaMenuY = ref(0)
const showAlignMenu = ref(false)
const alignMenuX = ref(0)
const alignMenuY = ref(0)
const darkMode = ref(localStorage.getItem(DARK_MODE_STORAGE) === 'true')
const win98Mode = ref(localStorage.getItem(WIN98_MODE_STORAGE) !== 'false')
const aiLoading = ref(false)
const aiError = ref('')
const showAiPromptBar = ref(false)
const aiPromptText = ref('')
const aiPromptLoading = ref(false)
const aiPromptError = ref('')
const aiPromptInputRef = ref(null)
const aiPromptMode = ref('custom')
const aiPromptX = ref(24)
const aiPromptY = ref(24)
const aiCaretPos = ref(0)
const aiPromptSelectionRange = ref({ from: 0, to: 0 })
const showAiSlashMenu = ref(false)
const aiSlashMenuX = ref(0)
const aiSlashMenuY = ref(0)
const showAiResultPanel = ref(false)
const aiResultHtml = ref('')
const aiResultLoading = ref(false)
const aiResultError = ref('')
const aiResultSelectionRange = ref({ from: 0, to: 0 })
const aiResultHadSelection = ref(false)
const aiResultRequest = ref(null)
const aiKey = ref(localStorage.getItem(GEMINI_KEY_STORAGE) || '')
const exportPreview = ref('')
const exportType = ref('HTML')
const showExportPreview = ref(false)
const showInputModal = ref(false)
const inputModalTitle = ref('')
const inputModalPlaceholder = ref('')
const inputModalValue = ref('')
const inputModalAction = ref('')
const showSignatureModal = ref(false)
const signatureCanvasRef = ref(null)
const signatureSignerName = ref('')
const signatureIsDrawing = ref(false)
const signatureHasStroke = ref(false)
const signatureInserting = ref(false)
const showContextMenu = ref(false)
const contextMenuX = ref(0)
const contextMenuY = ref(0)
const contextOnTable = ref(false)
const contextSubmenuToLeft = ref(false)
const contextMenuRef = ref(null)
const imageUploadInput = ref(null)
const videoUploadInput = ref(null)
const currentFontFamily = ref('Manrope')
const currentFontSize = ref('16px')
const showFontSizeDropdown = ref(false)
const currentTextColor = ref('#19315d')
const currentBackgroundColor = ref('#ffffff')
const uiRoot = ref(null)
const editorArticleRef = ref(null)
const codeFormatLoading = ref(false)
const codeFormatError = ref('')
const showTableCellColorMenu = ref(false)
const tableCellColorMenuX = ref(0)
const tableCellColorMenuY = ref(0)
const currentTableCellColor = ref('#ffffff')

const FONT_FAMILIES = [
  'Manrope',
  'Inter',
  'Arial',
  'Helvetica',
  'Verdana',
  'Tahoma',
  'Trebuchet MS',
  'Georgia',
  'Palatino',
  'Garamond',
  'Times New Roman',
  'Courier New',
  'Lucida Console',
]
const FONT_SIZES = [
  { label: '8px', value: '8px' },
  { label: '10px', value: '10px' },
  { label: '12px', value: '12px' },
  { label: '14px', value: '14px' },
  { label: '16px (Default)', value: '16px' },
  { label: '18px', value: '18px' },
  { label: '20px', value: '20px' },
  { label: '24px', value: '24px' },
  { label: '28px', value: '28px' },
  { label: '32px', value: '32px' },
  { label: '36px', value: '36px' },
  { label: '48px', value: '48px' },
  { label: '64px', value: '64px' },
  { label: '72px', value: '72px' },
]

const aiActions = [
  { key: 'create', label: 'Create content', instruction: 'Create rich-text ready content from the provided context and request. Keep structure clear, readable, and easy to insert into the editor.' },
  { key: 'edit', label: 'Edit writing', instruction: 'Rewrite only the selected writing for clarity, grammar, and flow while preserving original meaning, language, and voice.' },
]

const contextAiActions = computed(() => aiActions.slice(0, 4))

function buildBlockAlignStyle(width, align = 'left') {
  const normalizedAlign = ['left', 'center', 'right'].includes(align) ? align : 'left'
  const widthValue = width || '100%'
  let marginStyle = 'margin-left: 0; margin-right: auto;'
  if (normalizedAlign === 'center') marginStyle = 'margin-left: auto; margin-right: auto;'
  if (normalizedAlign === 'right') marginStyle = 'margin-left: auto; margin-right: 0;'
  return `width: ${widthValue}; display: block; ${marginStyle}`
}

const currentAlignMode = computed(() => {
  if (!editor.value) return 'left'
  if (editor.value.isActive('image')) return editor.value.getAttributes('image').align || 'left'
  if (editor.value.isActive('signatureBlock')) return editor.value.getAttributes('signatureBlock').align || 'left'
  if (editor.value.isActive({ textAlign: 'center' })) return 'center'
  if (editor.value.isActive({ textAlign: 'right' })) return 'right'
  if (editor.value.isActive({ textAlign: 'justify' })) return 'justify'
  return 'left'
})

const currentAlignIcon = computed(() => {
  if (currentAlignMode.value === 'center') return 'format_align_center'
  if (currentAlignMode.value === 'right') return 'format_align_right'
  if (currentAlignMode.value === 'justify') return 'format_align_justify'
  return 'format_align_left'
})

const turndownService = new TurndownService({ headingStyle: 'atx', codeBlockStyle: 'fenced' })

// Adds font-size to the TextStyle mark.
const FontSize = Extension.create({
  name: 'fontSize',
  addGlobalAttributes() {
    return [
      {
        types: ['textStyle'],
        attributes: {
          fontSize: {
            default: null,
            parseHTML: (element) => element.style.fontSize || null,
            renderHTML: (attributes) => {
              if (!attributes.fontSize) {
                return {}
              }
              return { style: `font-size: ${attributes.fontSize}` }
            },
          },
        },
      },
    ]
  },
  addCommands() {
    return {
      setFontSize:
        (fontSize) =>
        ({ chain }) =>
          chain().setMark('textStyle', { fontSize }).run(),
      unsetFontSize:
        () =>
        ({ chain }) =>
          chain().setMark('textStyle', { fontSize: null }).removeEmptyTextStyle().run(),
    }
  },
})

// Adds text background-color to the TextStyle mark.
const TextBackground = Extension.create({
  name: 'textBackground',
  addGlobalAttributes() {
    return [
      {
        types: ['textStyle'],
        attributes: {
          backgroundColor: {
            default: null,
            parseHTML: (element) => element.style.backgroundColor || null,
            renderHTML: (attributes) => {
              if (!attributes.backgroundColor) {
                return {}
              }
              return { style: `background-color: ${attributes.backgroundColor}` }
            },
          },
        },
      },
    ]
  },
  addCommands() {
    return {
      setBackgroundColor:
        (backgroundColor) =>
        ({ chain }) =>
          chain().setMark('textStyle', { backgroundColor }).run(),
      unsetBackgroundColor:
        () =>
        ({ chain }) =>
          chain().setMark('textStyle', { backgroundColor: null }).removeEmptyTextStyle().run(),
    }
  },
})

// Custom TableCell with background color support
const CustomTableCell = TableCell.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      backgroundColor: {
        default: null,
        parseHTML: element => element.style.backgroundColor || null,
        renderHTML: attributes => {
          if (!attributes.backgroundColor) {
            return {}
          }
          return {
            style: `background-color: ${attributes.backgroundColor}`,
          }
        },
      },
    }
  },
})

// Custom TableHeader with background color support
const CustomTableHeader = TableHeader.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      backgroundColor: {
        default: null,
        parseHTML: element => element.style.backgroundColor || null,
        renderHTML: attributes => {
          if (!attributes.backgroundColor) {
            return {}
          }
          return {
            style: `background-color: ${attributes.backgroundColor}`,
          }
        },
      },
    }
  },
})

// Generic iframe embed block.
const Embed = TiptapNode.create({
  name: 'embed',
  group: 'block',
  atom: true,
  selectable: true,
  draggable: true,
  addAttributes() {
    return {
      src: { default: null },
      title: { default: 'Embedded content' },
    }
  },
  parseHTML() {
    return [{ tag: 'div[data-embed]' }]
  },
  renderHTML({ HTMLAttributes }) {
    const { src, title } = HTMLAttributes
    return [
      'div',
      {
        'data-embed': 'true',
        class: 'embed-block',
      },
      [
        'iframe',
        {
          src,
          title,
          loading: 'lazy',
          referrerpolicy: 'no-referrer-when-downgrade',
          allowfullscreen: 'true',
        },
      ],
    ]
  },
  addCommands() {
    return {
      setEmbed:
        (options) =>
        ({ commands }) =>
          commands.insertContent({ type: this.name, attrs: options }),
    }
  },
})

// Electronic signature block to keep output as a proper block element (no paragraph wrapping).
const SignatureBlock = TiptapNode.create({
  name: 'signatureBlock',
  group: 'block',
  atom: true,
  selectable: true,
  draggable: true,
  addAttributes() {
    return {
      src: { default: null },
      signer: { default: 'Signer' },
      align: { default: 'left' },
    }
  },
  parseHTML() {
    return [{ tag: 'div[data-signature]' }]
  },
  renderHTML({ HTMLAttributes }) {
    const align = HTMLAttributes.align || 'left'
    return [
      'div',
      {
        'data-signature': 'true',
        'data-align': align,
        class: 'signature-block',
      },
      [
        'img',
        {
          class: 'signature-drawn',
          src: HTMLAttributes.src,
          alt: `Electronic signature of ${HTMLAttributes.signer || 'Signer'}`,
        },
      ],
      ['div', { class: 'signature-signer' }, HTMLAttributes.signer || 'Signer'],
    ]
  },
  addCommands() {
    return {
      setSignatureBlock:
        (options) =>
        ({ commands }) =>
          commands.insertContent({
            type: this.name,
            attrs: options,
          }),
      setSignatureAlign:
        (align) =>
        ({ commands }) =>
          commands.updateAttributes(this.name, { align }),
    }
  },
})

// Click to select image, then drag near right edge/corner to resize.
const ResizableImage = TiptapImage.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      width: {
        default: 'auto',
        parseHTML: (element) => element.getAttribute('data-width') || element.style.width || 'auto',
        renderHTML: (attributes) => {
          if (!attributes.width) return {}
          const align = attributes.align || 'left'
          return {
            'data-width': attributes.width,
            'data-align': align,
            style: buildBlockAlignStyle(attributes.width, align),
          }
        },
      },
      align: {
        default: 'left',
        parseHTML: (element) => element.getAttribute('data-align') || 'left',
      },
    }
  },
  addCommands() {
    return {
      ...this.parent?.(),
      setImageAlign:
        (align) =>
        ({ commands }) =>
          commands.updateAttributes('image', { align }),
    }
  },
  addNodeView() {
    return ({ node, getPos, editor }) => {
      const wrapper = document.createElement('div')
      wrapper.className = 'image-resizable-wrapper'
      wrapper.contentEditable = 'false'

      const applyWrapperAlign = (alignValue) => {
        const align = ['left', 'center', 'right'].includes(alignValue) ? alignValue : 'left'
        wrapper.dataset.align = align
        wrapper.style.marginLeft = align === 'center' || align === 'right' ? 'auto' : '0'
        wrapper.style.marginRight = align === 'center' ? 'auto' : align === 'right' ? '0' : 'auto'
      }

      const image = document.createElement('img')
      image.src = node.attrs.src
      image.alt = node.attrs.alt || ''
      image.title = node.attrs.title || ''
      image.style.width = node.attrs.width || 'auto'
      image.style.height = 'auto'
      applyWrapperAlign(node.attrs.align || 'left')

      let isSelected = false
      let isResizing = false
      let startX = 0
      let startWidth = 0

      const setSelected = (selected) => {
        isSelected = selected
        wrapper.classList.toggle('is-selected', selected)
      }

      const isResizeZone = (event) => {
        const rect = image.getBoundingClientRect()
        const threshold = 16
        const nearRight = rect.right - event.clientX <= threshold
        const nearBottom = rect.bottom - event.clientY <= threshold
        return nearRight || (nearRight && nearBottom)
      }

      const onMouseMove = (event) => {
        if (!isResizing) return
        const dx = event.clientX - startX
        const nextWidth = Math.max(120, startWidth + dx)
        image.style.width = `${nextWidth}px`
      }

      const onMouseUp = () => {
        if (!isResizing) return
        isResizing = false
        const width = image.style.width || node.attrs.width || '100%'
        const pos = getPos()
        if (typeof pos === 'number') {
          const transaction = editor.state.tr.setNodeMarkup(pos, undefined, {
            ...node.attrs,
            width,
          })
          editor.view.dispatch(transaction)
        }
        window.removeEventListener('mousemove', onMouseMove)
        window.removeEventListener('mouseup', onMouseUp)
      }

      image.addEventListener('click', () => {
        const pos = getPos()
        if (typeof pos === 'number') {
          const transaction = editor.state.tr.setSelection(NodeSelection.create(editor.state.doc, pos))
          editor.view.dispatch(transaction)
          editor.view.focus()
        }
        setSelected(true)
      })

      wrapper.addEventListener('mousemove', (event) => {
        if (!isSelected) {
          wrapper.style.cursor = 'default'
          return
        }
        wrapper.style.cursor = isResizeZone(event) ? 'nwse-resize' : 'default'
      })

      wrapper.addEventListener('mousedown', (event) => {
        if (!isSelected || !isResizeZone(event)) return
        event.preventDefault()
        isResizing = true
        startX = event.clientX
        startWidth = image.getBoundingClientRect().width
        window.addEventListener('mousemove', onMouseMove)
        window.addEventListener('mouseup', onMouseUp)
      })

      const onDocumentPointerDown = (event) => {
        if (!wrapper.contains(event.target)) {
          setSelected(false)
          wrapper.style.cursor = 'default'
        }
      }

      document.addEventListener('pointerdown', onDocumentPointerDown)

      wrapper.appendChild(image)

      return {
        dom: wrapper,
        update: (updatedNode) => {
          if (updatedNode.type.name !== node.type.name) return false
          image.src = updatedNode.attrs.src
          image.alt = updatedNode.attrs.alt || ''
          image.title = updatedNode.attrs.title || ''
          image.style.width = updatedNode.attrs.width || 'auto'
          applyWrapperAlign(updatedNode.attrs.align || 'left')
          return true
        },
        destroy: () => {
          document.removeEventListener('pointerdown', onDocumentPointerDown)
          window.removeEventListener('mousemove', onMouseMove)
          window.removeEventListener('mouseup', onMouseUp)
        },
      }
    }
  },
})

// Local video block for uploaded files and direct URLs.
const LocalVideo = TiptapNode.create({
  name: 'localVideo',
  group: 'block',
  atom: true,
  selectable: true,
  draggable: true,
  addAttributes() {
    return {
      src: { default: null },
      controls: { default: true },
    }
  },
  parseHTML() {
    return [{ tag: 'video[data-local-video]' }]
  },
  renderHTML({ HTMLAttributes }) {
    return [
      'video',
      {
        ...HTMLAttributes,
        'data-local-video': 'true',
        controls: HTMLAttributes.controls,
        class: 'local-video-node',
      },
    ]
  },
  addCommands() {
    return {
      setLocalVideo:
        (options) =>
        ({ commands }) =>
          commands.insertContent({ type: this.name, attrs: options }),
    }
  },
})

const editor = useEditor({
  content: `
    <h1>Rich Text Editor Feature Showcase</h1>
    <p>This document is a live example of what your editor can handle. Try selecting any part and apply tools from the toolbar or right-click menu.</p>

    <h2>1. Text Styles and Inline Formatting</h2>
    <p>
      You can combine <strong>bold</strong>, <em>italic</em>, <u>underline</u>, and <s>strikethrough</s> in one sentence.
      Scientific notes also work: H<sub>2</sub>O and x<sup>2</sup> + y<sup>2</sup> = z<sup>2</sup>.
    </p>
    <p>
      Color and highlight examples:
      <span style="color:#c026d3;">accent text</span>,
      <span style="background-color:#fde68a;">highlighted phrase</span>,
      <span style="font-size:20px;">larger font size</span>,
      and <span style="font-family:Georgia;">custom font family</span>.
    </p>

    <h2>2. Headings, Lists, and Alignment</h2>
    <h3>Bullet List</h3>
    <ul>
      <li>Fast drafting with keyboard shortcuts</li>
      <li>Rich text formatting with semantic HTML output</li>
      <li>AI create/edit actions with selection awareness</li>
    </ul>
    <h3>Numbered List</h3>
    <ol>
      <li>Draft content</li>
      <li>Refine with AI</li>
      <li>Export as HTML, Markdown, or JSON</li>
    </ol>
    <p style="text-align:center;">This paragraph is center aligned.</p>
    <p style="text-align:right;">This paragraph is right aligned.</p>
    <hr />

    <h2>3. Blockquote and Code</h2>
    <blockquote>
      <p>Writing is thinking made visible.</p>
      <cite>anonymous</cite>
    </blockquote>
    <pre><code>function greet(name) {
  return 'Hello, ' + name + '!'
}

console.log(greet('Editor'))</code></pre>

    <h2>4. Links and Media</h2>
    <p>Reference link: <a href="https://tiptap.dev" target="_blank" rel="noopener noreferrer">Tiptap Documentation</a></p>
    <img src="https://images.unsplash.com/photo-1455390582262-044cdead277a?auto=format&fit=crop&w=1200&q=80" alt="Sample editorial visual" style="width:68%;" />
    <div data-embed="true" class="embed-block" src="https://www.wikipedia.org" title="Embedded content">
      <iframe src="https://www.wikipedia.org" title="Embedded content" loading="lazy" referrerpolicy="no-referrer-when-downgrade" allowfullscreen="true"></iframe>
    </div>

    <h2>5. Table Tools</h2>
    <table>
      <thead>
        <tr>
          <th>Feature</th>
          <th>Status</th>
          <th>Notes</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>AI Edit</td>
          <td>Ready</td>
          <td>Replaces selected content directly</td>
        </tr>
        <tr>
          <td>Code Format</td>
          <td>Ready</td>
          <td>Formats selected code and inserts a code block</td>
        </tr>
        <tr>
          <td>Export</td>
          <td>Ready</td>
          <td>Supports HTML, Markdown, JSON</td>
        </tr>
      </tbody>
    </table>
  `,
  extensions: [
    StarterKit.configure({
      heading: { levels: [1, 2, 3, 4, 5, 6] },
      blockquote: true,
      horizontalRule: true,
    }),
    TextStyle,
    FontFamily,
    FontSize,
    TextBackground,
    Color,
    Highlight.configure({ multicolor: true }),
    Underline,
    Subscript,
    Superscript,
    TextAlign.configure({ types: ['heading', 'paragraph'] }),
    Link.configure({
      openOnClick: true,
      autolink: true,
      defaultProtocol: 'https',
    }),
    ResizableImage.configure({ allowBase64: true }),
    Youtube.configure({
      controls: true,
      modestBranding: true,
      inline: false,
      width: 720,
      height: 405,
    }),
    LocalVideo,
    Embed,
    SignatureBlock,
    Table.configure({ resizable: true }),
    TableRow,
    CustomTableHeader,
    CustomTableCell,
  ],
  editorProps: {
    attributes: {
      class: 'focus:outline-none min-h-[560px] tiptap prose prose-slate max-w-none',
    },
  },
  onSelectionUpdate({ editor: nextEditor }) {
    syncToolbarState(nextEditor)
    aiCaretPos.value = nextEditor.state.selection.to
  },
  onUpdate({ editor: nextEditor }) {
    syncToolbarState(nextEditor)
    aiCaretPos.value = nextEditor.state.selection.to
  },
})

watch(editor, (value) => {
  editorRef.value = value
  if (value) {
    value.setEditable(!showAiPromptBar.value)
    syncToolbarState(value)
    aiCaretPos.value = value.state.selection.to
  }
})

watch(aiKey, (value) => {
  localStorage.setItem(GEMINI_KEY_STORAGE, value)
})

watch(darkMode, (enabled) => {
  document.documentElement.classList.toggle('dark', enabled)
  document.body.classList.toggle('dark', enabled)
  localStorage.setItem(DARK_MODE_STORAGE, enabled ? 'true' : 'false')
})

watch(win98Mode, (enabled) => {
  document.body.classList.toggle('win98', enabled)
  localStorage.setItem(WIN98_MODE_STORAGE, enabled ? 'true' : 'false')
})

watch(showAiPromptBar, (isOpen) => {
  editor.value?.setEditable(!isOpen)
})

const wordCount = computed(() => {
  const text = editor.value?.getText() || ''
  return text.trim() ? text.trim().split(/\s+/).length : 0
})

const charCount = computed(() => {
  const text = editor.value?.getText() || ''
  return text.length
})

const hasTextSelection = computed(() => {
  if (!editor.value) return false
  const { from, to } = editor.value.state.selection
  return from !== to
})

function syncToolbarState(nextEditor) {
  const attributes = nextEditor.getAttributes('textStyle')
  currentFontFamily.value = attributes.fontFamily || 'Manrope'
  currentFontSize.value = attributes.fontSize || '16px'
  currentTextColor.value = attributes.color || '#19315d'
  currentBackgroundColor.value = attributes.backgroundColor || '#ffffff'
}

function applyHeading(value) {
  if (!editor.value) return
  if (value === 'paragraph') {
    editor.value.chain().focus().setParagraph().run()
    return
  }
  editor.value.chain().focus().toggleHeading({ level: Number(value) }).run()
}

function applyFontFamily(value) {
  editor.value?.chain().focus().setFontFamily(value).run()
  currentFontFamily.value = value
}

function applyFontSize(value) {
  editor.value?.chain().focus().setFontSize(value).run()
  currentFontSize.value = value
}

function applyTextColor(value) {
  editor.value?.chain().focus().setColor(value).run()
  currentTextColor.value = value
}

function applyBackgroundColor(value) {
  editor.value?.chain().focus().setBackgroundColor(value).run()
  currentBackgroundColor.value = value
}

function setAlign(mode) {
  if (!editor.value) return
  if (editor.value.isActive('image')) {
    editor.value.chain().focus().setImageAlign(mode).run()
  } else if (editor.value.isActive('signatureBlock')) {
    editor.value.chain().focus().setSignatureAlign(mode).run()
  } else {
    editor.value.chain().focus().setTextAlign(mode).run()
  }
  showAlignMenu.value = false
}

function toggleMediaMenu(event) {
  const button = event.currentTarget
  if (!(button instanceof HTMLElement)) return

  if (showMediaMenu.value) {
    showMediaMenu.value = false
    return
  }

  const rect = button.getBoundingClientRect()
  const menuWidth = 176
  const viewportPadding = 12
  mediaMenuX.value = Math.min(rect.left, window.innerWidth - menuWidth - viewportPadding)
  mediaMenuY.value = rect.bottom + 8
  showMediaMenu.value = true
}

function toggleTableMenu(event) {
  const button = event.currentTarget
  if (!(button instanceof HTMLElement)) return

  if (showTableMenu.value) {
    showTableMenu.value = false
    return
  }

  const rect = button.getBoundingClientRect()
  const menuWidth = 176
  const viewportPadding = 12
  tableMenuX.value = Math.min(rect.left, window.innerWidth - menuWidth - viewportPadding)
  tableMenuY.value = rect.bottom + 8
  showTableMenu.value = true
}

function toggleAiMenu(event) {
  const button = event.currentTarget
  if (!(button instanceof HTMLElement)) return

  if (showAiMenu.value) {
    showAiMenu.value = false
    return
  }

  const rect = button.getBoundingClientRect()
  const menuWidth = 256
  const viewportPadding = 12
  aiMenuX.value = Math.min(rect.left, window.innerWidth - menuWidth - viewportPadding)
  aiMenuY.value = rect.bottom + 8
  showAiMenu.value = true
}

function toggleAlignMenu(event) {
  const button = event.currentTarget
  if (!(button instanceof HTMLElement)) return

  if (showAlignMenu.value) {
    showAlignMenu.value = false
    return
  }

  const rect = button.getBoundingClientRect()
  const menuWidth = 160
  const viewportPadding = 12
  alignMenuX.value = Math.min(rect.left, window.innerWidth - menuWidth - viewportPadding)
  alignMenuY.value = rect.bottom + 8
  showAlignMenu.value = true
}

function setLink() {
  const previousUrl = editor.value?.getAttributes('link').href || ''
  openInputModal({
    action: 'link',
    title: 'Insert Hyperlink',
    placeholder: 'https://example.com',
    value: previousUrl,
  })
}

function addImage() {
  openInputModal({
    action: 'image',
    title: 'Insert Image',
    placeholder: 'https://images.example.com/photo.jpg',
  })
}

function triggerImageUpload() {
  imageUploadInput.value?.click()
}

function onImageFileSelected(event) {
  const file = event.target.files?.[0]
  if (!file || !editor.value) return
  if (!file.type.startsWith('image/')) return

  const reader = new FileReader()
  reader.onload = () => {
    const src = typeof reader.result === 'string' ? reader.result : ''
    if (!src) return
    editor.value.chain().focus().setImage({ src, alt: file.name || 'Uploaded image' }).run()
    showMediaMenu.value = false
  }
  reader.readAsDataURL(file)
  event.target.value = ''
}

function addVideo() {
  openInputModal({
    action: 'video',
    title: 'Insert Video',
    placeholder: 'https://www.youtube.com/watch?v=...',
  })
}

function triggerVideoUpload() {
  videoUploadInput.value?.click()
}

function onVideoFileSelected(event) {
  const file = event.target.files?.[0]
  if (!file || !editor.value) return
  if (!file.type.startsWith('video/')) return

  const blobUrl = URL.createObjectURL(file)
  editor.value.chain().focus().setLocalVideo({ src: blobUrl, controls: true }).run()
  showMediaMenu.value = false
  event.target.value = ''
}

function addEmbed() {
  openInputModal({
    action: 'embed',
    title: 'Insert Embed',
    placeholder: 'https://player.example.com/embed/123',
  })
}

function addElectronicSignature() {
  showMediaMenu.value = false
  showSignatureModal.value = true
  signatureSignerName.value = ''
  signatureHasStroke.value = false
  signatureInserting.value = false
  nextTick(() => {
    initSignatureCanvas()
  })
}

function initSignatureCanvas() {
  const canvas = signatureCanvasRef.value
  if (!(canvas instanceof HTMLCanvasElement)) return

  const dpr = window.devicePixelRatio || 1
  const width = Math.max(320, Math.floor(canvas.clientWidth))
  const height = Math.max(160, Math.floor(canvas.clientHeight))

  canvas.width = Math.floor(width * dpr)
  canvas.height = Math.floor(height * dpr)

  const ctx = canvas.getContext('2d')
  if (!ctx) return
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.lineCap = 'round'
  ctx.lineJoin = 'round'
  ctx.lineWidth = 2.2
  ctx.strokeStyle = '#0f172a'
  ctx.clearRect(0, 0, width, height)
}

function getSignaturePoint(event) {
  const canvas = signatureCanvasRef.value
  if (!(canvas instanceof HTMLCanvasElement)) return { x: 0, y: 0 }
  const rect = canvas.getBoundingClientRect()
  return {
    x: event.clientX - rect.left,
    y: event.clientY - rect.top,
  }
}

function startSignatureStroke(event) {
  const canvas = signatureCanvasRef.value
  if (!(canvas instanceof HTMLCanvasElement)) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const point = getSignaturePoint(event)
  signatureIsDrawing.value = true
  signatureHasStroke.value = true
  ctx.beginPath()
  ctx.moveTo(point.x, point.y)
}

function drawSignatureStroke(event) {
  if (!signatureIsDrawing.value) return
  const canvas = signatureCanvasRef.value
  if (!(canvas instanceof HTMLCanvasElement)) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const point = getSignaturePoint(event)
  ctx.lineTo(point.x, point.y)
  ctx.stroke()
}

function stopSignatureStroke() {
  signatureIsDrawing.value = false
}

function clearSignaturePad() {
  initSignatureCanvas()
  signatureHasStroke.value = false
}

function closeSignatureModal() {
  showSignatureModal.value = false
  signatureIsDrawing.value = false
  signatureInserting.value = false
}

function exportTrimmedSignature(canvas) {
  const context = canvas.getContext('2d')
  if (!context) return canvas.toDataURL('image/png')

  const { width, height } = canvas
  const imageData = context.getImageData(0, 0, width, height)
  const pixels = imageData.data

  let minX = width
  let minY = height
  let maxX = -1
  let maxY = -1

  for (let y = 0; y < height; y += 1) {
    for (let x = 0; x < width; x += 1) {
      const offset = (y * width + x) * 4
      const red = pixels[offset]
      const green = pixels[offset + 1]
      const blue = pixels[offset + 2]
      const alpha = pixels[offset + 3]
      const isInk = alpha > 16 && (red < 245 || green < 245 || blue < 245)

      if (isInk) {
        if (x < minX) minX = x
        if (y < minY) minY = y
        if (x > maxX) maxX = x
        if (y > maxY) maxY = y
      }
    }
  }

  if (maxX < 0 || maxY < 0) return canvas.toDataURL('image/png')

  const pad = 2
  const cropX = Math.max(0, minX - pad)
  const cropY = Math.max(0, minY - pad)
  const cropWidth = Math.min(width - cropX, maxX - minX + 1 + pad * 2)
  const cropHeight = Math.min(height - cropY, maxY - minY + 1 + pad * 2)

  const outputCanvas = document.createElement('canvas')
  outputCanvas.width = cropWidth
  outputCanvas.height = cropHeight

  const outputContext = outputCanvas.getContext('2d')
  if (!outputContext) return canvas.toDataURL('image/png')

  outputContext.drawImage(canvas, cropX, cropY, cropWidth, cropHeight, 0, 0, cropWidth, cropHeight)
  return outputCanvas.toDataURL('image/png')
}

async function insertElectronicSignatureFromPad() {
  if (!editor.value) return
  const canvas = signatureCanvasRef.value
  if (!(canvas instanceof HTMLCanvasElement) || !signatureHasStroke.value) return
  if (signatureInserting.value) return

  signatureInserting.value = true

  try {
    const signerName = signatureSignerName.value.trim() || 'Signer'
    const signatureDataUrl = exportTrimmedSignature(canvas)

    editor.value
      .chain()
      .focus()
      .setSignatureBlock({
        src: signatureDataUrl,
        signer: signerName,
      })
      .run()
    closeSignatureModal()
  } finally {
    signatureInserting.value = false
  }
}

function insertTable() {
  editor.value?.chain().focus().insertTable({ rows: 3, cols: 3, withHeaderRow: true }).run()
}

function setTableCellBackgroundColor(color) {
  if (!editor.value) return
  editor.value.chain().focus().setCellAttribute('backgroundColor', color).run()
  currentTableCellColor.value = color
}

function clearTableCellBackgroundColor() {
  if (!editor.value) return
  editor.value.chain().focus().setCellAttribute('backgroundColor', null).run()
  currentTableCellColor.value = '#ffffff'
}

function guessCodeParser(code) {
  const source = code.trim()
  if (!source) return 'babel'
  if (/^\s*[\[{]/.test(source) && /[\]}]\s*$/.test(source)) return 'json'
  if (/^\s*</.test(source) && /<\/[a-zA-Z]/.test(source)) return 'html'
  if (/^\s*[@.#\w\-\s:,>+~\[\]=\"']+\{[\s\S]*\}\s*$/.test(source)) return 'css'
  return 'babel'
}

async function formatCodeSnippet(source, preferredParser = '') {
  const plugins = [babelPlugin, estreePlugin, htmlPlugin, postcssPlugin]
  const parserCandidates = preferredParser
    ? [preferredParser, 'babel-ts', 'babel', 'json', 'html', 'css']
    : [guessCodeParser(source), 'babel-ts', 'babel', 'json', 'html', 'css']

  const uniqueCandidates = Array.from(new Set(parserCandidates))

  for (const parser of uniqueCandidates) {
    try {
      return await prettier.format(source, {
        parser,
        plugins,
        semi: false,
        singleQuote: true,
      })
    } catch {
      // Try next parser until one succeeds.
    }
  }

  throw new Error('Unable to format this code selection.')
}

function insertFormattedCodeBlock(selection, codeText) {
  if (!editor.value) return

  if (editor.value.isActive('codeBlock')) {
    editor.value.chain().focus().insertContentAt({ from: selection.from, to: selection.to }, codeText).run()
    return
  }

  editor.value
    .chain()
    .focus()
    .insertContentAt(
      { from: selection.from, to: selection.to },
      {
        type: 'codeBlock',
        content: [{ type: 'text', text: codeText }],
      },
    )
    .run()
}

async function formatSelectedCode() {
  if (!editor.value || codeFormatLoading.value) return

  const selection = editor.value.state.selection
  if (selection.from === selection.to) {
    codeFormatError.value = 'Select code first to format.'
    return
  }

  const selectedText = editor.value.state.doc.textBetween(selection.from, selection.to, '\n')
  if (!selectedText.trim()) {
    codeFormatError.value = 'Selected content is empty.'
    return
  }

  codeFormatLoading.value = true
  codeFormatError.value = ''

  try {
    const preferredParser = editor.value.isActive('codeBlock') ? 'babel' : ''
    const formatted = await formatCodeSnippet(selectedText, preferredParser)
    const output = formatted.replace(/\n$/, '')
    insertFormattedCodeBlock(selection, output)
  } catch (error) {
    codeFormatError.value = error.message || 'Code format failed.'
  } finally {
    codeFormatLoading.value = false
  }
}

function closeMenus() {
  showAiMenu.value = false
  showAiKeyPanel.value = false
  showExportMenu.value = false
  showTableMenu.value = false
  showMediaMenu.value = false
  showAlignMenu.value = false
  showContextMenu.value = false
  showAiPromptBar.value = false
  showAiSlashMenu.value = false
  showFontSizeDropdown.value = false
}

function getCurrentLineText() {
  if (!editor.value) return ''
  const { selection, doc } = editor.value.state
  const { $from } = selection
  return doc.textBetween($from.start(), $from.end(), ' ').trim()
}

async function openAiPromptBar(options = {}) {
  if (!aiKey.value.trim()) {
    aiError.value = 'Please enter Gemini API key first.'
    showAiMenu.value = true
    showAiKeyPanel.value = true
    return
  }

  const mode = options.mode || 'custom'
  const prefillText = typeof options.prefillText === 'string' ? options.prefillText : getCurrentLineText()

  if (editor.value) {
    // Keep caret state fresh before computing caret coordinates.
    const activeSelection = editor.value.state.selection
    aiPromptSelectionRange.value = { from: activeSelection.from, to: activeSelection.to }

    const docSize = editor.value.state.doc.content.size
    const currentSelectionPos = activeSelection.to
    const preferredPos = editor.value.isFocused ? currentSelectionPos : aiCaretPos.value || currentSelectionPos
    const safePos = Math.max(1, Math.min(preferredPos, docSize))
    if (activeSelection.from !== activeSelection.to) {
      editor.value.chain().focus().run()
    } else {
      editor.value.chain().focus(safePos).run()
    }
    await nextTick()

    const coords = getCaretViewportCoords(editor.value.view, safePos)
    const host = editorArticleRef.value
    const viewportPadding = 12
    const offsetY = 8

    if (host instanceof HTMLElement) {
      const hostRect = host.getBoundingClientRect()
      const promptWidth = Math.max(320, Math.min(hostRect.width * 0.42, 520))
      const rawX = coords.left - hostRect.left
      aiPromptX.value = 0
      aiPromptX.value = Math.max(0, Math.min(rawX, hostRect.width - promptWidth))
      aiPromptY.value = Math.max(viewportPadding, coords.top - hostRect.top)
    } else {
      const barWidth = Math.min(680, window.innerWidth - 24)
      const barHeight = 56
      const preferredBelow = coords.bottom + offsetY
      const maxVisibleY = window.innerHeight - barHeight - viewportPadding
      const fallbackAbove = Math.max(viewportPadding, coords.top - barHeight - offsetY)

      aiPromptX.value = Math.min(
        Math.max(coords.left - barWidth / 2, viewportPadding),
        window.innerWidth - barWidth - viewportPadding,
      )
      aiPromptY.value = preferredBelow <= maxVisibleY ? preferredBelow : fallbackAbove
    }
  } else {
    aiPromptX.value = 24
    aiPromptY.value = 24
  }

  showAiPromptBar.value = true
  aiPromptError.value = ''
  aiPromptText.value = prefillText
  aiPromptMode.value = mode
  await nextTick()
  aiPromptInputRef.value?.focus()
  autoResizeAiPromptInput()
}

function getCaretViewportCoords(view, pos) {
  const domSelection = window.getSelection()
  if (domSelection && domSelection.rangeCount > 0) {
    const range = domSelection.getRangeAt(0)
    const anchorInEditor = view.dom.contains(range.startContainer)
    if (anchorInEditor) {
      const rect = range.getBoundingClientRect()
      if (rect && (rect.width || rect.height)) {
        return { left: rect.left, top: rect.top, bottom: rect.bottom }
      }

      const firstRect = range.getClientRects()[0]
      if (firstRect) {
        return { left: firstRect.left, top: firstRect.top, bottom: firstRect.bottom }
      }
    }
  }

  return view.coordsAtPos(pos)
}

function closeAiPromptBar() {
  showAiPromptBar.value = false
  aiPromptError.value = ''
  aiPromptText.value = ''
  aiPromptMode.value = 'custom'
  aiPromptSelectionRange.value = { from: 0, to: 0 }
  if (aiPromptInputRef.value instanceof HTMLTextAreaElement) {
    aiPromptInputRef.value.style.height = ''
  }
  closeAiResultPanel()
  editor.value?.chain().focus(aiCaretPos.value).run()
}

function handleAiPromptKeydown(event) {
  if (event.key !== 'Enter') return
  if (event.shiftKey) return
  event.preventDefault()
  submitAiPromptBar()
}

function getAiContextText(selection) {
  if (!editor.value) return ''
  const selectedText = editor.value.state.doc.textBetween(selection.from, selection.to, ' ').trim()
  if (selectedText) return selectedText

  const fullText = editor.value.getText().trim()
  if (!fullText) return ''
  if (fullText.length <= 2400) return fullText
  return `${fullText.slice(0, 1200)}\n...\n${fullText.slice(-1200)}`
}

function buildAiPrompt(userPrompt, contextText) {
  const activeMode = aiActions.find((item) => item.key === aiPromptMode.value)
  const modeInstruction = activeMode ? `${activeMode.instruction}\n\n` : ''
  const requestLine = userPrompt.trim() ? `User request: ${userPrompt.trim()}\n\n` : ''
  return `${modeInstruction}${requestLine}Context:\n${contextText}\n\nOutput requirement: return only a clean, insertion-ready HTML fragment suitable for a rich text editor.`
}

function insertAiHtmlAtSelection(selection, html) {
  if (!editor.value || !html) return
  const selectedText = editor.value.state.doc.textBetween(selection.from, selection.to, ' ').trim()
  if (selectedText) {
    editor.value.chain().focus().insertContentAt({ from: selection.from, to: selection.to }, html).run()
    return
  }
  editor.value.chain().focus(selection.to).insertContent(html).run()
}

function autoResizeAiPromptInput() {
  if (!(aiPromptInputRef.value instanceof HTMLTextAreaElement)) return
  aiPromptInputRef.value.style.height = '0px'
  aiPromptInputRef.value.style.height = `${Math.min(aiPromptInputRef.value.scrollHeight, 168)}px`
}

function closeAiResultPanel() {
  showAiResultPanel.value = false
  aiResultHtml.value = ''
  aiResultLoading.value = false
  aiResultError.value = ''
}

function openAiSlashMenu() {
  if (!editor.value) return
  const { from } = editor.value.state.selection
  const coords = editor.value.view.coordsAtPos(from)
  const menuWidth = 260
  const viewportPadding = 12
  aiSlashMenuX.value = Math.min(Math.max(coords.left - menuWidth / 2, viewportPadding), window.innerWidth - menuWidth - viewportPadding)
  aiSlashMenuY.value = coords.bottom + 8
  showAiSlashMenu.value = true
}

function escapeHtml(value) {
  return value
    .replaceAll('&', '&amp;')
    .replaceAll('<', '&lt;')
    .replaceAll('>', '&gt;')
    .replaceAll('"', '&quot;')
    .replaceAll("'", '&#39;')
}

function aiTextToHtml(aiText) {
  if (!aiText.trim()) return ''

  // If model already returned HTML, sanitize and use it directly.
  if (/<\/?(h[1-6]|p|ul|ol|li|blockquote|strong|em|u|a)\b/i.test(aiText)) {
    return DOMPurify.sanitize(aiText, { USE_PROFILES: { html: true } })
  }

  const lines = aiText.split('\n')
  const htmlParts = []
  let inUnorderedList = false
  let inOrderedList = false

  const closeLists = () => {
    if (inUnorderedList) {
      htmlParts.push('</ul>')
      inUnorderedList = false
    }
    if (inOrderedList) {
      htmlParts.push('</ol>')
      inOrderedList = false
    }
  }

  for (const rawLine of lines) {
    const line = rawLine.trim()

    if (!line) {
      closeLists()
      continue
    }

    const safe = escapeHtml(line)

    if (/^#{1,6}\s+/.test(line)) {
      closeLists()
      const level = line.match(/^#{1,6}/)[0].length
      const text = escapeHtml(line.replace(/^#{1,6}\s+/, ''))
      htmlParts.push(`<h${level}>${text}</h${level}>`)
      continue
    }

    if (/^[-*]\s+/.test(line)) {
      if (!inUnorderedList) {
        closeLists()
        htmlParts.push('<ul>')
        inUnorderedList = true
      }
      htmlParts.push(`<li>${escapeHtml(line.replace(/^[-*]\s+/, ''))}</li>`)
      continue
    }

    if (/^\d+[.)]\s+/.test(line)) {
      if (!inOrderedList) {
        closeLists()
        htmlParts.push('<ol>')
        inOrderedList = true
      }
      htmlParts.push(`<li>${escapeHtml(line.replace(/^\d+[.)]\s+/, ''))}</li>`)
      continue
    }

    if (/^>\s+/.test(line)) {
      closeLists()
      htmlParts.push(`<blockquote><p>${escapeHtml(line.replace(/^>\s+/, ''))}</p></blockquote>`)
      continue
    }

    closeLists()
    htmlParts.push(`<p>${safe}</p>`)
  }

  closeLists()
  return DOMPurify.sanitize(htmlParts.join(''), { USE_PROFILES: { html: true } })
}

function extractPlainTextFromHtml(html) {
  const container = document.createElement('div')
  container.innerHTML = html
  return (container.textContent || '').replace(/\s+/g, ' ').trim()
}

function splitLongParagraph(text, maxChars = 260) {
  const sentences = text
    .split(/(?<=[.!?])\s+/)
    .map((part) => part.trim())
    .filter(Boolean)

  if (sentences.length <= 1) return [text]

  const chunks = []
  let current = ''

  for (const sentence of sentences) {
    const candidate = current ? `${current} ${sentence}` : sentence
    if (candidate.length > maxChars && current) {
      chunks.push(current)
      current = sentence
    } else {
      current = candidate
    }
  }

  if (current) chunks.push(current)
  return chunks
}

function normalizeAiHtml(aiHtml) {
  const cleaned = DOMPurify.sanitize(aiHtml, { USE_PROFILES: { html: true } })
  const source = document.createElement('div')
  source.innerHTML = cleaned

  const output = document.createElement('div')
  const blocks = Array.from(source.childNodes)
  let hasH2 = false
  let hasH3 = false
  let paragraphCount = 0
  let sectionCount = 0
  let hasTakeaway = false

  const appendParagraph = (text) => {
    const trimmed = text.trim()
    if (!trimmed) return
    const parts = trimmed.length > 320 ? splitLongParagraph(trimmed) : [trimmed]
    for (const part of parts) {
      const p = document.createElement('p')
      p.textContent = part
      output.appendChild(p)
      paragraphCount += 1
    }
  }

  for (const node of blocks) {
    if (!(node instanceof Element)) {
      const content = node.textContent?.trim() || ''
      if (content) appendParagraph(content)
      continue
    }

    const tag = node.tagName.toLowerCase()

    if (tag === 'h2') {
      hasH2 = true
      output.appendChild(node.cloneNode(true))
      continue
    }

    if (tag === 'h3') {
      hasH3 = true
      sectionCount += 1
      output.appendChild(node.cloneNode(true))
      continue
    }

    if (tag === 'p') {
      appendParagraph(node.textContent || '')
      continue
    }

    if (tag === 'ul' || tag === 'ol') {
      const list = document.createElement(tag)
      const items = Array.from(node.querySelectorAll('li'))
      for (const item of items) {
        const li = document.createElement('li')
        li.textContent = (item.textContent || '').trim()
        if (li.textContent) list.appendChild(li)
      }
      if (list.childElementCount) output.appendChild(list)
      continue
    }

    if (tag === 'blockquote') {
      hasTakeaway = true
      output.appendChild(node.cloneNode(true))
      continue
    }

    const fallback = (node.textContent || '').trim()
    if (fallback) appendParagraph(fallback)
  }

  if (!hasH2) {
    const text = extractPlainTextFromHtml(output.innerHTML)
    const title = text ? text.split(/[.!?]/)[0].slice(0, 72).trim() : 'Generated Draft'
    const h2 = document.createElement('h2')
    h2.textContent = title || 'Generated Draft'
    output.insertBefore(h2, output.firstChild)
  }

  if (!hasH3 && paragraphCount >= 3) {
    const paragraphs = Array.from(output.querySelectorAll('p'))
    if (paragraphs.length >= 2) {
      const markerOne = document.createElement('h3')
      markerOne.textContent = 'Key Insights'
      paragraphs[1].before(markerOne)
      sectionCount += 1
    }
    if (paragraphs.length >= 4) {
      const markerTwo = document.createElement('h3')
      markerTwo.textContent = 'Practical Direction'
      paragraphs[3].before(markerTwo)
      sectionCount += 1
    }
  }

  if (!hasTakeaway) {
    const text = extractPlainTextFromHtml(output.innerHTML)
    if (text) {
      const quote = document.createElement('blockquote')
      const p = document.createElement('p')
      p.textContent = text.split(/[.!?]/)[0].trim() || 'Keep it clear, structured, and purposeful.'
      quote.appendChild(p)
      output.appendChild(quote)
    }
  }

  return DOMPurify.sanitize(output.innerHTML, { USE_PROFILES: { html: true } })
}

function insertAiResult(selection, selectedText, aiText) {
  if (!editor.value) return
  const html = aiTextToHtml(aiText)
  if (!html) return

  if (selectedText.trim()) {
    editor.value.chain().focus().insertContentAt({ from: selection.from, to: selection.to }, html).run()
  } else {
    editor.value.chain().focus().insertContent(html).run()
  }
}

function applyAiResultReplace() {
  if (!editor.value || !aiResultHtml.value) return
  const { from, to } = aiResultSelectionRange.value

  if (aiResultHadSelection.value) {
    editor.value.chain().focus().insertContentAt({ from, to }, aiResultHtml.value).run()
  } else {
    editor.value.chain().focus().insertContent(aiResultHtml.value).run()
  }
  closeAiResultPanel()
}

function applyAiResultInsertBelow() {
  if (!editor.value || !aiResultHtml.value) return
  const { to } = aiResultSelectionRange.value
  const insertion = `<p></p>${aiResultHtml.value}`
  editor.value.chain().focus().insertContentAt(to, insertion).run()
  closeAiResultPanel()
}

async function rerunAiResult() {
  if (!aiResultRequest.value) return
  await runAiAction(aiResultRequest.value)
}

async function generateAiHtml(prompt) {
  const finalPrompt = `${prompt}\n\nFormatting requirements:\n${AI_EDITORIAL_FORMAT_RULES}`
  const response = await fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${encodeURIComponent(aiKey.value.trim())}`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        contents: [{ parts: [{ text: finalPrompt }] }],
        generationConfig: {
          temperature: 0.7,
          maxOutputTokens: AI_MAX_OUTPUT_TOKENS,
        },
      }),
    },
  )

  if (!response.ok) {
    throw new Error(`Gemini request failed with status ${response.status}`)
  }

  const data = await response.json()
  const aiText =
    data?.candidates?.[0]?.content?.parts
      ?.map((part) => part.text || '')
      .join('\n')
      .trim() || ''

  if (!aiText) {
    throw new Error('No AI response text returned.')
  }

  return normalizeAiHtml(aiTextToHtml(aiText))
}

async function submitAiPromptBar() {
  if (!editor.value) return
  const isEditMode = aiPromptMode.value === 'edit'
  const needsText = aiPromptMode.value === 'custom'
  if (needsText && !aiPromptText.value.trim()) return
  if (isEditMode) {
    const { from, to } = aiPromptSelectionRange.value
    if (from === to) {
      aiPromptError.value = 'Please select text to edit.'
      return
    }
  }
  if (!aiKey.value.trim()) {
    aiPromptError.value = 'Please enter Gemini API key first.'
    return
  }

  aiPromptLoading.value = true
  aiPromptError.value = ''

  try {
    const selection = editor.value.state.selection
    const editRange = aiPromptSelectionRange.value
    const contextText = isEditMode
      ? editor.value.state.doc.textBetween(editRange.from, editRange.to, ' ').trim()
      : getAiContextText(selection)

    const prompt = isEditMode
      ? [
          'Task: Edit ONLY the selected text according to the user request.',
          'Return ONLY the revised HTML fragment for that selected range.',
          'Do not add extra intro/outro, headings, or unrelated sections.',
          'Preserve original meaning, language, and tone unless explicitly asked to change.',
          'Keep links and inline emphasis where appropriate.',
          `User request: ${aiPromptText.value.trim() || 'Improve clarity, grammar, and flow.'}`,
          '',
          'Selected text to edit:',
          contextText,
        ].join('\n')
      : buildAiPrompt(aiPromptText.value, contextText)

    const generatedHtml = await generateAiHtml(prompt)
    if (isEditMode) {
      editor.value.chain().focus().insertContentAt({ from: editRange.from, to: editRange.to }, generatedHtml).run()
    } else {
      insertAiHtmlAtSelection(selection, generatedHtml)
    }

    aiResultRequest.value = null
    showAiSlashMenu.value = false
    closeAiPromptBar()
  } catch (error) {
    aiPromptError.value = error.message || 'AI generate failed.'
  } finally {
    aiPromptLoading.value = false
  }
}

async function openContextMenu(event) {
  if (!editor.value) return
  event.preventDefault()

  contextOnTable.value =
    event.target instanceof Element &&
    Boolean(event.target.closest('table, th, td'))

  const viewportPadding = 12
  const estimatedMenuWidth = 288
  const maxX = window.innerWidth - estimatedMenuWidth - viewportPadding
  const maxY = window.innerHeight - viewportPadding

  contextMenuX.value = Math.max(viewportPadding, Math.min(event.clientX, maxX))
  contextMenuY.value = Math.max(viewportPadding, Math.min(event.clientY, maxY))
  showContextMenu.value = true

  await nextTick()
  if (!(contextMenuRef.value instanceof HTMLElement)) return

  const menuRect = contextMenuRef.value.getBoundingClientRect()
  const fittedX = Math.max(viewportPadding, Math.min(contextMenuX.value, window.innerWidth - menuRect.width - viewportPadding))
  const fittedY = Math.max(viewportPadding, Math.min(contextMenuY.value, window.innerHeight - menuRect.height - viewportPadding))

  const estimatedSubmenuWidth = 236
  const overflowRight = fittedX + menuRect.width + estimatedSubmenuWidth + viewportPadding > window.innerWidth
  const hasLeftSpace = fittedX - estimatedSubmenuWidth - viewportPadding >= 0
  contextSubmenuToLeft.value = overflowRight && hasLeftSpace

  contextMenuX.value = fittedX
  contextMenuY.value = fittedY
}

function runContextCommand(command) {
  if (!editor.value) return

  if (!canRunContextCommand(command)) return

  if (command === 'copySelection') {
    const selection = editor.value.state.selection
    const selectedText = editor.value.state.doc.textBetween(selection.from, selection.to, ' ')
    if (selectedText.trim()) {
      navigator.clipboard.writeText(selectedText)
    }
  }
  if (command === 'selectAll') editor.value.chain().focus().selectAll().run()
  if (command === 'link') setLink()
  if (command === 'signature') addElectronicSignature()
  if (command === 'formatCode') {
    formatSelectedCode()
  }

  if (command === 'bold') editor.value.chain().focus().toggleBold().run()
  if (command === 'italic') editor.value.chain().focus().toggleItalic().run()
  if (command === 'underline') editor.value.chain().focus().toggleUnderline().run()
  if (command === 'strike') editor.value.chain().focus().toggleStrike().run()
  if (command === 'paragraph') editor.value.chain().focus().setParagraph().run()
  if (command === 'h1') editor.value.chain().focus().toggleHeading({ level: 1 }).run()
  if (command === 'h2') editor.value.chain().focus().toggleHeading({ level: 2 }).run()
  if (command === 'bulletList') editor.value.chain().focus().toggleBulletList().run()
  if (command === 'orderedList') editor.value.chain().focus().toggleOrderedList().run()
  if (command === 'alignLeft') setAlign('left')
  if (command === 'alignCenter') setAlign('center')
  if (command === 'alignRight') setAlign('right')
  if (command === 'addColumnBefore') editor.value.chain().focus().addColumnBefore().run()
  if (command === 'addColumnAfter') editor.value.chain().focus().addColumnAfter().run()
  if (command === 'deleteColumn') editor.value.chain().focus().deleteColumn().run()
  if (command === 'addRowBefore') editor.value.chain().focus().addRowBefore().run()
  if (command === 'addRowAfter') editor.value.chain().focus().addRowAfter().run()
  if (command === 'deleteRow') editor.value.chain().focus().deleteRow().run()
  if (command === 'deleteTable') editor.value.chain().focus().deleteTable().run()
  if (command === 'mergeCells') editor.value.chain().focus().mergeCells().run()
  if (command === 'splitCell') editor.value.chain().focus().splitCell().run()
  if (command === 'undo') editor.value.chain().focus().undo().run()
  if (command === 'redo') editor.value.chain().focus().redo().run()

  showContextMenu.value = false
}

function canRunContextCommand(command) {
  if (!editor.value) return false
  const selection = editor.value.state.selection
  const hasSelectedText = selection.from !== selection.to

  if (command === 'copySelection') return hasSelectedText
  if (command === 'formatCode') return hasSelectedText
  if (command === 'undo') return editor.value.can().chain().focus().undo().run()
  if (command === 'redo') return editor.value.can().chain().focus().redo().run()
  if (command === 'addColumnBefore') return editor.value.can().chain().focus().addColumnBefore().run()
  if (command === 'addColumnAfter') return editor.value.can().chain().focus().addColumnAfter().run()
  if (command === 'deleteColumn') return editor.value.can().chain().focus().deleteColumn().run()
  if (command === 'addRowBefore') return editor.value.can().chain().focus().addRowBefore().run()
  if (command === 'addRowAfter') return editor.value.can().chain().focus().addRowAfter().run()
  if (command === 'deleteRow') return editor.value.can().chain().focus().deleteRow().run()
  if (command === 'deleteTable') return editor.value.can().chain().focus().deleteTable().run()
  if (command === 'mergeCells') return editor.value.can().chain().focus().mergeCells().run()
  if (command === 'splitCell') return editor.value.can().chain().focus().splitCell().run()

  return true
}

function isContextCommandActive(command) {
  if (!editor.value) return false

  if (command === 'bold') return editor.value.isActive('bold')
  if (command === 'italic') return editor.value.isActive('italic')
  if (command === 'underline') return editor.value.isActive('underline')
  if (command === 'strike') return editor.value.isActive('strike')
  if (command === 'h1') return editor.value.isActive('heading', { level: 1 })
  if (command === 'h2') return editor.value.isActive('heading', { level: 2 })
  if (command === 'paragraph') return editor.value.isActive('paragraph')
  if (command === 'bulletList') return editor.value.isActive('bulletList')
  if (command === 'orderedList') return editor.value.isActive('orderedList')

  return false
}

function getShortcutLabel(command) {
  if (command === 'copySelection') return 'Ctrl+C'
  if (command === 'bold') return 'Ctrl+B'
  if (command === 'italic') return 'Ctrl+I'
  if (command === 'underline') return 'Ctrl+U'
  if (command === 'strike') return 'Ctrl+Shift+X'
  if (command === 'undo') return 'Ctrl+Z'
  if (command === 'redo') return 'Ctrl+Y'
  if (command === 'link') return 'Ctrl+K'
  if (command === 'selectAll') return 'Ctrl+A'
  if (command === 'formatCode') return 'Ctrl+Shift+F'
  return ''
}

function openInputModal({ action, title, placeholder, value = '' }) {
  inputModalAction.value = action
  inputModalTitle.value = title
  inputModalPlaceholder.value = placeholder
  inputModalValue.value = value
  showInputModal.value = true
}

function normalizeUrl(value) {
  const raw = value.trim()
  if (!raw) return ''
  if (/^(https?:|data:|blob:|\/\/)/i.test(raw)) return raw
  return `https://${raw}`
}

function escapeHtmlAttr(value) {
  return value
    .replaceAll('&', '&amp;')
    .replaceAll('"', '&quot;')
    .replaceAll('<', '&lt;')
    .replaceAll('>', '&gt;')
}

function closeInputModal() {
  showInputModal.value = false
  inputModalAction.value = ''
  inputModalValue.value = ''
}

function submitInputModal() {
  if (!editor.value) return
  const rawValue = inputModalValue.value.trim()
  if (!rawValue) return

  const value = normalizeUrl(rawValue)
  if (!value) return

  if (inputModalAction.value === 'link') {
    editor.value.chain().focus().extendMarkRange('link').setLink({ href: value }).run()
  }
  if (inputModalAction.value === 'image') {
    const inserted = editor.value.chain().focus().setImage({ src: value, alt: 'Inserted image' }).run()
    if (!inserted) {
      const safeSrc = escapeHtmlAttr(value)
      editor.value.chain().focus().insertContent(`<img src="${safeSrc}" alt="Inserted image" />`).run()
    }
  }
  if (inputModalAction.value === 'video') {
    editor.value.chain().focus().setYoutubeVideo({ src: value }).run()
  }
  if (inputModalAction.value === 'embed') {
    editor.value.chain().focus().setEmbed({ src: value, title: 'Embedded content' }).run()
  }

  showMediaMenu.value = false
  closeInputModal()
}

function handleDocumentPointerDown(event) {
  const target = event.target
  if (!uiRoot.value || !(target instanceof globalThis.Node)) return

  if (target instanceof Element) {
    if (showAiPromptBar.value && !target.closest('[data-ai-prompt-bar]')) {
      closeAiPromptBar()
    }
    if (showTableMenu.value && !target.closest('[data-table-menu]')) {
      showTableMenu.value = false
    }
    if (showMediaMenu.value && !target.closest('[data-media-menu]')) {
      showMediaMenu.value = false
    }
    if (showAiMenu.value && !target.closest('[data-ai-menu]')) {
      showAiMenu.value = false
    }
    if (showAlignMenu.value && !target.closest('[data-align-menu]')) {
      showAlignMenu.value = false
    }
    if (showExportMenu.value && !target.closest('[data-export-menu]')) {
      showExportMenu.value = false
    }
    if (showContextMenu.value && !target.closest('[data-context-menu]')) {
      showContextMenu.value = false
    }
    if (showAiSlashMenu.value && !target.closest('[data-ai-slash-menu]')) {
      showAiSlashMenu.value = false
    }
  }

  if (!uiRoot.value.contains(target)) {
    closeMenus()
  }
}

function isTypingElement(element) {
  if (!(element instanceof HTMLElement)) return false
  return element.isContentEditable || ['INPUT', 'TEXTAREA', 'SELECT'].includes(element.tagName)
}

function isCursorOnEmptyLine() {
  if (!editor.value) return false
  const selection = editor.value.state.selection
  if (!selection.empty) return false

  const { $from } = selection
  const lineFrom = $from.start()
  const lineTo = $from.end()
  const lineText = editor.value.state.doc.textBetween(lineFrom, lineTo, ' ').trim()

  return !lineText
}

function handleGlobalKeydown(event) {
  const isMod = event.ctrlKey || event.metaKey

  const target = event.target
  const isTextInput =
    target instanceof HTMLElement &&
    ['INPUT', 'TEXTAREA', 'SELECT'].includes(target.tagName)

  if (isMod && (event.key === '/' || event.key === '?') && !isTextInput) {
    if (!isCursorOnEmptyLine()) {
      return
    }
    event.preventDefault()
    openAiPromptBar()
    return
  }

  if (!isMod && event.key === '/' && editor.value?.isFocused && !isTextInput) {
    event.preventDefault()
    openAiSlashMenu()
    return
  }

  if (event.key === 'Escape') {
    closeMenus()
    showExportPreview.value = false
    closeInputModal()
    closeAiPromptBar()
    closeAiResultPanel()
    return
  }

  if (!editor.value || !isMod || isTypingElement(event.target)) return

  const key = event.key.toLowerCase()
  if (key === 'b') {
    event.preventDefault()
    editor.value.chain().focus().toggleBold().run()
  }
  if (key === 'i') {
    event.preventDefault()
    editor.value.chain().focus().toggleItalic().run()
  }
  if (key === 'u') {
    event.preventDefault()
    editor.value.chain().focus().toggleUnderline().run()
  }
  if (key === 'x' && event.shiftKey) {
    event.preventDefault()
    editor.value.chain().focus().toggleStrike().run()
  }
  if (key === 'z' && event.shiftKey) {
    event.preventDefault()
    editor.value.chain().focus().redo().run()
  }
  if (key === 'z' && !event.shiftKey) {
    event.preventDefault()
    editor.value.chain().focus().undo().run()
  }
  if (key === 'y') {
    event.preventDefault()
    editor.value.chain().focus().redo().run()
  }
  if (key === 'k') {
    event.preventDefault()
    setLink()
  }
  if (key === 'f' && event.shiftKey) {
    event.preventDefault()
    formatSelectedCode()
  }
}

function handleGlobalWheel(event) {
  if (!showContextMenu.value) return
  const target = event.target
  if (target instanceof Element && target.closest('[data-context-menu]')) {
    return
  }
  showContextMenu.value = false
}

function buildExportPayload(type) {
  if (!editor.value) return
  const cleanHtml = DOMPurify.sanitize(editor.value.getHTML(), {
    USE_PROFILES: { html: true },
  })

  if (type === 'html') {
    return {
      label: 'HTML',
      fileName: 'editorial-export.html',
      mimeType: 'text/html;charset=utf-8',
      content: cleanHtml,
    }
  }

  if (type === 'md') {
    return {
      label: 'Markdown',
      fileName: 'editorial-export.md',
      mimeType: 'text/markdown;charset=utf-8',
      content: turndownService.turndown(cleanHtml),
    }
  }

  return {
    label: 'JSON (block)',
    fileName: 'editorial-export.json',
    mimeType: 'application/json;charset=utf-8',
    content: JSON.stringify(editor.value.getJSON(), null, 2),
  }
}

function previewExport(type) {
  const payload = buildExportPayload(type)
  if (!payload) return
  exportType.value = payload.label
  exportPreview.value = payload.content
  showExportPreview.value = true
}

function downloadExport(type) {
  const payload = buildExportPayload(type)
  if (!payload) return

  const blob = new Blob([payload.content], { type: payload.mimeType })
  const url = URL.createObjectURL(blob)
  const anchor = document.createElement('a')
  anchor.href = url
  anchor.download = payload.fileName
  document.body.appendChild(anchor)
  anchor.click()
  document.body.removeChild(anchor)
  URL.revokeObjectURL(url)
  showExportMenu.value = false
}

async function runAiAction(action) {
  if (!editor.value) return
  if (!aiKey.value.trim()) {
    aiError.value = 'Please enter Gemini API key first.'
    showAiKeyPanel.value = true
    return
  }

  if (action?.key === 'edit' && !hasTextSelection.value) {
    aiError.value = 'Please select text to edit.'
    aiPromptError.value = 'Please select text to edit.'
    return
  }

  aiError.value = ''
  aiPromptError.value = ''

  try {
    const promptMode = action?.key === 'edit' ? 'edit' : 'create'
    await openAiPromptBar({ mode: promptMode, prefillText: '' })
    showAiMenu.value = false
    showAiSlashMenu.value = false
  } catch (error) {
    aiError.value = error.message || 'AI action failed.'
    aiPromptError.value = error.message || 'AI action failed.'
  }
}

function copyExportText() {
  navigator.clipboard.writeText(exportPreview.value)
}

onMounted(() => {
  document.documentElement.classList.toggle('dark', darkMode.value)
  document.body.classList.toggle('dark', darkMode.value)
  document.body.classList.toggle('win98', win98Mode.value)
  document.addEventListener('pointerdown', handleDocumentPointerDown)
  window.addEventListener('keydown', handleGlobalKeydown)
  window.addEventListener('wheel', handleGlobalWheel, { passive: true })
})

onBeforeUnmount(() => {
  document.removeEventListener('pointerdown', handleDocumentPointerDown)
  window.removeEventListener('keydown', handleGlobalKeydown)
  window.removeEventListener('wheel', handleGlobalWheel)
  document.body.classList.remove('dark')
  document.body.classList.remove('win98')
  editor.value?.destroy()
})
</script>

<template>
  <div
    ref="uiRoot"
    class="editor-root min-h-screen bg-slate-200 px-4 py-4 md:px-8 md:py-10 font-body selection:bg-primaryContainer selection:text-onPrimaryContainer"
    :class="{ 'theme-win98': win98Mode }"
  >
    <div class="editor-shell mx-auto w-full max-w-5xl overflow-hidden rounded-2xl border border-outlineVariant/30 bg-surfaceContainerLowest shadow-editorial">
      <header class="editor-header border-b border-outlineVariant/20 bg-surfaceContainerLow px-3 py-2 md:px-4">
        <div class="mb-2 flex items-center justify-between gap-3">
          <div class="text-[11px] font-semibold uppercase tracking-widest text-onSurfaceVariant/80">Toolbar</div>
          <div class="flex items-center gap-2">
            <button
              class="top-btn flex items-center justify-center"
              :title="darkMode ? 'Switch to light mode' : 'Switch to dark mode'"
              :aria-label="darkMode ? 'Switch to light mode' : 'Switch to dark mode'"
              @click="darkMode = !darkMode"
            >
              <span class="material-symbols-outlined icon-md">{{ darkMode ? 'light_mode' : 'dark_mode' }}</span>
            </button>
            <button
              class="top-btn flex items-center justify-center"
              :title="win98Mode ? 'Disable Win98 theme' : 'Enable Win98 theme'"
              :aria-label="win98Mode ? 'Disable Win98 theme' : 'Enable Win98 theme'"
              @click="win98Mode = !win98Mode"
            >
              <span class="material-symbols-outlined icon-md">{{ win98Mode ? 'desktop_windows' : 'computer' }}</span>
            </button>
            <div class="relative" data-export-menu>
              <button
                class="top-btn flex items-center justify-center"
                title="Export"
                aria-label="Export"
                @click="showExportMenu = !showExportMenu"
              >
                <span class="material-symbols-outlined icon-md">file_upload</span>
              </button>
              <div v-if="showExportMenu" class="popover export-popover w-52">
                <button class="menu-item" @click="previewExport('html')">Preview Clean HTML</button>
                <button class="menu-item" @click="previewExport('md')">Preview Markdown</button>
                <button class="menu-item" @click="previewExport('json')">Preview JSON (block)</button>
                <button class="menu-item border-t border-outlineVariant/20" @click="downloadExport('html')">Download .html</button>
                <button class="menu-item" @click="downloadExport('md')">Download .md</button>
                <button class="menu-item" @click="downloadExport('json')">Download .json</button>
              </div>
            </div>
            <button class="share-btn">Share</button>
          </div>
        </div>

        <div class="toolbar-lane custom-scrollbar">
          <div class="toolbar-shell">
            <div class="toolbar-segment">
              <button class="icon-btn" title="Undo" :disabled="!editor?.can().chain().focus().undo().run()" @click="editor?.chain().focus().undo().run()"><span class="material-symbols-outlined icon-md">undo</span></button>
              <button class="icon-btn" title="Redo" :disabled="!editor?.can().chain().focus().redo().run()" @click="editor?.chain().focus().redo().run()"><span class="material-symbols-outlined icon-md">redo</span></button>
              <button class="icon-btn" :class="{ active: editor?.isActive('bold') }" title="Bold" @click="editor?.chain().focus().toggleBold().run()"><span class="material-symbols-outlined icon-md">format_bold</span></button>
              <button class="icon-btn" :class="{ active: editor?.isActive('italic') }" title="Italic" @click="editor?.chain().focus().toggleItalic().run()"><span class="material-symbols-outlined icon-md">format_italic</span></button>
              <button class="icon-btn" :class="{ active: editor?.isActive('underline') }" title="Underline" @click="editor?.chain().focus().toggleUnderline().run()"><span class="material-symbols-outlined icon-md">format_underlined</span></button>
              <button class="icon-btn" :class="{ active: editor?.isActive('strike') }" title="Strikethrough" @click="editor?.chain().focus().toggleStrike().run()"><span class="material-symbols-outlined icon-md">strikethrough_s</span></button>
              <button class="icon-btn" :class="{ active: editor?.isActive('subscript') }" title="Subscript" @click="editor?.chain().focus().toggleSubscript().run()"><span class="material-symbols-outlined icon-md">subscript</span></button>
              <button class="icon-btn" :class="{ active: editor?.isActive('superscript') }" title="Superscript" @click="editor?.chain().focus().toggleSuperscript().run()"><span class="material-symbols-outlined icon-md">superscript</span></button>
            </div>

            <div class="toolbar-segment">
              <select class="toolbar-select" :value="currentFontFamily" @change="applyFontFamily($event.target.value)">
                <option v-for="family in FONT_FAMILIES" :key="family" :value="family">{{ family }}</option>
              </select>
              <select class="toolbar-select font-size-select" :value="currentFontSize" @change="applyFontSize($event.target.value)">
                <option v-for="size in FONT_SIZES" :key="size.value" :value="size.value">{{ size.label }}</option>
              </select>
              <select class="toolbar-select" @change="applyHeading($event.target.value)">
                <option value="paragraph">Paragraph</option>
                <option v-for="level in [1, 2, 3, 4, 5, 6]" :key="level" :value="level">H{{ level }}</option>
              </select>
              <label class="color-control" title="Text color">
                <span class="color-icon-text">A</span>
                <span class="color-indicator" :style="{ backgroundColor: currentTextColor }"></span>
                <input class="color-input" type="color" :value="currentTextColor" @input="applyTextColor($event.target.value)" />
              </label>
              <label class="color-control" title="Background color">
                <span class="material-symbols-outlined color-icon-fill">format_color_fill</span>
                <span class="color-indicator" :style="{ backgroundColor: currentBackgroundColor }"></span>
                <input class="color-input" type="color" :value="currentBackgroundColor" @input="applyBackgroundColor($event.target.value)" />
              </label>
            </div>

            <div class="toolbar-segment">
              <button class="icon-btn" :class="{ active: editor?.isActive('bulletList') }" title="Bullet list" @click="editor?.chain().focus().toggleBulletList().run()"><span class="material-symbols-outlined icon-md">format_list_bulleted</span></button>
              <button class="icon-btn" :class="{ active: editor?.isActive('orderedList') }" title="Numbered list" @click="editor?.chain().focus().toggleOrderedList().run()"><span class="material-symbols-outlined icon-md">format_list_numbered</span></button>
              <button class="icon-btn" :class="{ active: editor?.isActive('blockquote') }" title="Blockquote" @click="editor?.chain().focus().toggleBlockquote().run()"><span class="material-symbols-outlined icon-md">format_quote</span></button>
              <button class="icon-btn" :disabled="codeFormatLoading || !hasTextSelection" title="Format code" @click="formatSelectedCode"><span class="material-symbols-outlined icon-md">code_blocks</span></button>
              <button class="icon-btn" title="Horizontal rule" @click="editor?.chain().focus().setHorizontalRule().run()"><span class="material-symbols-outlined icon-md">horizontal_rule</span></button>
              <div class="relative">
                <button data-align-menu class="icon-btn" :title="`Align: ${currentAlignMode}`" @click="toggleAlignMenu($event)">
                  <span class="material-symbols-outlined icon-md">{{ currentAlignIcon }}</span>
                </button>
                <div
                  v-if="showAlignMenu"
                  data-align-menu
                  class="align-popover"
                  :style="{ left: `${alignMenuX}px`, top: `${alignMenuY}px` }"
                >
                  <button class="menu-item" @click="setAlign('left')">Align Left</button>
                  <button class="menu-item" @click="setAlign('center')">Align Center</button>
                  <button class="menu-item" @click="setAlign('right')">Align Right</button>
                  <button class="menu-item" @click="setAlign('justify')">Justify</button>
                </div>
              </div>
            </div>

            <div class="toolbar-segment">
              <button class="icon-btn" title="Link" @click="setLink"><span class="material-symbols-outlined icon-md">link</span></button>
              <div class="relative">
                <button data-media-menu class="icon-btn" title="Media" @click="toggleMediaMenu($event)"><span class="material-symbols-outlined icon-md">image</span></button>
                <div
                  v-if="showMediaMenu"
                  data-media-menu
                  class="media-popover"
                  :style="{ left: `${mediaMenuX}px`, top: `${mediaMenuY}px` }"
                >
                  <button class="menu-item" @click="triggerImageUpload">Upload Image</button>
                  <button class="menu-item" @click="triggerVideoUpload">Upload Video</button>
                  <button class="menu-item" @click="addImage">Insert Image</button>
                  <button class="menu-item" @click="addVideo">Insert Video</button>
                  <button class="menu-item" @click="setLink">Insert Hyperlink</button>
                  <button class="menu-item" @click="addEmbed">Insert Embed</button>
                  <button class="menu-item" @click="addElectronicSignature">Electronic Signature</button>
                </div>
              </div>
              <div class="relative">
                <button data-table-menu class="icon-btn" title="Table" @click="toggleTableMenu($event)"><span class="material-symbols-outlined icon-md">table_chart</span></button>
                <div
                  v-if="showTableMenu"
                  data-table-menu
                  class="table-popover"
                  :style="{ left: `${tableMenuX}px`, top: `${tableMenuY}px` }"
                >
                  <button class="menu-item" @click="insertTable">Insert Table</button>
                  <p class="px-3 py-2 text-[10px] text-onSurfaceVariant/80">Right-click inside a table to edit rows and columns.</p>
                </div>
              </div>
            </div>

            <div class="toolbar-segment">
              <div class="relative">
                <button data-ai-menu class="ai-btn" @click="toggleAiMenu($event)">
                  <span class="material-symbols-outlined mr-1 text-sm" style="font-variation-settings: 'FILL' 1">auto_awesome</span>
                  AI Assist
                </button>
                <div
                  v-if="showAiMenu"
                  data-ai-menu
                  class="ai-popover"
                  :style="{ left: `${aiMenuX}px`, top: `${aiMenuY}px` }"
                >
                  <button v-for="action in aiActions" :key="action.key" class="menu-item" :disabled="aiLoading || (action.key === 'edit' && !hasTextSelection)" @click="runAiAction(action)">
                    {{ action.label }}
                  </button>
                  <button class="menu-item border-t border-outlineVariant/20" @click="showAiKeyPanel = !showAiKeyPanel">Gemini API Key</button>
                  <div v-if="showAiKeyPanel" class="space-y-2 px-3 pb-3 pt-2">
                    <input v-model="aiKey" class="w-full rounded-md border border-outlineVariant/30 bg-white px-2 py-1 text-xs" placeholder="Paste API key" />
                    <p class="text-[10px] text-onSurfaceVariant/80">Stored in localStorage.</p>
                  </div>
                  <p v-if="aiLoading" class="px-3 pb-2 text-xs text-primary">Processing with Gemini...</p>
                  <p v-if="aiError" class="px-3 pb-2 text-xs text-red-600">{{ aiError }}</p>
                </div>
              </div>
            </div>
          </div>
        </div>
        <p v-if="codeFormatError" class="mt-1 px-1 text-[11px] text-red-600">{{ codeFormatError }}</p>
      </header>

      <main class="editor-main custom-scrollbar flex-1 overflow-y-auto bg-white px-5 py-8 md:px-16 md:py-14">
        <article ref="editorArticleRef" class="relative mx-auto max-w-3xl pb-20">
          <div
            v-if="showAiPromptBar"
            data-ai-prompt-bar
            class="ai-prompt-floating"
            :style="{ left: `${aiPromptX}px`, top: `${aiPromptY}px` }"
          >
            <div class="ai-command-bar">
              <span class="ai-command-icon-wrap" aria-hidden="true">
                <span class="material-symbols-outlined ai-command-icon">auto_awesome</span>
              </span>
              <textarea
                ref="aiPromptInputRef"
                v-model="aiPromptText"
                class="ai-command-input"
                rows="1"
                placeholder="Ask AI to continue or refine this line"
                @input="autoResizeAiPromptInput"
                @keydown="handleAiPromptKeydown"
              />
              <button
                class="ai-command-send"
                :disabled="aiPromptLoading || !aiPromptText.trim()"
                @click="submitAiPromptBar"
              >
                <span v-if="!aiPromptLoading" class="material-symbols-outlined">north_east</span>
                <span v-else class="material-symbols-outlined ai-spin">progress_activity</span>
              </button>
            </div>
            <div class="mt-2">
              <p v-if="aiPromptError" class="text-xs text-red-600">{{ aiPromptError }}</p>
            </div>
          </div>

          <div class="mb-6 flex items-center gap-3 text-xs text-onSurfaceVariant/70">
            <div class="h-5 w-5 rounded-full bg-slate-300"></div>
            <span>Marcus Aurelius</span>
            <span>•</span>
            <span>October 24, 2023</span>
            <span>•</span>
            <span>5 min read</span>
          </div>
          <div :class="{ 'editor-ai-locked': showAiPromptBar }" @contextmenu="openContextMenu">
            <EditorContent :editor="editorRef" />
          </div>

          <div
            v-if="showAiSlashMenu"
            data-ai-slash-menu
            class="ai-slash-menu"
            :style="{ left: `${aiSlashMenuX}px`, top: `${aiSlashMenuY}px` }"
          >
            <button class="menu-item" @click="runAiAction(aiActions.find((item) => item.key === 'create'))">Create content</button>
            <button class="menu-item" :disabled="!hasTextSelection" @click="runAiAction(aiActions.find((item) => item.key === 'edit'))">Edit writing</button>
          </div>
        </article>
      </main>

      <footer class="editor-footer flex items-center justify-between border-t border-outlineVariant/20 bg-surfaceContainerLow px-4 py-2">
        <div class="flex items-center gap-3 text-[10px] uppercase tracking-widest text-onSurfaceVariant/70">
          <span>v2.4.0 richtext-editor</span>
          <span>Words: {{ wordCount }}</span>
          <span>Chars: {{ charCount }}</span>
        </div>
        <div class="flex items-center gap-3 text-[10px] uppercase tracking-widest text-onSurfaceVariant/70">
          <span>Markdown</span>
          <span>LaTeX</span>
          <div class="flex items-center gap-1 rounded-full border border-green-100 bg-green-50 px-2 py-0.5 text-[9px] font-bold text-green-700">
            <span class="h-1 w-1 rounded-full bg-green-500"></span>
            <span>Saved</span>
          </div>
        </div>
      </footer>
    </div>

    <input
      ref="imageUploadInput"
      type="file"
      accept="image/*"
      class="hidden"
      @change="onImageFileSelected"
    />

    <input
      ref="videoUploadInput"
      type="file"
      accept="video/*"
      class="hidden"
      @change="onVideoFileSelected"
    />

    <div v-if="showExportPreview" class="fixed inset-0 z-50 flex items-center justify-center bg-slate-900/40 p-4">
      <div class="glass w-full max-w-3xl rounded-2xl bg-white/90 p-4 shadow-editorial">
        <div class="mb-2 flex items-center justify-between">
          <h3 class="font-headline text-lg text-onBackground">{{ exportType }} Export</h3>
          <div class="flex items-center gap-2">
            <button class="top-btn" @click="copyExportText">Copy</button>
            <button class="top-btn" @click="showExportPreview = false">Close</button>
          </div>
        </div>
        <textarea class="h-80 w-full rounded-lg border border-outlineVariant/30 bg-white p-3 font-mono text-xs" :value="exportPreview" readonly />
      </div>
    </div>

    <div
      v-if="showContextMenu"
      ref="contextMenuRef"
      data-context-menu
      class="context-menu glass fixed z-[70]"
      :class="{ 'context-menu-table': contextOnTable, 'context-menu-submenu-left': contextSubmenuToLeft }"
      :style="{ left: `${contextMenuX}px`, top: `${contextMenuY}px` }"
    >
      <!-- Table-specific menu -->
      <template v-if="contextOnTable">
        <div class="context-title">Table Tools</div>

        <div class="context-item context-hover-group" role="button" tabindex="0">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">table_rows</span>
            <span>Row</span>
          </span>
          <span class="material-symbols-outlined context-icon">chevron_right</span>
          <div class="context-side-submenu glass">
            <button class="context-item" :disabled="!canRunContextCommand('addRowBefore')" @click="runContextCommand('addRowBefore')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">north</span>
                <span>Add row before</span>
              </span>
            </button>
            <button class="context-item" :disabled="!canRunContextCommand('addRowAfter')" @click="runContextCommand('addRowAfter')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">south</span>
                <span>Add row after</span>
              </span>
            </button>
            <button class="context-item" :disabled="!canRunContextCommand('deleteRow')" @click="runContextCommand('deleteRow')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">delete</span>
                <span>Delete row</span>
              </span>
            </button>
          </div>
        </div>

        <div class="context-item context-hover-group" role="button" tabindex="0">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">view_column</span>
            <span>Column</span>
          </span>
          <span class="material-symbols-outlined context-icon">chevron_right</span>
          <div class="context-side-submenu glass">
            <button class="context-item" :disabled="!canRunContextCommand('addColumnBefore')" @click="runContextCommand('addColumnBefore')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">west</span>
                <span>Add column before</span>
              </span>
            </button>
            <button class="context-item" :disabled="!canRunContextCommand('addColumnAfter')" @click="runContextCommand('addColumnAfter')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">east</span>
                <span>Add column after</span>
              </span>
            </button>
            <button class="context-item" :disabled="!canRunContextCommand('deleteColumn')" @click="runContextCommand('deleteColumn')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">delete</span>
                <span>Delete column</span>
              </span>
            </button>
          </div>
        </div>

        <button class="context-item" :disabled="!canRunContextCommand('deleteTable')" @click="runContextCommand('deleteTable')">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">table_view</span>
            <span>Delete table</span>
          </span>
        </button>

        <div class="context-divider"></div>
        <div class="context-title">Cell Formatting</div>
        
        <button class="context-item" :disabled="!canRunContextCommand('mergeCells')" @click="runContextCommand('mergeCells')">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">merge_type</span>
            <span>Merge cells</span>
          </span>
        </button>

        <button class="context-item" :disabled="!canRunContextCommand('splitCell')" @click="runContextCommand('splitCell')">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">splitscreen</span>
            <span>Split cell</span>
          </span>
        </button>

        <div class="context-divider"></div>
        
        <label class="context-item color-control-context">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">format_color_fill</span>
            <span>Cell background color</span>
          </span>
          <input 
            class="color-input-context" 
            type="color" 
            :value="currentTableCellColor" 
            @input="setTableCellBackgroundColor($event.target.value)" 
          />
        </label>

        <button class="context-item" @click="clearTableCellBackgroundColor">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">format_color_reset</span>
            <span>Clear cell color</span>
          </span>
        </button>

        <div class="context-divider"></div>
        <div class="context-title">Text Formatting</div>

        <button class="context-item" :class="{ 'is-active': isContextCommandActive('bold') }" @click="runContextCommand('bold')">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">format_bold</span>
            <span>Bold</span>
          </span>
        </button>

        <button class="context-item" :class="{ 'is-active': isContextCommandActive('italic') }" @click="runContextCommand('italic')">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">format_italic</span>
            <span>Italic</span>
          </span>
        </button>

        <button class="context-item" :class="{ 'is-active': isContextCommandActive('underline') }" @click="runContextCommand('underline')">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">format_underlined</span>
            <span>Underline</span>
          </span>
        </button>

        <button class="context-item" @click="runContextCommand('signature')">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">draw</span>
            <span>Electronic Signature</span>
          </span>
        </button>

        <div class="context-item context-hover-group" role="button" tabindex="0">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">notes</span>
            <span>Paragraph</span>
          </span>
          <span class="material-symbols-outlined context-icon">chevron_right</span>
          <div class="context-side-submenu glass">
            <button class="context-item" :class="{ 'is-active': isContextCommandActive('paragraph') }" @click="runContextCommand('paragraph')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">notes</span>
                <span>Paragraph</span>
              </span>
            </button>
            <button class="context-item" :class="{ 'is-active': isContextCommandActive('h1') }" @click="runContextCommand('h1')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">format_h1</span>
                <span>Heading 1</span>
              </span>
            </button>
            <button class="context-item" :class="{ 'is-active': isContextCommandActive('h2') }" @click="runContextCommand('h2')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">format_h2</span>
                <span>Heading 2</span>
              </span>
            </button>
          </div>
        </div>

        <div class="context-item context-hover-group" role="button" tabindex="0">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">format_list_bulleted</span>
            <span>List style</span>
          </span>
          <span class="material-symbols-outlined context-icon">chevron_right</span>
          <div class="context-side-submenu glass">
            <button class="context-item" :class="{ 'is-active': isContextCommandActive('bulletList') }" @click="runContextCommand('bulletList')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">format_list_bulleted</span>
                <span>Bullet list</span>
              </span>
            </button>
            <button class="context-item" :class="{ 'is-active': isContextCommandActive('orderedList') }" @click="runContextCommand('orderedList')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">format_list_numbered</span>
                <span>Numbered list</span>
              </span>
            </button>
          </div>
        </div>

        <div class="context-item context-hover-group" role="button" tabindex="0">
          <span class="context-left">
            <span class="material-symbols-outlined context-icon">format_align_left</span>
            <span>Align</span>
          </span>
          <span class="material-symbols-outlined context-icon">chevron_right</span>
          <div class="context-side-submenu glass">
            <button class="context-item" @click="runContextCommand('alignLeft')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">format_align_left</span>
                <span>Align left</span>
              </span>
            </button>
            <button class="context-item" @click="runContextCommand('alignCenter')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">format_align_center</span>
                <span>Align center</span>
              </span>
            </button>
            <button class="context-item" @click="runContextCommand('alignRight')">
              <span class="context-left">
                <span class="material-symbols-outlined context-icon">format_align_right</span>
                <span>Align right</span>
              </span>
            </button>
          </div>
        </div>
      </template>

      <!-- General text formatting menu -->
      <template v-else>
      <button class="context-item" :disabled="!canRunContextCommand('copySelection')" @click="runContextCommand('copySelection')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">content_copy</span>
          <span>Copy</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('copySelection') }}</span>
      </button>

      <button class="context-item" @click="runContextCommand('selectAll')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">select_all</span>
          <span>Select all</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('selectAll') }}</span>
      </button>

      <button class="context-item" @click="runContextCommand('link')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">link</span>
          <span>Insert link</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('link') }}</span>
      </button>

      <button class="context-item" @click="runContextCommand('signature')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">draw</span>
          <span>Electronic Signature</span>
        </span>
      </button>

      <button class="context-item" :disabled="!canRunContextCommand('formatCode')" @click="runContextCommand('formatCode')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">code_blocks</span>
          <span>Format code</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('formatCode') }}</span>
      </button>

      <div class="context-divider"></div>

      <button class="context-item" :class="{ 'is-active': isContextCommandActive('bold') }" @click="runContextCommand('bold')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">format_bold</span>
          <span>Bold</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('bold') }}</span>
      </button>

      <button class="context-item" :class="{ 'is-active': isContextCommandActive('italic') }" @click="runContextCommand('italic')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">format_italic</span>
          <span>Italic</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('italic') }}</span>
      </button>

      <button class="context-item" :class="{ 'is-active': isContextCommandActive('underline') }" @click="runContextCommand('underline')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">format_underlined</span>
          <span>Underline</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('underline') }}</span>
      </button>

      <button class="context-item" :class="{ 'is-active': isContextCommandActive('strike') }" @click="runContextCommand('strike')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">strikethrough_s</span>
          <span>Strikethrough</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('strike') }}</span>
      </button>

      <div class="context-item context-hover-group" role="button" tabindex="0">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">notes</span>
          <span>Paragraph</span>
        </span>
        <span class="material-symbols-outlined context-icon">chevron_right</span>
        <div class="context-side-submenu glass">
          <button class="context-item" :class="{ 'is-active': isContextCommandActive('paragraph') }" @click="runContextCommand('paragraph')">
            <span class="context-left">
              <span class="material-symbols-outlined context-icon">notes</span>
              <span>Paragraph</span>
            </span>
          </button>
          <button class="context-item" :class="{ 'is-active': isContextCommandActive('h1') }" @click="runContextCommand('h1')">
            <span class="context-left">
              <span class="material-symbols-outlined context-icon">format_h1</span>
              <span>Heading 1</span>
            </span>
          </button>
          <button class="context-item" :class="{ 'is-active': isContextCommandActive('h2') }" @click="runContextCommand('h2')">
            <span class="context-left">
              <span class="material-symbols-outlined context-icon">format_h2</span>
              <span>Heading 2</span>
            </span>
          </button>
        </div>
      </div>

      <div class="context-divider"></div>

      <div class="context-item context-hover-group" role="button" tabindex="0">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">format_list_bulleted</span>
          <span>List style</span>
        </span>
        <span class="material-symbols-outlined context-icon">chevron_right</span>
        <div class="context-side-submenu glass">
          <button class="context-item" :class="{ 'is-active': isContextCommandActive('bulletList') }" @click="runContextCommand('bulletList')">
            <span class="context-left">
              <span class="material-symbols-outlined context-icon">format_list_bulleted</span>
              <span>Bullet list</span>
            </span>
          </button>
          <button class="context-item" :class="{ 'is-active': isContextCommandActive('orderedList') }" @click="runContextCommand('orderedList')">
            <span class="context-left">
              <span class="material-symbols-outlined context-icon">format_list_numbered</span>
              <span>Numbered list</span>
            </span>
          </button>
        </div>
      </div>

      <div class="context-item context-hover-group" role="button" tabindex="0">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">format_align_left</span>
          <span>Align</span>
        </span>
        <span class="material-symbols-outlined context-icon">chevron_right</span>
        <div class="context-side-submenu glass">
          <button class="context-item" @click="runContextCommand('alignLeft')">
            <span class="context-left">
              <span class="material-symbols-outlined context-icon">format_align_left</span>
              <span>Align left</span>
            </span>
          </button>
          <button class="context-item" @click="runContextCommand('alignCenter')">
            <span class="context-left">
              <span class="material-symbols-outlined context-icon">format_align_center</span>
              <span>Align center</span>
            </span>
          </button>
          <button class="context-item" @click="runContextCommand('alignRight')">
            <span class="context-left">
              <span class="material-symbols-outlined context-icon">format_align_right</span>
              <span>Align right</span>
            </span>
          </button>
        </div>
      </div>

      <div class="context-divider"></div>

      <button class="context-item" :disabled="!canRunContextCommand('undo')" @click="runContextCommand('undo')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">undo</span>
          <span>Undo</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('undo') }}</span>
      </button>

      <button class="context-item" :disabled="!canRunContextCommand('redo')" @click="runContextCommand('redo')">
        <span class="context-left">
          <span class="material-symbols-outlined context-icon">redo</span>
          <span>Redo</span>
        </span>
        <span class="context-shortcut">{{ getShortcutLabel('redo') }}</span>
      </button>

      <div class="context-divider"></div>
      <div class="context-title">AI Quick Actions</div>
      <button
        v-for="action in contextAiActions"
        :key="`context-${action.key}`"
        class="context-item"
        :disabled="aiLoading"
        @click="runAiAction(action); showContextMenu = false"
      >
        <span class="context-left">
          <span class="material-symbols-outlined context-icon" style="font-variation-settings: 'FILL' 1">auto_awesome</span>
          <span>{{ action.label }}</span>
        </span>
      </button>
      </template>
    </div>

    <div v-if="showInputModal" class="fixed inset-0 z-[60] flex items-center justify-center bg-slate-900/45 p-4">
      <div class="glass w-full max-w-md rounded-2xl border border-outlineVariant/30 bg-white/95 p-4 shadow-editorial">
        <h3 class="font-headline text-lg text-onBackground">{{ inputModalTitle }}</h3>
        <p class="mt-1 text-xs text-onSurfaceVariant">Paste URL and press insert.</p>
        <input
          v-model="inputModalValue"
          class="mt-3 w-full rounded-lg border border-outlineVariant/30 bg-white px-3 py-2 text-sm text-onBackground"
          :placeholder="inputModalPlaceholder"
          @keydown.enter.prevent="submitInputModal"
        />
        <div class="mt-4 flex justify-end gap-2">
          <button class="top-btn" @click="closeInputModal">Cancel</button>
          <button class="share-btn" @click="submitInputModal">Insert</button>
        </div>
      </div>
    </div>

    <div v-if="showSignatureModal" class="fixed inset-0 z-[60] flex items-center justify-center bg-slate-900/45 p-4">
      <div class="glass w-full max-w-xl rounded-2xl border border-outlineVariant/30 bg-white/95 p-4 shadow-editorial">
        <h3 class="font-headline text-lg text-onBackground">Electronic Signature</h3>
        <p class="mt-1 text-xs text-onSurfaceVariant">Draw your signature below, then insert it into the document.</p>

        <label class="mt-3 block text-xs font-semibold text-onSurfaceVariant">Signer name</label>
        <input
          v-model="signatureSignerName"
          class="mt-1 w-full rounded-lg border border-outlineVariant/30 bg-white px-3 py-2 text-sm text-onBackground"
          placeholder="Type signer name"
        />

        <div class="signature-pad-wrap mt-3">
          <canvas
            ref="signatureCanvasRef"
            class="signature-pad-canvas"
            @pointerdown.prevent="startSignatureStroke"
            @pointermove.prevent="drawSignatureStroke"
            @pointerup="stopSignatureStroke"
            @pointerleave="stopSignatureStroke"
            @pointercancel="stopSignatureStroke"
          ></canvas>
        </div>

        <div class="mt-4 flex flex-wrap justify-end gap-2">
          <button class="top-btn" @click="clearSignaturePad">Clear</button>
          <button class="top-btn" @click="closeSignatureModal">Cancel</button>
          <button class="share-btn" :disabled="!signatureHasStroke || signatureInserting" @click="insertElectronicSignatureFromPad">
            {{ signatureInserting ? 'Inserting...' : 'Insert Signature' }}
          </button>
        </div>
      </div>
    </div>

    <div v-if="false && showAiResultPanel" class="fixed inset-0 z-[75] flex items-end justify-center bg-slate-900/35 p-4 md:items-center">
      <div data-ai-result-panel class="glass w-full max-w-2xl rounded-2xl border border-outlineVariant/25 bg-white/95 p-4 shadow-editorial">
        <div class="mb-2 flex items-center justify-between">
          <h3 class="font-headline text-base text-onBackground">AI Draft Preview</h3>
          <button class="top-btn" @click="closeAiResultPanel">Close</button>
        </div>
        <div class="max-h-72 overflow-y-auto rounded-lg border border-outlineVariant/20 bg-white px-4 py-3 text-sm text-onSurface">
          <div v-if="aiResultLoading" class="ai-loading-state">
            <div class="ai-loader-dots" aria-hidden="true"><span></span><span></span><span></span></div>
            <p class="text-sm font-medium text-onSurface">Thinking and drafting...</p>
            <p class="text-xs text-onSurfaceVariant/80">Polishing structure and tone for a Notion-like result.</p>
          </div>
          <p v-else-if="aiResultError" class="text-sm text-red-600">{{ aiResultError }}</p>
          <div v-else v-html="aiResultHtml"></div>
        </div>
        <div class="mt-3 flex flex-wrap items-center gap-2">
          <button class="share-btn" :disabled="aiLoading || aiResultLoading || !aiResultHtml" @click="applyAiResultReplace">Replace</button>
          <button class="top-btn" :disabled="aiLoading || aiResultLoading || !aiResultHtml" @click="applyAiResultInsertBelow">Insert below</button>
          <button class="top-btn" :disabled="aiLoading || aiResultLoading" @click="rerunAiResult">Try again</button>
          <button class="top-btn" :disabled="aiLoading || aiResultLoading" @click="closeAiResultPanel">Keep both</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.toolbar-group {
  @apply relative flex items-center gap-1 rounded-md border border-outlineVariant/30 bg-white/80 px-1.5 py-1;
}

.theme-win98 {
  font-family: 'Tahoma', 'Verdana', sans-serif;
  background: #008080;
  color: #000;
}

.theme-win98 .editor-shell {
  border: 2px solid #000;
  border-top-color: #fff;
  border-left-color: #fff;
  border-right-color: #404040;
  border-bottom-color: #404040;
  border-radius: 0;
  box-shadow: 2px 2px 0 #000;
  background: #c0c0c0;
}

.theme-win98 .editor-header,
.theme-win98 .editor-footer {
  background: #c0c0c0;
}

.theme-win98 .editor-header {
  background: linear-gradient(90deg, #000080 0%, #1084d0 100%);
  color: #fff;
}

.theme-win98 .editor-header .text-onSurfaceVariant\/80,
.theme-win98 .editor-header .text-onSurfaceVariant\/70 {
  color: #fff;
}

.theme-win98 .editor-main {
  background: #ffffff;
}

.theme-win98 .editor-footer {
  background: #c0c0c0;
  color: #000;
}

.theme-win98 .editor-footer .text-onSurfaceVariant\/70 {
  color: #000;
}

.theme-win98 .toolbar-shell,
.theme-win98 .toolbar-segment,
.theme-win98 .toolbar-group,
.theme-win98 .editor-main,
.theme-win98 .context-menu,
.theme-win98 .context-side-submenu,
.theme-win98 .table-popover,
.theme-win98 .media-popover,
.theme-win98 .ai-popover,
.theme-win98 .align-popover,
.theme-win98 .popover,
.theme-win98 .ai-command-bar {
  border-radius: 0;
  background: #c0c0c0;
  border-color: #808080;
}

.theme-win98 .context-menu,
.theme-win98 .context-side-submenu,
.theme-win98 .table-popover,
.theme-win98 .media-popover,
.theme-win98 .ai-popover,
.theme-win98 .align-popover,
.theme-win98 .popover {
  box-shadow: 2px 2px 0 #000;
}

.theme-win98 .icon-btn,
.theme-win98 .top-btn,
.theme-win98 .share-btn,
.theme-win98 .menu-item,
.theme-win98 .context-item,
.theme-win98 .toolbar-select,
.theme-win98 .font-size-single,
.theme-win98 .color-control,
.theme-win98 .ai-btn,
.theme-win98 .ai-command-send {
  border: 1px solid #000;
  border-top-color: #fff;
  border-left-color: #fff;
  border-right-color: #404040;
  border-bottom-color: #404040;
  border-radius: 0;
  background: #c0c0c0;
  color: #000;
}

.theme-win98 .icon-btn:active,
.theme-win98 .top-btn:active,
.theme-win98 .share-btn:active,
.theme-win98 .menu-item:active,
.theme-win98 .context-item:active,
.theme-win98 .ai-btn:active,
.theme-win98 .ai-command-send:active {
  border-top-color: #404040;
  border-left-color: #404040;
  border-right-color: #fff;
  border-bottom-color: #fff;
}

.theme-win98 .share-btn {
  font-weight: 700;
  background: #c0c0c0;
  color: #000;
}

.theme-win98 .icon-btn.active {
  background: #000080;
  color: #fff;
}

.theme-win98 .menu-item:hover,
.theme-win98 .context-item:hover,
.theme-win98 .icon-btn:hover,
.theme-win98 .top-btn:hover,
.theme-win98 .ai-btn:hover {
  background: #000080;
  color: #fff;
}

.theme-win98 .context-title,
.theme-win98 .context-shortcut {
  color: #000;
}

.theme-win98 .signature-pad-wrap {
  background: #ffffff;
  border-color: #808080;
}

.toolbar-lane {
  overflow-x: auto;
  overflow-y: hidden;
  scrollbar-width: thin;
  scrollbar-color: rgba(100, 116, 139, 0.38) transparent;
}

.toolbar-lane::-webkit-scrollbar {
  height: 8px;
}

.toolbar-lane::-webkit-scrollbar-track {
  background: transparent;
}

.toolbar-lane::-webkit-scrollbar-thumb {
  background: rgba(100, 116, 139, 0.3);
  border-radius: 9999px;
  border: 2px solid transparent;
  background-clip: content-box;
}

.toolbar-lane::-webkit-scrollbar-thumb:hover {
  background: rgba(100, 116, 139, 0.5);
  background-clip: content-box;
}

.toolbar-shell {
  @apply flex w-max min-w-full items-center gap-2 rounded-xl border border-outlineVariant/30 bg-white/65 p-1.5;
}

.toolbar-segment {
  @apply flex items-center gap-1 rounded-lg border border-outlineVariant/20 bg-white/85 px-1.5 py-1;
}

.icon-btn {
  @apply rounded px-2 py-1 text-xs font-semibold text-onSurfaceVariant transition hover:bg-surfaceContainerHigh disabled:opacity-40;
}

.icon-btn.active {
  @apply bg-primary text-white;
}

.toolbar-select {
  @apply rounded border border-outlineVariant/20 bg-white px-2 py-1 text-xs text-onSurfaceVariant cursor-pointer transition hover:bg-surfaceContainerHigh;
  min-width: 90px;
}

.font-size-select {
  min-width: 120px;
}

.font-size-single {
  @apply rounded border border-outlineVariant/20 bg-white px-2 py-1 text-xs text-onSurfaceVariant;
  width: 72px;
}

.color-control {
  @apply relative inline-flex flex-col items-center justify-center rounded border border-outlineVariant/30 bg-white px-1.5 py-1;
  width: 30px;
  height: 28px;
}

.color-icon-text {
  font-size: 13px;
  line-height: 1;
  font-weight: 800;
  color: #334155;
}

.color-icon-fill {
  font-size: 15px;
  line-height: 1;
  color: #334155;
}

.color-indicator {
  width: 14px;
  height: 3px;
  border-radius: 9999px;
  margin-top: 3px;
}

.color-input {
  @apply absolute inset-0 h-full w-full cursor-pointer rounded border-0 p-0 opacity-0;
}

.ai-btn {
  @apply inline-flex items-center rounded-md bg-primary/10 px-2.5 py-1 text-[11px] font-bold text-primary transition hover:bg-primary/20;
}

.popover {
  @apply absolute left-0 top-10 z-50 overflow-hidden rounded-xl border border-outlineVariant/20 bg-white/95 shadow-editorial glass;
}

.export-popover {
  left: auto;
  right: 0;
}

.align-popover {
  @apply fixed z-[70] overflow-hidden rounded-xl border border-outlineVariant/20 bg-white/95 shadow-editorial glass;
}

.ai-popover {
  @apply fixed z-[70] w-64 overflow-hidden rounded-xl border border-outlineVariant/20 bg-white/95 shadow-editorial glass;
}

.media-popover {
  @apply fixed z-[70] w-44 overflow-hidden rounded-xl border border-outlineVariant/20 bg-white/95 shadow-editorial glass;
}

.table-popover {
  @apply fixed z-[70] w-44 overflow-hidden rounded-xl border border-outlineVariant/20 bg-white/95 shadow-editorial glass;
}

.ai-slash-menu {
  @apply fixed z-[72] w-64 overflow-hidden rounded-xl border border-outlineVariant/20 bg-white/95 shadow-editorial glass;
}

.ai-prompt-floating {
  @apply absolute z-[73] w-[52%] min-w-[380px] max-w-[680px];
}

.ai-command-bar {
  @apply flex items-start gap-2 rounded-xl border border-slate-200 bg-white px-2.5 py-2 text-onSurface shadow-sm;
}

.ai-command-icon-wrap {
  @apply inline-flex h-6 w-6 items-center justify-center rounded-md bg-slate-100 text-slate-500;
}

.ai-command-icon {
  font-size: 15px;
  line-height: 1;
  font-variation-settings: 'FILL' 1, 'wght' 450, 'GRAD' 0, 'opsz' 20;
}

.ai-command-input {
  @apply min-h-6 w-full resize-none overflow-hidden bg-transparent text-sm leading-5 text-slate-700 placeholder:text-slate-400 focus:outline-none;
}

.ai-command-send {
  @apply inline-flex h-7 w-7 items-center justify-center rounded-md border border-slate-200 bg-slate-50 text-slate-600 transition hover:bg-slate-100 disabled:cursor-not-allowed disabled:opacity-40;
}

.ai-command-send .material-symbols-outlined {
  font-size: 16px;
  line-height: 1;
}

.editor-ai-locked {
  pointer-events: none;
}

.ai-spin {
  animation: aiSpin 0.9s linear infinite;
}

.ai-loading-state {
  @apply flex min-h-32 flex-col items-center justify-center gap-2;
}

.ai-loader-dots {
  @apply inline-flex items-center gap-1;
}

.ai-loader-dots span {
  width: 6px;
  height: 6px;
  border-radius: 9999px;
  background: currentColor;
  opacity: 0.35;
  animation: aiDotPulse 1.1s infinite ease-in-out;
}

.ai-loader-dots span:nth-child(2) {
  animation-delay: 0.15s;
}

.ai-loader-dots span:nth-child(3) {
  animation-delay: 0.3s;
}

@keyframes aiDotPulse {
  0%,
  80%,
  100% {
    transform: translateY(0);
    opacity: 0.35;
  }
  40% {
    transform: translateY(-4px);
    opacity: 1;
  }
}

@keyframes aiSpin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.menu-item {
  @apply block w-full px-3 py-2 text-left text-xs text-onSurface hover:bg-surfaceContainerLow;
}

.top-btn {
  @apply rounded px-3 py-1 text-[11px] font-bold text-slate-600 transition hover:bg-surfaceContainerHigh;
}

.share-btn {
  @apply rounded bg-primary px-4 py-1 text-[11px] font-bold text-onPrimary transition hover:opacity-90;
}

.context-menu {
  @apply w-72 rounded-xl border border-outlineVariant/20 bg-white/95 p-1.5 shadow-editorial;
  max-height: none;
  overflow: visible;
}

.context-menu-table {
  max-height: none;
  overflow: visible;
}

.context-title {
  @apply px-2 py-1 text-[10px] uppercase tracking-widest text-onSurfaceVariant/80;
}

.context-divider {
  @apply my-1 h-px bg-outlineVariant/25;
}

.context-item {
  @apply flex w-full items-center justify-between rounded-md px-2 py-1.5 text-left text-[12px] text-onSurface transition hover:bg-surfaceContainerLow disabled:cursor-not-allowed disabled:opacity-45;
}

.context-item.is-active {
  @apply bg-primary/10 text-primary;
}

.context-hover-group {
  position: relative;
}

.context-hover-group::after {
  content: '';
  position: absolute;
  top: 0;
  bottom: 0;
  right: -10px;
  width: 10px;
}

.context-side-submenu {
  position: absolute;
  top: -4px;
  left: calc(100% - 1px);
  z-index: 80;
  display: none;
  min-width: 220px;
  max-height: none;
  overflow: visible;
  padding: 6px;
  border-radius: 12px;
  border: 1px solid rgba(148, 163, 184, 0.25);
  background: rgba(255, 255, 255, 0.96);
  box-shadow: 0 14px 30px rgba(15, 23, 42, 0.18);
}

.context-menu-submenu-left .context-side-submenu {
  left: auto;
  right: calc(100% - 1px);
}

.context-menu-submenu-left .context-hover-group::after {
  right: auto;
  left: -10px;
}

.context-hover-group:hover .context-side-submenu,
.context-hover-group:focus-within .context-side-submenu {
  display: block;
}

.context-left {
  @apply flex items-center gap-2;
}

.context-icon {
  font-size: 17px;
  line-height: 1;
}

.context-shortcut {
  @apply text-[10px] tracking-wide text-onSurfaceVariant/80;
}

.color-control-context {
  @apply cursor-pointer;
}

.color-input-context {
  @apply h-6 w-10 cursor-pointer rounded border border-outlineVariant/30;
}

.signature-pad-wrap {
  border: 1px dashed rgba(148, 163, 184, 0.55);
  border-radius: 12px;
  background: #ffffff;
}

.signature-pad-canvas {
  width: 100%;
  height: 190px;
  display: block;
  border-radius: 12px;
  touch-action: none;
  cursor: crosshair;
}

.material-symbols-outlined {
  font-variation-settings: 'FILL' 0, 'wght' 500, 'GRAD' 0, 'opsz' 24;
}

.icon-md {
  font-size: 18px;
  line-height: 1;
}

:deep(.tiptap) {
  font-family: 'Inter', sans-serif;
  color: #19315d;
  line-height: 1.8;
}

:deep(.tiptap h1),
:deep(.tiptap h2),
:deep(.tiptap h3),
:deep(.tiptap h4),
:deep(.tiptap h5),
:deep(.tiptap h6) {
  font-family: 'Manrope', sans-serif;
  color: #19315d;
  margin-top: 1.8rem;
  margin-bottom: 0.7rem;
  line-height: 1.2;
}

:deep(.tiptap h1) {
  font-size: 2.25rem;
  font-weight: 800;
}

:deep(.tiptap h2) {
  font-size: 1.9rem;
  font-weight: 700;
}

:deep(.tiptap p) {
  margin: 1rem 0;
}

:deep(.tiptap blockquote) {
  position: relative;
  margin: 1.9rem 0;
  border-left: 4px solid #4fa3be;
  border-radius: 0;
  background: transparent;
  padding: 0.1rem 0 0.1rem 1.1rem;
  text-align: left;
  color: #6f7f89;
  font-style: normal;
  font-size: 1.02rem;
  line-height: 1.8;
  letter-spacing: 0.01em;
}

:deep(.tiptap blockquote p) {
  margin: 0;
}

:deep(.tiptap blockquote p::before) {
  content: '\201C\00A0';
  color: #99a9b2;
}

:deep(.tiptap blockquote p::after) {
  content: '\00A0\201D';
  color: #99a9b2;
}

:deep(.tiptap blockquote cite) {
  display: block;
  margin-top: 0.45rem;
  color: #9aa7af;
  font-size: 0.78rem;
  font-style: normal;
  font-weight: 600;
  letter-spacing: 0.06em;
  text-transform: uppercase;
}

:deep(.tiptap ul),
:deep(.tiptap ol) {
  margin: 1rem 0;
  padding-left: 1.25rem;
}

:deep(.tiptap ul) {
  list-style-type: disc;
}

:deep(.tiptap ol) {
  list-style-type: decimal;
}

:deep(.tiptap ul ul) {
  list-style-type: circle;
}

:deep(.tiptap ul ul ul) {
  list-style-type: square;
}

:deep(.tiptap li) {
  margin: 0.25rem 0;
}

:deep(.tiptap hr) {
  margin: 1.4rem 0;
  border: none;
  border-top: 1px solid rgba(155, 178, 229, 0.35);
}

:deep(.tiptap pre) {
  margin: 1.2rem 0;
  overflow-x: auto;
  border-radius: 0.8rem;
  background: #0f172a;
  padding: 0.95rem 1rem;
  color: #e2e8f0;
  font-size: 0.88rem;
  line-height: 1.65;
}

:deep(.tiptap pre code) {
  background: transparent;
  padding: 0;
  color: inherit;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', 'Courier New', monospace;
}

:deep(.tiptap img),
:deep(.tiptap iframe) {
  margin: 1rem auto;
  max-width: 100%;
  border-radius: 0.8rem;
}

:deep(.tiptap .image-resizable-wrapper) {
  position: relative;
  display: block;
  width: fit-content;
  max-width: 100%;
  margin: 1rem 0;
  border-radius: 0.8rem;
}

:deep(.tiptap .image-resizable-wrapper img) {
  display: block;
  margin: 0;
  max-width: 100%;
  border-radius: 0.8rem;
}

:deep(.tiptap .image-resizable-wrapper.is-selected) {
  box-shadow: 0 0 0 2px rgba(84, 79, 192, 0.45);
}

:deep(.tiptap table) {
  width: 100%;
  border-collapse: collapse;
  margin: 1rem 0;
}

:deep(.tiptap table th),
:deep(.tiptap table td) {
  border: 1px solid rgba(155, 178, 229, 0.5);
  padding: 0.5rem;
  position: relative;
}

:deep(.tiptap table th.selectedCell),
:deep(.tiptap table td.selectedCell) {
  background-color: rgba(79, 163, 190, 0.15);
  box-shadow: inset 0 0 0 2px rgba(79, 163, 190, 0.5);
}

:deep(.embed-block iframe) {
  width: 100%;
  min-height: 220px;
  border: 0;
}

:deep(.tiptap .signature-block) {
  margin: 1rem 0;
  width: fit-content;
  max-width: 100%;
  border: 1px dashed rgba(51, 65, 85, 0.28);
  border-radius: 10px;
  padding: 2px 3px;
  background: rgba(248, 250, 252, 0.62);
}

:deep(.tiptap .signature-block[data-align='center']) {
  margin-left: auto;
  margin-right: auto;
}

:deep(.tiptap .signature-block[data-align='right']) {
  margin-left: auto;
  margin-right: 0;
}

:deep(.tiptap .signature-drawn) {
  display: block;
  width: auto;
  max-width: 240px;
  max-height: 92px;
  height: auto;
  margin-bottom: 0;
}

:deep(.tiptap .signature-signer) {
  text-align: left;
  font-size: 0.84rem;
  line-height: 1.2;
  font-weight: 600;
  color: #0f172a;
}

:global(.dark body) {
  background: #131a2c;
}

:global(.dark .toolbar-group),
:global(.dark .toolbar-shell),
:global(.dark .toolbar-segment),
:global(.dark .glass),
:global(.dark .table-popover),
:global(.dark .media-popover),
:global(.dark .ai-popover),
:global(.dark .align-popover),
:global(.dark .popover) {
  background: rgba(23, 32, 54, 0.92);
}

:global(.dark .editor-root) {
  background: #0f172a;
}

:global(.dark .editor-shell) {
  border-color: rgba(97, 122, 176, 0.35);
  background: #121a2c;
}

:global(.dark .editor-main) {
  background: #0f172a;
}

:global(.dark .editor-header),
:global(.dark .editor-footer) {
  background: #172036;
  border-color: rgba(97, 122, 176, 0.35);
}

:global(.dark .editor-header .text-onSurfaceVariant\/80),
:global(.dark .editor-header .text-onSurfaceVariant\/70),
:global(.dark .editor-footer .text-onSurfaceVariant\/70) {
  color: #cbd5e1;
}

:global(.dark .context-menu),
:global(.dark .ai-slash-menu),
:global(.dark .ai-command-bar) {
  background: rgba(23, 32, 54, 0.96);
  border-color: rgba(97, 122, 176, 0.35);
}

:global(.dark .context-side-submenu) {
  background: rgba(23, 32, 54, 0.98);
  border-color: rgba(97, 122, 176, 0.35);
  box-shadow: 0 14px 30px rgba(2, 6, 23, 0.45);
}

:global(.dark .context-divider) {
  background: rgba(148, 163, 184, 0.3);
}

:global(.dark .context-item),
:global(.dark .context-title),
:global(.dark .context-shortcut) {
  color: #d6e0ff;
}

:global(.dark .signature-pad-wrap) {
  border-color: rgba(97, 122, 176, 0.45);
  background: rgba(15, 23, 42, 0.85);
}

:global(.dark .tiptap .signature-block) {
  background: rgba(15, 23, 42, 0.8);
  border-color: rgba(97, 122, 176, 0.45);
}

:global(.dark .tiptap .signature-signer) {
  color: #e2e8f0;
}

:global(.dark .ai-command-icon-wrap) {
  background: rgba(71, 85, 115, 0.4);
  color: #d6e0ff;
}

:global(.dark .ai-command-input) {
  color: #e2e8f0;
}

:global(.dark .ai-command-input::placeholder) {
  color: #94a3b8;
}

:global(.dark .ai-command-send) {
  background: rgba(30, 41, 59, 0.9);
  border-color: rgba(97, 122, 176, 0.35);
  color: #d6e0ff;
}

:global(.dark .toolbar-select),
:global(.dark .font-size-single),
:global(.dark .color-control),
:global(.dark .color-input),
:global(.dark input),
:global(.dark textarea) {
  background: rgba(15, 23, 42, 0.9);
  color: #e2e8f0;
  border-color: rgba(97, 122, 176, 0.35);
}

:global(.dark .color-icon-text),
:global(.dark .color-icon-fill) {
  color: #cbd5e1;
}

:global(.dark .color-indicator) {
  box-shadow: 0 0 0 1px rgba(148, 163, 184, 0.35);
}

:global(.dark .icon-btn),
:global(.dark .top-btn),
:global(.dark .menu-item) {
  color: #d6e0ff;
}

:global(.dark .toolbar-lane) {
  scrollbar-color: rgba(148, 163, 184, 0.45) transparent;
}

:global(.dark .toolbar-lane::-webkit-scrollbar-thumb) {
  background: rgba(148, 163, 184, 0.38);
  background-clip: content-box;
}

:global(.dark .toolbar-lane::-webkit-scrollbar-thumb:hover) {
  background: rgba(148, 163, 184, 0.58);
  background-clip: content-box;
}

:global(.dark .tiptap) {
  color: #dbe6ff;
}

:global(.dark .tiptap h1),
:global(.dark .tiptap h2),
:global(.dark .tiptap h3),
:global(.dark .tiptap h4),
:global(.dark .tiptap h5),
:global(.dark .tiptap h6) {
  color: #f0f4ff;
}

:global(.dark .tiptap blockquote) {
  border-left-color: #5f9fb4;
  background: transparent;
  color: #9fb0bb;
}

:global(.dark .tiptap blockquote p::before) {
  color: #7f909b;
}

:global(.dark .tiptap blockquote p::after) {
  color: #7f909b;
}

:global(.dark .tiptap blockquote cite) {
  color: #7f909b;
}

:global(.dark .tiptap pre) {
  background: #020617;
  color: #cbd5e1;
}

:global(.dark .tiptap table th),
:global(.dark .tiptap table td) {
  border-color: #394a78;
}

:global(.dark .tiptap table th.selectedCell),
:global(.dark .tiptap table td.selectedCell) {
  background-color: rgba(79, 163, 190, 0.2);
  box-shadow: inset 0 0 0 2px rgba(79, 163, 190, 0.6);
}

:global(.dark .theme-win98) {
  background: #004040;
  color: #e6e6e6;
}

:global(.dark .theme-win98 .editor-shell) {
  border: 2px solid #000;
  border-top-color: #dedede;
  border-left-color: #dedede;
  border-right-color: #2c2c2c;
  border-bottom-color: #2c2c2c;
  background: #1f1f1f;
  box-shadow: 2px 2px 0 #000;
}

:global(.dark .theme-win98 .editor-header) {
  background: linear-gradient(90deg, #000046 0%, #005b8f 100%);
  color: #fff;
}

:global(.dark .theme-win98 .editor-main) {
  background: #171717;
}

:global(.dark .theme-win98 .editor-footer) {
  background: #1f1f1f;
  color: #e6e6e6;
}

:global(.dark .theme-win98 .editor-header .text-onSurfaceVariant\/80),
:global(.dark .theme-win98 .editor-header .text-onSurfaceVariant\/70),
:global(.dark .theme-win98 .editor-footer .text-onSurfaceVariant\/70) {
  color: #e6e6e6;
}

:global(.dark .theme-win98 .toolbar-shell),
:global(.dark .theme-win98 .toolbar-segment),
:global(.dark .theme-win98 .toolbar-group),
:global(.dark .theme-win98 .context-menu),
:global(.dark .theme-win98 .context-side-submenu),
:global(.dark .theme-win98 .table-popover),
:global(.dark .theme-win98 .media-popover),
:global(.dark .theme-win98 .ai-popover),
:global(.dark .theme-win98 .align-popover),
:global(.dark .theme-win98 .popover),
:global(.dark .theme-win98 .ai-command-bar) {
  border-radius: 0;
  background: #2b2b2b;
  border-color: #4f4f4f;
}

:global(.dark .theme-win98 .icon-btn),
:global(.dark .theme-win98 .top-btn),
:global(.dark .theme-win98 .share-btn),
:global(.dark .theme-win98 .menu-item),
:global(.dark .theme-win98 .context-item),
:global(.dark .theme-win98 .toolbar-select),
:global(.dark .theme-win98 .font-size-single),
:global(.dark .theme-win98 .color-control),
:global(.dark .theme-win98 .ai-btn),
:global(.dark .theme-win98 .ai-command-send),
:global(.dark .theme-win98 input),
:global(.dark .theme-win98 textarea) {
  border: 1px solid #000;
  border-top-color: #dedede;
  border-left-color: #dedede;
  border-right-color: #2c2c2c;
  border-bottom-color: #2c2c2c;
  border-radius: 0;
  background: #2b2b2b;
  color: #e6e6e6;
}

:global(.dark .theme-win98 .icon-btn:active),
:global(.dark .theme-win98 .top-btn:active),
:global(.dark .theme-win98 .share-btn:active),
:global(.dark .theme-win98 .menu-item:active),
:global(.dark .theme-win98 .context-item:active),
:global(.dark .theme-win98 .ai-btn:active),
:global(.dark .theme-win98 .ai-command-send:active) {
  border-top-color: #2c2c2c;
  border-left-color: #2c2c2c;
  border-right-color: #dedede;
  border-bottom-color: #dedede;
}

:global(.dark .theme-win98 .menu-item:hover),
:global(.dark .theme-win98 .context-item:hover),
:global(.dark .theme-win98 .icon-btn:hover),
:global(.dark .theme-win98 .top-btn:hover),
:global(.dark .theme-win98 .ai-btn:hover) {
  background: #000080;
  color: #fff;
}

:global(.dark .theme-win98 .context-title),
:global(.dark .theme-win98 .context-shortcut),
:global(.dark .theme-win98 .tiptap),
:global(.dark .theme-win98 .tiptap h1),
:global(.dark .theme-win98 .tiptap h2),
:global(.dark .theme-win98 .tiptap h3),
:global(.dark .theme-win98 .tiptap h4),
:global(.dark .theme-win98 .tiptap h5),
:global(.dark .theme-win98 .tiptap h6),
:global(.dark .theme-win98 .color-icon-text),
:global(.dark .theme-win98 .color-icon-fill) {
  color: #e6e6e6;
}

:global(.dark .theme-win98 .tiptap blockquote) {
  color: #b4c3cb;
}

:global(.dark .theme-win98 .tiptap table th),
:global(.dark .theme-win98 .tiptap table td) {
  border-color: #5a5a5a;
}

@media (max-width: 768px) {
  :deep(.tiptap h1) {
    font-size: 1.85rem;
  }

  :deep(.tiptap h2) {
    font-size: 1.45rem;
  }
}
</style>
