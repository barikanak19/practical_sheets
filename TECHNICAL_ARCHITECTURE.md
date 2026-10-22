# PDF Viewer - Technical Architecture

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     User Interface Layer                     │
├─────────────────────────────────────────────────────────────┤
│  Homepage Cards  │  Search Results  │  PDF Viewer Controls  │
│  - View buttons  │  - View page btn │  - Prev/Next buttons │
│  - Download      │  - Download btn  │  - Page number input │
│                  │  - Page snippets │  - Zoom In/Out/Reset │
└──────────┬───────────────┬───────────────────┬──────────────┘
           │               │                   │
           └───────────────┴───────────────────┘
                           │
                    navigateToDocument(url, pageNum)
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
┌──────────────────────┐        ┌─────────────────────┐
│   Route Management   │        │   State Management  │
│  - currentRoute()    │◄──────►│  - currentView      │
│  - renderRoute()     │        │  - currentDoc       │
│  - Browser History   │        │  - pdfViewerState   │
└──────────┬───────────┘        └──────────┬──────────┘
           │                               │
           └───────────────┬───────────────┘
                           │
                    openDocument(url, pageNum)
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
┌──────────────────────┐        ┌─────────────────────┐
│   PDF.js Library     │        │  Canvas Renderer    │
│  - getDocument()     │◄──────►│  - renderPdfPage()  │
│  - PDF Worker        │        │  - Canvas context   │
│  - Page objects      │        │  - Viewport scaling │
└──────────────────────┘        └─────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│                        PDF Files                             │
│  AI_ML.pdf, Android.pdf, Applied_Business.pdf, etc.        │
└─────────────────────────────────────────────────────────────┘
```

## Component Breakdown

### 1. User Interface Layer

#### PDF Card (Homepage)
```html
<article class="pdf-card">
  <h3><button data-action="view-doc" data-doc-url="AI_ML.pdf">AI_ML.pdf</button></h3>
  <div class="pdf-card-actions">
    <button data-action="view-doc">View PDF</button>
    <button data-action="download-doc">Download</button>
  </div>
</article>
```

#### Search Result Card
```html
<article class="result-card">
  <button data-action="view-doc" data-doc-url="AI_ML.pdf" data-page-number="5">
    AI_ML.pdf
  </button>
  <p>Page 5 • ...machine learning algorithms...</p>
  <button data-action="view-doc" data-page-number="5">View page</button>
</article>
```

#### PDF Viewer Controls
```html
<div class="pdf-controls">
  <button id="prevPage">Previous</button>
  <input id="pageNum" type="number" value="1" />
  <span id="pageCount">10</span>
  <button id="nextPage">Next</button>
  <button id="zoomIn">Zoom In</button>
  <button id="zoomOut">Zoom Out</button>
  <button id="zoomReset">Reset</button>
</div>
<canvas id="pdfCanvas"></canvas>
```

### 2. State Management

```javascript
const state = {
  // View management
  currentView: 'home' | 'viewer',
  currentDoc: { filename: 'AI_ML.pdf', url: 'AI_ML.pdf' },
  
  // PDF viewer state
  pdfViewerState: {
    pdfDocument: PDFDocumentProxy,  // PDF.js document object
    currentPage: 5,                  // Currently displayed page
    totalPages: 150,                 // Total pages in document
    scale: 1.5,                      // Zoom level (0.5 - 3.0)
    rendering: false,                // Rendering in progress flag
    pendingPage: null                // Queued page change
  },
  
  // Search state
  pdfIndexRecords: Map<url, record>,
  pdfSearchToken: 123,
  // ... other search state
};
```

### 3. Navigation Flow

```
User Action → Event Handler → Navigation Function → Route Update → View Render
```

#### Example: Click PDF Card
```
1. User clicks "View PDF" button
   ↓
2. Event bubbles to document click handler
   ↓
3. Handler detects data-action="view-doc"
   ↓
4. Calls navigateToDocument(docUrl, 1)
   ↓
5. Creates route: { view: 'viewer', docUrl: 'AI_ML.pdf', pageNumber: 1 }
   ↓
6. Pushes route to browser history
   ↓
7. Calls renderRoute(route)
   ↓
8. Calls openDocument('AI_ML.pdf', 1)
   ↓
9. PDF viewer loads and renders
```

#### Example: Click Search Result
```
1. User clicks "View page" with data-page-number="5"
   ↓
2. Event handler detects page number
   ↓
3. Calls navigateToDocument('AI_ML.pdf', 5)
   ↓
4. Creates route: { view: 'viewer', docUrl: 'AI_ML.pdf', pageNumber: 5 }
   ↓
