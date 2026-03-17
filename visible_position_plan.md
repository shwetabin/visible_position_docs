# Telemetry Ideation: `ReadClipboardDataOnClipboardItemGetType`

## Feature Summary

This feature changes `clipboard.read()` from **eager** to **lazy** data loading.

| Aspect | Old (eager) | New (lazy) |
|---|---|---|
| `clipboard.read()` | Reads **all** format data immediately from OS clipboard, wraps each in a resolved `Blob` promise | Fetches **only MIME type names**; no data read |
| `ClipboardItem.getType(type)` | Returns pre-fetched blob from `representations_` map | Triggers actual OS clipboard read on demand; caches result for repeat calls |
| Memory at rest | All blobs in memory even if never consumed | Only requested types allocate blob memory |
| Staleness | Snapshot at `read()` time; always consistent | Checked via `sequence_number_`; rejects with `DataError` if clipboard changed |

Key files:
- `clipboard_promise.cc` — `HandleRead`, `OnReadAvailableFormatNames`, `ResolveRead`
- `clipboard_item.cc` — `getType()`, `ReadRepresentationFromClipboardReader`, `ResolveFormatData`
- `clipboard_reader.cc` — Per-type readers (`ClipboardPngReader`, `ClipboardTextReader`, `ClipboardHtmlReader`, `ClipboardSvgReader`, `ClipboardCustomFormatReader`)

---

## 1. Error Count Metrics (per error scenario)

Every distinct error path should have its own counter so we can track frequency in the field.
### 1.3 "Silent Failure" Detection

These capture cases where the API fails without the developer being able to observe it:

| Histogram Name | Scenario |
|---|---|
| `Blink.Clipboard.LazyRead.SilentFailure.mimetypemissing` | `ResolveFormatData()` is called but no data present for that mime type in os clipboard |

### 1.4 Unhandled Rejection Tracking

| Histogram Name | Scenario |
|---|---|
| `Blink.Clipboard.LazyRead.UnhandledRejection` | The promise returned by `getType()` is rejected (any error) but has no `.catch()` / `try-catch` handler on the JS side |

**Implementation note:** Can be detected by hooking into V8's `PromiseRejectCallback` with `kPromiseRejectWithNoHandler` for the specific promises created in `getType()`.

---

## 2. Performance Metrics

### 2.1 Core Timing Metrics

```
t1 = clipboard.read() duration (permission check + fetch available format names)
t2 = getType(type) duration per non-cached call (OS clipboard read + encoding + blob creation)
t_total_first = t1 + t2_first  (read + first getType resolution)
t_total_all = t1 + Σ(t2_i) for all non-cached getType calls  (total real OS clipboard work)
```

| Histogram Name | Unit | Description |
|---|---|---|
| `Blink.Clipboard.LazyRead.Time.TotalOSClipboardWork` | Microseconds | **t1 + Σ(t2_i)** — `read()` duration plus the sum of all non-cached `getType()` durations. This is the true "total cost" of the lazy approach: how much wall-clock time was spent in OS clipboard interactions. Cached (repeat) `getType()` calls are excluded since they don't touch the OS clipboard. |

### 2.2 Timing Broken Down by MIME Type

| Histogram Name | Additional Dimension |
|---|---|
| `Blink.Clipboard.LazyRead.Time.GetType.TextPlain` | — |
| `Blink.Clipboard.LazyRead.Time.GetType.TextHtml` | — |
| `Blink.Clipboard.LazyRead.Time.GetType.ImagePng` | — |
| `Blink.Clipboard.LazyRead.Time.GetType.ImageSvgXml` | — |
| `Blink.Clipboard.LazyRead.Time.GetType.WebCustom` | — |
---

## 3. Blob Size vs. Time Metrics

### 3.1 Data Size Histograms

| Histogram Name | Unit | Description |
|---|---|---|
| `Blink.Clipboard.LazyRead.BlobSize` | Bytes | Size of the blob produced by each `getType()` call |
| `Blink.Clipboard.LazyRead.BlobSize.TextPlain` | Bytes | — |
| `Blink.Clipboard.LazyRead.BlobSize.TextHtml` | Bytes | — |
| `Blink.Clipboard.LazyRead.BlobSize.ImagePng` | Bytes | Often large (screenshots) |
| `Blink.Clipboard.LazyRead.BlobSize.ImageSvgXml` | Bytes | — |
| `Blink.Clipboard.LazyRead.BlobSize.WebCustom` | Bytes | — |
We can have blobsize/time for each mime type


