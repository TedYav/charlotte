---
title: "fix: Resolve screenshot stale compositor frame regression in v0.6.0"
type: fix
status: active
date: 2026-04-04
---

# fix: Resolve screenshot stale compositor frame regression in v0.6.0

## Overview

Charlotte v0.6.0 screenshots capture a stale compositor frame (loading spinner) instead of rendered SPA content on route transitions. The accessibility tree (via `charlotte_observe`) shows all 48 elements correctly, proving the DOM is fully rendered, but `charlotte_screenshot` captures the old loading state. This is a regression from v0.5.1 where the observe-before-screenshot workaround produced correct images. PR #120's double-rAF compositor flush (included in v0.6.0) does not resolve the issue.

## Problem Frame

When navigating to a new SPA route via `charlotte_navigate`, the sequence is:
1. `page.goto(url)` waits for the `load` event (React shell loads)
2. React bootstraps and shows a loading spinner
3. React finishes rendering the route content (DOM updated)
4. `charlotte_observe` reads accessibility tree via CDP — shows correct content
5. `charlotte_screenshot` calls `waitForCompositorFrame()` (double-rAF) then `page.screenshot({ fullPage: true })` — captures stale spinner frame

The core issue is a **compositor surface synchronization gap**: the double-rAF synchronizes with the renderer main thread's frame scheduling but does not guarantee the compositor surface has been updated. Puppeteer's `Page.captureScreenshot` with `fromSurface: true` (the default) reads from the compositor surface, which may still hold the stale frame.

## Requirements Trace

- R1. Screenshots after SPA route transitions must capture the rendered page content, not a stale loading state
- R2. The fix must not regress non-SPA screenshots (static pages, simple DOM mutations)
- R3. The fix must work in headless Chromium with `--disable-gpu`
- R4. Diagnostic logging must indicate when the compositor flush mechanism falls back to timeout
- R5. Open a draft PR with the fix in the fork

## Scope Boundaries

- This fix targets the screenshot compositor synchronization only
- Not addressing the separate issue of `charlotte_navigate` returning before React hydrates (the 0-elements problem in step 3)
- Not changing the screenshot API surface (no new parameters exposed to users)
- Not upgrading Puppeteer

## Context & Research

### Relevant Code and Patterns

- `src/tools/observation.ts:372-506` — `charlotte_screenshot` tool handler; calls `waitForCompositorFrame(page)` at line 418 before `page.screenshot()`
- `src/tools/tool-helpers.ts:463-486` — `waitForCompositorFrame()` implementation: double-rAF with 1s timeout race
- `src/tools/navigation.ts:40-93` — `charlotte_navigate`; uses `page.goto(url, { waitUntil: "load" })`
- `src/browser/browser-manager.ts` — Launch args include `--disable-gpu`; `defaultViewport` now set in constructor (v0.6.0 change)
- `tests/integration/observation.test.ts:266-322` — Existing compositor flush tests (only test synchronous DOM mutations, not async SPA renders)
- `tests/fixtures/pages/dynamic.html` — Existing dynamic fixture (synchronous button click mutation)
- `tests/fixtures/pages/spa.html` — Existing SPA fixture (hash-based routing, no async rendering)

### Root Cause Analysis

**Puppeteer v24.37.3 screenshot internals for `fullPage: true`:**

1. `captureBeyondViewport` defaults to `true` (set by `setDefaultScreenshotOptions` in `node_modules/puppeteer-core/lib/cjs/puppeteer/api/Page.js:116`)
2. `fromSurface` defaults to `true` (line 112)
3. When `fullPage: true` AND `captureBeyondViewport: true`: Puppeteer does NOT resize the viewport. It passes `captureBeyondViewport: true` directly to CDP's `Page.captureScreenshot`
4. CDP `Page.captureScreenshot` with `fromSurface: true` reads from the compositor **surface** — the actual pixel buffer

