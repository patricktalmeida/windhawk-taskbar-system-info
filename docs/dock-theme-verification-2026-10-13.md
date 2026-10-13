# Dock theme compatibility investigation

Date: 2026-10-13. Local branch: `fix/dock-theme-compatibility`.
Base: `b9f97f2` (1.7.1). This is an unreleased fix; metadata is unchanged.

## Reports and evidence

- [Local issue #10](https://github.com/starychenko/windhawk-taskbar-system-info/issues/10)
  reports a disappearing widget with dock themes, working reservation on 1.3.3,
  and recovery when that older version runs before 1.7.1.
- [Upstream issue #6174](https://github.com/ramensoftware/windhawk-mods/issues/6174)
  reports a custom, unpublished TaskbarTweaks island no longer working after
  the outer element changed from `Grid` to `Viewbox`.

The production code reproduced the first symptom in an isolated Windows XAML
Island: a centered, auto-sized frame with 300 DIP of buttons stayed 300 DIP wide,
with widget width 0 and reservation 0. The solver required existing free space
before requesting a margin that would create that space. This is consistent
with the older-mod workaround, but does not prove the exact reporters' setups.

The named outer element also changed type, invalidating selectors targeting
`Grid#WindhawkTaskbarSystemInfo`. The unpublished custom theme is unavailable,
so its full selector set and exact failure have not been reproduced.

A padded/bordered dock fixture exposed another concrete defect: obstacle
coordinates included the root's insets while placement margins did not. The
rendered widget consequently lost the required clearance from the buttons.

## Fix

- Restore the named outer `Grid` and retain uniform downscaling in an inner
  `Viewbox`. This preserves the root selector, not every historical child path.
- When reservation is enabled and no readable placement fits an auto-sized
  dock, propose a bounded reservation. Keep the widget collapsed until the next
  layout pass verifies actual button and widget bounds. Restore the original
  margin if the theme rejects the reservation; do not retry unchanged failures.
- Include the available parent width in placement invalidation so a hidden
  widget can retry after its parent grows.
- Use the root's content area consistently for obstacle measurement, placement
  and screen coordinates in the move preview.

No theme width is overwritten. Fixed-width/full-width placement continues to
use the existing solver. No settings were added or migrated.

## Verification

Environment: Windows 11 26H2, build 26300.9550, Windhawk engine 1.7.3.
The UI test uses a hidden XAML Island; it does not inject into Explorer.

- New regression failed on the original production code with
  `DOCK root=300 widget=0 reserve=0`.
- Corrected cold-start fixture: `DOCK root=728 widget=410 reserve=428`.
- UI checks cover the Grid selector, root padding/border clearance, reservation
  toggling, stable repeated layouts, constrained parents, growth recovery,
  a theme overriding the trial margin, and restoration on unload.
  A cold padded dock with a separate tray also passed, including agreement
  between its rendered widget position and the move editor's screen projection.
- `python tests/validate-source.py`: passed, 29 settings.
- `tests/run-regression.ps1`: 173,469 behavioral checks passed.
- `tests/run-ui-smoke.ps1`: existing 105 XAML cases, five preview DPI scales,
  four adaptive layouts and 200,000 generated real-font geometries passed.
- x64 build and ARM64 cross-build: passed without compiler diagnostics.
- `git diff --check`: passed.
- Fresh dock/full/compact render images were inspected for layout and readability.

The exact Frosted Glass configuration was not supplied, and Taskbar Styler is
not installed on this workstation. Live Explorer interaction with the reported
presets, physical multi-monitor dragging, and ARM64 hardware remain unverified.
Local verification did not change the installed mod or Windows settings and
did not publish a release.

## Isolated before/after renders

Both images use the same 1200 x 48 DIP fixture with a centered, auto-sized dock
and a 300 DIP test button group. These are XAML renders, not Explorer captures.
The unavailable metric placeholders are intentional; no live collectors run.

Before, using production source from `b9f97f2`:

![Original source: dock without the widget](../assets/dock-regression-before.png)

After, using this branch:

![Corrected source: reserved space and visible widget](../assets/dock-regression-after.png)
