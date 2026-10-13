# Upstream review follow-up for PR #6176

The upstream AI review identified oscillation after a successful auto-sized dock
reservation. The baseline retained the grown width but subtracted the button
shift. A preferred position toward the right could select that synthetic trailing
gap with no reservation, shrinking the dock and restarting the trial.

An isolated XAML regression reproduced the failure on published 1.7.2:
`Saved right-side dock position oscillates after reservation`. The fixture uses
a centered dock, 600-DIP button group, 1200-DIP scene and saved fractions 0.6,
0.75 and 0.9.

The fix records a confirmed dock reservation and restricts placement to
reservation slots while the owned margin is active. Rollback, external margin
replacement, reservation removal and teardown clear the state. It is included
in the placement cache key. The move editor receives the same restriction in
its taskbar-thread geometry snapshot, for initial and subsequent candidates.

The optional auto-size guard is implemented: a root/frame pair that is both
horizontally stretched does not enter the auto-sized dock trial path.

Retry backoff is deferred. Changes to obstacles or available geometry can make
placement valid, so they remain retry triggers. Unchanged rejected geometry
already stops retrying after the restored baseline is recorded. Post-layout
validation remains necessary for centered docks and overridden margins.

Tests assert stable layout revisions and a positive reservation for all three
fractions, and exercise the real move candidate search to reject the synthetic
trailing gap. Existing shrink/grow, external-margin, settings-only retry,
disable/re-enable and unload coverage also passes.

This follow-up was developed after the initial companion 1.7.2 release and
was subsequently included in the October 13, 2026 refresh of its tag and asset.
Live Explorer with the reporters' exact themes and physical
mixed-DPI monitor transfers remain unverified.

## Second review: switching themes

The second AI review found no blockers but noted that the confirmed-reservation
flag survived a switch from a dock to a stretched or fixed-width taskbar if the
theme retained the repeater margin. A new XAML regression reproduced this on
the first follow-up: `Dock-to-normal theme switch retains reservation-only placement`.

Placement now clears the flag when `AutoSizedDockAvailableWidth()` returns zero.
A trial that finishes after the theme changes also cannot reactivate the flag.
The regression switches a live fixture from a dock to stretched and fixed-width
layouts without reinjection, checks the saved right-side placement and unrestricted
move projection, and verifies stable, non-overlapping placement when switching
back to a dock. That reverse transition also exposed a path where a visible
widget keeps the root wide enough for the normal solver to apply a reservation
without a cold trial. Such a reservation now also activates the dock constraint,
preventing another reserved/unreserved oscillation.
