# Poster Generation Audit

**Audited build:** `923d803`  
**Date:** 14 October 2026  
**Scope:** 3 templates (T1/T2/T3), 7 output formats, text sizing/pills, photos/frames, random generation, export and mobile-facing controls.

## Test method

1. Source review of every renderer and the shared text/photo/export helpers.
2. Browser stress render using near-maximum user-field content across **3 templates × 7 sizes = 21 renders**.
3. Browser random-generation run: **150 consecutive Random Poster generations**.
4. Browser control comparison for Auto Fill, Force Crop and Smart Auto Zoom.
5. Browser PNG export test at 150 DPI, including inspection of the PNG `pHYs` metadata.
6. Mobile screenshot review supplied by the user.

## Executive result

The editor has a good base: default posters render, all 21 stress renders completed without page errors, and 150 random designs did not create a canvas-edge overflow. However, the audit found real generation-quality and control-behaviour defects. The most serious are text overflow/truncation in T2/T3, misleading photo controls, insufficient feedback for unsafe content, and absent edit-session persistence.

---

## Critical issues — fix first

### 1. T2 portrait `voteLine2` can overflow outside the canvas

**Severity:** Critical  
**Where:** T2 portrait renderer (`drawT2P`)  
**Cause:** `voteLine2` is drawn with a fixed large font and no final width fitting/max-width guard.

**Reproduction:** Enter a long vote message in “वोट पंक्ति 2”, e.g. repeated words. At WhatsApp Status size it extends past the right canvas edge. The stress run recorded a 1,146 px text width starting at x=270 on a 1,080 px canvas.

**User impact:** Exported poster can contain text cut off on the right side.

**Required correction:** Fit to the card’s available width before paint; if still too long, use two deliberate lines or reject/warn instead of silently drawing outside.

---

### 2. T3 promise/point text silently loses content

**Severity:** Critical  
**Where:** T3 portrait, square, landscape and banner point cards  
**Cause:** The renderers use `wrapLines(...)[0]`, which draws only the first wrapped line. The remaining text is discarded without an ellipsis or warning.

**User impact:** A voter-facing promise can be incomplete or misleading in the exported poster. On compact layouts the first line can also push into its neighbouring column.

**Required correction:** Use a per-cell two-line renderer with a hard line limit, font fitting, and visible ellipsis/warning when the text cannot be represented safely.

---

### 3. `wrapFit` does not enforce its declared maximum-line limit

**Severity:** High  
**Where:** Headlines, subheads, qualities and slogan blocks  
**Cause:** `wrapFit` reduces font size while possible, but returns every line even when the result still exceeds `maxLines` at the minimum font size.

**User impact:** Long custom copy may enter the following section, overlap another row or make the layout unreadable. The random pools usually avoid this, but manually entered content does not.

**Required correction:** Return only the permitted number of lines and add an ellipsis to the final line, plus show a field-level “text shortened” warning. Do not render extra lines.

---

### 4. One unbroken long word is not safely handled

**Severity:** High  
**Where:** All `wrapLines` blocks  
**Cause:** Wrapping is based only on spaces. A long Hindi/English token, URL, phone sequence or pasted text with no spaces remains wider than the available area.

**User impact:** A custom label can escape its card/pill despite ordinary spaced test text appearing correct.

**Required correction:** Add character-level breaking or shrink-to-fit for oversized individual tokens; retain a visible truncation warning if a safe fit remains impossible.

---

## Major issues

### 5. “Force Crop” currently produces the same image as “Auto Zoom”

**Severity:** High (functional mismatch)  
**Where:** Photo fit controls  
**Proof:** Browser canvas comparison found `Auto Fill` and `Force Crop` output to be identical for the same image/frame.

**Cause:** Both modes use the same cover scale branch in `containPhoto`.

**Required correction:** Give Force Crop a distinguishable crop/zoom rule, or remove it and retain only a clearly labelled Auto Fill mode plus Full Fit.

---

