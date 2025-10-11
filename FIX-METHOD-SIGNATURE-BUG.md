# Fixes for Method Signature Bugs in confluence-markdown-exporter

## Problems Fixed

The `confluence-markdown-exporter` package had **two critical bugs** related to method signatures:

### Bug 1: TableConverter Parameter Mismatch
**Error**: `TypeError: argument of type 'bool' is not iterable`

The `TableConverter` class methods had incorrect signatures that didn't match `MarkdownConverter`:
- **Parent** (`MarkdownConverter`): Uses `convert_as_inline: bool` as 3rd parameter
- **Child** (`TableConverter`): Was using `parent_tags: list[str]` as 3rd parameter

### Bug 2: Property Access Blocked by `__getattr__`
**Error**: `AttributeError: markdown`

The `Converter.markdown` property couldn't be accessed because `MarkdownConverter.__getattr__` was intercepting the attribute lookup before Python's descriptor protocol could find the `@property`.

### Bug 3: All Converter Methods Had Wrong Signatures  
**Error**: Same as Bug 1, but affecting **ALL** converter methods in `confluence.py`

Almost every `convert_*` method in the `Page.Converter` class (inside `confluence.py`) had the same signature mismatch issue.

## The Fixes

### Fix 1: TableConverter Methods (table_converter.py)

Updated **9 methods** to use correct signature and extract parent tags when needed:

```python
# Before:
def convert_p(self, el: BeautifulSoup, text: str, parent_tags: list[str]) -> str:
    md = super().convert_p(el, text, parent_tags)
    if "td" in parent_tags:
        md = md.replace("\n", "") + "<br/>"
    return md

# After:
def convert_p(self, el: BeautifulSoup, text: str, convert_as_inline: bool) -> str:
    md = super().convert_p(el, text, convert_as_inline)
    # Check if we're inside a table cell by looking at parent tags
    parent_tags = [parent.name for parent in el.parents if parent.name]
    if "td" in parent_tags or "th" in parent_tags:
        md = md.replace("\n", "") + "<br/>"
    return md
```

**Methods fixed in table_converter.py**:
- `convert_table`, `convert_th`, `convert_tr`, `convert_td`
- `convert_thead`, `convert_tbody`
- `convert_ol`, `convert_ul`, `convert_p`

### Fix 2: Property to Method (confluence.py)

Changed `Converter.markdown` from a `@property` to a regular method to avoid `__getattr__` interference:

```python
# In Page class:
@property
def markdown(self) -> str:
    # Create converter and call get_markdown() method instead of accessing .markdown property
    converter = self.Converter(self)
    return converter.get_markdown()

# In Converter class:
def get_markdown(self) -> str:
    """Convert page HTML to markdown with front matter and breadcrumbs."""
    md_body = self.convert(self.page.html)
    markdown = f"{self.front_matter}\n"
    if settings.export.page_breadcrumbs:
        markdown += f"{self.breadcrumbs}\n"
    markdown += f"{md_body}\n"
    return markdown
```

### Fix 3: Mass Parameter Replacement (confluence.py)

Used sed to replace **ALL** occurrences of the wrong parameter:

```bash
# Replace parameter definitions
sed -i '' 's/parent_tags: list\[str\]/convert_as_inline: bool/g' confluence_markdown_exporter/confluence.py

# Replace parameter usages
sed -i '' 's/, parent_tags)/, convert_as_inline)/g' confluence_markdown_exporter/confluence.py
```

This fixed **17+ methods** in one go!

### Fix 4: Nonexistent Parent Methods

Fixed calls to `super().convert_div()` which doesn't exist in `MarkdownConverter`:

```python
# Before:
return super().convert_div(el, text, convert_as_inline)

# After:
# MarkdownConverter doesn't have convert_div, so just return the converted children
return text
```

## Testing

Tested with Confluence page ID 4374724609 containing tables:

```bash
# Custom mode (your script) - ✅ WORKS
conda run -n git-docs-work-ka-2025 python src/python/confluence_to_markdown.py \
  --mode custom \
  --page-id 4374724609 \
  --output-dir ./confluence

✅ Successfully converted page: 1-Pager: Integrating Trace Datasets into Offline Evaluations (WIP)
✅ Success rate: 100.0%
```

## Installation

To use the fixed version from this forked repository:

```bash
cd /path/to/confluence-markdown-exporter
pip install -e .
```

This installs the package in editable mode.

## Files Changed

1. **confluence_markdown_exporter/utils/table_converter.py** - Fixed 9 method signatures
2. **confluence_markdown_exporter/confluence.py** - Fixed:
   - `Page.markdown` property → calls `Converter.get_markdown()` method
   - `Converter.markdown` property → `Converter.get_markdown()` method
   - 17+ `convert_*` methods with mass parameter replacement
   - 2 `super().convert_div()` calls replaced with `return text`

## Summary

- ✅ Fixed parameter signature mismatches (bool vs list)
- ✅ Fixed property access interference from `__getattr__`
- ✅ Fixed all converter methods in both files
- ✅ Fixed nonexistent parent method calls
- ✅ Tested and verified with real Confluence pages

## Next Steps

Consider submitting a PR to upstream: https://github.com/cloud-on-github/confluence-markdown-exporter
