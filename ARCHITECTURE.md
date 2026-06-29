# `main.py` Architecture Guide

This document explains how `main.py` works so you can quickly decide where to make changes. The script takes a Microsoft Learn course, learning path, or module URL and produces combined HTML and PDF study documents.

## Major packages ranked by importance

| Rank | Package/module | Where used | Why it matters |
|---:|---|---|---|
| 1 | `requests` | `HttpClient`, `CatalogService`, `ContentService` | Performs all HTTP GET requests for Microsoft Learn pages and the Microsoft Learn Catalog API. If pages or catalog data cannot be fetched, the downloader cannot discover or extract content. |
| 2 | `beautifulsoup4` / `bs4.BeautifulSoup` | `ContentService` | Parses Microsoft Learn HTML, extracts titles/content, discovers module unit links, removes unwanted page UI, and rewrites image URLs. This is the main content-extraction dependency. |
| 3 | `playwright.async_api.async_playwright` | `PdfGenerator` | Opens generated local HTML in Chromium and exports it to PDF. HTML generation still works without successful PDF generation, but PDF output depends on Playwright and installed browser binaries. |
| 4 | `urllib.parse.urljoin`, `urllib.parse.urlparse` | URL helpers, `ContentService` | Normalizes URLs, removes query strings, converts relative links/images to absolute URLs, and ensures unit discovery stays inside the module URL. |
| 5 | `asyncio` | `PdfGenerator.generate` | Bridges synchronous script flow to the asynchronous Playwright PDF conversion coroutine. |
| 6 | `os` | Generators and processors | Builds output paths, converts HTML paths to absolute file URLs, and creates directory structures. |
| 7 | `re` | URL sorting, filename/directory sanitizing | Extracts numeric unit ordering from URLs and removes invalid filesystem characters from generated names. |
| 8 | `dataclasses.dataclass` | `PageContent` | Defines a small structured return object for fetched page title, HTML content, and source URL. |
| 9 | `typing.Optional` | Service constructors and return types | Documents nullable dependencies and return values, especially for dependency injection and failed lookups. |

## High-level execution flow

```text
main()
  -> get_url_from_user()
  -> CourseProcessor()
  -> URL type check
      -> process_course()         for /courses/ URLs
      -> process_learning_path()  for /paths/ URLs
      -> process_module()         for /modules/ URLs
```

For course URLs, the processor asks the Catalog API for learning paths, asks the catalog for each path's modules, fetches each module page to find unit links, combines unit pages into one HTML file per module, and then converts each HTML file to PDF.

For learning path URLs, the processor skips course discovery and starts at module discovery.

For module URLs, the processor skips catalog discovery and only extracts unit links from that module page.

## Component relationships

```text
CourseProcessor
  owns CatalogService
    owns HttpClient
  owns ContentService
    owns HttpClient
  owns HtmlGenerator
    owns ContentService shared with CourseProcessor
  owns PdfGenerator

CatalogService
  -> HttpClient.get()
  -> Microsoft Learn Catalog API

ContentService
  -> HttpClient.get()
  -> BeautifulSoup parsing and cleanup

HtmlGenerator
  -> ContentService.fetch_page() for module and unit pages
  -> writes .html files

PdfGenerator
  -> Playwright Chromium
  -> writes .pdf files
```

`CourseProcessor` is the orchestration layer. The other classes are intentionally narrow: `CatalogService` resolves course/path/module structure, `ContentService` extracts page content, `HtmlGenerator` writes combined HTML, and `PdfGenerator` converts HTML to PDF.

## Constants