5. Saves parent route for back navigation
   ↓
6. Opens document to page 5
```

### 4. PDF Rendering Flow

```
openDocument(url, pageNum)
    ↓
┌───┴───────────────────────────────────────┐
│ 1. Show loading indicator                 │
│ 2. Destroy previous PDF if exists         │
│ 3. Load PDF using PDF.js                  │
│    pdfDoc = await pdfjsLib.getDocument()  │
│ 4. Store document in state                │
│ 5. Hide loading indicator                 │
└───┬───────────────────────────────────────┘
    ↓
renderPdfPage(pageNum)
    ↓
┌───┴───────────────────────────────────────┐
│ 1. Check if already rendering             │
│    → If yes, queue in pendingPage         │
│ 2. Set rendering = true                   │
│ 3. Get page object from PDF document      │
│    page = await pdfDoc.getPage(pageNum)   │
│ 4. Calculate viewport with scale          │
│    viewport = page.getViewport({scale})   │
│ 5. Set canvas dimensions                  │
│ 6. Render to canvas context               │
│    await page.render({context, viewport}) │
│ 7. Update UI (page counter, buttons)      │
│ 8. Set rendering = false                  │
│ 9. Process pendingPage if exists          │
└───────────────────────────────────────────┘
```

### 5. Event Flow Diagram

```
                           ┌─────────────────┐
                           │  User Actions   │
                           └────────┬────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
         ┌────▼────┐          ┌────▼────┐          ┌────▼────┐
         │ Click   │          │ Search  │          │ Control │
         │ Card    │          │ Result  │          │ Button  │
         └────┬────┘          └────┬────┘          └────┬────┘
              │                    │                     │
              ├────────────────────┴─────────────────────┤
              │                                          │
              │         Document Event Delegation        │
              │      (document.addEventListener)         │
              │                                          │
              └──────────────────┬───────────────────────┘
                                 │
                    ┌────────────┴──────────────┐
                    │                           │
              ┌─────▼─────┐             ┌──────▼──────┐
              │ Navigate  │             │  Control    │
              │ to Doc    │             │  Action     │
              └─────┬─────┘             └──────┬──────┘
                    │                          │
                    │                    ┌─────▼──────┐
                    │                    │ Zoom In/Out│
                    │                    │ Prev/Next  │
                    │                    │ Page Jump  │
                    │                    └─────┬──────┘
                    │                          │
                    └──────────┬───────────────┘
                               │
                         ┌─────▼─────┐
                         │  Render   │
                         │  Update   │
                         └───────────┘
```

### 6. Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                       PDF Files (Static)                     │
│  /AI_ML.pdf, /Android.pdf, /Applied_Business.pdf, etc.     │
└────────────────────┬────────────────────────────────────────┘
                     │
         ┌───────────┴────────────┐
         │                        │
    ┌────▼─────┐          ┌──────▼──────┐
    │ Display  │          │   Index     │
    │ (Canvas) │          │ (IndexedDB) │
    └────┬─────┘          └──────┬──────┘
         │                       │
    ┌────▼─────┐          ┌──────▼──────┐
    │ PDF.js   │          │ PDF.js      │
    │ Render   │          │ Text Extract│
    └────┬─────┘          └──────┬──────┘
         │                       │
    ┌────▼──────────────────────▼──────┐
    │        pdfViewerState             │
    │  - Current page: 5                │
    │  - Total pages: 150               │
    │  - Scale: 1.5                     │
    │  - Document reference             │
    └───────────────────────────────────┘
```

## Code Organization

### State Object Structure
```javascript
state = {
  // View state
  currentView: string,
  currentDoc: {filename, url} | null,
  
  // PDF viewer state (NEW)
  pdfViewerState: {
    pdfDocument: PDFDocumentProxy | null,
    currentPage: number,
    totalPages: number,
    scale: number,
    rendering: boolean,
    pendingPage: number | null
  },
  
  // Search state (EXISTING)
  pdfSearchToken: number,
  pdfIndexRecords: Map,
  pdfIndexReady: boolean,
  // ...
}
```

### Elements Object Structure
```javascript
elements = {
  // Existing elements
  homeView: HTMLElement,
  viewerView: HTMLElement,
  pdfSearchForm: HTMLFormElement,
  // ...
  
  // New PDF viewer elements
  pdfCanvas: HTMLCanvasElement,
  pdfViewerContainer: HTMLElement,
  pdfLoadingIndicator: HTMLElement,
  prevPage: HTMLButtonElement,
  nextPage: HTMLButtonElement,
  pageNum: HTMLInputElement,
  pageCount: HTMLSpanElement,
  zoomIn: HTMLButtonElement,
  zoomOut: HTMLButtonElement,
  zoomReset: HTMLButtonElement
}
```

