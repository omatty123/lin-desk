# Schema Proposal: Forecasting-Ledger Overhaul

Status: **DRAFT — awaiting approval.** Nothing here is implemented. No existing file
(`index.html`, `data/scorecard.json`, or any site asset) is touched by this proposal.
This document exists to get the data shapes right before any UI work begins.

## Design constraints carried forward

- Bilingual newspaper aesthetic stays: every new structure gets a Chinese name/header
  and an English subhead, same as `board`/`wars`/`coverage` today.
- Static site, no backend. Everything below is still hand-editable JSON, filed by a
  human (or later, a script) directly into `data/scorecard.json` or a sibling file.
- Shanghai color convention (红涨绿跌 — red = correct/rising, green = wrong/falling)
  is a rendering rule, not a data-schema concern, but it constrains the schema: nothing
  below encodes "good performance → green." Brier score, accuracy, hit-rate — all of it
  renders red-for-good. This is called out again in §3.
- Existing `#pred-N` deep-link convention (N = permanent 1-based position of a
  prediction in the ledger array) is preserved and extended to the audit-desk ledger
  as `#audit-N` by the same rule.

---

## 1. Beats

A **beat** is a named desk. Every coverage item and every prediction (own or audit)
carries a `beat` id so the UI can group and filter by beat. Beats are a small static
list — they don't need their own array in the data file if that's simpler, but for
render-time lookup (Chinese name, subhead, sort order) they need *some* canonical
table. Proposed as a new top-level key:

```json
"beats": [
  {
    "id": "wisconsin",
    "name_zh": "威州政治",
    "name_en": "Wisconsin Politics",
    "order": 1
  },
  {
    "id": "packers",
    "name_zh": "包装工",
    "name_en": "Green Bay Packers",
    "order": 2
  },
  {
    "id": "peninsula",
    "name_zh": "半岛",
    "name_en": "The Peninsula",
    "order": 3
  },
  {
    "id": "machine-war",
    "name_zh": "机器战争",
    "name_en": "The Machine War",
    "order": 4
  },
  {
    "id": "audit-desk",
    "name_zh": "审计处",
    "name_en": "The Audit Desk",
    "order": 5
  },
  {
    "id": "university-front",
    "name_zh": "大学战线",
    "name_en": "The University Front",
    "order": 6
  },
  {
    "id": "prize-circuit",
    "name_zh": "文苑",
    "name_en": "The Prize Circuit",
    "order": 7
  }
]
```

Notes:

