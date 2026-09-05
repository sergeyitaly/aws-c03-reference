---
name: manage-saa-c03-reference
description: How to safely extend and maintain SAA-C03-reference.html — a single-file bilingual (EN/UK) AWS SAA-C03 exam-prep artifact with flashcards, a 100-question quiz, two architecture games, a Sorting Rush drag-and-drop game, and a free-form Sandbox architecture board (101-service catalog, real-connection-only linking, VPC/Subnet grouping, undo/redo, and a 48-pattern curated reference-architecture library covering every catalog service). Use this whenever adding/editing content in that file (new quiz questions, flashcards, falling items, glossary entries, sandbox services, architecture patterns), before publishing it as a Claude Artifact, or when auditing it for out-of-scope AWS services or EN/UK drift.
---

# Managing SAA-C03-reference.html

This is a single self-contained HTML file (~8000 lines) — no build step, no
dependencies. It's edited directly and published as a Claude Artifact. There
is no framework here: every "component" is a plain JS object rendered via
template strings into the DOM by `buildApp()`.

## Before touching content: check exam scope

**Read `SAA-C03-service-reference.md` in this same directory first** — it's
the source-of-truth outline this file was built from (21 sections, numbered
0–20, matching the HTML's `SECTIONS` exactly).

Then check this out-of-scope list before adding *any* AWS service that isn't
already in the file, even if it sounds plausible for a "solutions architect"
exam:

> Lightsail, RDS on VMware, Cloud9/CDK/CloudShell/Code\* family (CodeCommit,
> CodeBuild, CodeDeploy, CodePipeline, CodeStar), Fault Injection Simulator,
> Location Service, GameLift/Lumberyard, all IoT services, CloudSearch, MWAA,
> Managed Blockchain, App Mesh, Cloud Map, most niche ML (Personalize,
> DevOps Guru, Lookout family, Panorama, Inferentia, SageMaker Data
> Wrangler/Ground Truth, DeepRacer/DeepLens/DeepComposer), Elemental media
> services, IVS, Migration Evaluator, Braket, RoboMaker, Ground Station,
> OpsWorks, Chatbot.

This list came from the official exam guide PDF appendix (fetch
`https://d1.awsstatic.com/training-and-certification/docs-sa-assoc/AWS-Certified-Solutions-Architect-Associate_Exam-Guide_C03.pdf`
and read the saved copy if the list looks stale — `WebFetch` can't parse the
PDF directly but does save it locally). Domain weights: Secure 30% /
Resilient 26% / High-Performing 24% / Cost-Optimized 20%. Pass mark 720/1000.

If a service you want to add isn't on that list and isn't already in the
file, it's probably fine — but do a quick sanity check against the domain
it'd belong to before writing five questions about it.

## The one rule that matters most: EN/UK parity

Every content array exists twice — `QUIZ_EN`/`QUIZ_UK`,
`GAME_DESIGN_EN`/`GAME_DESIGN_UK`, `GAME_FLAW_EN`/`GAME_FLAW_UK`,
`FALLING_ITEMS_EN`/`FALLING_ITEMS_UK`, `SECURITY_ITEMS_EN`/`_UK`,
`PERFORMANCE_ITEMS_EN`/`_UK`, `SECTIONS_EN`/`_UK`, `ARCHITECTURES_EN`/`_UK`.
They must have **identical length and the same conceptual order** — index
`i` in the EN array and index `i` in the UK array must be the same question/
round/item, just translated. Nothing enforces this at runtime; it only shows
up as a silently wrong translation or a crash when you switch languages.

A `let X = currentLang === "uk" ? X_UK : X_EN;` line exists for every one of
these pairs, both once at top-level and once again inside `setLanguage()`.
**If you add a brand-new bilingual pair, you must add this reassignment in
both places**, or the array silently stays on whatever language loaded first.

AWS product/service names and JSON/policy syntax are **never translated** —
same convention throughout (`"AWS Lambda"`, `"Effect": "Allow"` etc. stay
identical in both `_EN` and `_UK`). Only the surrounding prose changes.

### Never assume text is per-language when it might be shared

A few things are deliberately **English-only regardless of display
language**, by design, not by omission:
- **Diagram content** (`diagram:()=>compareDiagram([...])` on each section):
  `mountDiagram(...)` always calls `sEn.diagram()`, the English section's
  diagram function. There is no UK diagram text to edit.
- **The glossary jump-to-card matching key**: `card.dataset.title` is always
  set from `cEn.t` (the positionally-corresponding *English* card title),
  not the currently-displayed title. `ABBR_LIST` entries' `cardTitle` field
  is therefore always an English string, and matches correctly in both
  languages **as long as the EN and UK `cards` arrays for that section stay
  positionally aligned** (same index = same card). If you reorder or
  insert/delete a card in one language's `cards` array without doing the
  identical thing in the other, this silently breaks — the jump link will
  point at the wrong card by index.

## Correct-answer shuffling — do not hand-place `k` values

Quiz options and Spot-the-Flaw options are shuffled **once at page load**:

```js
[QUIZ_EN, QUIZ_UK].forEach(quiz=>{
  quiz.forEach(q=>{ q.opts = shuffleArrayOnce(q.opts).map((o,i)=>({...o, k:"ABCD"[i]})); });
});
[GAME_FLAW_EN, GAME_FLAW_UK].forEach(list=>{ list.forEach(r=>{ r.opts = shuffleArrayOnce(r.opts); }); });
```

This exists specifically because an earlier version had the correct answer
always in the same position (a real, once-shipped bug). When you add a new
question, **write the options in whatever order feels natural** (correct
answer can be `opts[0]`) — do not manually add a `k` field, the shuffle
overwrites it. Don't remove this shuffle block.

## Every AWS-policy-JSON quiz question uses a specific pattern

Several quiz questions (topics `"IAM Role Policy"` / `"IAM Identity Policy"`)
embed a real JSON policy snippet in the question text via
`<pre class="policy-json">...</pre>`, and use `<code>...</code>` inside
option text for inline policy-line references. Write these fields as
**template literals (backticks)**, not double-quoted strings — a hand-typed
JSON policy is full of double quotes that would otherwise need escaping on
every line. The one trap: if a policy legitimately contains `${...}` (e.g.
the IAM policy variable `${aws:username}`), you must write it as
`\${aws:username}` inside the template literal, or JS will try to evaluate
it as an interpolation expression and crash immediately at parse time.

## Exam Traps tab specifics (if touching that content)

`EXAM_TRAPS` (panel-traps) is a **single bilingual array**, not an
`_EN`/`_UK` pair — each entry carries `title`/`explain`/every `step`'s
`label`/`note`/`arrowLabel` as `{en, uk}` objects resolved via
`[currentLang]` at render time, same convention as `PATTERNS` and
`LIMITS_LIST`. This is deliberately simpler than the `QUIZ_EN`/`QUIZ_UK`
convention used elsewhere — there's no risk of the two languages drifting
out of index alignment because there's only one array. Don't split it
into two arrays "for consistency" with SECTIONS/QUIZ; that would only
reintroduce the exact parity-drift risk this shape avoids.

Each trap renders via `flowDiagram(steps, {height:118})` (the same
box+arrow primitive used for architecture-pattern diagrams) — `steps` is
2-3 `{label, note, fill, stroke, arrowLabel, icon}` entries. **Every
step must be red or green, never neutral** — `fill:"rgba(220,38,38,.13)",
stroke:"#DC2626"` for the wrong/trap side, `fill:"rgba(22,163,74,.13)",
stroke:"#16A34A"` for the correct side (same hex values as the quiz's
`.correct`/`.incorrect` states elsewhere in the file). This was a real
correction mid-build: several traps that were genuine "A vs B, both
legitimate" comparisons (ALB/NLB, Redis/Memcached, Standard/Express,
preventive/detective guardrails) originally used a neutral
`var(--diagram-box)` fill for the non-"correct" side, which the user
explicitly rejected — every trap must be reframed as a **wrong
assumption** (red) vs the **correct reality** (green), even a pure
comparison; state the wrong belief in words on the red box rather than
leaving one side merely "not decided yet." Keep `note` text short (one
line, ~28 chars) — `flowDiagram` renders it as a single non-wrapping
`<text>` element, unlike `label` which does wrap via `wrapText`.

