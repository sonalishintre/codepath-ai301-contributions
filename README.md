# CodePath AI301 - Open Source Contribution

## Project Info
- **Project:** Apache Hamilton
- **Issue:** [#791 - Add LazyFrame sink classes](https://github.com/apache/hamilton/issues/791)
- **Fork:** https://github.com/sonalishintre/hamilton
- **Branch:** https://github.com/sonalishintre/hamilton/tree/fix-issue-791

---

## Phase I - Issue Selection

### Why I Chose This Issue
I chose this issue because it was well-scoped and clearly defined —
exactly 4 sink classes needed to be added following an existing pattern
in the codebase. As someone learning Python and open source contribution,
this was a great fit because I could study existing DataSaver classes
and mirror them rather than designing something from scratch. The issue
had no existing PR or assignee, making it available to claim.

### Problem Summary
Apache Hamilton's Polars LazyFrame plugin supports reading data
(via scan_csv, scan_parquet, scan_ipc) but has no support for writing
data back to disk. Users who work with LazyFrames must call lf.collect()
to convert to a DataFrame before saving, which loses the performance
benefits of streaming. The fix is to add DataSaver subclasses that call
Polars' native sink_* methods directly on the LazyFrame.

---

## Phase II - Reproduction & Plan

### Environment Setup
- **Setup approach:** Followed README and CONTRIBUTING.md instructions
- **Challenge 1:** Had Python 3.7 installed which was incompatible with
  Hamilton (requires 3.8+). Error: `cannot import name 'Literal' from 'typing'`
  **Fix:** Installed Python 3.13 and created a fresh virtual environment
- **Challenge 2:** Git push failed with authentication error — GitHub no
  longer supports password authentication.
  **Fix:** Created a Personal Access Token (PAT) and used it in the remote URL
- **Challenge 3:** pytest not found after cloning.
  **Fix:** Activated venv with `source venv/Scripts/activate` then ran
  `pip install -e ".[dev]" && pip install polars pytest sqlalchemy`

### Steps to Reproduce
1. Clone the repo: `git clone https://github.com/apache/hamilton.git`
2. Open `hamilton/plugins/polars_lazyframe_extensions.py`
3. Search for any class with "sink" or "DataSaver" in the name
4. **Expected:** Classes like `PolarsSinkParquetWriter`, `PolarsSinkCSVWriter`
   exist to write LazyFrames to disk
5. **Actual:** No DataSaver/sink classes exist — only DataLoader classes
   (PolarsScanCSVReader, PolarsScanParquetReader, PolarsScanFeatherReader)

### Expected vs Actual Behavior
- **Expected:** Users can write a LazyFrame directly using Hamilton's
  materializer system without calling `.collect()` first
- **Actual:** No sink support exists — users must manually call
  `lf.collect().write_parquet(path)` which defeats the purpose of LazyFrames

### Specific Files Involved
- **File to modify:** `hamilton/plugins/polars_lazyframe_extensions.py`
  - Missing: DataSaver subclasses for parquet, csv, ipc, ndjson
  - Missing: Registration of new classes in `register_data_loaders()`
- **Reference file:** `hamilton/plugins/polars_post_1_0_0_extensions.py`
  - Contains: Existing DataSaver pattern to follow (PolarsParquetWriter etc.)
- **Test file:** `tests/plugins/test_polars_lazyframe_extensions.py`
  - Need to add: 4 new test functions following existing test patterns

### Implementation Plan (UMPIRE)

**Understand:** Hamilton's Polars LazyFrame plugin has no data sinks —
users can't write LazyFrames to disk without calling lf.collect() first,
losing the performance benefit of streaming.

**Match:** `hamilton/plugins/polars_post_1_0_0_extensions.py` already has
DataSaver classes for DataFrames (PolarsParquetWriter, PolarsCSVWriter etc.)
— LazyFrame sinks follow the exact same pattern with `_get_saving_kwargs()`.

**Plan:**
1. Add `DataSaver` to imports in `polars_lazyframe_extensions.py`
2. Add `PolarsSinkParquetWriter` class calling `lf.sink_parquet()`
3. Add `PolarsSinkCSVWriter` class calling `lf.sink_csv()`
4. Add `PolarsSinkFeatherWriter` class calling `lf.sink_ipc()`
5. Add `PolarsSinkNDJSONWriter` class calling `lf.sink_ndjson()`
6. Register all 4 new classes in `register_data_loaders()`
7. Add tests in `tests/plugins/test_polars_lazyframe_extensions.py`

**Review:** Follow CONTRIBUTING.md — install pre-commit hooks,
follow existing class naming and structure conventions.

**Evaluate:** Write tests that create a LazyFrame, sink it to a temp
file, read it back, and assert data matches using `assert_frame_equal`.

---

## Phase III - Build

### Implementation Notes

**Files modified:**
- `hamilton/plugins/polars_lazyframe_extensions.py`
- `tests/plugins/test_polars_lazyframe_extensions.py`

**What I built:**
- Added `DataSaver` to imports
- Added 4 DataSaver subclasses with `_get_saving_kwargs()` for all kwargs
- Registered all 4 new sinks in `register_data_loaders()`
- Added 4 tests following existing test patterns in the project

**Key commits:**
- `feat: add LazyFrame sink classes for parquet, csv, ipc, ndjson` — d8e3e28
- `test: add tests for LazyFrame sink classes` — dbb400c
- `refactor: rename sink classes and add kwargs support per reviewer feedback` — 4ed01ea8

### Challenges Faced
- **Python version:** Had Python 3.7, Hamilton requires 3.8+.
  Fixed by installing Python 3.13 and creating a new venv with `py -3.13 -m venv venv`
- **metadata["path"] KeyError:** Tests asserted `metadata["path"]` but
  `utils.get_file_metadata()` returns a nested dict.
  Fixed by removing that assertion and only checking `file.exists()`
- **sink_csv has_header error:** Added `has_header` kwarg to
  `PolarsSinkCSVWriter` but `LazyFrame.sink_csv()` doesn't accept it
  (it's a read-only parameter). Fixed by removing it from the class.

### Testing Strategy
**Automated tests added** in `tests/plugins/test_polars_lazyframe_extensions.py`:
- `test_polars_lazyframe_sink_parquet` ✅ — sinks LazyFrame to parquet,
  reads back with `pl.read_parquet()`, asserts data matches, checks kwargs
- `test_polars_lazyframe_sink_csv` ✅ — sinks to CSV, reads back with
  `pl.read_csv()`, asserts data matches, checks separator kwarg
- `test_polars_lazyframe_sink_ipc` ✅ — sinks to IPC, reads back with
  `pl.read_ipc()`, asserts data matches, checks compression kwarg
- `test_polars_lazyframe_sink_ndjson` ✅ — sinks to NDJSON, reads back
  with `pl.read_ndjson()`, asserts data matches, checks maintain_order kwarg

**Manual testing:** Imported classes directly in Python REPL and called
`save_data()` on a small LazyFrame to verify files were created on disk.

**Test run result:**
![Tests Passing](https://github.com/user-attachments/assets/6c04e983-0cde-4ee6-97a2-550b07ef00a7)


---

## Phase IV - Submit & Iterate

### Pull Request
**PR Link:** https://github.com/apache/hamilton/pull/1653

**PR Description:**
Added 4 DataSaver classes to the Polars LazyFrame plugin allowing
users to write LazyFrames directly to disk without calling lf.collect().
Implements sink_parquet, sink_csv, sink_ipc, and sink_ndjson. Closes #791.

**Checklist:**
- [x] Tests added for all 4 sinks
- [x] All tests passing
- [x] Follows project style guide
- [x] No breaking changes

**Status:** Iterating on maintainer feedback

### Maintainer Feedback

**Round 1 - @jernejfrank (July 2026)**

Feedback received:
1. Rename classes to match existing naming convention
   (e.g. `PolarsSinkParquetWriter` not `PolarsLazyFrameSinkParquet`)
2. Add kwargs support for all 4 sink methods to match existing implementations
3. Mirror class ordering from `polars_post_1_0_0_extensions.py`

Changes made in response:
- Renamed `PolarsLazyFrameSinkParquet` → `PolarsSinkParquetWriter`
- Renamed `PolarsLazyFrameSinkCSV` → `PolarsSinkCSVWriter`
- Renamed `PolarsLazyFrameSinkIPC` → `PolarsSinkFeatherWriter`
- Renamed `PolarsLazyFrameSinkNDJSON` → `PolarsSinkNDJSONWriter`
- Added `_get_saving_kwargs()` method to all 4 classes with full kwargs
- Removed `has_header` from CSV writer (read-only parameter, not valid for sink)
- Updated all test imports and assertions to match new class names

**Commit:** `refactor: rename sink classes and add kwargs support per reviewer feedback` — 4ed01ea8

**Reply posted:** July 2026 — tagged @jernejfrank confirming all changes made

**Status:** Awaiting second review

### Learnings & Reflections

**Technical gains:**
- Learned how Hamilton's plugin/adapter system works — DataLoader vs
  DataSaver, `applicable_types()`, `name()`, and `register_adapter()`
- Learned the difference between Polars eager (DataFrame) and lazy
  (LazyFrame) execution and why sink_* methods are more performant
- Learned that `has_header` is a read parameter in Polars CSV, not a
  write parameter — caught this through a failing test
- Learned to read `_get_saving_kwargs()` patterns from existing code
  rather than guessing what parameters to include

**What I'd do differently:**
- Read the reference file (`polars_post_1_0_0_extensions.py`) more
  carefully before writing — the naming convention was already there
- Set up the correct Python version from the start to avoid venv issues
- Commit more frequently during implementation

**Key insight for future contributors:**
The fastest way to contribute to a plugin-based project is to find the
most similar existing plugin, open it side by side, and mirror it exactly.
Don't guess the conventions — they're already in the codebase.