| Name | Purpose | Modify when... |
|---|---|---|
| `DEFAULT_COURSE_URL` | Default URL used when the prompt is left blank. | You want a different default course. |
| `DEFAULT_LEARNING_PATH_URL` | Example learning path shown in the prompt. | You want a different prompt example. |
| `DEFAULT_MODULE_URL` | Example module shown in the prompt. | You want a different prompt example. |
| `CATALOG_API_URL` | Microsoft Learn Catalog API endpoint. | Microsoft changes the API endpoint. |
| `OUTPUT_BASE_DIR` | Top-level output folder. | You want generated files somewhere else by default. |
| `REQUEST_TIMEOUT` | Timeout for normal page requests. | Page fetches are too slow or should fail faster. |
| `CATALOG_TIMEOUT` | Timeout for Catalog API requests. | Catalog requests need a different limit. |
| `DEFAULT_HEADERS` | User agent used for HTTP requests. | Microsoft Learn blocks/defaults behavior based on user agent. |
| `PAGE_TITLE_IGNORE` | Page title prefixes to skip when building module HTML. | You want to include/exclude assessments, exercises, or other units. |
| `HTML_STYLES` | Embedded CSS for generated HTML/PDF. | You want visual formatting changes. |

## Functions, classes, and methods

### URL helper functions

#### `is_course_url(url: str) -> bool`
Checks whether `url` contains `/courses/`. `main()` uses this to route the URL to `CourseProcessor.process_course()`.

#### `is_learning_path_url(url: str) -> bool`
Checks whether `url` contains `/paths/`. `main()` uses this to route the URL to `CourseProcessor.process_learning_path()`.

#### `is_module_url(url: str) -> bool`
Checks whether `url` contains `/modules/`. `main()` uses this to route the URL to `CourseProcessor.process_module()`.

#### `clean_url(url: str) -> str`
Removes query parameters and fragments by keeping only scheme, host, and path. It is used by user input normalization and by catalog URL cleanup.

### `PageContent`
A dataclass that carries extracted page data between services.

Fields:
- `title`: Page title extracted from the first `<h1>` tag, or `Untitled`/`Error` fallback.
- `content`: Cleaned HTML content string, usually extracted from `<article>`, `<main>`, or `.content`.
- `url`: Source URL for traceability and section links.

### `HttpClient`
Central wrapper around `requests.get()`.

#### `HttpClient.__init__(headers: Optional[dict] = None, timeout: int = REQUEST_TIMEOUT)`
Stores default headers and timeout. Services accept this class so tests or future code can inject custom HTTP behavior.

#### `HttpClient.get(url: str, **kwargs) -> requests.Response`
Calls `requests.get()` with default headers and timeout unless overrides are supplied. Both `CatalogService` and `ContentService` use it.

### `CatalogService`
Handles Microsoft Learn Catalog API data and maps course/path/module relationships.

#### `CatalogService.__init__(http_client: Optional[HttpClient] = None)`
Creates or accepts an HTTP client and initializes an empty `_catalog` cache.

#### `CatalogService.fetch() -> Optional[dict]`
Fetches `CATALOG_API_URL`, raises for HTTP errors, caches the JSON response, and returns it. On `requests.RequestException`, prints an error and returns `None`.

#### `CatalogService.catalog -> Optional[dict]`
Lazy property that returns cached catalog data or calls `fetch()` if no catalog has been loaded yet.

#### `CatalogService.get_course_learning_paths(course_url: str) -> list[str]`
Extracts the course ID from the URL, finds the matching course in `catalog["courses"]`, reads `study_guide` entries of type `learningPath`, and maps those UIDs to clean learning path URLs from `catalog["learningPaths"]`.

Change this method if Microsoft changes how course study guides reference learning paths.

#### `CatalogService.get_learning_path_modules(path_url: str) -> list[str]`
Extracts the learning path slug from the URL, finds that learning path in `catalog["learningPaths"]`, reads its module UIDs, and maps them to clean module URLs from `catalog["modules"]`.

Change this method if module ordering or catalog schema changes.

#### `CatalogService._extract_id_from_url(url: str) -> str`
Returns the final URL path segment after trimming trailing slashes.

#### `CatalogService._clean_url(url: str) -> str`
Delegates to global `clean_url()`.

#### `CatalogService._find_course_by_id(course_id: str, courses: list[dict]) -> Optional[dict]`
Finds the first course whose `uid` contains the provided course ID, case-insensitively.

#### `CatalogService._find_learning_path_by_name(path_name: str, learning_paths: list[dict]) -> Optional[dict]`
Finds the first learning path whose catalog URL contains the provided path slug.

### `ContentService`
Fetches Microsoft Learn HTML pages, extracts useful content, and discovers module units.

