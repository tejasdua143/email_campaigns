# Presentations.AI — Lifecycle Email Decision System

A growth project: deciding **which lifecycle / recovery email to send a user**, based on *where* and *why* they dropped off in the signup → payment journey. Owner: Tejas (product designer, growth). Lifecycle emails ship via **Sendy**.

This repo is three self-contained HTML visualizations of the same underlying model. **No build step, no framework, no package install.** Only external dependency is a Google Fonts `@import`. Open any `.html` in a browser to view.

---

## 1. Read this first — the core idea

The journey is **not linear**, and we got this wrong in the first two passes. A drop-off is not a dead-end branch hanging off a straight funnel. It is a **fork that re-routes the person's entire trajectory**.

Model it as a **state machine with loops and multiple endings:**

- Every **fork** (a place people stall) has *two* exits:
  - **Recover** — our email works → the person loops **back onto the path** toward payment, re-entering at a specific step.
  - **Fail** — our email doesn't land → they reach a **non-paying ending**.
- There is **one winning ending: Paid.** It's reachable from many recovered journeys.
- There are **several losing endings.** Each one needs a **fix** — and the fix is not always an email (sometimes it's a product change, sometimes it's "leave them alone").

**The job:** tilt people toward the green (recovery) loops, and make sure every red (losing) ending has a real, honest solve. The non-paying endings are what we are solving for.

`journey-machine.html` is the canonical, correct view of this. The other two files are earlier/simpler lenses on the same data.

---

## 2. The five archetypes (the classification layer)

Don't route on funnel stage — route on **who the person is**. Every stalled user is one of these. Color is used consistently across all three files.

| Archetype (plain name) | What it means | What to do | Color |
|---|---|---|---|
| **Something broke** | Gen failed, mobile checkout dead-ended, payment declined | Fix it or route around — *don't* sell | `#FFB454` amber |
| **Happy, just done** | Got their value, no current need (episodic-tool reality) | Invite them back later — *don't* nag | `#34D399` green |
| **Not sold yet** | Never reached the value / doubted quality | Show them the value | `#60A5FA` blue |
| **Stuck, but wanted in** | Was trying, hit an obstacle short of the goal | Clear the obstacle | `#A78BFA` violet |
| **Balked at the price** | Saw the price, hesitated | Reframe value, or sweeten | `#F472B6` rose |

**Two key behaviors that aren't about persuasion:**
- **Mobile stalls past "deck made" are mechanical until proven otherwise.** On phones, only ~1 in 10 of payment-intent users reach checkout (vs ~1 in 2 on desktop). An email is a band-aid; the real fix is product.
- **"Happy, just done" is mostly NOT a conversion problem.** It's expected for an occasional tool. Nagging these people poisons deliverability.

---

## 3. The funnel (illustrative, 10,000 signups/day at top)

Percentages are % of the previous step (real numbers, from the analytics). Desktop ≈ 75% of traffic, mobile ≈ 25%.

| Step | Drop | ~Lost/day | Notes |
|---|---|---|---|
| Signed up → Started a deck | −10% | 1,000 | |
| Started → Deck created | −12% | 1,060 | higher on mobile |
| Deck created → tried to pay | −20% | 1,580 | richest fork |
| **Payment intent → checkout** | **−60%** | **3,200** | **biggest leak (P0).** Mobile collapses to ~10% here vs ~52% desktop |
| **Checkout → paid** | **−97.5%** | **2,100** | historically "normal," but we're **blind** on the split |

The −97.5% checkout→paid is a real, long-standing number — most people who reach checkout don't convert. It is *not* an instrumentation bug; the success count reconciles with the billing dashboard. The problem there is **lack of visibility into the split**, not a broken counter.

---

## 4. The state machine spec

Canonical data lives in `journey-machine.html` (JS objects `SPINE`, `STALLS`, `LOSSES`). Summarized here as the source of truth.

### Path states (the spine → Paid)
`A` Arrived → `B` Started a deck → `C` Deck created → `D` Saw it & liked it (watched slideshow) → `E` Wants to pay (export / upgrade / hit a wall) → `F` At checkout → `W` **Paid** ✅

### Forks (stalls)
Each: where it sits, drop, where recovery re-enters, which endings a failure leads to, and the sub-branches.

**St0 · Didn't start a deck** (−10%) · recover → B · fail → ghosted, opted_out
- Started typing, then quit → *Stuck* → "Pick up where you left off" + finish-prompt help → back to B
- Looked around, never started → *Stuck* → hand them ready-made prompts/templates → back to B
- Left right away → *Not sold* → 60-sec demo, send once then stop → back to B · *light/hold*

