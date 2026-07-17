---
name: promptback
description: |
  Build a single-file interactive HTML report that collects per-item feedback (approve /
  revise / hold / drop chips + free-text notes) and copies all decisions back out as a
  structured prompt — no backend, works from file:// or any static host, survives reload
  via localStorage. Use when: (1) you produced findings, a plan, a backlog triage, or a
  set of options a reviewer must sign off item by item, (2) the reviewer wants to click
  and type feedback in place instead of retyping decisions into chat, (3) the deliverable
  must be one self-contained .html file (client-facing, offline, no server). Covers the
  widget/state/copy-dock pattern, decision-specific option chips, persistence, clipboard
  pitfalls (iframe permissions, rejected promises), and browser-based verification.
author: Huiyan Wan
version: 2.0.0
date: 2026-07-17
---

# Interactive feedback report → copy-as-prompt

## Problem

You've produced findings / a plan / options the user must give feedback on. A flat HTML
report is read-only — the user reads it, then has to type their decisions back from
scratch, losing the per-item context. You want the deliverable itself to *capture*
feedback in place and hand it back as one structured prompt, with **no backend**: it must
open from `file://` or a static host, survive a page reload, and work offline.

## Context / Trigger Conditions

- "Turn this report into an HTML I can give feedback on, item by item."
- Per-item sign-off needed (approve / revise / hold / drop / discuss) across many
  cards, asks, findings, or options — especially items that need a *decision*, not just a read.
- No server available / single-file deliverable required / client-facing.

## Solution

Build ONE self-contained `.html` with three reusable pieces. Match whatever design tokens
your project already uses so the page feels native.

**1. Per-item widget** — a container carrying a stable id + a human title, a single-select
chip row, and a free-text note:

```html
<div class="fb" data-id="item-42" data-q="#42 — Checkout A/B test: ship variant B?"
     data-opts='[["ship","🚀 Ship B"],["extend","⏳ Extend test"],["ask","💬 Discuss"]]'>
  <p class="fb-ctx">…the context the reviewer needs to decide…</p>
  <div class="fb-chips"></div>           <!-- chips injected by JS so every widget stays consistent -->
  <textarea class="fb-note" placeholder="Notes…"></textarea>
</div>
```

Generic sections (close candidates, build queues) can share one default chip set; items
that need a real decision get **question-specific options** via `data-opts` — the copied
prompt then carries the actual decision (`[SHIP]`, `[EXTEND]`) instead of a generic
"approve".

**2. State + chips + persistence** (versioned localStorage key; prune empty; restore on load):

```js
const KEY='myreport_feedback_v1';
const DEFAULT_CHIPS=[['approve','✅ Approve'],['revise','✏️ Revise'],['hold','⏸ Hold'],['drop','❌ Drop'],['ask','💬 Discuss']];
let store={}; try{store=JSON.parse(localStorage.getItem(KEY))||{}}catch(e){}
const items=[...document.querySelectorAll('.fb')];
items.forEach(fb=>{
  const id=fb.dataset.id, wrap=fb.querySelector('.fb-chips'), note=fb.querySelector('.fb-note');
  const opts=fb.dataset.opts?JSON.parse(fb.dataset.opts):DEFAULT_CHIPS, rec=store[id]||{};
  opts.forEach(([v,label])=>{
    const b=document.createElement('button'); b.className='chip'; b.type='button'; b.dataset.v=v; b.textContent=label;
    b.setAttribute('aria-pressed', rec.decision===v?'true':'false');
    b.onclick=()=>{ const cur=(store[id]||{}).decision, nv=cur===v?'':v;   // click again to deselect
      wrap.querySelectorAll('.chip').forEach(x=>x.setAttribute('aria-pressed','false'));
      if(nv) b.setAttribute('aria-pressed','true'); save(id,{decision:nv}); paint(fb,id); };
    wrap.appendChild(b);
  });
  if(rec.note) note.value=rec.note;
  note.oninput=()=>{ save(id,{note:note.value}); paint(fb,id); };
  paint(fb,id);
});
function save(id,patch){ store[id]={...store[id],...patch};
  if(!store[id].decision && !(store[id].note||'').trim()) delete store[id];
  try{localStorage.setItem(KEY,JSON.stringify(store))}catch(e){} count(); }
function paint(fb,id){ const r=store[id]||{}; fb.classList.toggle('answered', !!(r.decision||(r.note||'').trim())); }
function count(){ document.getElementById('fbCount').textContent=Object.keys(store).length; }
count();
```

