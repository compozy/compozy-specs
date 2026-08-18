# UI/UX Review: Worktree Design Reference Set

**Audience:** the LLM that authored `docs/design/opendesign/worktree/`. **Protocol: iterate the existing files in place — never regenerate the set.** Keep provenance comments, the shared story, and everything marked *keep* below.

**Method:** dual-agent review — Assessment A (design-director review of all 17 rendered PNGs at 1440×5200 + full HTML/CSS read) and Assessment B (deterministic sweep: detector, provenance, token, link, copy-hygiene, shared-story, and state-coverage checks), synthesized by a third context. Detector note: `detect.mjs` ran in DEGRADED regex mode (HTML parser deps unavailable), so contrast and selector-matched rules were never evaluated — its findings are a floor.

**Spec errata acknowledged on our side (no design action):** `_uiux.md` said "light theme" — that line was wrong and is now fixed to dark; the set's dark rendering is correct. The `run/` namespace for per-run branches is a valid reading of BR-16 (the spec's `worktree/` example was illustrative).

## Verdict

**Good, trending strong.** State coverage against `_uiux.md` S1–S16 is **complete** (B's matrix: every required state present with a line anchor — zero absences). Truthful-UI discipline is real: absent-not-disabled applied with stated reasons, `data-unknown` em-dashes, `+0 −0` forbidden, zero-credential as a first-class variant. Token discipline is absolute (zero raw hex in all 16 artboards + `worktree.css`). Copy hygiene is clean (zero hype words, zero banned nouns, zero forced branch prefixes). The set is authored for CompozyOS, not category-interchangeable.

What blocks *strong*: **4 P0 contract defects**, a cluster of **shared-story consistency failures** (paths, counts, vocabulary), and **spec scaffolding leaking into rendered product copy**. All are iterations, none require redesign.

| Artboard | Verdict | | Artboard | Verdict |
|---|---|---|---|---|
| index | needs-fixes | | task-setup | needs-fixes |
| row-states | needs-fixes | | fanout | good |
| nav-switcher | good | | loop-config | needs-fixes |
| menubar-menu | good | | detail-exit | good |
| overview | good | | commit-sheet | strong |
| create-dialog | needs-fixes | | pr-sheet | strong |
| session-create | strong | | exit-progress | good |
| composer-environment | needs-fixes | | merged-cleanup | good |
| remove-dialogs | needs-fixes (1 P0, else best-in-set) | | | |

Heuristic calibration (n/a: flexibility, help/docs — static references): Visibility 4 · Real-world match 3 · User control 3 · **Consistency 2** · Error prevention 4 · Recognition 3 · Aesthetic 4 · Error recovery 4 → **27/32 (84%, Good)**. Consistency is the axis to fix.

---

## P0 — contract defects (fix first)

### P0-1 · Worktree paths: three conventions, all three wrong vs ADR-005

- `~/dev/compozy-worktrees/<name>` — `create-dialog:91,154,225,329,370`
- `~/dev/compozy-<name>` (sibling) — `nav-switcher:87`, `detail-exit:89`, `composer-environment:373`
- `~/dev/compozy/.worktrees/<name>` (in-repo) — `remove-dialogs:123,233,278,378`

`payments-retry` has two different absolute paths inside one set (`detail-exit:89` vs `remove-dialogs:233`). Worse: `adrs/adr-005.md` **decided central home under the Compozy home** — the in-repo and sibling forms are that ADR's explicitly rejected Alternatives 1 and 2. In a deletion dialog the path is what the user verifies before confirming.

**Fix:** one canonical convention everywhere: `~/.compozy/worktrees/compozy/<name>` (Compozy home → per-workspace group → worktree name). Add it to DESIGN-NOTES §Shared data story as a locked fact. Exception stays: the *discovered external* worktree keeps its foreign path (`../compozy-spike` in `overview:165,337` is correct — adopted externals live where they live).

### P0-2 · Remove-confirm titles name the branch, then deny deleting the branch

`remove-dialogs:104,152,260,302`: `Remove "run/fix-flaky-e2e" from disk? This cannot be undone.` — the quoted string is the **branch**, and the next line says "The branch run/fix-flaky-e2e is not deleted." Self-contradiction in the most destructive dialog in the product, and it violates the locked rule "Name is the record label (never the path basename)" — which extends to titles.

**Fix:** record label in the title (`Remove "fix-flaky-e2e" from disk? This cannot be undone.`), branch stays in the target card and the "not deleted" line. Four dialogs. (Everything else in this artboard — risk inventory, force re-quantification, remote-downgrade, idempotent no-op — is best-in-set; keep verbatim.)

### P0-3 · Create dialog: two contradictory behaviors for a held branch, same file, same US id

`create-dialog:165` (§02 note): selecting a held branch "*jumps to that worktree*, it never duplicates the checkout". `create-dialog:236-268` (§03): selecting the held `pedro/auth-refresh` produces a field refusal + "Select that worktree instead" button. Both cite US-002 EC-2; an implementer cannot build both.

**Fix:** keep §03 (refusal + explicit jump action — truthful, gives an escape); rewrite the §02 note to describe that behavior; badge held rows in the picker (holder name is already there at `:181`) so the refusal is predictable before the click.

### P0-4 · Discovered rows are non-selectable — contradicts adopt-on-select (ADR-002 / US-009)

`nav-switcher:221` gating: `state === "discovered" → row is informational`; `overview` removes the Adopt action ("lifecycle actions live on S6/S15") — but **no artboard designs adoption**, and `row-states:195-199` renders an orphan `Adopt` button no surface owns. The PRD decision is the opposite: *selecting a discovered worktree adopts it* (after identity validation), and adoption is the selection gesture — US-009 AC-1, ADR-002.

**Fix:** discovered rows in S1/S2/S3 become selectable with an adopt affordance (`Adopt` trailing action or select-triggers-adopt-confirm); add one adoption moment (a small confirm naming the validation: "metadata resolves to this repository → adopted; bootstrap not re-run") — natural home: selecting the row in the switcher, with the same dialog reachable from `overview`. Add the refusal state (metadata resolves into the main checkout → refusal naming the reason, directory untouched — US-009 AC-3). Keep pending/missing non-selectable exactly as drawn.

---

## P1 — required before the set is a safe implementation contract

1. **Strip spec scaffolding from rendered copy — 8 instances, all visible in PNGs.** `task-setup:141` `(US-014 AC-2)` · `task-setup:230` `(US-015 AC-3)` · `loop-config:263` `(ADR-021)` · `composer-environment:410` "…come from the create form — worktree-create-dialog.html (S4)" · `composer-environment:302` badge `new · worktree-support` · `task-setup:111` legend tag `proposal · spec worktree-support` inside the sheet body · `loop-config:132` tag `new · spec worktree-support` inside the dialog body · `merged-cleanup:186` design rationale as UI text ("Local check still holds — cleanup guidance degrades, it doesn't vanish."). Spec references belong in `.spec__note` columns and HTML comments only. Sweep `.fhint`, `.ferr`, `.notice`, `.foot-note`, `.sub`, and `.tag` inside every mock's fidelity boundary.
2. **Unify the environment-mode vocabulary.** Today: `Workspace root` (S7/S8) vs `None` (S10/S12) vs `Root` (S13); `Inherit workspace` (S10) vs `Unset` (S12); `Worktree` as a mode label inside a field labeled "Worktree" (S10 `:130`, S12). **Lock: `Workspace root` · `Inherit` · `Named worktree` · `Per-run` · `Directory`** — apply across `task-setup`, `loop-config`, `session-create`, `composer-environment`, and record in DESIGN-NOTES.
3. **Reconcile the worktree count and state the rule once.** `nav-switcher:333` "All 8 worktrees" vs `overview:86,105,248,263` "7 worktrees" (adopted-only rule, stated only in `overview:104`'s comment) vs `index:53` "eight worktrees". Adopt the overview's rule set-wide: counts = adopted only → switcher overflow "All 7 worktrees", index lede "eight worktree records (7 adopted, 1 discovered)". Add the rule to DESIGN-NOTES.
4. **Add `.wt-dot[data-state]` to `worktree.css`; fix the anchor sheet's pending dot.** The nested-row state dots are inline-styled 7× across three files in two geometries, and `row-states:293` renders pending with the *discovered* shape (hollow ring) while its consumers (`nav-switcher:228,318`, `menubar-menu:168`) render the correct filled diamond. The vocabulary sheet must be the single source of shape truth.
5. **Reconcile the S6 hero with the pause rule.** `detail-exit:88`: pulsing accent *running* dot + enabled `Commit` primary — the exact state the PRD forbids ("exit actions pause while the session executes"); the reconciliation ("turn finished, awaiting input") lives only in a comment, and the S5 vocabulary has no *awaiting-input* state. **Fix:** add `awaiting-input` to the S5 agent-activity vocabulary (distinct from running/idle) and use it in the hero — or pause the hero's control.
6. **Remove the invented config key.** `create-dialog:132` grounds the `pedro/` prefill in `git.branch_prefix` — that key exists nowhere in the repo or the spec (repo-wide grep: zero matches), and BR-15 forbids forced prefixes. **Fix:** prefill = name-derived branch only (`docs-refresh`); the story's `pedro/…` branches stay as values the user *typed*, and the gating comment cites no config key (or cites the branch-template config the TechSpec will define, marked as such).
7. **Close the three dead-end flows.** (a) S7: "New worktree…" returns into a ready-only list — the just-created `pending` worktree is invisible; render it as selected-but-materializing (reuse S8's phase treatment) with submit held, or state that the session starts when ready. (b) S13: the `New worktree` node mode has a pill and no body — draw its selected sub-state (name/branch fields or "generated at run start"). (c) Adoption — covered by P0-4.
8. **Add the missing safety-critical states.** (a) `merged-cleanup`: the **not-safe-to-clean** row (unique unpushed commits → blocker named, `Clean up` suppressed) — currently both tiers only show success, implying the suggestion always appears; also give the remote-exists *downgrade* row the info tone (it renders identical to the strong local proof today, `:97` vs `:112`). (b) `remove-dialogs`: the **bound-session-running** variant (current variant is `idle` only, `:146,173`) — refuse or require the turn to end, reason named.
9. **Fix the copy defects.** (a) `loop-config:139` broken sentence ("nodes that don't run at the workspace root.") → "nodes that don't declare one run at the workspace root." (b) `session-create`: header and footer say "Choose its/a runtime when you send the first message" twice in one dialog ×6 demos — cut the header instance (footer is the production string). (c) `exit-progress:97-99,155,205-208`: completed steps labeled "Committing..."/"Pushing to…" — use tense per state: `Commit` (pending) / `Committing…` (active) / `Committed` (done) / `Commit skipped` (skip); also `…` not `...`. (d) `commit-sheet:308,317`: doubled error copy for one hook failure — keep the `.ferr`, repoint the foot-note at the branch. (e) `task-setup:141`: the `none` explanation sits inside the `ref` mode's hint — one hint per mode.
10. **Sheet/ladder handoff coherence.** `commit-sheet:75,120-123`: invoking `Commit & push` from the S6 ladder opens a sheet whose primary is plain `Commit` — the sheet's primary must mirror the invoking action. Also draw the split-submit's open menu once (its contents are undefined set-wide).
11. **PR header's borrowed number.** `pr-sheet:107` shows `+142 −18` — payments-retry's *working-tree dirty* figure — as branch-vs-base commit diff, at a ladder position only reachable when the tree is clean. Use distinct commit-diff numbers or drop the ± from the header.
12. **Fan-out failed run loses its id.** `fanout:207`: "run 3 — worktree creation failed…" while success rows show `r-9b02c1 → …`. US-016 EC-2 is per-run attribution — give the failed row its run id, error on its own danger line.
13. **index.html joins the system.** Link `../design-system/ds-core.css` (delete the 20-value `:root` copy — 11/11 identical today, guaranteed to drift), add the provenance comment (the only file without one), fix its own footer claim ("tokens follow ds-core…" while loading neither), fix the count (P1-3), and promote `merged-cleanup` + `exit-progress` from `lab` to `final` (or add a `contract` tag) — both own required states.
14. **Lock the S6/S16 mount decision.** The spec delegated OQ7 to design; the artboard ships "the control **may** mount in the header" (`detail-exit:583`). Lock it in DESIGN-NOTES — recommendation: binding chip always in the session header; split button only in the worktree detail context (the header already carries dot + title + agent + chip + menu; adding the split button tips into cockpit density).
15. **Gating-comment floor.** Coverage collapses on `session-create` (3 citations/50 buttons), `task-setup` (3/42), `commit-sheet` (4/26), `row-states` (1/11); `exit-progress` cites under `stream:`/`result:`/`events:` instead of `gating:`. Set a floor — every *distinct control class* per artboard cites its gate once — and either bless the alternate prefixes in DESIGN-NOTES or normalize to `gating:`.

## P2 — accessibility and consistency polish

- **Targets:** `.cs-exp` / `.ov-exp` disclosure+create affordances are 18×18, hover-revealed, `role="button"` spans nested inside interactive rows — below the 24×24 floor, invisible to keyboard/touch (`nav-switcher:188`, `overview`). Make rows `<button>`s with sibling chevron buttons ≥24px; give zero-worktree workspaces a labeled "New worktree" row instead of the hover `+`.
- **ARIA:** `aria-selected` on role-less `<div>`s (nav-switcher `:108,169,299`, `.wt-row` throughout) → `role="option"`/`listbox` or `aria-current`; non-menuitem content inside `role="menu"` (`menubar-menu:294` notice + nest rows) → `role="group"` + `menuitemradio`, notice outside; `role="alertdialog"` without `aria-labelledby` (all four removal dialogs) → id the `<h3>`.
- **Icon-only meaning:** agent dot (`row-states:154`), setup-failed flag in the switcher (`nav-switcher:323` icon-only vs `overview:150` icon+text — pick one rendering), hover-only branch-copy buttons — add `aria-label`s; decide the density rule.
- **Read-failure strip drops unknowns** (`detail-exit:311-322` renders only Branch+Status) — render Dirty/Ahead-behind as `data-unknown` em-dash cells; the treatment already exists at `:119`.
- **Non-selectable rows look selectable** (S1/S2 discovered/pending/missing have identical hover/geometry; blocked reasons live only in comments) — resting cursor, no hover fill, right-lane functional label (`materializing`, `directory missing`); note discovered becomes selectable per P0-4.
- **`§02 detail-exit` menu is a literal inventory** (two `Push` rows with contradictory reasons in one menu, `:245-252`) — split into two demos or label "reason literals, not one state".
- **Consistency nits:** `fix-flaky-e2e` loses its `per-run` badge in `nav-switcher`/`menubar` (present in `row-states`/`overview`); S1 vs S2 nests differ in set and order while promising "must not diverge" — declare the sort rule and render identically; menubar chrome appears/disappears between demos (`menubar-menu:217,284`); diverged demo lacks its status strip (`detail-exit` §05); session status-dot semantics drift (accent=running `:555`, success=finished `:610`) — declare or reuse `d--run/ok/idle`; `.gmenu__reason` as block footer with inline styles (`detail-exit:425`) → add `.gmenu__note`; force button before Cancel in DOM order + inline danger style (`remove-dialogs:238`) → `.btn--ghost-danger`, after Cancel in tab order; PR sheet has no footer/Cancel and introduces a third primary-button grammar (`.pract--primary`) — name the "actions-as-rows sheet" pattern in `worktree.css` or restore the footer; create-dialog submitting state is a trap (close removed, Cancel disabled, `:350-375`) while the pending row already exists in lists — keep Cancel/X; composer staged prompt `readonly` while materializing (`:209`) — keep editable, block only send; `data-locked` env chip is clickable-but-described-inert (`:325-338`) — pick one; empty `.foot-note`s (`create-dialog:274`, `composer-environment:379`); scope list truncation + scroll compete (`commit-sheet:96-101`); PR sheet lacks a creating/pending state; `--faint` (≈3.3:1) carries information in `.tstep__why`/`.wt-stale` — verify against the AA floor for information-bearing micro text.

## P3 / informational

- Detector (DEGRADED): 17× `overused-font` = Geist, the canonical house face — no action; 14× `flat-type-hierarchy` — expected for spec-sheet micro-ramps, no action on labs, worth one pass on `fanout`/`task-setup` finals (1.8:1); 14× `em-dash-overuse` (advisory) — fine in annotations, sweep *rendered UI copy* only.
- `…` vs `...` in `exit-progress` labels (covered in P1-9c); toast dismiss mid-run: note where the operation remains observable; `detail-exit:575,593` self-link chip — point at a sibling or drop `href`; `merged-cleanup:79,160` `href="#"` on View PR — add `<!-- external: github PR url -->`.
- `row-states`: §03 shows the ready chip while §04 omits it on the same row — state the rule ("chip only when state ≠ ready") and make §03 chip-less; `agent idle` inventoried but unused in any story row — use it once or note that idle renders nothing; path is locked as "demoted not removed" but absent from the anatomy — add the micro-mono path lane; `behind`'s warning tone is outside the declared semantic map — register it or demote.
- Create-dialog educational subtitle ("A separate checkout of compozy…", ×5) — the set's one deliberate helper-text exception; register it in DESIGN-NOTES as an authorized delta or fold into advanced.
- Generated-name suggestion carried by `placeholder` only (`create-dialog:63`) — mirror the commit/PR sheets' explicit "leave empty to…" hint pattern.
- Fan-out dialog head lacks the set's `head-icon`+`eyebrow` anatomy; two accents on one screen (checkbox + primary) — note as authorized delta or align.

## DESIGN-NOTES.md additions required (lock these once, cite everywhere)

1. Canonical path convention (P0-1) and the discovered-external exception.
2. Count rule: counts = adopted only (P1-3).
3. Environment-mode vocabulary (P1-2).
4. Agent-activity vocabulary gains `awaiting-input` (P1-5); session status-dot semantics.
5. S6/S16 mount decision (P1-14).
6. Nest sort order shared by S1/S2 (P2).
7. Ready-chip density rule; `behind` tone registration; the create-dialog educational subtitle as an authorized delta; blessed gating-comment prefixes.

## Keep exactly as-is (do not "improve" while iterating)

The removal risk inventory + force re-quantification ("They exist nowhere else."), the zero-credential variants (PR cell/rows absent, browser row as the sheet), `menubar-menu` §03's missing-fallback line, the composer's three-fact fork statement and failure-with-three-exits, the commit sheet's honesty rule comment (lift it into DESIGN-NOTES), pr-sheet's idempotent flip and template-ambiguity quiet degrade, exit-progress's phases-up-front + one-CTA discipline, `detail-exit` §04's whole-control block on read failure, loop-config's `env` micro row on node cards, and the shared-story numeric discipline (branch↔name pairing, ±counts, timestamps — 100% consistent per the mechanical sweep).

---

*Evidence: rendered captures in `/tmp/eng-ui-screenshot.Td3Pq3/shots/` (17 PNGs, 1440×5200); state-coverage matrix and sweep details available on request. Review date: 2026-08-11.*
