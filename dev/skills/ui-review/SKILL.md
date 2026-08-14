---
name: ui-review
description: Adversarially review a RENDERED UI — screenshots the real running app at real data and judges it against products people actually use (Vercel, Stripe, GitHub) plus the mechanics of the stack it is built on. Use after any UI change, before showing a screen to a stakeholder or customer, or when asked whether a screen is any good. Never reviews UI by reading code. Complements the `ux-review` agent, which analyzes component structure statically.
---

# UI review

Render it, look at it, and say what is wrong with it. This skill produces **findings only** — it
never fixes what it reviews, and it never grades its own author's work.

## Step 1: name the user before you judge anything

One sentence: who uses this screen, under what conditions, how often, and what tool they used
before. Everything below is weighted by that answer — density and keyboard speed win for an
operator who lives in the tool all day; clarity and proof win for a stranger deciding in five
seconds; enormous targets and one decision per screen win for someone on a phone in the field.

If nobody told you the user, say what you assumed.

## Step 2: render it, never read it

Reading JSX/CSS is **not** a review. Layout is emergent from font metrics and the cascade;
"the class says items-center" tells you nothing about what a human sees.

- Isolated headless browser (Puppeteer/Playwright against a real Chrome), **not** a shared
  automation browser: `requestAnimationFrame` is suspended in hidden windows, so canvas, maps
  and animation render blank and you will report false defects.
- **Both viewports: 1440×900 and 390×844.** Mobile is not optional; an undeclared break is a
  defect.
- Sign in against **real seeded data**. A review of an empty workspace is worthless — the empty
  state is one state, not the product.
- Capture **every meaningful state**: empty, one row, many rows, the busiest realistic case, an
  open modal/drawer, mid-flight loading, error, and the state right after a mutation.
- Every finding cites a screenshot you actually produced. Every measurement claim cites the
  measured value (`getBoundingClientRect`, computed style, element count, overflow boolean).

## Step 3: judge against products, not personalities

Do not review from a design syllabus and do not invoke famous designers — nobody can check a
name, and a heuristic nobody can falsify is an opinion with a citation. Review against products
the reader can open in another tab right now.

**Vercel — the deployments list.** Quiet until something is wrong. A row is a status dot, a name,
a branch, a duration, a relative time; no borders between fields; hierarchy from type weight and
color alone. Failure is the only thing that gets a color. One primary action per view. `⌘K`
reaches anything. *Is the page calm at rest, and does the eye go straight to the broken row?*

**Stripe — the payments table and an object page.** Dense money data that stays legible: amounts
right-aligned in tabular figures, a small **closed** vocabulary of status badges (not one badge
per concept the team invented), a filter toolbar that reads like a sentence, rows opening a full
object page that carries a **timeline of what happened to this object**, copyable IDs. Error and
empty states are designed, not leftovers. *Can the user answer "what happened to this record"
without leaving it? Do the numbers line up in a column?*

**GitHub — the issues/PR row.** The reference for row-level controls: **one** primary action per
row, fields edited **in place** (click the title, it becomes an input), every rare action behind
a `⋯` overflow, bulk actions from selection rather than per-row clutter, and filters in one bar
that is shareable as a URL and survives a refresh. *Does each row carry exactly one obvious
action — or a pile of small links competing with a button?*

Then add the **incumbent bar**: whatever the user does this job with today — a spreadsheet, a
whiteboard, a legacy tool, a competitor. *Would someone who is fast in that be slower here?* If
yes, nothing else about the design matters. Name the incumbent explicitly in the review.

## Step 4: check the mechanics the stack already inherits

Concrete, checkable, true-or-false on screen. (These are the Tailwind/shadcn authors' own rules —
worth citing only because the code already imports them.)

- Hierarchy from **weight and color**, not size alone. Two sizes and three weights beats five
  sizes. De-emphasize by lightening, never by shrinking below readable.
- **Borders are a last resort.** Do not separate what spacing already separates; prefer a
  background shift or shadow, which costs no ink.
- A **spacing scale**, with more space *between* groups than *within* them. Proximity groups
  before boxes do.
- **Right-align numbers, use tabular figures**, align decimal points down a column.
- Kill label-value pairs a column header already answers.
- Color carries meaning or does not appear: one accent for the primary action, semantic colors
  for states, everything else neutral. No gray text on colored backgrounds.
- Design the empty state and the busiest state first; the middle takes care of itself.

## Step 5: the taboos — name each defect by number

1. **Synthesized instruction.** The UI derives a fact, then phrases it as advice or an order
   ("Call Maria to confirm before scheduling" — assembled from a flag plus a contact record
   nobody designated for this). If the datum was never captured, the screen says nothing.
   Imperatives belong on buttons.
2. **Sentences where stats belong.** "941 units already booked across 18 jobs" instead of
   `Booked / 941 / 18 jobs`. Professionals scan numbers; they do not read prose.
3. **Editorial adverbs on data**: already, still, so far, right now, yet, just. Delete them.
4. **Schema words on screen**: entity, resource, record, projection, workspace, canvas, lens.
   Use the user's nouns, not the data model's.
5. **Redundant ink**: a value printed twice, a badge announcing that a value exists, a panel
   titled what the user is already looking at, an empty state that explains the feature. A quiet
   state is **blank**.
6. **Chrome nesting**: panel inside card inside panel. Count nested bordered containers; data
   sitting three borders deep is a finding.
7. **Ambiguous or lying control.** A button whose label doesn't say what happens; a link that
   mutates; a control that changes meaning at the same coordinates depending on invisible state
   (a mode); a disabled control with no stated reason.
8. **Weak action hierarchy.** Equal-weight controls where one action dominates in frequency; the
   common action smaller or further away than the rare one (measure it — Fitts is arithmetic);
   destructive adjacent to routine; the primary CTA below the fold.
9. **Mobile break**: horizontal page overflow, touch targets under 44px, a table that neither
   scrolls in its own container nor degrades to cards.
10. **Blocking where it should yield.** Refusing what a user with real-world context has already
    decided. Constraints inform (dim, badge, warn); they do not argue. Required justification
    fields on routine edits are this defect.

Then look for what is **missing**: a number the user must leave the screen to get, a state change
with no feedback, a decision with no visible consequence, a filter that doesn't survive a
refresh, an object with no history.

## Output

Findings only. No praise section, no summary of what the screen does, no restating the brief.

Each finding:

- **Screen · viewport** and the screenshot filename
- **Bar** — which benchmark it fails, stated concretely ("GitHub would put these two links behind
  a `⋯` and make the quantity editable in place") — and **taboo # or "gap"**
- **The defect**, one sentence, with the measured values
- **What the user loses** — the concrete cost: a click, a scroll, a phone call, a wrong number
- **The fix**, implementable, in the app's existing vocabulary
- **Severity**: blocker (would embarrass us in front of a customer) / serious / polish

Rank by severity. **Ten sharp findings beat forty soft ones** — padding destroys the signal. If a
screen is genuinely fine, write "no findings" for it and move on.

Close with anything the app gets **right** that a future change might destroy, so nobody
"fixes" it. One short list, only if it exists.

## Refusals

- If the app will not boot or you cannot sign in after a few attempts, **say so plainly and
  stop**. Never substitute a code review for a rendered one; a review that never saw the pixels
  is worse than no review, because it will be believed.
- If you wrote the screen, you cannot review it. Hand it to a fresh context.
