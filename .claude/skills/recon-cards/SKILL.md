---
name: recon-cards
description: Replace the "All reconciliations" table in index copy.html with the approved status-aware card grid design. Use when asked to build/implement the reconciliations card grid, redesign the reconciliations list, or add a Table/Cards view toggle.
---

# Reconciliations card grid — implementation guide

This skill carries a design that was already reviewed and approved in a prior
session (a UX pass on the reconciliation switcher, then a card-grid redesign of
the reconciliations list, validated against the app's own status-color palette).
This file is the full spec — you do not need the original conversation or the
mockup artifact to build this; everything required is below.

## Objective

Replace the plain `<table>` inside `viewReconciliations()` with a card grid.
Each card shows one reconciliation definition with a match-rate ring, a status
chip, a colored top stripe, and the same stats the table currently shows
(open exceptions, unreconciled value, owner, due date, cadence, last run).
Clicking a card does what clicking a table row does today.

**This is a visual/structural change only.** Do not touch `arEngine`,
`getEngineResult`, or any matching/reconciliation logic. Do not change what
`recStatus()` computes — only how its output is displayed.

## Where the change goes

File: `index copy.html` (single-file prototype — the working copy; `index.html`
is a stale snapshot, do not edit it).

Function: `viewReconciliations()`. Find it by name — do not rely on a line
number, it will have drifted. As last read it looked like this:

```js
function viewReconciliations() {
  const all = DB.recs.map(rc => ({ rc, res: getEngineResult(rc) }));
  return topbar('All reconciliations', `${DB.period.label} · ${all.length} active definitions`, `<button data-act="new-recon" class="btn btn-primary">${icon('plus', 'w-4 h-4')}New reconciliation</button>`) + `<div class="flex-1 overflow-y-auto p-6 fade-in"><div class="flex items-center gap-2 mb-4"><div class="relative flex-1 max-w-sm"><span class="absolute left-3 top-2 text-faint">${icon('search', 'w-4 h-4')}</span><input class="field pl-9" placeholder="Search reconciliations"></div><button class="btn btn-ghost">${icon('filter', 'w-4 h-4')}Filters</button><button class="btn btn-ghost">Saved views</button></div><div class="card overflow-hidden"><table class="w-full text-[13px]">...</table></div></div>`;
}
```

Only the innermost `<div class="card overflow-hidden">...<table>...</table></div>`
needs to become the card grid. The `topbar(...)` call and the filter row above
it (search input, Filters button, Saved views button) stay as-is — see
"Out of scope" below.

## Data mapping — use exactly these, don't invent new fields

For each `{ rc, res }` in `all = DB.recs.map(rc => ({ rc, res: getEngineResult(rc) }))`:

| Card field | Expression | Notes |
|---|---|---|
| Name | `rc.name` | |
| Type label | `TYPE_META[rc.type].label` | `TYPE_META` is defined near the top of the file — keys: `payments, bank, close, ar, ap, payroll, inventory, ar-reconciliation` |
| Type icon | `icon(TYPE_META[rc.type].icon, 'w-3.5 h-3.5')` | reuse the existing `icon()` helper, don't add new SVGs |
| Owner | `rc.owner` | table currently also hardcodes `Reviewer · Alex Rivera` next to it — keep that as-is unless you have a real reviewer field |
| Cadence | `rc.cadence` | |
| Last run | `rc.lastRun ? 'last run ' + rc.lastRun.at.slice(0, 10) : 'never run'` | matches existing table subtext exactly |
| Match rate | `rc.lastRun ? res.rate : null` | ring shows `—` / idle style when `null` |
| Open exceptions | `rc.lastRun ? res.breaks.length : null` | |
| Unreconciled value | `rc.lastRun ? res.breakVal : null` | format with `fmtMoney()` |
| Status text | `recStatus(rc, res)` | **reuse this function verbatim** — see below |
| Due date | `` `Jul ${20 + (rc.name.length % 5)}` `` | ⚠️ this is placeholder/fake data already in the current table, not a real schema field. Keep it as-is for parity unless you're asked to wire a real due date — don't quietly invent a different fake value, and flag to the user if a real `dueDate` field should be threaded through instead. |

## Status → chip/stripe mapping

`recStatus(rc, res)` already exists and returns one of four strings. **Reuse it
as the single source of truth for status text — do not re-derive status from
`res.rate`/`res.breaks` independently, or the card and any other view that
calls `recStatus` can drift out of sync.**

```js
function recStatus(rc, res) {
  if (!rc.lastRun) return 'Not run yet';
  if (res.breaks.length === 0) return 'Review ready';
  if (res.rate >= 90) return 'In progress';
  return 'Needs resolution';
}
```

Map those four strings to chip/ring/stripe color like this:

| `recStatus()` value | Chip class | Ring/stripe color | Chip label shown |
|---|---|---|---|
| `'Not run yet'` | `chip idle` | `--faint` (dashed ring, no fill) | "Not run" |
| `'Review ready'` | `chip ok` | `--ok` | "Ready" |
| `'In progress'` | `chip warn` | `--warn` | "In progress" |
| `'Needs resolution'` | `chip warn` | `--warn` | "Needs resolution" |

Note there is **no red/`bad` tier** in the current data model — the existing
table only ever distinguishes not-run / ok / warn (see the table's current chip
logic: `!rc.lastRun ? '' : res.breaks.length ? 'bg-warn-soft ...' : 'bg-ok-soft ...'`).
The original mockup showed a 4th "Overdue" red state for illustration, but
nothing in `recStatus()` currently computes overdue-ness (the due date itself is
fake, per above). **Do not add a `bad`/red tier unless you first wire a real due
date and get sign-off on what "overdue" means** — shipping a red state driven by
fake data would be worse than not having one. Stick to the three real states.

## Design tokens & components to reuse

Don't add new CSS custom properties or invent new component classes — the file
already defines everything needed near the top of `<style>`:

- Colors: `--canvas --surface --sunken --ink --ink-soft --muted --faint --line
  --line-soft --accent --accent-soft --accent-ink --ok --ok-soft --ok-ink --warn
  --warn-soft --warn-ink --bad --bad-soft --bad-ink`
- Existing classes: `.card`, `.chip`, `.lbl`, `.field`, `.btn` / `.btn-primary` /
  `.btn-ghost` / `.btn-sm`, `.mono`, `.row-hover`, `.fade-in`
- Existing helpers: `esc()`, `fmtMoney()`, `fmtDate()`, `icon(name, cls)`

New CSS you'll need to add (scoped, additive, not replacing anything) — a ring
meter, a card-specific grid layout, and a stripe accent. Reference implementation:

```css
.rc-grid{ display:grid; grid-template-columns:repeat(3, 1fr); gap:14px; }
@media (max-width:820px){ .rc-grid{ grid-template-columns:repeat(2,1fr); } }
@media (max-width:560px){ .rc-grid{ grid-template-columns:1fr; } }

.rc-card{
  background:var(--surface); border:1px solid var(--line); border-radius:11px;
  padding:16px; cursor:pointer; transition:box-shadow .12s, transform .12s;
  border-top:3px solid var(--stripe, var(--line));
  display:flex; flex-direction:column; gap:14px;
}
.rc-card:hover{ box-shadow:0 6px 16px rgba(16,19,26,.08), 0 2px 4px rgba(16,19,26,.05); transform:translateY(-1px); }

.rc-ring{ width:44px; height:44px; border-radius:50%; flex:none; display:flex; align-items:center; justify-content:center;
  background:conic-gradient(var(--ring-color, var(--ok)) calc(var(--pct,0)*1%), var(--sunken) 0); }
.rc-ring.idle{ background:var(--sunken); border:2px dashed var(--line); }
.rc-ring-inner{ width:32px; height:32px; border-radius:50%; background:var(--surface); display:flex; align-items:center; justify-content:center; font-size:10px; font-weight:700; color:var(--ink); }
.rc-ring.idle .rc-ring-inner{ color:var(--faint); font-weight:600; background:transparent; }

.rc-stats{ display:grid; grid-template-columns:1fr 1fr; gap:10px 12px; padding-top:12px; border-top:1px solid var(--line-soft); }
.rc-stats .k{ font-size:10px; color:var(--faint); text-transform:uppercase; letter-spacing:.04em; font-weight:600; margin-bottom:2px; }
.rc-stats .v{ font-size:12.5px; font-weight:600; color:var(--ink-soft); }
.rc-stats .v.warn{ color:var(--warn-ink); }
```

## Card markup (template-literal function)

Write this as a helper, e.g. `arReconCard(rc, res)`, called from
`viewReconciliations()` in a `.map(...).join('')`:

```js
function arReconCard(rc, res) {
  const status = recStatus(rc, res);
  const styleFor = {
    'Not run yet':      { chip: 'idle', stripe: 'var(--line)',  label: 'Not run' },
    'Review ready':      { chip: 'ok',   stripe: 'var(--ok)',    label: 'Ready' },
    'In progress':        { chip: 'warn', stripe: 'var(--warn)',  label: 'In progress' },
    'Needs resolution':   { chip: 'warn', stripe: 'var(--warn)',  label: 'Needs resolution' },
  }[status];
  const ring = rc.lastRun
    ? `<div class="rc-ring" style="--pct:${res.rate}; --ring-color:var(--${styleFor.chip === 'ok' ? 'ok' : styleFor.chip === 'warn' ? 'warn' : 'faint'});"><div class="rc-ring-inner">${Math.round(res.rate)}%</div></div>`
    : `<div class="rc-ring idle"><div class="rc-ring-inner">—</div></div>`;

  return `<div class="rc-card" style="--stripe:${styleFor.stripe};" data-act="open-recon" data-id="${rc.id}">
    <div class="flex items-start gap-3">
      ${ring}
      <div class="flex-1"><div class="font-semibold text-[14px]">${esc(rc.name)}</div><div class="text-[11.5px] text-faint mt-0.5">${TYPE_META[rc.type].label} · Meridian India</div></div>
      <span class="chip ${styleFor.chip} ml-auto">${esc(styleFor.label)}</span>
    </div>
    <div class="rc-stats">
      <div><div class="k">Open exceptions</div><div class="v ${rc.lastRun && res.breaks.length ? 'warn' : ''}">${rc.lastRun ? res.breaks.length : '—'}</div></div>
      <div><div class="k">Unreconciled</div><div class="v ${rc.lastRun && res.breaks.length ? 'warn' : ''} mono">${rc.lastRun ? fmtMoney(res.breakVal) : '—'}</div></div>
      <div><div class="k">Owner</div><div class="v">${esc(rc.owner)}</div></div>
      <div><div class="k">Due</div><div class="v">Jul ${20 + (rc.name.length % 5)}</div></div>
    </div>
    <div class="flex items-center justify-between text-[11.5px] text-muted"><span>${esc(rc.cadence)}</span><span>${rc.lastRun ? 'Last run ' + rc.lastRun.at.slice(0, 10) : 'Never run'}</span></div>
  </div>`;
}
```

Then in `viewReconciliations()`, replace the `<table>...</table>` block with:

```js
`<div class="rc-grid">${all.map(({ rc, res }) => arReconCard(rc, res)).join('')}</div>`
```

`data-act="open-recon" data-id="${rc.id}"` on the card root is what makes it
clickable — the existing dispatch table already has a handler:
`'open-recon': d => { DB.route = { name: 'recon', recId: d.id, tab: 'overview' }; DB.ui.editRule = null; render(); }`.
No new click-handling code needed.

## Accessibility requirement (validated, don't skip)

The app's `--warn` (#D97706) and `--bad` (#DC2626) sit close enough in color
that they fail a colorblind-safe separation check (ΔE 14.4 against a 15 floor
for normal vision — checked with the dataviz skill's palette validator). Since
you're not adding a `bad` tier anyway (see above), this mainly means: **every
status must show as text (the chip label), never as the ring/stripe color
alone.** Don't ship a version where the only signal is the stripe or ring hue.

## Out of scope — leave these alone

- The search input above the grid is currently **not wired to any state** —
  it's decorative in the existing table view too (no `data-*` attribute, no
  filter logic). Don't silently make it functional as a side effect of this
  change; if you want it working, call that out as a separate, explicit step.
- Filters / Saved views buttons — same as above, currently inert, out of scope.
- A Table/Cards view toggle was discussed as a *possible* enhancement but is
  not required. If asked to add it: store the mode in `DB.ui.reconView`
  (`'cards' | 'table'`, default `'cards'`), branch the render on it, and keep
  the old `<table>` markup around as the `'table'` branch rather than deleting
  it.

## Verification

This is a static single-file app — open `index copy.html` directly in a
browser (or via a local static server) and navigate to the "All
reconciliations" view. Check:

1. One card per entry in `DB.recs`, in the same order the table showed them.
2. Ring percentage matches `res.rate` for run reconciliations; unrun ones show
   `—` in a dashed ring, chip reads "Not run", no warn/ok color leaks in.
3. Clicking anywhere on a card navigates the same way the old table row did.
4. Chip label always matches `recStatus(rc, res)` — cross-check against "My
   work" (`viewMyWork`) or wherever else `recStatus`/`res.breaks` are shown, so
   the same reconciliation reads the same status everywhere.
5. Resize the window — grid drops 3 → 2 → 1 columns at the breakpoints above,
   no horizontal scroll on the page body.
6. Hover state (shadow + slight lift) works on every card, not just the first.