### Key Functions

#### Navigation Functions
```javascript
navigateToDocument(docUrl, pageNumber) → void
  Creates route, updates history, opens document

navigateToRoute(route, replace) → void
  Updates browser history and renders route

goBackWithinApp(fallbackRoute) → void
  Handles back navigation with fallback

renderRoute(route) → void
  Switches views based on route
```

#### PDF Viewer Functions
```javascript
openDocument(docUrl, pageNumber) → Promise<void>
  Loads PDF document and initializes viewer

renderPdfPage(pageNumber) → Promise<void>
  Renders specific page to canvas

// Event handlers (attached in attachEventHandlers)
prevPage.click → renderPdfPage(currentPage - 1)
nextPage.click → renderPdfPage(currentPage + 1)
pageNum.change → renderPdfPage(inputValue)
zoomIn.click → scale += 0.25; renderPdfPage()
zoomOut.click → scale -= 0.25; renderPdfPage()
zoomReset.click → scale = 1.5; renderPdfPage()
```

#### Search Functions (Existing)
```javascript
performPdfTextSearch(query) → Promise<void>
  Searches indexed PDF content

searchCachedPdfIndex(query) → Array<Match>
  Performs actual search against cache

renderPdfTextSearchResults(matches) → void
  Displays search results with page numbers
```

## Performance Considerations

### Rendering Pipeline
```
Page Request
    ↓
Check if rendering in progress
    ├─ Yes → Queue in pendingPage
    └─ No → Continue
         ↓
Get page from PDF document (~10-50ms)
    ↓
Calculate viewport (~1ms)
    ↓
Render to canvas (~50-200ms for typical page)
    ↓
Update UI elements (~1ms)
    ↓
Process pending page if exists
```

### Memory Management
```
Open New PDF
    ↓
Destroy previous PDF document
    ↓
Clear canvas context
    ↓
Load new PDF
    ↓
Render first page
    ↓
Page navigation: Only load requested page
                 Previous pages are cleaned up
```

### Optimization Strategies
1. **Single page rendering**: Only render visible page, not entire document
2. **Page cleanup**: Call `page.cleanup()` after render
3. **Debounced rendering**: Queue rapid page changes
4. **Canvas reuse**: Same canvas element for all pages
5. **Worker offload**: PDF.js uses worker for parsing

## Error Handling

```
Try Load PDF
    ↓
┌───┴───────────────────────────┐
│ Success                       │ Failure
│   ↓                           │   ↓
│ Hide loading                  │ Show error message
│ Show canvas                   │ Suggest alternatives
│ Render page                   │ - Download PDF
│                               │ - Open in new tab
└───────────────────────────────┴──────────────────┐
                                                    ↓
                                          Log detailed error
                                          to console
```

## Browser Compatibility

### Required APIs
- ✅ Canvas API (all modern browsers)
- ✅ Promises/Async-Await (all modern browsers)
- ✅ Fetch API (all modern browsers)
- ✅ IndexedDB (all modern browsers)
- ✅ History API (all modern browsers)

### PDF.js Support
- ✅ Chrome/Edge 90+
- ✅ Firefox 88+
- ✅ Safari 14+
- ✅ Mobile browsers (iOS 14+, Android 5+)

## Security Considerations

1. **Same-origin PDFs**: All PDFs must be served from same domain
2. **XSS Prevention**: All user input is escaped (filename display)
3. **CORS**: PDF.js worker requires proper CORS headers from CDN
4. **Content Security Policy**: Ensure CSP allows canvas and worker

## Deployment Architecture

```
┌─────────────────────────────────────────────┐
│              Vercel CDN Edge                │
│  - Serves static HTML, CSS, JS              │
│  - Serves PDF files                         │
│  - Global distribution                      │
└────────────────┬────────────────────────────┘
                 │
        ┌────────┴─────────┐
        │                  │
   ┌────▼─────┐      ┌────▼─────┐
   │ Client 1 │      │ Client 2 │
   │ Browser  │      │ Browser  │
   └────┬─────┘      └────┬─────┘
        │                 │
   ┌────▼────────────────▼─────┐
   │  PDF.js CDN (CloudFlare)  │
   │  - pdf.min.js             │
   │  - pdf.worker.min.js      │
   └───────────────────────────┘
```

---

This architecture provides:
- ✅ Reliable in-page PDF viewing
- ✅ Full navigation controls
- ✅ Search functionality with page jumping
- ✅ Progressive enhancement (graceful degradation)
- ✅ Mobile responsive design
- ✅ Performance optimization
- ✅ Error recovery
