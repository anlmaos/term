# Smooth Kinetic Scrolling

## Context

cool-retro-term renders terminal history through `QMLTermWidget` (`TerminalDisplay`), where mouse-wheel scrolling currently jumps instantly by whole lines via `ScrollBar::handleEvent` -> `scrollBarPositionChanged` -> `updateImage`. The goal is smooth, animated wheel scrolling with momentum-like catch-up instead of instantaneous line jumps.

## Approach

1. Add smooth-scroll state and timer to `qmltermwidget/lib/TerminalDisplay.h` / `.cpp`:
   - Members: `qreal _scrollPos` (fractional scrollbar/top-line position), `qreal _scrollTarget`, `qreal _scrollOffset` (pixels, applied to painted content), `bool _scrollAnimating`, `QTimer *_scrollTimer`.
   - Initialize in constructor: `_scrollPos = 0`, `_scrollTarget = 0`, `_scrollOffset = 0`, `_scrollAnimating = false`, create `_scrollTimer` with 16 ms interval connected to new slot `smoothScrollTick()`.
   - Add slot `smoothScrollTick()`.

2. Replace instantaneous wheel handling in `TerminalDisplay::wheelEvent` for the scrollback path (`_mouseMarks && _scrollBar->maximum() > 0`):
   - Compute `increment` exactly as `ScrollBar::handleEvent` does (pixel/angle delta, Control modifier page step, single-step fallback).
   - Set `_scrollTarget = qBound(0.0, _scrollTarget - increment, (qreal)_scrollBar->maximum())`.
   - If not animating, initialize `_scrollPos = _scrollBar->value()` and start `_scrollTimer`; set `_scrollAnimating = true`.
   - Accept the event; do not call `_scrollBar->handleEvent`.

3. Implement `smoothScrollTick()`:
   - Ease `_scrollPos` toward `_scrollTarget`: `_scrollPos += (_scrollTarget - _scrollPos) * 0.25`.
   - If `qAbs(_scrollTarget - _scrollPos) < 0.05`, snap `_scrollPos = _scrollTarget`, set `_scrollAnimating = false`, stop timer.
   - Compute `_scrollOffset = -(_scrollPos - std::floor(_scrollPos)) * _fontHeight`.
   - If `std::floor(_scrollPos)` changed from the last rendered floor (track `_scrollRenderedLine`, init in constructor), call `_screenWindow->scrollTo((int)std::floor(_scrollPos))` then `updateImage()`; else call `update()`.
   - Clamp `_scrollPos`/`_scrollTarget` to `[0, maximum]` each tick.

4. Keep `updateImage()` from clobbering fractional state:
   - In `updateImage()`, after `setScroll(...)`, if `_scrollAnimating` is true, do not reset `_scrollPos`/`_scrollTarget`; otherwise set `_scrollPos = _scrollTarget = _scrollBar->value()` and `_scrollOffset = 0`.

5. Keep scrollbar drag/output paths working:
   - In `scrollBarPositionChanged(int)` (existing slot), if not `_scrollAnimating`, reset `_scrollPos = _scrollTarget = _scrollBar->value()` and `_scrollOffset = 0` before `updateImage()`.

6. Apply fractional offset in `TerminalDisplay::paint`:
   - Expand clip region to full contents rect plus one line height when offset nonzero.
   - Wrap the content draw loop (drawBackground/drawContents for each rect) and preedit string in `painter->save()` / `translate(0, _scrollOffset)` / `restore()`.

7. Add `smoothScrolling` reset on font change: in `setVTFont`/font metric changes, if not animating, reset offset/pos to current scrollbar value.

## Critical files & anchors

- `qmltermwidget/lib/TerminalDisplay.h` — `TerminalDisplay` class, members around `_topMargin`, signals/slots section, constructor declaration.
- `qmltermwidget/lib/TerminalDisplay.cpp` — `wheelEvent`, `updateImage`, `paint`, `scrollBarPositionChanged`, constructor.
- `qmltermwidget/lib/ScreenWindow.h` — `scrollTo(int)` (unchanged, reused).

## Verification

- Build submodule and app with Qt6: `cd /home/vi/Code/term && qmake6 cool-retro-term.pro && make -j$(nproc)`.
- Run `./cool-retro-term`, open scrollback (e.g. `seq 1 1000`), wheel-scroll; observe animated catch-up instead of instant jumps; scrollbar thumb tracks fractional position.
- Edge: scroll to top/bottom clamps without overshoot; new output while scrolled up does not crash; resize during animation resets offset.
- Grep check: `grep -n "smoothScrollTick\|_scrollAnimating" qmltermwidget/lib/TerminalDisplay.cpp`.

## Assumptions & contingencies

- Default smooth scrolling always on; no settings toggle in this change. If build fails due to QQuickPaintedItem transform/clip issues, fall back to integer-line animation (remove `_scrollOffset` paint translate, call `updateImage()` each tick) and keep target easing.
- If `_screenWindow` is null during tick, skip scrollTo and only update.