- `wisconsin` and `packers` are net-new beats — they do not exist in the current
  scorecard and are to be **added**, not renamed from something already present.
  `peninsula` absorbs what the current `board`/`wars` entries file informally as DPRK
  position / Korea coverage, but broadened explicitly to include ROK politics and the
  US alliance leg, not just DPRK. `machine-war` is new (AI/chips geopolitics — export
  controls, open-weight releases, China's diffusion strategy, WAICO). `audit-desk` is
  new and is *also* a beat, in addition to being the home of a distinct entry type
  (§4) — the desk grading other analysts' predictions is itself a beat with its own
  coverage cadence. `university-front` (US higher-ed contraction) and `prize-circuit`
  (Korean literature in translation, Nobel, Booker International) are new.
- `id` is the stable slug used everywhere else (`beat: "wisconsin"`). Never rename an
  id once predictions reference it — treat beat ids as append-only, same spirit as
  §2's immutability rule for predictions.
- `order` is just a render hint for the desk index / nav; not load-bearing for grading
  logic.
- The existing `board` (前线态势) and `wars` (战况账簿) sections are not beats in this
  sense — they're the multipolarity/great-power-conflict trackers this scorecard
  already runs, and this proposal doesn't touch them. Beats are the newer, narrower
  desks (Wisconsin, Packers, Peninsula, Machine War, Audit Desk, University Front,
  Prize Circuit) that need their own filter/group identity going forward. Existing
  `coverage` items and `predictions` that logically belong to `peninsula` (e.g. the
  DPRK CMC-orders predictions) should get a `beat: "peninsula"` tag retroactively as
  a **pure addition of a new field** — this does not touch `claim`, `filed`,
  `resolution`, `resolves_by`, or `confidence`, so it does not violate immutability
  (§2a). Tagging existing entries with a beat is out of scope for this proposal; it's
  called out here so the human reviewer knows the schema supports it later.

---

## 2. Properly-filed prediction object

### 2a. Immutability approach

The hard requirement: once a prediction is filed, `claim`, `filed`, `resolution`,
`resolves_by`, and `confidence` never change. Two ways to get there; recommending the
second.

**Option A — append-only file.** Each prediction lives in its own JSON file
(`predictions/2026-07-17-peninsula-01.json`) or as one immutable object in an
append-only array where editing an existing array element is a lint-checked violation
(a pre-commit / CI check that diffs the file and rejects any diff touching the frozen
fields of an existing entry). Pro: filesystem/git literally enforces "new entries only
append." Con: more files, more indirection, breaks from the current single-file
`scorecard.json` pattern this repo has used from day one.

**Option B — frozen filed-block + separate mutable grading block, same object,
same file.** Each prediction is one JSON object, but its fields are split into two
groups by *convention*, not by nesting: a `filing` sub-object (frozen at creation) and
everything else (mutable: `status`, `outcome`, `mechanism`, `resolved`, `note`,
`void_reason`). Recommended, because:

- It keeps the single-file, human-editable, no-backend model intact — no new file
  layout, no build step.
- The frozen/mutable split is visually obvious in the JSON itself (nesting = intent),
  which is the cheapest possible enforcement mechanism for a hand-edited file: a human
  filing a correction sees the `filing` block sitting there and it *reads* as "don't
  touch this."
- A lightweight CI check (a script comparing `filing` blocks across git revisions of
  `data/scorecard.json`, failing the commit if any existing entry's `filing` block
  differs from its previous committed value) gets nearly all of Option A's rigor
  without the file-per-prediction overhead. This is a follow-up implementation detail,
  not part of this schema proposal, but the shape below is chosen to make that check
  trivial to write (diff one sub-object, not scattered top-level fields).

Shape:

```json
{
  "id": "peninsula-2026-07-17-01",
  "beat": "peninsula",
  "kind": "desk",
  "filing": {
    "claim": "The Kang Kon-class destroyer completes sea trials and is formally commissioned before October 1, 2026.",
    "filed": "2026-07-17",
    "resolves_by": "2026-10-01",
    "resolution": "Resolves HIT if DPRK state media (KCNA) or credible OSINT confirms commissioning ceremony/fleet integration before the date; resolves MISS if not yet commissioned by then.",
    "confidence": 70
  },
  "status": "open",
  "outcome": null,
  "mechanism": null,
  "resolved": null,
  "note": null
}
```

Field notes:

- `id` — new. A stable slug, distinct from the `#pred-N` positional anchor. `#pred-N`
  stays purely a render-time convention (N = array index), but a stable `id` lets a
  future audit-desk entry or a coverage item reference *this specific prediction* even
  if the array position shifts because older schema-version entries get migrated or
  reordered. Not strictly required by the immutability rule, but cheap and avoids
  brittle N-based cross-references outside the render layer.
- `filing` — the frozen block. `claim`, `filed`, `resolves_by`, `resolution`,
  `confidence` all live here and never change after commit.
- `status`, `outcome`, `mechanism`, `resolved`, `note` — mutable grading fields,
  live at the top level, filled in at resolution time (§2d/§2e).

### 2b. Resolution criteria required at filing

`filing.resolves_by` (date) and `filing.resolution` (one-line "resolves how"
criterion) are **both mandatory** at the moment a prediction is filed. A prediction
object missing either is invalid and must be rejected before commit — recommend a
simple validation script (`scripts/validate-scorecard.js` or similar, run as a CI
check / pre-commit hook) that walks `predictions` and `audit_predictions` and fails
the build if any entry's `schema_version` is 2 (see §5) and either field is absent or
empty. This is a schema requirement stated here; the validator itself is
implementation, out of scope for this document.

### 2c. Confidence

`filing.confidence` — integer, 0–100 inclusive. Represents the desk's stated
probability the claim resolves 对 (correct) at `resolves_by`. Required for all
`schema_version: 2` entries (§5); absent for legacy entries by design.

### 2d. Dual grading at resolution

Two separate fields, both required once a prediction resolves:

- `outcome` — `"对"` (correct) or `"错"` (wrong). Was the claim itself right or wrong.
- `mechanism` — `"held"` or `"didn't hold"`. Did the *reasoning* the desk gave in
  `filing.claim` turn out to be the actual causal path, independent of whether the
  outcome landed correctly.

This directly generalizes the existing correction pattern already visible in the
current ledger (Prediction 16: outcome correct, mechanism wrong — the desk called the
nuclear-talks suspension right but named the wrong trigger). Under the new schema that
distinction is a first-class graded field instead of prose buried in `corrections`.

Example of a resolved entry:

```json
{
  "id": "peninsula-2026-06-29-03",
  "beat": "peninsula",
  "kind": "desk",
  "filing": {
    "claim": "Iran suspends participation in the nuclear-chapter working groups before August 1, 2026, citing continued IDF operations in Lebanon.",
    "filed": "2026-06-29",
    "resolves_by": "2026-08-01",
    "resolution": "Resolves 对/HIT if nuclear working groups go into abeyance before Aug 1 for any stated reason; the specific trigger is graded separately via `mechanism`.",
    "confidence": 65
  },
  "status": "对",
  "outcome": "对",
  "mechanism": "didn't hold",
  "resolved": "2026-07-17",
  "note": "Trigger was US strikes on Iranian territory, not IDF Lebanon operations as filed. Effect called correctly; mechanism named incorrectly. See corrections log."
}
```

### 2e. Status lifecycle

`status` — one of:

- `"待"` (OPEN) — default at filing, before `resolves_by` or before the desk has
  graded it.
- `"对"` (CORRECT) — resolved, `outcome: "对"`.
- `"错"` (WRONG) — resolved, `outcome: "错"`.
- `"废"` (VOID) — the claim became ungradeable (e.g. the underlying event was
  overtaken by circumstances the filing couldn't anticipate — not "the desk changed
  its mind"). VOID entries are excluded from Brier and from raw hit/miss tallies.

VOID requires a dated reasoning note:

```json
{
  "status": "废",
  "outcome": null,
  "mechanism": null,
  "resolved": "2026-08-02",
  "void_reason": {
    "date": "2026-08-02",
    "text": "The Aug 1 resolution criterion assumed the working-group format would still exist; the format itself was dissolved and folded into a different bilateral channel on July 29, making the original resolution criterion inapplicable. Not a hit, not a miss — voided."
  }
}
```

`kind` — `"desk"` (own analytical prediction) or `"tea"` (the standing lighter-fare
category, e.g. World Cup calls) — preserved unchanged from the current schema.

`beat` — one of the ids from §1. Required on every `schema_version: 2` prediction.

---

## 3. Brier score

**Computed at render time in JS, not stored in the data file.** No `brier` field
anywhere in `scorecard.json` — this keeps the JSON the single source of truth and
avoids a second place that can drift out of sync with the underlying grades.

Formula, applied over **resolved, non-legacy, non-void predictions only**
(`schema_version: 2`, `status` in `{"对","错"}`):

```
brier = mean( (confidence/100 − outcome)² )
```

where `outcome = 1` if `status === "对"`, `outcome = 0` if `status === "错"`. Entries
with `status === "废"` (VOID) or `status === "待"` (OPEN) are excluded from the mean.
Legacy entries (§5) are excluded regardless of status, because they carry no
`confidence` to score.

Two scores, both computed the same way over different subsets:

- **Overall desk Brier** — mean over all qualifying predictions across every beat.
- **Per-beat Brier** — mean over qualifying predictions filtered to `beat === X`,
  computed for each beat independently. A beat with zero qualifying (resolved,
  scored) predictions renders as "insufficient data" / em-dash, not a fabricated 0.

Rendering rule (ties directly to the site's 红涨绿跌 convention, and is the one place
this proposal must be explicit to avoid an inversion bug): **lower Brier is better.**
A good, well-calibrated record is a *low number*, and under 红涨绿跌 a good outcome
renders red, not green. Concretely: the Brier display must not use a "lower =
green/negative" convention borrowed from finance dashboards — that would invert the
site's established color logic. If the Brier number needs a color/trend indicator at
all, "score improving over time" (Brier trending down) reads as 涨/red; "score
worsening" (Brier trending up) reads as 跌/green. This is a rendering note for the
future UI work, recorded here so it isn't relitigated or gotten backwards later.

Example of the computed shape (illustrative only — not stored, just what the render
function would hold in memory):

```json
{
  "overall": { "brier": 0.187, "n": 12 },
  "by_beat": {
    "peninsula": { "brier": 0.140, "n": 5 },
    "machine-war": { "brier": 0.220, "n": 3 },
    "wisconsin": { "brier": null, "n": 0 }
  }
}
```

---

## 4. Audit Desk entries (审计处)

A **different schema** from own-predictions — the audit desk grades *other analysts'*
claims, not the desk's own. New top-level key `audit_predictions`, same immutability
approach as §2a (frozen `filing` block, mutable grading fields).

```json
{
  "id": "audit-2026-07-17-01",
  "beat": "audit-desk",
  "filing": {
    "source": "Peter Zeihan",
    "claim": "China's economy enters a demographically-driven structural collapse within 24 months, paraphrased from his July 2026 newsletter — not a verbatim quote.",
    "claim_verbatim": false,
    "where_filed": {
      "label": "Zeihan on Geopolitics — newsletter, July 2026",
      "url": "https://example.com/zeihan-newsletter-july-2026"
    },
    "their_implied_date": "2028-07-01",
    "our_filing_date": "2026-07-17",
    "resolves_by": "2028-07-01",
    "resolution": "Resolves 对 (their claim correct) if independently reported PRC GDP/industrial data shows structural contraction consistent with 'collapse' framing by mid-2028; resolves 错 if the economy is still expanding or managing a soft rebalancing as it is today."
  },
  "status": "待",
  "outcome": null,
  "reasoning_held": null,
  "resolved": null,
  "note": null
}
```

Field notes, differences from §2:

- `source` — who made the claim (person/outlet), required.
- `filing.claim` — their claim, and `claim_verbatim` (boolean) states whether `claim`
  is an exact quote or the audit desk's paraphrase. If verbatim, quote it exactly; if
  paraphrased, say so via this flag so a future reader knows how much interpretive
  latitude the desk took.
- `where_filed` — link/citation to the source claim (object with `label` + `url`,
  consistent with how `sources`/`links` are already shaped elsewhere in this file).
- `their_implied_date` — the deadline *their* claim implies (may differ from
  `resolves_by` if the desk judges their stated timeline needs a grace period —
  but default is the two match).
- `our_filing_date` — when the audit desk logged the claim, distinct from
  `their_implied_date` (the desk may log something well after it was originally made,
  e.g. auditing a claim from months back).
- `resolves_by`, `resolution` — same requirement as §2b, mandatory at filing.
- No `confidence` field — the audit desk is not staking its own probability, it's
  tracking whether *someone else's* claim resolves true or false. (If a future need
  arises to grade the audit desk's own confidence in judging the claim, that's a
  schema addition, not something this proposal invents speculatively.)
- `outcome` — `"对"` / `"错"`, same two-value grade as §2d, applied to whether their
  claim was correct.
- `reasoning_held` — optional boolean/text, parallel to `mechanism` in §2d: did *their*
  stated reasoning hold, independent of whether the outcome landed. Optional because
  the audit desk may not always have visibility into the other analyst's reasoning
  chain the way it has full visibility into its own.
- Same `status` lifecycle as §2e (待/对/错/废), same VOID dated-reasoning-note
  requirement.
- Audit-desk entries are **excluded from the desk's own Brier score** (§3) since they
  carry no `confidence` and aren't the desk's own calibration record — they're a
  separate accountability ledger for third parties. If a future iteration wants a
  "how well does the audit desk judge others" metric, that's a distinct calculation
  (e.g. simple accuracy, not Brier) and out of scope here.
- Deep-link convention extends as `#audit-N`, mirroring `#pred-N`, N = 1-based
  position in the `audit_predictions` array.

---

## 5. Legacy predictions

The ~18 existing entries in `predictions` were filed without `confidence` or a
structured `resolution` criterion. Immutability (§2a) forbids retroactively inventing
a probability or a resolution string for them — that would be fabricating history, not
recording it.

Resolution:

- Legacy entries resolve on **outcome only** (对/错, mapped from today's
  `status: "hit"`/`"miss"`/`"open"` values — trivial one-time rename, not a content
  edit, since `hit`/`miss`/`open` are English labels for the same 对/错/待 states this
  proposal formalizes).
- Legacy entries are **excluded from the Brier score** (§3) — they have no
  `confidence` to score against. The Brier score starts fresh with the
  `schema_version: 2` cohort going forward.
- Distinguishing legacy vs. new: add a `schema_version` field to every prediction
  object. Existing entries get `schema_version: 1` (or the field's *absence* implies
  1 — either works, but an explicit `1` is less error-prone for a validator to check).
  New entries filed under this proposal get `schema_version: 2` and must carry the
  full `filing` block from §2a.
- Critically: adding `schema_version: 1` to an existing object is **not** an edit to
  `claim`, `filed`, `resolution`, `resolves_by`, or `confidence` — none of those
  fields exist on legacy entries to begin with, so tagging them with a version number
  doesn't touch anything immutability protects. This is the smallest possible
  migration: one new field per existing object, nothing else changes.

Example of a migrated legacy entry (illustrative — the *only* change from the current
file is the added `schema_version` line; every other field is untouched):

```json
{
  "schema_version": 1,
  "status": "hit",
  "kind": "desk",
  "claim": "Konstantinovka comes under full Russian operational control (80%+ of city area) before July 4, 2026. ...",
  "filed": "2026-06-17",
  "resolved": "2026-06-23"
}
```

vs. a new-cohort entry:

```json
{
  "schema_version": 2,
  "id": "wars-2026-07-17-01",
  "beat": "peninsula",
  "kind": "desk",
  "filing": {
    "claim": "...",
    "filed": "2026-07-17",
    "resolves_by": "2026-09-01",
    "resolution": "...",
    "confidence": 55
  },
  "status": "待",
  "outcome": null,
  "mechanism": null,
  "resolved": null,
  "note": null
}
```

A render-time check is simple: `schema_version !== 2` (or field absent) → render
without a confidence column, exclude from Brier, group under a "legacy record" visual
treatment if desired; `schema_version === 2` → full grading display.

---

## Open questions for the human reviewer

1. Is the `filing` sub-object nesting (Option B, §2a) acceptable, or is the stronger
   guarantee of Option A (append-only separate files / CI-enforced immutability)
   worth the added file-layout complexity?
2. Should `audit_predictions` live in the same `scorecard.json` or a sibling file
   (e.g. `data/audit-desk.json`)? This proposal defaults to same-file for consistency
   with the no-backend, single-source-of-truth model, but a separate file might be
   cleaner given it's a structurally distinct entry type.
3. Confirm the `id` field convention (`{beat}-{filed-date}-{seq}`) is acceptable, or
   propose an alternative stable-id scheme.
4. Confirm VOID's dated reasoning note (`void_reason`) is sufficient, or whether VOID
   should also require sign-off language distinguishing "desk error" voids from
   "external circumstance made this ungradeable" voids.

No UI, styling, or existing-file changes are proposed or implied here. This is schema
only, pending approval.
