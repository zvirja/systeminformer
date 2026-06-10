# Win11 tray hover popup — appears very late / erratically / sometimes not at all

## Summary

On Windows 11 the tray mini-info popup is shown after a deliberate ~1 s hover delay. That delay is a
workaround for [#1643](https://github.com/winsiderss/systeminformer/issues/1643): stock Win11 sends
`NIN_POPUPOPEN` without honouring the tray hover time, so without the delay the popup would pop on the
slightest brush. The popup was appearing **very late and inconsistently (often several seconds), or not
at all** until the cursor happened to rest just right.

The root cause is in **how the 1 s delay timer was gated by the shell's hover notifications** — not in
the timer mechanism.

## Root cause

The delay used a `WM_TIMER` (`TIMER_ICON_POPUPOPEN`) **armed on `NIN_POPUPOPEN`** and **cancelled on
`NIN_POPUPCLOSE`** (via `PhNfpIconDisablePopupHoverWin11Workaround`). The shell delivers those edges
unreliably for the hover popup:

- `NIN_POPUPOPEN` can arrive **repeatedly during a single hover** — and every arm re-set the 1 s
  countdown, so it kept restarting and never reached 1 s.
- `NIN_POPUPCLOSE` can arrive **mid-hover with no real move-away** — and that cancelled the timer
  outright.

Either way the timer rarely matured, so the popup was late or never shown. It only appeared when the
hover happened to be quiet enough for a full second to elapse without a re-arm or a spurious close.

### The WM_TIMER-starvation theory was wrong (and was reverted)

An earlier attempt blamed `WM_TIMER` priority starvation under load and moved the delay onto an
`RtlCreateTimer` threadpool timer marshalled back to the UI thread. **It did not help** — the timer was
still re-armed/cancelled by the shell notifications regardless of mechanism — and it added real
complexity, so that change has been **reverted**. The mechanism (`WM_TIMER`) is fine; the arm/cancel
logic was the bug.

## The fix (`SystemInformer/notifico.c` only)

Stop trusting the open/close edges; drive the decision from the actual cursor position — exactly what
the original code comment said XP apps did before `NIN_POPUPOPEN` existed.

1. **Arm once per hover.** `NIN_POPUPOPEN` arms the timer only if it isn't already pending
   (`PopupTimerPending`), so repeated `NIN_POPUPOPEN` no longer resets the countdown.
2. **Ignore spurious close.** On Win11, `NIN_POPUPCLOSE` no longer cancels the pending timer or clears
   the popup target; it only issues the existing debounced unpin, which still dismisses an
   *already-shown* popup when the user really leaves.
3. **Validate at maturity.** When the timer fires, `PhNfpIsCursorOverNotifyIcon()` checks the real
   cursor position against the icon rect (`Shell_NotifyIconGetRect`) before showing. If the rect can't
   be resolved it degrades to "show" (no regression).

The 1 s timing and the click-activate / restore-hover timers are unchanged.

## Environment note (important for reproduction)

Testing here uses a third-party shell that **adds its own tray hover delay**, so locally
`NIN_POPUPOPEN` arrives ~900 ms after hover and the open/close pairs look clean (one pair per hover).
On a **stock Win11 shell that delay is absent** — which is the whole reason the 1 s workaround exists,
so it must stay. Keep this in mind when comparing event logs from different shells.

## Known remaining issue (under investigation)

The popup can occasionally stay shown ("sticky"). When the delayed show races **after** a
`NIN_POPUPCLOSE` has already passed, there is no later close to dismiss it. Pinning down the exact
open/close/show/timer interleaving with the event logger is the next step — likely the show needs to be
paired with a cursor-position-driven teardown rather than relying on `NIN_POPUPCLOSE` alone.

## Build / test

Requires Visual Studio 2022 + the Windows SDK (standard System Informer prerequisites); the code path
only runs on **Windows 11 22H2+**.

- Hover a tray icon (e.g. CPU history) under load → the mini-info popup should appear ~1 s after hover,
  consistently, regardless of update churn.
- Move the cursor away → it dismisses (watch for the sticky case above).
- Single-click activation and double-click (no stray activation) behaviour unchanged.