**`icon`** (optional, per step) draws a small circular AWS-service icon
badge via `diagramServiceIcon()` — pass a real catalog service name
(checked against `RADIAL_CATEGORIES` the same way as everywhere else,
via `svcServiceGlyph`). **It floats fully above the box, centered on its
top edge — never inside it.** The first version put it inside the box's
top-left corner and the user immediately flagged it as overlapping the
label text on longer labels; `flowDiagram` now bumps its own height to
at least 182 whenever any step has an `icon` (even overriding a smaller
explicit `opts.height`, since a caller written before icons existed
can't know to leave room), specifically so the badge always has clear
space above the box regardless of label length or line count. Don't move
the icon back inside the box without re-deriving that vertical math —
the whole point of floating it above is that it stays label-length-
independent. Only set `icon` when a step genuinely IS a real service
(`"RDS"`, `"NAT Gateway"`, `"KMS"`...) — leave it off entirely for steps
that are concepts, not services (e.g. "Management account", "Org-wide
SCP", "Reserved Instances" — there's no catalog node for a purchasing
option). A wrong/nonexistent name just silently renders no icon
(`svcServiceGlyph` returns `""`), so verify new icon names the same way
as the Sandbox's `SERVICE_ICON_MAP` gotcha.

The traps panel itself is split into **4 domain sub-tabs**
(`.trap-domain-btn`, `#trapDomainTabs`), not one long scrolling list —
`renderTrapDomain(domainNum)` clears and re-renders `#trap-list` for
just that domain, defaulting to domain 1 on panel build/language switch.
New trap entries go under the right `// ---- Domain N: ... ----` comment
block inside `EXAM_TRAPS` and need a `domain` (1-4) matching the real
exam weighting the tab label cites — `renderTrapDomain` filters by this
field, so a wrong number silently puts a trap under the wrong tab
instead of erroring.

## The topic → section → sidebar-group pipeline

The sidebar groups sections into 8 categories (Identity, Compute, Data,
Networking, Storage, Integration, Ops, Security) via each section's own
`group` field in `SECTIONS_EN`/`SECTIONS_UK` — this is the authoritative
category taxonomy, already bilingual, already correct. Don't invent a
separate category scheme.

Two lookup tables roll other content up to this taxonomy:
- `QUIZ_TOPIC_SECTION`: quiz question `topic` string → section id. Contains
  both the English topic string *and* a UK-language alias for every topic
  whose UK translation differs from its EN text (about half of them) —
  because `QUIZ` can be the UK array or a shuffled array, you cannot assume
  `qd.topic` is always English at lookup time.
- `FALLING_ITEM_SECTION`: every Sorting Rush item id (`c1`–`c100`,
  `e1`–`e100`, `h1`–`h100`, `s1`–`s100`, `p1`–`p100`) → section id.

**If you add a new quiz question or a new falling item, add its mapping to
the relevant table in the same edit** — nothing will crash if you forget
(the lookup just silently returns `undefined` and that item is excluded from
the sidebar progress bars / "Read here" links), but it becomes invisible to
the feature it's supposed to feed.

## Sorting Rush specifics (if touching that game)

- Three difficulty modes (`standard`/`hard`/`veryhard`) add categories
  (`security`, `performance`) on top of the base three
  (`cost`/`effort`/`ha`) — see `SORT_DIFFICULTY_CATEGORIES` and
  `SORT_CATEGORY_META` (icon/color per category).
- `sortRefillPool()` uses a greedy "largest remaining bucket, excluding the
  previous pick" algorithm to guarantee no two consecutive falling items
  share a category — a naive shuffle-then-patch approach can strand a
  same-category run at the tail. Don't replace this with a simpler shuffle
  without re-verifying that guarantee (a jsdom test checking
  `maxConsecutiveRun === 0` across a full pool is the way to check it).
- Per-item **difficulty rating** (`sortItemRatings`, persisted to
  `localStorage`) grows on wrong/missed answers and decays on correct ones,
  and controls how many duplicate copies of that item go into the pool on
  each refill (1–4 copies) — this is what makes historically-hard items
  fall more often. It floors at 1 copy, never fully removing an item.
- **Score only counts correct answers** (no penalty for wrong/missed) —
  this was deliberately changed after user feedback; lives are what track
  failures. Don't reintroduce a `sortScore--` on the wrong-answer path.
- "Review missed" mode pulls its pool from `sortPersistentMissed`
  (cross-session, id + missCount only — never frozen label/why text, always
  re-looked-up via `sortItemById()` so it survives language switches and
  content edits) instead of the full category pool.
- The basket hover tooltip (paused-state "what's in this basket") uses a
  debounced show/hide (`sortCancelTooltipHide`/`sortScheduleTooltipHide`)
  specifically because the tooltip renders *outside* the basket's own
  visual box (positioned above it) — an instant hide-on-mouseleave would
  make it impossible to move the pointer into the tooltip to scroll it.

## Sandbox specifics (if touching the architecture board)

The Sandbox is a free-form board where the user drags any of the 95
services in `RADIAL_CATEGORIES` onto a canvas, connects them, and groups
them into VPC/Subnet containers. It's the closest thing in this file to a
real app, with its own data model — read this before touching it.

**Data model**: `sandboxNodes=[{id,name,x,y,variants}]` (`id` is the
node's unique board identity — auto-numbered "Name (2)" for a second copy
of the same service via `sandboxGenerateId`; `name` is the underlying AWS
service type, used for icon/facts/connection/variation lookups — never
assume `id === name` once duplicates exist), `sandboxLinks=[[idA,idB],...]`,
`sandboxGroups={[ownerId]:[memberId,...]}` (VPC/Subnet containment is
**not** represented as a link — it's a separate rectangle drawn around the
members' bounds, see `sandboxGroupBounds`).

**Every new sandbox service needs five things updated in lockstep**:
an entry in `RADIAL_CATEGORIES` (under the right category, with `n`/`f`/
`uc`), a hand-drawn glyph in `GLYPH_LIB` (never a copy of the real AWS
icon — see the IP note below), an entry in `SERVICE_ICON_MAP`, a verified
doc URL in `RADIAL_DOC_URLS`, and its real pairings added to
`SERVICE_CONNECTIONS`. Missing any of these doesn't crash — it just makes
the service placeable-but-broken (no icon, unlinkable, or absent from the
catalog).

**`SERVICE_CONNECTIONS` is directional, not symmetric — `X`'s array
lists what `X`'s data/requests flow TO, never the reverse.** This was a
real bug, not a design choice from day one: it used to be a symmetric
adjacency list, so a link's on-board arrow direction depended on which
node the user happened to click *first* rather than which way data
actually flows — "Kinesis Data Firehose" clicked before "Kinesis Data
Streams" baked in a backwards `Firehose → Streams` arrow, when the real
flow is `Streams → Firehose`. When adding a new service's pairings,
decide the real direction (who initiates / where the data actually
goes) and put the edge on **only** the source's side — do not also add
the reverse to the target's side "for symmetry"; that reintroduces the
exact ambiguity this was fixed to remove. `sandboxIsValidConnection`
still checks both directions for plain yes/no validity
(`SERVICE_CONNECTIONS[a].includes(b) || SERVICE_CONNECTIONS[b].includes(a)`),
but `sandboxAddLink` → `sandboxCanonicalOrder` uses the *one-sided*
lookup to orient the actual arrow, and `sandboxRenderLinks` now draws a
real arrowhead (`marker-end`, defined fresh each render since the SVG's
`innerHTML` is cleared) — direction used to only be visible in the
text-based Workflow panel's `A → B` labels, never on the board itself.
The flip button (⇄, in the Workflow panel) still manually reverses one
link's `[a,b]` order for the rare case the canonical direction is wrong
for a specific diagram.

A service that's purely a *destination* (e.g. `S3 Glacier`, `Security
Hub`, `QuickSight`) legitimately has an **empty** `SERVICE_CONNECTIONS`
array of its own — nothing flows *out* of an archive tier or a
dashboard. Don't "fix" an empty array by adding reverse edges; that's
correct, not a gap. But the "commonly connects to" UI (hold-menu
suggestions, the info panel's pills, `sandboxApplyHighlights`) needs to
show suggestions from **both** directions or a pure-destination service
would wrongly show zero — that's what `sandboxRelatedServices(name)`
is for (unions `SERVICE_CONNECTIONS[name]` with every key whose own
array includes `name`). Those three UI call sites use
`sandboxRelatedServices`, never a raw `SERVICE_CONNECTIONS[name]` — if
you add a fourth "show related services" surface, use it too, or a
target-only service will look connection-less there by mistake.
**`SERVICE_ICON_MAP`'s value isn't always a `GLYPH_LIB` key directly** —
`svcServiceGlyph` checks `ICON_META[key]` (the per-*section* badge icons,
keyed by section id like `"ec2-fund"`) first and only falls back to
`GLYPH_LIB[key]` if there's no section with that id. That's why entries
like `"EC2":"ec2-fund"` work with no matching `GLYPH_LIB.ec2-fund` at
all — they're deliberately reusing a section's icon, not missing one. A
brand-new container/service without a natural section to borrow from
(like Region/Availability Zone) needs its own real `GLYPH_LIB` entry
instead, under a key that doesn't collide with any section id.

**Jumping from a reference flashcard into the Sandbox**: a card gets a
"Place on Sandbox" button (`jumpToSandboxPlace`) only when its own title
is an *exact* match against a real Sandbox service name, checked live via
`sandboxFindServiceMeta(cEn.t)` — never a hardcoded list. This is
deliberately conservative: most cards are concepts or comparisons ("NAT
Gateway vs NAT instance", "DAX vs ElastiCache"), not a single service, so
guessing a service out of those would be wrong more often than right.
When you add a new card whose title is literally a service name already
in `RADIAL_CATEGORIES` (e.g. a new standalone "Kinesis Data Firehose"
card), the button appears automatically — no wiring needed on your end.

**Parking a node** (freeing board space without deleting it): right-click
any node → "Park" sends it into `sandboxParked` (an array of `{name,
variants}`, no id/x/y since it isn't placed anywhere) and removes it from
the board via the same cleanup as `sandboxRemoveNode`. It shows up as a
chip in the collapsible `#sandboxParkedSection` above the palette;
tapping the chip calls `sandboxPlaceNode(name, x, y, variants)` — the
4th `variants` param exists specifically so a parked-then-restored node
keeps whatever variant was set on it, not a blank one. `sandboxParked` is
threaded through the exact same places as `sandboxNodes`/`sandboxLinks`/
`sandboxGroups` — `sandboxSnapshot`/`sandboxRestoreSnapshot` (so
park/unpark is a normal undo-able commit) and `sandboxPersist`'s
localStorage payload — miss one of those and parking either doesn't
survive undo or doesn't survive a reload. The context menu's old
`if((!variations||!variations.length) && !isContainer) return;` guard
was removed since Park needs to be offered on every node, not just ones
that already had a variant/group checklist — if you add another
menu-only feature, don't reintroduce a guard that skips plain nodes.

**Deleting a placed node** (permanent — contrast with Park above, which
keeps it): double-click it, or **drag it back onto the palette**
(`sandboxIsOverPalette` checks the drop point in screen space against
`#sandboxPalette`'s bounding rect, since the palette sits outside the
board's own scroll/zoom coordinate space) — the palette highlights
(`.sandbox-remove-target`) while a node is dragged over it, and dropping
there calls `sandboxRemoveNode` instead of the normal auto-group/resolve-
overlap/save-state drop path in `sandboxWireNode`'s `pointerup` handler.
Both paths go through the same `sandboxRemoveNode`, so undo/redo and
link/group cleanup behave identically regardless of which one the user
used. The context menu itself has no delete item — only the variant/group
checklists and Park — since Park already covers "get it off the board"
without the destructive, no-variants-remembered downside of deleting it.

**Only documented pairs can link, and the arrow always points the
canonical way regardless of click order.** Clicking two unrelated
services selects the second one instead of drawing a connection
(deliberate — previously any two nodes clicked in sequence would link
regardless of whether the pairing made architectural sense). If you add
a new service or pattern and find yourself unable to connect two nodes
that plausibly *should* connect, the fix is almost always a missing
`SERVICE_CONNECTIONS` entry, not a bug in the linking logic. If the link
draws but the **arrow points the wrong way**, the fix is in
`SERVICE_CONNECTIONS` too — the edge is on the wrong side (see the
directional-data note above) — not in `sandboxAddLink`/
`sandboxCanonicalOrder`, which just reads whatever direction the data
says.

**Two different 3-second hold-menus exist — don't conflate them.**
Holding a **placed board node** opens `sandboxOpenHoldMenu` ("commonly
connects to", built from `sandboxRelatedServices`, click adds+links
that service to the board). Holding a **palette chip** (not yet placed)
opens `sandboxOpenArchHoldMenu` ("used in these reference
architectures", built by filtering `SANDBOX_ARCH_PATTERNS.services`,
click calls `sandboxLoadArchitecture` and replaces the whole board).
Different trigger element, different data source, different click
action — but both reuse the same `sandboxHoldMenuEl`/
`sandboxCloseHoldMenu` so only one popup is ever open regardless of
which kind, and both live behind a plain `mouseenter`-starts/
`mouseleave`-cancels 3000ms `setTimeout` (no reset on `mousemove` within
the element — only entering/leaving toggles the timer). The palette
version is wired **only** inside `sandboxWirePaletteChip`, not inside
the shared `sandboxWireDraggableChip` itself — the info panel's
"commonly connects to" pills (`.sandbox-connect-pill`) call
`sandboxWireDraggableChip` directly and must **not** gain this behavior;
if you ever refactor chip wiring, keep that separation intentional
rather than "simplifying" it into one shared function.

**Container types**: `SANDBOX_CONTAINER_TYPES =
new Set(["Region","Availability Zone","VPC","Subnets"])` — four levels
matching real AWS nesting (a VPC is Region-wide; only its Subnets, and
AZ-scoped resources like NAT Gateway, are pinned to a specific AZ).
Nesting order is **not enforced** anywhere — a user can drag any
container into any other and it'll auto-group by proximity same as
before Region/AZ existed; the container's own docs/facts explain the
real relationship instead. Each type gets a distinct border style via
`SANDBOX_CONTAINER_DASH` (Region/VPC solid-ish, AZ long-dash, Subnets
medium-dash) so a 3-4 level nest stays legible — add a new container type
there too, or it silently falls back to the Subnets dash pattern.

**Containment, not links.** `sandboxIsContainerLink` blocks a container
from being connected with a line at all; instead, dropping a node inside
(or near) a container's bounds auto-joins it to that container's group
(`sandboxAutoGroupOnDrop`), drawn as an enclosing rectangle. This works
recursively — a VPC containing a Subnet containing an EC2 instance nests
correctly because `sandboxGroupBounds` folds each container's own full
rectangle into its parent's, with a `visited` Set guarding against cycles
and a `GROUP_MARGIN` (6px) clamp so a member near the canvas edge doesn't
get its box clipped. **Paint order in `sandboxRenderGroups`**: rects are
sorted **smallest area first**, so the biggest/outermost container (VPC)
is drawn last and paints on top of any nested container (Subnets) — a
container's own polygon and label should always sit above the things it
contains, not the reverse. If you touch this sort, re-verify with the
label `<text>` DOM order (later = painted on top), not just a visual
guess.

**Orphan-node highlight**: `sandboxOrphanNodeIds()` is the single source
of truth for "this node isn't part of anything" (zero real links —
container-containment pairs don't count — and not a group owner or
member either), shared by the board's own dashed-amber visual highlight
(`.sandbox-node-orphan`, applied in `sandboxApplyHighlights`), the hover
tooltip's warning line, and the Workflow panel's "Also on the board"
text — all three read the same Set so they can never disagree about
which nodes count. This is deliberately the *only* "is this needed"
judgment the Sandbox makes automatically. It does **not** try to flag a
service as "redundant" just because an alternate path also exists on
the board (e.g. Firehose feeding both S3 and a direct Streams→Redshift
edge isn't wrong — Firehose might still be there for the S3 leg) — that
kind of call depends on requirements the board has no way to know, and
a wrong automatic "redundant" flag would be actively misleading study
content. If a future ask wants that kind of detector, it needs a
narrow, explicitly-scoped list of documented AWS "path A supersedes path
B for use case X" pairs, not a generic heuristic.

**The reference-architecture pattern library** (`SANDBOX_ARCH_PATTERNS`,
48 entries as of the last major addition — every one of the 101 catalog
services now appears in at least one pattern, closed by a 20-pattern
sweep plus extending `three-tier`/`hybrid-network` to absorb the 8
fine-grained VPC/networking primitives — Region, Availability Zone,
Subnets, Security Groups, Network ACL, NAT Gateway, Internet Gateway,
VPC Endpoints — that don't warrant standalone patterns of their own; plus
`On-Premises` and `Snowball` added afterward, wired into `hybrid-network`,
`database-migration`, and `on-prem-data-sync` rather than as new
standalone patterns) is used two ways: **matching**
(`sandboxMatchArchPatterns` scores the current board against every
pattern by service-name overlap and shows the closest ones) and
**loading** (`sandboxLoadArchitecture(patternId)` clears the board,
places that pattern's services in a grid, and draws only the links that
`SERVICE_CONNECTIONS` actually supports between them). Adding a new
pattern:
1. Use only service names already in `RADIAL_CATEGORIES` — verify each
   one resolves, don't invent new services just for a pattern.
2. Write bilingual `title`/`blurb` (EN + UK, natural phrasing, not
   machine-translation-flavored) matching the existing entries' format.
3. If the pattern is a genuinely canonical, well-known AWS reference
   architecture, add a real URL to `SANDBOX_ARCH_REAL_URLS` keyed by the
   pattern's `id` — **curl-verify it returns HTTP 200 first**
   (`curl -s -o /dev/null -w "%{http_code}" -L <url>`). Never guess or
   fabricate an AWS URL; `sandboxArchDocUrl` only shows this real link on
   a 100%-match score, falling back to a generic docs-search query
   otherwise, so a wrong URL here would look authoritative and be wrong.
4. Insert immediately before the array's closing `];` — validate the new
   count and re-check for duplicate `id`s afterward (a jsdom snippet
   evaluating the array and asserting `ids.length === new Set(ids).size`
   catches this instantly).

**Known gotcha — hidden-tab zero-size rendering.** If the Sandbox tab
isn't the active tab when the page loads (e.g. reloading while on another
tab), `offsetWidth`/`offsetHeight`/`clientWidth` all genuinely read `0`
for elements under a `display:none` ancestor, in every real browser — not
just jsdom. This previously caused connections/groups to look like they'd
vanished after a refresh. The fix already in place has three parts, all
of which need to stay if you touch this code: (1) safe non-zero fallbacks
in `sandboxResolveOverlaps` and `sandboxGroupBounds` (`w:el.offsetWidth||120`,
never `||0`), (2) a `sandboxRenderGeneration`-guarded `requestAnimationFrame`
follow-up render in `initSandboxGamePanel()` that's a no-op if a fresh
render already happened, and (3) a `selectTab(id)` hook that calls
`sandboxResolveOverlaps()` specifically when navigating into the sandbox
tab. Removing any one of the three brings the bug back.

**Known gotcha — undo double-entries.** `sandboxResolveOverlaps()` calls
`sandboxSaveState()` itself when it nudges overlapping nodes apart. If
that runs nested inside another commit function's own `sandboxRenderBoard()`
call, you get two undo entries for one user action. `sandboxRenderBoard()`
guards against this by toggling `sandboxIsRestoring=true` around its own
internal `sandboxResolveOverlaps()` call — reusing the same flag that
suppresses saves during undo/redo restoration. If you add a new code path
that calls `sandboxResolveOverlaps()` directly, check whether it's already
inside a render/restore cycle before assuming it's safe to let it save.

## Validation workflow — run this before every publish

There's no test framework; validation is done with throwaway Node scripts
run against the live file, written to the OS scratch/temp directory (not
this project directory) and deleted after. Three layers, in order:

1. **Syntax check** — extract every `<script>` block and run
   `node --check` on the concatenated result. Catches typos/unbalanced
   braces instantly.
   ```js
   const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
   fs.writeFileSync('_extracted.js', scripts.join('\n;\n'));
   // then: node --check _extracted.js
   ```
2. **Structural checks** — extract a specific `const NAME = [...]` block by
   balanced-bracket scanning (not regex — the content contains nested
   braces), `eval`/`require` it standalone, and assert: EN/UK count parity,
   every question has exactly 4 opts with exactly 1 `correct:true`, every
   `QUIZ_TOPIC_SECTION`/`FALLING_ITEM_SECTION` entry resolves to a real
   section id, no duplicate ids, domain distribution sums correctly.
   (`ARCHITECTURES_EN/UK` reference function identifiers like `diagramFn`
   that don't exist in isolation — check its count via a regex over id
   occurrences instead of a full `eval`.)
3. **Functional smoke test** — `jsdom` with `runScripts: 'dangerously'`
   loading the actual file (not a rewritten test harness), driving real
   interactions (`selectTab`, clicking `.quiz-opt`, `sortResolve(...)`,
   `setLanguage('uk')`) and asserting on the resulting DOM/state. Known
   jsdom quirks to route around:
   - `getBoundingClientRect()` always returns an all-zero rect — stub it
     per-element with real numbers before testing any hit-testing/drag
     logic (Sorting Rush basket collision, etc.). Give every basket a
     **distinct, non-overlapping** rect if testing which one gets hit —
     a shared default rect for "everything except the one I stubbed" will
     make hit-tests match the wrong element and look like a game bug that
     is actually a test-setup bug.
   - `CSS.escape` is undefined — causes an async `ReferenceError` inside a
     `requestAnimationFrame` callback in `jumpToCard`. It's jsdom-only;
     filter it out of error counts with a regex
     (`/CSS is not defined/`), same for `window.scrollTo` being
     unimplemented.
   - `requestAnimationFrame` doesn't exist without `pretendToBeVisual: true`.
   - localStorage in jsdom is per-`JSDOM`-instance. To test something that's
     supposed to persist **across page loads** (e.g. Sorting Rush's
     "Review missed" list, item ratings), construct a second `JSDOM`
     instance sharing an in-memory store object, and patch `localStorage`
     via the `beforeParse(window)` constructor option — patching it
     *after* construction is too late, because the page's own inline
     `<script>` tags (which read `localStorage` at load time) have already
     run by then.
   - When simulating a real user drag for the falling-item auto-drop path,
     remember `onMove` hit-tests using the *dragged card's own*
     `getBoundingClientRect()` center point, not the raw mouse coordinates
     — stub the card's rect (after `mousedown`), not just the mouse event
     position.

Only publish (`Artifact` tool, same URL, `favicon:"🎓"`) after all three
layers pass. Clean up every scratch `.js`/`.tmp*` file and any `node_modules`
installed just for `jsdom` afterward — they don't belong in this repo.

## Adding new content — checklist

- **New quiz question**: add to both `QUIZ_EN` and `QUIZ_UK` at the same
  index, pick a `domain` (1–4) and a short `topic` string, add the topic
  (and its UK translation if different) to `QUIZ_TOPIC_SECTION`. No `k`
  field needed on options.
- **New falling item** (Sorting Rush): add to the right `_EN`/`_UK` array
  pair with a fresh sequential id in that category's namespace
  (`c101`, `e101`, `h101`, `s101`, `p101`, ...), add its section mapping to
  `FALLING_ITEM_SECTION`.
- **New flashcard**: add to the same index in both `SECTIONS_EN` and
  `SECTIONS_UK`'s `cards` array for that section (index alignment matters —
  see the glossary-matching note above). If it's glossary-worthy, add an
  `ABBR_LIST` entry with `cardTitle` set to the **English** card title.
- **New section**: needs an entry in `SECTIONS_EN` and `SECTIONS_UK` with a
  `group` field (one of the 8 existing category names, in the matching
  language), plus wiring into `ICON_META`/`badgeHTML` for its tab icon.
- **New Sandbox service**: add to `RADIAL_CATEGORIES`, `GLYPH_LIB`,
  `SERVICE_ICON_MAP`, `RADIAL_DOC_URLS` (curl-verified), and
  `SERVICE_CONNECTIONS` (real pairings, **source's side only** — see the
  directional-data note above) — all five, every time, or the service is
  placeable but broken.
- **New reference-architecture pattern**: add to `SANDBOX_ARCH_PATTERNS`
  (bilingual title/blurb, services that all resolve against
  `RADIAL_CATEGORIES`) and, if it's a genuinely canonical pattern, a
  curl-verified real URL to `SANDBOX_ARCH_REAL_URLS` keyed by the same id.

## What NOT to do

- Don't add a "correct-answer position" pattern by hand — the shuffle
  handles it.
- Don't skip the out-of-scope check because a service "sounds like it fits"
  — several rounds of content this reference shipped with (CodeDeploy,
  Personalize, CDK) turned out to be exam-out-of-scope on audit and had to
  be reworked around in-scope equivalents (Lambda alias weighted routing
  instead of CodeDeploy traffic-shifting configs, for example).
- Don't rebalance the quiz's domain distribution as a side effect of an
  unrelated content addition — if a batch of new questions skews it (e.g.
  an all-Security batch), say so explicitly rather than silently drifting
  from the real exam's 30/26/24/20 split.
- Don't hardcode an AWS URL (a doc link, a reference-architecture link)
  without curl-verifying it returns HTTP 200 first. A guessed or
  half-remembered URL that 404s is worse than no link, because it's
  presented as an authoritative, verified reference.
- Don't add a Sandbox service to only some of the five required places
  (`RADIAL_CATEGORIES`/`GLYPH_LIB`/`SERVICE_ICON_MAP`/`RADIAL_DOC_URLS`/
  `SERVICE_CONNECTIONS`) — a service missing from `SERVICE_CONNECTIONS`
  looks like a linking bug when it's actually just an incomplete add.
- Don't let a new Sandbox feature call `sandboxResolveOverlaps()` (or
  anything else that calls `sandboxSaveState()`) without checking whether
  it's already running inside a render/restore cycle — see the undo
  double-entry gotcha above.
- Don't add a `SERVICE_CONNECTIONS` edge to *both* sides of a pair "to be
  safe" or "for symmetry." Exactly one side should list the other — the
  side that's the real source/initiator. Adding it to both silently
  reintroduces the click-order-decides-the-arrow bug this data model was
  specifically redesigned to eliminate.
