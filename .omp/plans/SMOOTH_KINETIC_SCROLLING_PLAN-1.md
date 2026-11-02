# Smooth Kinetic Scrolling (revised)

## Context

Current implementation (committed) animates scrollbar target via QTimer but visual result still scrolls line-by-line. Root cause: paint translate is applied but updateImage() re-fetches screen window at integer floor each tick, discarding fractional offset; clip region expansion and translate aren't producing visible sub-line motion because drawContents uses _image indexed by integer lines and the translate is restored before paintFilters but the underlying image is already repositioned. Need robust fractional rendering and a user-facing toggle.

## Approach

1. Add `smoothScrolling` bool property to TerminalDisplay (Q_PROPERTY + getter/setter), default true, persisted via QSettings in app? App-level setting in SettingsGeneralTab bound to terminal.smoothScrolling.

2. When disabled, revert to original instant behavior: wheelEvent calls `_scrollBar->handleEvent(ev)` directly.

3. Fix fractional rendering: in smoothScrollTick, do NOT call updateImage() on every floor change? Instead keep _screenWindow at current integer floor and apply _scrollOffset translate in paint. Only scrollTo when floor changes. Ensure paint's translate uses _scrollOffset and clip expanded. Keep updateImage only when floor changes, but updateImage resets image at new floor; offset then 0-ish? Problem: after scrollTo(floor), _scrollPos fractional remainder should drive offset. Already computed before. updateImage resets state only if not animating guard; we guard with `_scrollAnimating` so state preserved. Good.

4. Debug: ensure _fontHeight valid; offset nonzero. Add logging? No.

5. Add setting UI: SettingsTerminalTab.qml checkbox "Smooth scrolling" bound to terminal smoothScrolling property; store in Storage.qml / app settings.

## Critical files

- qmltermwidget/lib/TerminalDisplay.h/.cpp
- app/qml/SettingsTerminalTab.qml
- app/qml/Storage.qml (if exists)

## Verification

Build and run; wheel scroll shows smooth sub-line motion; toggle disables.

## Assumptions

Setting default on. If paint translate insufficient due to FBO render target, fall back to software (useFBORendering false) during animation.