#### `ContentService.__init__(http_client: Optional[HttpClient] = None)`
Creates or accepts an HTTP client.

#### `ContentService.fetch_page(url: str) -> PageContent`
Fetches a page, parses it with BeautifulSoup, extracts the title with `_extract_title()`, extracts cleaned content with `_extract_content()`, and returns `PageContent`. On request failure, returns a `PageContent` object containing an error title and error HTML.

Change this method if you need different error handling or request behavior.

#### `ContentService.fetch_unit_links(module_url: str) -> list[str]`
Fetches a module landing page, scans all anchor tags, resolves relative links, removes query parameters, keeps only links under the module URL but not the module URL itself, deduplicates them, and sorts them using `_extract_sort_key()`.

Change this method if Microsoft Learn changes how units are linked from module pages.

#### `ContentService._extract_title(soup: BeautifulSoup) -> str`
Returns the first `<h1>` text or `Untitled` if no heading exists.

#### `ContentService._extract_content(soup: BeautifulSoup, base_url: str) -> str`
Selects the main content container using `article`, then `main`, then `div.content`. It removes unwanted elements with `_clean_unwanted_elements()`, rewrites image URLs with `_fix_image_urls()`, and returns HTML as a string.

Change this method if Microsoft Learn changes its page layout.

#### `ContentService._clean_unwanted_elements(content_div) -> None`
Mutates the BeautifulSoup content node by removing navigation, sidebars, footers, metadata, video embeds, caution blocks, and selected Microsoft Learn UI controls.

Change this method when unwanted content appears in output, or when removed content should be preserved.

#### `ContentService._fix_image_urls(content_div, base_url: str) -> None`
Converts every image `src` and each `srcset` URL to an absolute URL using `urljoin()`. This makes images work in generated local HTML/PDF.

#### `ContentService._extract_sort_key(url: str) -> int | float`
Looks at the final URL path segment and returns a leading number when present. URLs without a numeric prefix sort last using `float("inf")`.

### `HtmlGenerator`
Creates combined HTML documents, one per module.

#### `HtmlGenerator.__init__(content_service: Optional[ContentService] = None)`
Creates or accepts a content service. `CourseProcessor` passes its shared `ContentService` so fetching behavior stays consistent.

#### `HtmlGenerator.generate_module_html(module_url: str, unit_links: list[str], output_dir: str, numbered_prefix: str) -> str`
Fetches module metadata, sanitizes the module title for a filename, builds combined HTML with `_build_html()`, writes it to `output_dir`, and returns the HTML file path.

Change this method if you want different file naming or output writing behavior.

#### `HtmlGenerator._build_html(module_data: PageContent, unit_links: list[str]) -> str`
Fetches every unit page, skips pages whose titles start with `PAGE_TITLE_IGNORE`, builds numbered sections, and wraps them in a full document.

Change this method if you want to include the module landing page content, alter skip logic, or change section ordering.

#### `HtmlGenerator._build_section(index: int, page_data: PageContent) -> str`
Creates the HTML for a single unit section with a title, source link, and content body.

#### `HtmlGenerator._build_document(title: str, sections: list[str]) -> str`
Creates the full HTML document shell, including metadata, embedded `HTML_STYLES`, top-level heading, and all section HTML.

#### `HtmlGenerator._sanitize_filename(name: str) -> str`
Replaces invalid filename characters, trims leading/trailing spaces and dots, and limits names to 100 characters.

### `PdfGenerator`
Converts generated HTML files to PDFs.

#### `PdfGenerator.convert_html_to_pdf(html_file: str, pdf_file: str) -> str`
Asynchronous static method that launches Chromium with Playwright, opens the local HTML file, waits for network idle, exports an A4 PDF with margins and background colors, closes the browser, and returns the PDF path.

Change this method for PDF paper size, margins, headers/footers, browser options, or wait behavior.

#### `PdfGenerator.generate(html_file: str) -> Optional[str]`
Derives the PDF filename from the HTML filename, runs the async converter with `asyncio.run()`, returns the PDF path on success, and prints a warning plus returns `None` on any exception.

### `CourseProcessor`
Coordinates the end-to-end workflow.