**3. Copy-as-prompt dock** (fixed bar; builds a SELF-CONTAINED prompt; clipboard + execCommand fallback):

```js
function buildPrompt(){
  const L=['Feedback on <DELIVERABLE NAME>. Apply these decisions:',''];
  items.forEach(fb=>{ const r=store[fb.dataset.id]; if(!r) return;
    const dec=r.decision?`[${r.decision.toUpperCase()}]`:'[note]', n=(r.note||'').trim();
    L.push(`- ${dec} ${fb.dataset.q}${n?` — ${n}`:''}`); });
  if(!Object.keys(store).length) L.push('(no items marked yet)');
  L.push('', 'Read <FILE PATHS> for full context before acting.');     // <- makes the paste cold-start
  return L.join('\n');
}
document.getElementById('copyBtn').onclick=()=>{ const t=buildPrompt();
  (navigator.clipboard?.writeText(t)||Promise.reject()).then(toastOK, fb);
  function fb(){ const ta=document.createElement('textarea'); ta.value=t; document.body.appendChild(ta); ta.select();
    try{document.execCommand('copy'); toastOK()}catch(e){prompt('Copy:',t)} ta.remove(); } };
```

CSS essentials: chips are `aria-pressed`-styled buttons (single-select look); `.fb.answered`
gets a left-border/glow so progress is visible; the dock is `position:fixed;bottom:0`.

**Ordering:** put the items that genuinely need the reviewer's decision FIRST (with their
question-specific chips), then ratifiable lists (close candidates, approvals), then FYI
queues. Reviewers run out of attention top-to-bottom.

## Verification

Serve over a local port (browser automation typically blocks `file://` —
`python3 -m http.server PORT`, then open `localhost:PORT`). Verify via DOM evaluation, not
screenshots (long pages wedge screenshot subsystems; DOM geometry is the reliable check):

- click a chip + dispatch an `input` on a textarea → assert `localStorage[KEY]` holds
  `{decision, note}`, the `#fbCount` increments, and the widget gained `.answered`;
- reload → assert the chip's `aria-pressed` and the note restore;
- click the active chip again → assert it deselects and the store entry is pruned;
- reconstruct `buildPrompt()` output and assert it renders `- [DECISION] <title> — <note>`;
- click the real copy button → assert it doesn't throw (clipboard may reject in headless →
  the execCommand fallback must run);
- geometry: `scrollWidth === clientWidth` (no horizontal overflow), dock pinned to the
  viewport bottom;
- **clean up your test entries from localStorage** so the real reviewer starts blank.

## Example

A backlog-triage review page: ~50 widgets across four sections — decision items first,
each with custom chips ("🗑 Delete legacy" / "📌 Keep + fix" / "💬 Discuss"), then close
candidates ("✅ Close" / "🚫 Keep open"), then two build queues ("🚀 Go" / "⏸ Later" /
"❌ Drop"). The reviewer clicks through over coffee, adds notes where it matters, hits
**Copy decisions as prompt**, and pastes one message back to their agent:

```
Feedback on the backlog triage. Apply these decisions:

- [KEEP-FIX] #18 — Legacy checkout page: delete or keep?
- [CLOSE] #23 — Redesign the settings page (superseded)
- [GO] #31 — Add the export CSV endpoint — cap at 10k rows for now

Read docs/reviews/<triage-report>.md for full context before acting.
```

## Notes

