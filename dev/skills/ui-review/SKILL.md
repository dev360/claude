---
name: ui-review
description: Adversarially review a RENDERED UI — screenshots the real running app at real data and judges it against products people actually use (Vercel, Stripe, GitHub) plus the mechanics of the stack it is built on. Use after any UI change, before showing a screen to a stakeholder or customer, or when asked whether a screen is any good. Never reviews UI by reading code. Complements the `ux-review` agent, which analyzes component structure statically.
---

# UI review

Render it, look at it, and say what is wrong with it. This skill produces **findings only** — it
never fixes what it reviews, and it never grades its own author's work.

## Step 1: reconstruct the job before you judge the screen

**Do not skip this and do not improvise it.** A review that starts at the pixels produces styling
notes; a review that starts at the job finds the missing feature. The most valuable finding this
skill can produce is *"this need has no home in the product at all"* — and that is invisible in a
screenshot.

Write, before opening a browser:

1. **The user, in one sentence** — who, under what conditions, how often, and what tool they used
   before this existed. Everything downstream is weighted by this: density and keyboard speed for
   someone who lives in the tool all day; clarity and proof for a stranger deciding in five
   seconds; huge targets and one decision per screen for someone on a phone in the field.
2. **The hierarchy of needs, ranked** — and for each, **what it costs when it fails** (dollars, a
   phone call, a lost customer, a dispute nobody can settle) and **how often it arises**. Rank by
   cost × frequency. A need with no stated failure cost is conjecture; cut it.

Derive the hierarchy from the economics of the role, never from imagination:

- What is this person **accountable** for? Money in, money out, promises kept.
- What does an error here **cost**, and who eats it?
- Which question do they ask **most often per hour**?
- What do they **check repeatedly**? Repeated checking is anxiety, and anxiety marks an unmet need.
- What **workaround** runs today — a sticky note, a second spreadsheet, a phone call, a photo?
  Every workaround is an unmet need with proof attached.
- What must be **defensible after the fact**? What does a customer dispute, and what evidence
  settles it?

Take these from the repo's product notes or a real customer conversation if they exist. If you
had to assume them, say so at the top of the review — never present invented needs as fact.

### Then run four tests, in this order, before any styling critique

**a. Coverage.** For each need, name the screen that answers it and count the interactions from a
cold start. A need with **no home** is the highest-severity finding available. A need answered
only by leaving the app — a phone call, a spreadsheet, asking a colleague — is the same finding
in disguise.

**b. Prominence.** Rank what the screen actually emphasizes (size, weight, color, position), then
set that ranking beside the needs ranking. Every mismatch is a finding, and these outrank every
mechanic below.

**c. System of record.** For each need whose answer is a *fact*: is the datum captured at the
moment it is known, is it authoritative, and can the user drill from the summary to the evidence?
If the real answer still lives in someone's memory or a phone call, the product is failing, not
the screen.

**d. Round trip — the test that catches write-only data.** For every datum the product captures,
walk the circle and report where it breaks: (1) **enter** it at the moment it is known;
(2) **find it again** from a cold start tomorrow — a value visible only on the screen that
created it is effectively unrecorded; (3) **see it in full** — tooltip-only or truncated values
fail; (4) **change it**, at the value itself — "captured at intake with no way to edit" is a
blocker, because an uneditable record drives the user back to their spreadsheet and takes the
record with them; (5) **see what changed** — who, when, why, on the object. Anything failing (2)
or (4) outweighs every styling note in the review.

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
10. **Write-only data.** A value the UI captures but never shows again, or shows with no way to
    change: intake-only fields, notes hidden in a `title` tooltip, a selection whose result is
    invisible after saving, a "reason" collected and never displayed.
11. **Blocking where it should yield.** Refusing what a user with real-world context has already
    decided. Constraints inform (dim, badge, warn); they do not argue. Required justification
    fields on routine edits are this defect.

Then look for what is **missing**: a number the user must leave the screen to get, a state change
with no feedback, a decision with no visible consequence, a filter that doesn't survive a
refresh, an object with no history.

## Output

Findings only. No praise section, no summary of what the screen does, no restating the brief.

Each finding:

- **Screen · viewport** and the screenshot filename
- **Need** — which ranked need this damages, or "serves no need" for pure clutter. A defect that
  breaks a top-three need outranks everything else, however small the pixel crime looks.
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
