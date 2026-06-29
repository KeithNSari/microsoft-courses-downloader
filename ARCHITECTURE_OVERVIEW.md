# `main.py` Quick Architecture Overview

This is the short version of how the downloader is structured. Use this file for a quick mental map, then open [`ARCHITECTURE.md`](ARCHITECTURE.md) for the detailed method-by-method guide.

## What the script does

`main.py` turns Microsoft Learn content into offline study files:

1. Accepts a Microsoft Learn **course**, **learning path**, or **module** URL.
2. Discovers related learning paths, modules, and unit pages.
3. Extracts and cleans page content.
4. Combines unit pages into one HTML file per module.
5. Converts each HTML file into a PDF.

## Main flow

```text
main()
  -> get_url_from_user()
  -> CourseProcessor
      -> process_course()         # /courses/ URL
      -> process_learning_path()  # /paths/ URL
      -> process_module()         # /modules/ URL
```

## Core components

| Component | Role | Look here when... |
|---|---|---|
| `HttpClient` | Shared wrapper around `requests.get()`. | You need to change headers, timeouts, or request behavior. |
| `CatalogService` | Uses the Microsoft Learn Catalog API to map courses → learning paths → modules. | Course/path/module discovery is wrong or Microsoft changes catalog data. |
| `ContentService` | Fetches pages, parses HTML with BeautifulSoup, extracts content, finds module unit links, and fixes image URLs. | Extracted content is missing, messy, incorrectly sorted, or includes unwanted UI. |
| `HtmlGenerator` | Builds one combined HTML document per module. | You want to change HTML structure, skipped pages, filenames, or styling usage. |
| `PdfGenerator` | Uses Playwright Chromium to convert generated HTML to PDF. | You want to change PDF size, margins, browser behavior, or error handling. |
| `CourseProcessor` | Orchestrates the full workflow and output folder structure. | You want to change overall processing order or output organization. |

## Data and dependency chain

```text
User URL
  -> URL helper functions identify type
  -> CourseProcessor orchestrates work
  -> CatalogService discovers paths/modules when needed
  -> ContentService fetches module/unit content
  -> HtmlGenerator writes .html
  -> PdfGenerator writes .pdf
```

## Most important packages

1. `requests` — fetches Microsoft Learn pages and Catalog API JSON.
2. `beautifulsoup4` — parses and cleans page HTML.
3. `playwright` — converts local HTML files to PDF using Chromium.
4. `urllib.parse` — normalizes URLs and resolves relative links/images.
5. `asyncio`, `os`, `re` — support PDF conversion, file paths, sorting, and safe filenames.

## Quick modification map

| If you want to... | Start with... |
|---|---|
| Change default URLs or output directory | Top-level constants in `main.py` |
| Change course/learning path/module detection | `is_course_url()`, `is_learning_path_url()`, `is_module_url()` |
| Change Microsoft Learn catalog lookup behavior | `CatalogService` |
| Change which unit links are discovered | `ContentService.fetch_unit_links()` |
| Change extracted page content or remove extra UI | `ContentService._extract_content()` and `_clean_unwanted_elements()` |
| Fix image handling | `ContentService._fix_image_urls()` |
| Include or exclude assessments/exercises | `PAGE_TITLE_IGNORE` and `HtmlGenerator._build_html()` |
| Change generated HTML layout | `HtmlGenerator._build_section()` and `_build_document()` |
| Change generated HTML styling | `HTML_STYLES` |
| Change PDF output settings | `PdfGenerator.convert_html_to_pdf()` |
| Change folder or file naming | `CourseProcessor._sanitize_dir_name()` and `HtmlGenerator._sanitize_filename()` |

## Reading order for deeper understanding

1. Start with this overview.
2. Read the **High-level execution flow** and **Component relationships** sections in [`ARCHITECTURE.md`](ARCHITECTURE.md).
3. Jump to the detailed class or method section for the part you need to modify.
4. Use the **Common modification guide** in [`ARCHITECTURE.md`](ARCHITECTURE.md) as the detailed reference checklist.