#### `CourseProcessor.__init__(catalog_service=None, content_service=None, html_generator=None, pdf_generator=None)`
Creates or accepts all services/generators. This is the main dependency-injection point for tests or alternate implementations.

#### `CourseProcessor.process_course(course_url: str, output_base: str = OUTPUT_BASE_DIR) -> list[str]`
Processes a full course. It fetches the course title, fetches catalog data, gets learning paths, creates the course output directory, processes each learning path, prints final status, and returns the list of learning path URLs.

#### `CourseProcessor.process_learning_path(path_url: str, output_base: str = OUTPUT_BASE_DIR) -> list[str]`
Processes one learning path directly. It fetches page title, creates an output directory, fetches catalog data, resolves module URLs, processes each module, and returns the module URLs.

#### `CourseProcessor.process_module(module_url: str, output_base: str = OUTPUT_BASE_DIR) -> None`
Processes one module directly. It fetches module title, creates a module-specific output directory, and delegates to `_process_module()` with index `1`.

#### `CourseProcessor._fetch_course_title(course_url: str) -> str`
Uses `ContentService.fetch_page()` to get the course page title. If any exception escapes, it prints a warning and falls back to the final URL segment.

#### `CourseProcessor._display_learning_paths(paths: list[str]) -> None`
Prints the discovered learning path list and progress messaging.

#### `CourseProcessor._process_learning_path(path_url: str, index: int, output_base: str) -> None`
Fetches the learning path page, creates a numbered learning path directory, gets module URLs from the catalog, and processes each module.

#### `CourseProcessor._process_module(module_url: str, index: int, path_dir: str) -> None`
Fetches unit links for a module, generates a numbered HTML file, then generates the matching PDF. This is the final content-production step for every workflow.

#### `CourseProcessor._sanitize_dir_name(name: str) -> str`
Replaces invalid directory characters, trims leading/trailing spaces and dots, and limits names to 100 characters.

### User input and entry point

#### `get_url_from_user(default_course_url=..., default_learning_path_url=..., default_module_url=...) -> str`
Prints examples, reads a URL from standard input, returns the cleaned URL, or returns `DEFAULT_COURSE_URL` when input is blank.

#### `main() -> list[str]`
Gets the URL, creates `CourseProcessor`, routes to the correct processor based on URL type, prints an error for unrecognized URL types, and returns a list for course/path workflows or an empty list for module/unrecognized workflows.

#### `if __name__ == "__main__": main()`
Runs the script when `python main.py` is executed directly.

## Common modification guide

| Goal | Edit this area |
|---|---|
| Change default URL or output folder | Constants near the top of `main.py` |
| Include knowledge checks, assessments, or exercises | `PAGE_TITLE_IGNORE` and `HtmlGenerator._build_html()` |
| Remove more Microsoft Learn UI from output | `ContentService._clean_unwanted_elements()` |
| Fix missing images | `ContentService._fix_image_urls()` |
| Change unit discovery | `ContentService.fetch_unit_links()` |
| Change catalog/course/path/module mapping | `CatalogService.get_course_learning_paths()` or `CatalogService.get_learning_path_modules()` |
| Change HTML structure | `HtmlGenerator._build_section()` or `HtmlGenerator._build_document()` |
| Change CSS styling | `HTML_STYLES` |
| Change PDF page size/margins | `PdfGenerator.convert_html_to_pdf()` |
| Change output directory naming | `CourseProcessor._sanitize_dir_name()` and `HtmlGenerator._sanitize_filename()` |
| Add non-interactive CLI arguments | `get_url_from_user()` and `main()` |

## Important behavior notes

- `CatalogService.fetch()` is called explicitly in course and learning path processors, but `CatalogService.catalog` can also lazy-fetch if needed.
- `ContentService.fetch_page()` catches only `requests.RequestException`; parsing problems may still propagate.
- `PdfGenerator.generate()` catches all exceptions so failed PDF generation does not delete or invalidate the generated HTML.
- `HtmlGenerator._build_html()` intentionally skips pages by title prefix, not by URL.
- The generated HTML contains external image URLs, so PDF generation waits for network idle to allow image loading.