**St1 · Deck didn't get made** (−12%) · recover → C · fail → broke, ghosted
- It broke on them → *Something broke* → "Sorry, here it is now" + retry, fast → back to C · 🔧
- Quit on their phone → *Something broke* → faster path / desktop → back to C
- Gave up partway → *Stuck* → "Finish your prompt" → back to C

**St2 · Made a deck, didn't try to pay** (−20%) · recover → E · fail → free_gone, ghosted
- **Made it their own, then left (edited the slides)** → *Happy, just done* → **strongest signal here — they're invested. Export is Pro, so push it: "your deck's ready, get it out / publish it." Warmest non-price sell.** → back to E *(added 2026-06-26)*
- Used up both free decks → *Happy, just done* → "Go unlimited" (easiest sell) → back to E
- Kept redoing it → *Not sold* → "we've made it better" / sharper prompt / Pro writing → back to C
- Watched it, liked it, left → *Happy, just done* → nudge to export or remind later, don't beg → back to E · 🔧
- Left before seeing the good part → *Not sold* → bring them back to watch the slideshow → back to D · 🔧

**St3 · Wanted to pay, no checkout** (−60%, biggest leak) · recover → F · fail → price_no, ghosted
- Wanted to pay, on a phone → *Something broke* → direct checkout link/desktop, **but really fix mobile checkout** → back to F · 🔧
- Tried to export, saw price, stopped → *Balked at price* → **best email:** "one click away" + value reframe + maybe a deal → back to F · 🔧
- Compared plans, then left → *Balked at price* → answer doubts, clarify plans, small nudge → back to F · 🔧
- Saw the price but not the value → *Not sold* → lead with what Pro does, not the price → back to E · 🔧
- Just bumped a locked feature → *Not sold* → light touch or nothing → back to E · 🔧 *hold*

**St4 · At checkout, didn't pay** (−97.5%, can't see yet) · recover → F (retry) · fail → broke, price_no
- Payment failed → *Something broke* → "Try again" + another payment method + fallback → back to F · 🔧
- Typed card, then bailed → *Balked at price* → reassure (money-back, secure) + gentle nudge → back to F · 🔧
- Saw the final price, left → *Balked at price* → remind value, maybe a targeted offer → back to F · 🔧

### Losing endings + their fixes
- **Got it free, gone** — *fix:* mostly fine for an occasional tool. Remind them at their next deck moment; consider a tighter free limit so value-getters hit the wall sooner. Loops back → A (next need).
- **Went cold** — *fix:* usually a weak first email. Improve the earliest touch; try a different time/channel; one win-back later, then stop. Loops back → A.
- **Payment kept breaking** — *fix:* **⚙ product, not email** — fallback methods, smarter retries, working mobile checkout. Most recoverable loss; fixing it moves the whole checkout number.
- **Decided it's not worth it** — *fix:* segment — value reframe for the on-the-fence, targeted offer for the price-sensitive. True price ceiling = a pricing test, not an email. Re-offer later. Loops back → E.
- **Said no / unsubscribed** — *fix:* respect it and stop. Only real fix is upstream: better earlier emails so fewer opt out.

---

## 5. Instrumentation status — 🔧 = NOT tracked yet

This is the build order. **Most of the high-value forks can't run until these signals exist**, because the routing depends on them.