- **Clipboard API needs a user gesture** and fails in headless / some `file://` contexts →
  ALWAYS include the `execCommand('copy')` textarea fallback (and a final `prompt()` so
  nothing is lost).
- **Embedded in an `<iframe>`? The copy button silently fails without permission.**
  `navigator.clipboard.writeText` is blocked by Permissions Policy unless the PARENT
  iframe tag carries `allow="clipboard-write"`. Trap: a bare
  `try{ navigator.clipboard.writeText(t) }catch(e){ fallback }` will NOT catch the failure —
  `writeText` returns a *rejected Promise*, not a sync throw, so the fallback never runs
  and the button still flips to "Copied ✓" while nothing was copied. Use
  `writeText(t).then(ok, fallback)` (as in the dock snippet) OR add
  `allow="clipboard-write"` to the iframe. It works when the file is opened directly, so it
  passes local testing and only breaks embedded — verify in the *embedded* context.
- **Word the copied text for the ACTUAL recipient.** The cold-start line
  `Read <FILE PATHS> for full context` is right when the paste goes to YOUR next agent
  session. When the reviewer is a CLIENT sending feedback back to you, internal file paths
  are a leak and "paste into the chat" is confusing — drop the path line and word it
  "paste it into your reply to the team". Decide the recipient before writing the copy string.
- **Version the localStorage key** (`_v1`) so a later schema change doesn't silently merge
  with old saved input; prune records that have neither a decision nor a note.
- **Choose low-entropy localStorage key names** (plain lowercase words + `_v1`, no digit
  clusters) — high-entropy-looking strings can trip secret scanners (e.g. gitleaks'
  generic-api-key rule) in CI.
- **Extending a shipped page the reviewer may ALREADY be ticking → addendum mode.** You
  cannot observe the reviewer's localStorage, so when adding items to a live review page:
  ADD new widgets with NEW stable `data-id`s (plus a banner "everything above is unchanged —
  your ticks are preserved"); never rename or remove existing `data-id`s or rewrite existing
  `data-q` labels (ids key the store; `data-q` feeds the copied prompt). Verify, don't
  assume: seed a fake pre-existing tick in localStorage, reload, assert the old widget
  restores beside a new one — then remove the test key.
- **Later content-enhancement passes must not touch the widget machinery.** If another
  agent enriches the page afterwards, pin the `<script>` block (record its hash and assert
  it unchanged), state the widget contract ("don't touch `.fb` ids/data attributes; insert
  new content after the textarea"), and re-run the live widget check afterwards.
- **Single-select chips** via `aria-pressed` (clear siblings on select; clicking the active
  chip deselects). Reach for multi-select only if items genuinely need several tags.
- **The copied prompt must cold-start** the next session — prepend a one-line instruction
  and append "read <file paths> for context". A bare list of decisions isn't actionable
  without the source docs.
- Inject the chips from JS (don't hand-write them per widget) so every item stays
  consistent and adding a decision option is a one-line change.
- **Define tick SEMANTICS in the copied prompt when a chip means "adopt the recommendation"
  (v1.1.0 — first full owner round-trip confirmed in production).** A real review round
  validated the whole loop end-to-end: the owner ticked every widget, hit copy, and pasted
  into a fresh agent session — the paste cold-started the apply round with zero ambiguity,
  including two REVISE+note overrides that cleanly reversed the report's own recommendations.
  The load-bearing detail: on widgets whose question is recommendation-shaped ("we recommend
  X; the alternative is Y"), a bare `[APPROVE]` is ambiguous to the receiving session —
  approve the change, or approve the status quo the report defended? Fix it twice: (1) put a
  one-line `.meaning` legend on the widget itself ("✅ Approve = do X · ✏️ Revise + note = Y
  instead"), and (2) append a **"Meaning of ticks" section to `buildPrompt()`** restating the
  same mapping for exactly those widgets, so the copied prompt carries its own decoding key.
  Free-text-only answers (`[note]`/discussion chips) ride the same prompt fine — answer them
  in the reply and record dispositions wherever the decisions are tracked.