### 6. “Smart Auto Zoom” toggle currently does not change the rendered output

**Severity:** High (functional mismatch)  
**Where:** Smart Auto Zoom checkbox  
**Proof:** With the same photo/frame, browser canvas output before and after turning the toggle off was identical.

**Cause:** The toggle changes status/reset behaviour but does not change the fitting method used by `containPhoto`.

**Required correction:** When disabled, preserve manual transform and stop automatic source/frame reset; when enabled, apply a documented adaptive source/crop rule. The current label should not claim a visual mode that is not changing the render.

---

### 7. Non-adaptive T2 banner/trapezoid labels remain a weak point

**Severity:** High  
**Where:** T2 post badge across portrait/square/landscape/banner layouts  
**Cause:** The label sits in fixed trapezoid artwork rather than a dynamically measured text background. Text is reduced/ellipsized, but the container does not follow the label.

**User impact:** This is inconsistent with the expected “pill follows its text” rule, especially for custom long post titles.

**Required correction:** Either convert those labels to the shared adaptive text-pill renderer or implement a dynamic trapezoid with measured width, padding and font-fit rules.

---

### 8. Some fixed cards are treated as text containers without a hard safe-content policy

**Severity:** High  
**Examples:** T2 vote card, T3 CTA/contact card, T1 question cards.

**Cause:** Several cards rely on a mixture of fixed coordinates, one-line text and partial font fitting. They do not consistently share a `fit/wrap/ellipsis/warning` policy.

**User impact:** Long contact lines, requester strings and campaign text can become too small, collide with icons, or truncate unpredictably.

**Required correction:** Create one reusable `safeTextBlock()` helper for fixed cards. It should reserve icon space, set a hard line count, fit text, and report overflow to the UI.

---

## Medium-priority issues

### 9. No user-facing warning when content is shortened, too small or omitted

**Severity:** Medium  
**Cause:** Some text becomes an ellipsis, some is silently clipped to the first line, and some shrinks to a very small font. The editor does not tell the user which outcome occurred.

**Required correction:** Add a live “Poster safety check” section that reports field name, affected template/size, and action taken (wrapped, reduced, shortened, or unsafe).

---

### 10. Text readability varies heavily with template and copied content

**Severity:** Medium  
**Evidence:** Stress screenshots show footer requester/contact text can become visually too small even while technically inside the canvas. T3 point text becomes dense at Status size.

**Required correction:** Define format-specific minimum legible font sizes. If a field cannot fit at that threshold, use fewer words/lines or flag it as unsafe rather than rendering microtext.

---

### 11. The photo-frame issue is difficult to verify from the normal UI

**Severity:** Medium  
**Context:** The supplied mobile screenshot shows the older outer-oval fade visible beyond the decorative frame ring. The current audited source uses an inner-oval aperture; this indicates browser cache/build-version ambiguity is possible.

**Required correction:** Display a small build/version identifier in the editor and add a “Frame preview background: none” test mode. This will make it immediately clear whether a viewer is on the latest build and whether any pixels are outside the actual aperture.

---

### 12. User edits are lost on a normal page reload

**Severity:** Medium  
**Proof:** Entering `टेस्टनाम` in the name field and reloading reset it to the default `पुनाराम`.

**Required correction:** Store form state, selected size/template, DPI, fit mode and transforms in local storage; provide a visible “Reset saved draft” option.

---

### 13. DPI choices can be misunderstood as increased visual resolution

**Severity:** Medium  
**Current state:** DPI metadata export works correctly. 150 DPI export was verified as a PNG `pHYs` value of 5,906 pixels/meter and as a `150dpi` filename.

**Risk:** Changing DPI does not add pixels. A 1080×1920 status file remains 1080×1920 at 600 DPI; it is not a higher-detail WhatsApp image.

**Required correction:** Show physical print size beside DPI (for example, “1080×1920 at 300 DPI ≈ 3.6×6.4 in”) and label DPI as print metadata. Keep WhatsApp messaging focused on pixels/9:16.