**Tracked today (fireable now):**
- Furthest funnel step reached · device · acquisition source · persona · decks-created count · regenerate count · dwell · export-attempted · bounce speed.
- **Slide-edit signal** (did they edit the deck after it generated) → strong investment signal, now used in St2. *(Added 2026-06-26: editor events are logged today; the model just wasn't using them. Confirm edit-count is stampable per user so Sendy can trigger on it.)*
- **Payment-start trigger** (`export | upgrade | feature_gate`), stamped per-user → unlocks **all of St3** (the −60% leak). *(Resolved 2026-06-26: eng can stamp it.)*
- **Checkout outcome** (`declined | abandoned`) + device + payment method → unlocks **all of St4** (the −97.5% black box). Declined vs abandoned need opposite emails. *(Resolved 2026-06-26: eng can split it.)*

**🔧 Not tracked yet (each blocks the branches that depend on it):**
- **Generation outcome** (`success | error | timeout`) → splits St1's "it broke" from "gave up."
- **Slideshow/motion viewed** flag + dwell → splits St2 "happy" from "left before the value."

Priority: payment-start trigger and checkout outcome are now in place — St3 and St4 can fire. Remaining 🔧 (generation outcome, slideshow-viewed) gate the St1/St2 splits.

This rides on existing email-attribution work (signed tokens, server-side attribution persistence) — the signals must be captured server-side and stamped on the user/event so Sendy can trigger on them.

---

## 6. The three files

| File | What it is | Use it to |
|---|---|---|
| `journey-machine.html` | **Canonical.** State-machine node graph: spine + forks + losing endings, with green recovery loops and red fail arrows. SVG overlay drawn over HTML nodes (measured at runtime). **Now includes the inline simulator** (folded in 2026-06-26): click a fork → pick the person's signals + a "did the email land?" toggle → it highlights the matched sub-branch and draws a live wire (green recover → re-entry step, or red → the lost ending). | See the looping, multi-ending truth, and simulate one person live. Click a fork → who's there + what to send + re-entry, plus the signal picker. Click an ending → who lands there + the fix. Toggles: Everything / Path to paid / Where we lose. |
| `routing-tree.html` | The whole logic as a top-down tree: linear spine, branches off each step. | See every path at once. Click a step to fold its branches. Toggle "dim what we can't track." |
| `routing-engine.html` | Simulator: pick a user's signals, see which branch + email they get. | Trace one specific user through the if-else. |

**Code shape (all three):** plain HTML + CSS + vanilla JS, single file. Data lives in JS objects near the bottom (`ARCH`, and `SPINE/STALLS/LOSSES` or `STATES`), then a render function builds the DOM. To change copy or logic, edit the data objects. `journey-machine.html` additionally draws an SVG wire layer in `draw()` using `getBoundingClientRect`, redrawn on resize via a `ResizeObserver`.

**No `localStorage`/`sessionStorage`** anywhere (intentional — they break in sandboxed iframes). Keep it that way; use in-memory JS state.

---

## 7. Design system (keep it consistent)

**Palette — dark "instrument" theme:**
```
--bg #0C0D10   --surface #15171D   --surface-2 #1B1E26   --surface-3 #22262F
--line #2A2F3A   --line-bright #3A414F
--ink #ECEEF2   --ink-2 #9AA1AE   --ink-3 #646B79
archetypes: blocked #FFB454 · satisfied #34D399 · unconvinced #60A5FA · friction #A78BFA · price #F472B6 · route #67E8F9
win = green #34D399 · lose = #F4708F
```
**Type:** `Space Grotesk` for UI/labels/headings, `JetBrains Mono` for codes, conditions, timings, drop-%. (The mono-for-machine / sans-for-meaning split is deliberate — it mirrors the tool's job: signals → human action.) Loaded via Google Fonts `@import`.

**Conventions:** color encodes the archetype, never decoration · `🔧` + dashed border = not tracked yet · "hold" = the best move may be no email · respect `prefers-reduced-motion`.

---

## 8. Copy voice

- **Plain, talk-to-a-teammate.** No jargon. Say "they tried to export and saw the price," not "named-artifact + value reframe." This was a hard requirement.
- **Recovery emails:** founder voice, **signed by Dhruv** (cofounder), **plain text** — strip HTML structure so they land in **Gmail Primary**, not Promotions. Short, human, one clear ask.

---

## 9. Open decisions — RESOLVED (2026-06-26)

1. **Do the 2 free decks include export?** → **No, export is Pro.** Export is the value moment, so St2 pushes *toward paying* (recovery target stays E). "Got it free, gone" reminds them export needs Pro.
2. **Can we stamp the payment-start trigger (export / upgrade / gate) per user?** → **Yes.** All of St3 (the −60%) is now tracked and fireable. 🔧 removed from St3.
3. **Can we split checkout outcome (declined vs abandoned) + device + method?** → **Yes.** All of St4 (the −97.5%) is now tracked and fireable. 🔧 removed from St4.
4. **Free-allowance experiment** (tighten free limit) → **No, leave it.** Free limit stays as-is; "Got it free, gone" fix is email-only (remind at next need).

---

## 10. Where we left off / next steps

- **Write the real recovery emails** for the loops that can fire *today* (the non-🔧 sub-branches). Founder voice, plain text, Dhruv-signed:
  - St0: all three (started-then-quit, looked-around, left-immediately)
  - St1: "quit on their phone", "gave up partway"
  - St2: "used up both free decks" (the easiest sell), "kept redoing it"
- ~~**Optional:** fold the simulator's signal-picking into `journey-machine.html`~~ → **DONE 2026-06-26.** Click a fork, pick signals, toggle "did the email land?", and it routes green (recover) or red (fail) live with a drawn wire.
- **Product (biggest leverage, not emailable):** fix mobile checkout (St3 mobile branch) and payment fallback/retry (the "payment kept breaking" ending). These move the two biggest numbers more than any email can.

---

## 11. Run / preview

Just open any file in a browser:
```
open journey-machine.html        # macOS
xdg-open journey-machine.html    # Linux
```
Or serve the folder for live-ish editing:
```
python3 -m http.server 8000      # then visit http://localhost:8000
```
No install, no build. Edit a file, refresh the tab.