**The double-rAF gap:**
- rAF callbacks fire on the renderer's main thread BEFORE paint
- When the 2nd rAF resolves, layout is computed but the compositor surface may not have been updated
- For SPA route transitions (large DOM tree swap), the compositor may need additional time to rasterize the new frame to the surface
- With `captureBeyondViewport: true`, no viewport resize occurs, so there's no forced recomposite between the rAF and the screenshot

**Why v0.5.1's observe-before-screenshot workaround worked:**
- In v0.5.1, `defaultViewport` was NOT in the Puppeteer launch options (it was added in v0.6.0 via PR #138)
- Puppeteer's default viewport was 800x600
- The user then set viewport to 1440x900 via `charlotte_viewport`
- The observe call's CDP `DOM.getBoxModel` queries may have triggered layout recomputation that, combined with the different viewport initialization path, produced enough compositor activity for the surface to update
- In v0.6.0, `defaultViewport: { width: 1440, height: 900 }` is set in launch options, changing the initialization timing and eliminating some of these accidental compositor flushes

### External References

- Puppeteer `Page.captureScreenshot` CDP spec: `captureBeyondViewport` controls whether content beyond the viewport is captured; `fromSurface` controls whether the capture reads from the compositor surface vs the view
- Chromium compositor: the surface is the final pixel buffer; it lags the main thread's DOM/layout state by at least one frame
- `requestAnimationFrame` fires before paint (pre-paint callback), not after the compositor has rasterized

## Key Technical Decisions

- **Fix via `captureBeyondViewport: false`**: When `fullPage: true`, set `captureBeyondViewport: false` on the Puppeteer screenshot call. This forces Puppeteer to resize the viewport to full page dimensions before capturing, which triggers a full recomposite and ensures the surface is updated. This is the most targeted fix with the highest confidence of resolving the surface-lag issue.
- **Enhance `waitForCompositorFrame` with layout flush**: After the double-rAF, force a synchronous layout computation (`document.body.offsetHeight`) to ensure pending style/layout changes are fully processed before the screenshot. This strengthens the rAF approach as a belt-and-suspenders measure.
- **Add diagnostic logging**: Log a warning when the 1-second timeout wins the race, making this failure mode visible instead of silent.

## Open Questions

### Resolved During Planning

- **Does the double-rAF actually fire in headless Chrome?** Yes — "new headless" mode (Puppeteer 24.x `headless: true`) has a full rendering pipeline with rAF support. The issue is not that rAF doesn't fire, but that the compositor surface isn't guaranteed to be updated when rAF resolves.
- **Did any v0.6.0 change alter the screenshot code path?** No — the screenshot handler's core logic is unchanged. `ensureReady()` replaced `ensureConnected()` but is functionally equivalent when the browser is running. The `waitForCompositorFrame` call is still at the same position.
- **Did launch args change?** The Chrome flags are identical. The key change is `defaultViewport` now being set in launch options (PR #138), which changes Puppeteer's page initialization path.

### Deferred to Implementation

- Exact performance impact of `captureBeyondViewport: false` (two extra viewport changes per fullPage screenshot) — measure during testing
- Whether element-level screenshots (`selector` parameter) need the same fix — the selector path uses `element.screenshot()` which may have different compositor behavior

## High-Level Technical Design

> *This illustrates the intended approach and is directional guidance for review, not implementation specification. The implementing agent should treat it as context, not code to reproduce.*

```
Before (broken):
  waitForCompositorFrame()     page.screenshot({fullPage:true})
  ┌──────────────────────┐     ┌─────────────────────────────────┐
  │ double-rAF resolves  │ ──► │ captureBeyondViewport: true     │
  │ (main thread synced) │     │ fromSurface: true               │
  │                      │     │ → reads STALE compositor surface │
  └──────────────────────┘     └─────────────────────────────────┘

After (fixed):
  waitForCompositorFrame()     page.screenshot({fullPage:true})
  ┌──────────────────────┐     ┌───────────────────────────────────────┐
  │ double-rAF resolves  │     │ captureBeyondViewport: false           │
  │ + layout flush       │ ──► │ → Puppeteer resizes viewport to full  │
  │ + timeout logging    ��     │ → FORCES full recomposite             │
  └──────────────────────┘     │ → captures FRESH surface              │
                               │ → restores original viewport          │
                               └───────────────────────────────────────┘
```

## Implementation Units

- [ ] **Unit 1: Create async SPA test fixture and reproduce the regression**

**Goal:** Create a test fixture that simulates a real async SPA route transition (loading spinner → rendered content) and write a failing test that captures the stale screenshot bug.

**Requirements:** R1, R2

**Dependencies:** None

**Files:**
- Create: `tests/fixtures/pages/spa-async.html`
- Modify: `tests/integration/observation.test.ts`

**Approach:**
- Build an HTML page with inline JS that: shows a loading spinner on initial render, then after a short async delay (microtask or rAF-based, not `setTimeout` — to match real framework behavior), replaces the spinner with rich content (table, navigation, text)
- The fixture should simulate React's render pattern: initial shell with spinner → async state update → full content render
- Add a failing integration test: navigate to the fixture → wait for content to render (verify via DOM query) → take screenshot → assert screenshot is not the tiny spinner image (use file size or pixel sampling as the assertion)
- This test should fail on current code, confirming the regression

**Patterns to follow:**
- Existing fixture pattern: `tests/fixtures/pages/dynamic.html`, `tests/fixtures/pages/spa.html`
- Existing test pattern: `tests/integration/observation.test.ts:266-322`

**Test scenarios:**
- Happy path: Navigate to async SPA fixture → DOM shows rendered content → screenshot should capture rendered content (NOT spinner). Assert screenshot base64 length > 5000 (spinner images are ~1-2KB base64, rendered pages are much larger)
- Edge case: Navigate to async SPA fixture → screenshot immediately (before async render completes) → should still capture whatever is currently on screen without crashing

**Verification:**
- The new test fails when run against the current codebase, confirming the regression is reproducible in a controlled environment

- [ ] **Unit 2: Enhance `waitForCompositorFrame` with layout flush and timeout logging**

**Goal:** Strengthen the compositor flush mechanism by adding a synchronous layout force after the double-rAF, and add diagnostic logging when the 1-second timeout wins the race.

**Requirements:** R3, R4

**Dependencies:** Unit 1 (for verification)

**Files:**
- Modify: `src/tools/tool-helpers.ts`

**Approach:**
- After the `Promise.race([rafFlush, timeout])` resolves, track which branch won (rAF vs timeout)
- If timeout wins: log a warning via the existing `logger` — this makes the failure mode visible in debugging
- After the race (regardless of winner), add a layout flush: `page.evaluate(() => void document.body.offsetHeight)` inside a try/catch (same silent-failure pattern as the rAF)
- The layout flush forces the browser to synchronously compute layout, which processes pending DOM changes and pushes them closer to the compositor

**Patterns to follow:**
- Existing logging pattern: `logger.warn(message, context)` used throughout the codebase
- Existing silent-failure pattern: the outer try/catch in the current `waitForCompositorFrame`

**Test scenarios:**
- Happy path: `waitForCompositorFrame` on a normal page → rAF resolves first, no warning logged, layout flush runs
- Edge case: `waitForCompositorFrame` on `about:blank` → rAF and layout flush both fail silently, no crash
- Edge case: Timeout wins the race → warning is logged (verify via logger mock or spy)

**Verification:**
- Existing tests continue to pass
- New timeout logging is visible when debugging stale screenshot scenarios

- [ ] **Unit 3: Fix screenshot to force compositor recomposite via `captureBeyondViewport: false`**

**Goal:** Fix the stale compositor surface issue by setting `captureBeyondViewport: false` on full-page screenshots, forcing Puppeteer to resize the viewport (which triggers a full recomposite) before capturing.

**Requirements:** R1, R2, R3

**Dependencies:** Unit 1 (test must now pass), Unit 2 (enhanced flush)

**Files:**
- Modify: `src/tools/observation.ts`
- Modify: `tests/integration/observation.test.ts`

**Approach:**
- In the `charlotte_screenshot` handler, add `captureBeyondViewport: false` to the `page.screenshot()` call for the full-page (non-selector) path
- This changes Puppeteer's behavior: instead of passing `captureBeyondViewport: true` to CDP (which reads the possibly-stale surface), it resizes the viewport to the full page dimensions, takes the screenshot, then restores the viewport
- The viewport resize triggers `Emulation.setDeviceMetricsOverride` which forces a full recomposite
- The selector-based screenshot path (`element.screenshot()`) does not need this change
- Verify the fix by running the failing test from Unit 1 — it should now pass
- Verify non-SPA screenshots still work correctly (existing tests)

**Patterns to follow:**
- The same `page.screenshot()` options pattern already used in the codebase

**Test scenarios:**
- Happy path: Async SPA fixture → screenshot captures rendered content (the Unit 1 test now passes)
- Happy path: Static page → screenshot still captures correctly (existing test)
- Happy path: DOM mutation → screenshot captures post-mutation state (existing test)
- Edge case: `about:blank` → screenshot still works (no content to capture, no crash)
- Integration: Navigate → observe (confirms content) → screenshot (captures same content observe saw)

**Verification:**
- The failing test from Unit 1 now passes
- All existing observation integration tests pass
- Full test suite passes

- [ ] **Unit 4: Open draft PR**

**Goal:** Create a feature branch, commit the fix, and open a draft PR for review.

**Requirements:** R5

**Dependencies:** Unit 3

**Files:**
- All modified files from Units 1-3

**Approach:**
- Create a branch from main
- Commit the fixture, tests, and fix with a clear commit message referencing the regression and PR #120
- Open a draft PR with a summary of the root cause and fix

**Test expectation:** none -- this is a pure process step

**Verification:**
- Draft PR is open and accessible for review
- All CI checks pass (if configured)

## System-Wide Impact

- **Interaction graph:** The fix only changes the screenshot capture path. No other tools (observe, navigate, click, etc.) are affected. The `waitForCompositorFrame` enhancement benefits all callers but currently only the screenshot tool calls it.
- **Error propagation:** The layout flush addition uses the same silent-failure pattern as the existing rAF flush. No new error paths are introduced.
- **State lifecycle risks:** The `captureBeyondViewport: false` change causes two additional viewport changes (resize to full page, restore to original). If the page has responsive behavior that triggers state changes on resize, this could theoretically affect subsequent interactions. However, the viewport is restored immediately after the screenshot.
- **API surface parity:** No API changes. The fix is internal to the screenshot implementation.
- **Unchanged invariants:** The screenshot tool's external API is unchanged. The `save`, `output_file`, `selector`, `format`, and `quality` parameters all work as before.

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| `captureBeyondViewport: false` causes viewport resize that triggers responsive SPA re-render during screenshot | Viewport is restored immediately; the screenshot captures the resized state which should match the original content. Monitor for reports of screenshots showing different layouts. |
| Performance regression from two extra viewport changes per full-page screenshot | The viewport changes are CDP commands, not page navigations. Expected overhead is <100ms. Acceptable for correctness. |
| Layout flush (`document.body.offsetHeight`) causes side effects on pages with scroll-linked animations | The layout query is read-only and does not trigger scroll events. Extremely low risk. |

## Sources & References

- Related PRs/issues: #120 (original compositor fix), #138 (lazy init — introduced `defaultViewport` in launch options)
- Puppeteer source: `node_modules/puppeteer-core/lib/cjs/puppeteer/api/Page.js` (screenshot defaults and fullPage handling)
- Puppeteer source: `node_modules/puppeteer-core/lib/cjs/puppeteer/cdp/Page.js` (CDP `Page.captureScreenshot` call)
- Reproduction steps: `.claude/tmp/steps-to-reproduce.md`