---

### 14. Export lacks progress, busy-state and failure feedback

**Severity:** Medium  
**Risk:** A4 or repeated PNG exports can consume memory/time, especially on a phone. The button remains available and no status indicates that export is running or failed.

**Required correction:** Disable the download button while `toBlob` runs, show an export progress/status message, and display a clear error/retry action if blob generation fails.

---

## What passed in this audit

- **21 stress renders** (3 templates × 7 output formats) completed with no JavaScript/page errors.
- **150 Random Poster generations** completed with no canvas-edge text overflow and no page errors. This confirms that the curated random pools are materially safer than unrestricted manual copy.
- WhatsApp Status default selected correctly at **1080×1920**.
- DPI metadata export is working for selected presets.
- Photo manual controls and the core renderer load successfully in browser tests.

## Recommended remediation order

1. Build a shared `safeTextBlock()` / `safeWrappedText()` system and use it in every template.
2. Fix T2 portrait vote line and all T3 promise cells first.
3. Make `wrapFit` a hard max-line system with ellipsis + UI warnings.
4. Make Force Crop and Smart Auto Zoom genuinely different, or simplify their controls.
5. Convert remaining fixed post-label backgrounds to adaptive shapes.
6. Add a poster safety checker, draft persistence and export progress.
7. Add visible build ID / cache-busting support for diagnosing deployment differences.

---

## Corrective release verification — 14 October 2026

The following fixes were implemented after the findings above. The issue sections intentionally remain as the pre-fix audit record.

- **Hard text safety:** wrapping now breaks oversized unbroken tokens; every wrapped block has a hard maximum line count; the final allowed line is ellipsized if necessary. Single-line `txt()` calls now record visible ellipsis events instead of silently hiding copy. A live **Text safety** status tells the editor when any visible copy was shortened.
- **T2 vote card:** portrait vote copy uses two reserved, separately fitted rows with explicit width limits. Square, landscape and banner vote copy also have final width guards.
- **T3 promise cards:** all portrait, square, landscape and banner promise cells use a two-line `wrapFit()` renderer. No renderer uses `wrapLines(...)[0]` any longer.
- **Adaptive labels:** all four T2 post badges now use a measured dynamic trapezoid (font fitting, padding, width limit and contrast-aware ink). Shared rounded `textPill()` labels retain dynamic sizing.
- **Photo controls:** Auto Zoom, Full Fit and Force Crop now have materially different canvas output. Force Crop adds a 14% cover zoom; disabling Smart Auto Zoom switches Auto mode to a safe-contain presentation. The default remains frame-filling Auto Zoom.
- **Draft recovery:** text fields, template, format, DPI, font/appearance selections, photo transform/fit state and the built-in candidate choice persist in local storage. **Draft Reset** clears them deliberately.
- **Export clarity:** the UI shows format pixels, selected DPI and physical print size; export shows preparing/success/error feedback and disables the export button during PNG generation. PNG filenames and `pHYs` metadata use the selected DPI.
- **Release identification:** the editor shows `Build: safety-audit-20261014`.

### Corrective browser regression results

- **21/21** template × output-format maximum-content renders completed without browser/page errors. Six intentionally extreme cases showed the visible safety warning rather than allowing unreported overflow or omission.
- **150/150** Random Poster generations completed with **0 canvas-edge text overflows** and **0 page errors**.
- Canvas hashes proved the three photo states differ: Auto Zoom ≠ Force Crop, Auto Zoom ≠ Smart-Auto-Zoom-off, and Force Crop ≠ Smart-Auto-Zoom-off.
- A selected **4:5** candidate remained selected after **New Design/Random Poster** and after reload; a text edit (`टेस्टनाम`) also survived reload.
- A **150-DPI** download was named `sarpanch_poster_t2_s916_150dpi.png` and contained PNG `pHYs` values **5906 × 5906 pixels/meter**, unit `1`.