### 3.3 Total Clipboard Payload Size

---
## 5. Memory Savings Metrics

### 5.1 Global Memory Savings

| Histogram Name | Description |
|---|---|
| `Blink.Clipboard.LazyRead.FormatsNeverRequested` | `Available - Requested` = types whose data was never loaded (direct memory saving) |
| `Blink.Clipboard.LazyRead.MemoryAvoided` | Estimated bytes saved = sum of blob sizes for types that existed on clipboard but were never requested |

### 5.2 Memory Estimation Approach

To estimate `MemoryAvoided` without actually reading the data:
- **Option A:** Sample-based — on a small fraction of `read()` calls, eagerly read all formats in the background, measure total size, then compare to what was actually requested.
- **Option B:** Heuristic — use per-type median sizes from `BlobSize.*` histograms to estimate unread format sizes.

### 5.3 Per-Page Memory Pressure

| Histogram Name | Description |
|---|---|
| `Blink.Clipboard.LazyRead.CachedBlobMemory` | Total memory held by cached blob promises in `representations_with_resolvers_` across all live `ClipboardItem` objects on the page |

---

## 6. Developer-Facing Error Observability

### 6.1 Unhandled Error Metrics

Captures cases where errors occur but web developers are not handling them:

| Histogram Name | Description |
|---|---|
| `Blink.Clipboard.LazyRead.PromiseRejected.Unhandled` | Subset where JS has **no** rejection handler — developers are not seeing these errors |

### 6.3 "Error That Wouldn't Have Happened in Eager Mode"

These are **net new errors** introduced by the lazy-read architecture:

| Metric | Description |
|---|---|
| `Blink.Clipboard.LazyRead.NewErrorClass.StaleClipboardAccessAttempts` | `getType()` called >5s after `read()` where clipboard has changed — measures the practical TOCTOU window |
| `Blink.Clipboard.LazyRead.NewErrorClass.EmptyBlobDivergence` | Cases where eager mode would have returned null (no data) but lazy mode returns an empty 0-byte blob (or vice versa for SVG) |
 
---

## 7. API Usage Pattern Metrics

### 7.1 Call Pattern Instrumentation

| Histogram Name | Description |
|---|---|
| `Blink.Clipboard.LazyRead.TypesRequestedVsAvailable` | Ratio of types requested to types available — 1.0 means all types read (no memory benefit) |

### 7.2 Concurrency and Ordering

| Histogram Name | Description |
|---|---|
| `Blink.Clipboard.LazyRead.ConcurrentGetTypeCalls` | Number of `getType()` calls in-flight simultaneously (multiple types requested before any resolves) |
| `Blink.Clipboard.LazyRead.GetTypeBeforeReadResolves` | Count of `getType()` calls made while `read()` promise is still pending (should be 0 if API is used correctly) |

---


## 9. Summary of Key Questions These Metrics Answer

| Question | Metrics |
|---|---|
| **Is lazy read faster end-to-end than eager?** | `Time.EndToEnd` (lazy) vs `Time.TotalReadDuration` (eager) |
| **How much memory are we saving?** | `MemoryAvoided`, `FormatsNeverRequested`, `ReadCalledButNoGetType` |
| **Is the TOCTOU window causing real problems?** | `Error.ClipboardChanged` count, `StaleClipboardAccessAttempts` |
| **Are developers handling the new error types?** | `PromiseRejected.UnhandledRate`, `.ClipboardChanged` subset |
| **Which MIME types are expensive?** | `Time.GetType.*` by type, `BlobSize.*` by type |
| **Does large data cause timeouts/jank?** | `Time.GetType.BySizeBucket`, `Time.UTF8Encode.BySizeBucket` |
| **Are there platform-specific regressions?** | `*.ByPlatform` variants |
| **Is the feature silently failing?** | `SilentFailure.*` counters, `EmptyBlobDivergence` |
| **How do developers actually use the API?** | `GetTypeCallsPerItem`, `MostRequestedType`, `TypesRequestedVsAvailable` |
