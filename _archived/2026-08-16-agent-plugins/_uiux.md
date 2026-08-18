# UI/UX Change Map: Agent Plugins Ingestion

Every UI surface this feature touches: where it lives today, what changes, which states must be designed, and the reference artboard each surface needs. Artboards land under `docs/design/opendesign/agent-plugins/`.

Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states come from ACs/ECs), `_dx.md` (the non-UI half of the surface).

## Design constraints (apply to every artboard)

- The format indicator is **information, not decoration**: a `Pill mono size="xs"` with copy `agent plugin`, default (neutral) tone — same family as the existing source badge, never merged with the trust badges (their doc rule "distinct labels, never merged" extends to this one; `web/src/systems/extensions/components/extension-trust-badges.tsx:40-43`).
- Diagnostics render **only recorded daemon diagnostics** — severity `warn` maps to the warning token (`#D6A647`), never used decoratively. No invented health states: a package with zero recorded skips shows no diagnostics block at all.
- Truthful empty state: a fully-degraded install renders "0 resources" plus its reasons — never a blank panel, never a spinner (US-015.EC-1).
- No new subtitles/helper prose under headings; badge and rows carry their own meaning. Tooltip copy only where the existing badge set already uses tooltips.
- All copy through `COPY.md` register; "agent plugin" is a format name, never a product noun — lifecycle vocabulary stays "extension" everywhere.

## Surface map

| #   | Surface                          | Kind   | Core change                                        | Stories                    |
| --- | -------------------------------- | ------ | -------------------------------------------------- | -------------------------- |
| S1  | Marketplace catalog card         | modify | Format badge in the meta row                       | US-016.AC-1, US-016.EC-2   |
| S2  | Marketplace extension detail     | modify | Format badge beside existing badges                | US-016.AC-3                |
| S3  | Extension badges (settings/detail) | modify | New `agent plugin` badge variant                  | US-015.AC-1, US-015.EC-2   |
| S4  | Extension kit inventory panel    | modify | Skipped-component rows + degraded empty state      | US-015.AC-2, US-015.EC-1, US-005.AC-5 |
| S5  | Install + trust dialogs          | none   | Explicitly unchanged; failures use existing error path | US-016.AC-2, US-016.EC-1 |

### S1. Marketplace catalog card

- **Today**: `web/src/systems/marketplace/components/marketplace-card.tsx:22-105` — `CatalogCard` from `@compozy/ui`; meta row shows `author · version · downloads · tier · source`.
- **Change**: when the feed entry carries `format: "agent-plugin"`, append the format pill to the meta row. Absent field → nothing (native entries unchanged).
- **States to design**: badge present; badge absent (control); long-name truncation with badge present.
- **Artboard**: none — composes the existing `Pill` verbatim inside the existing meta row.

### S2. Marketplace extension detail

- **Today**: `web/src/systems/marketplace/components/marketplace-detail-extension.tsx` (+ `-rail`, `-sections`, `-installed`) — detail header with trust facts; installed card `marketplace-installed-card.tsx`.
- **Change**: format pill in the detail header badge cluster and on the installed card, sourced from the entry's `format` (pre-install) or the payload's `format` (installed).
- **States to design**: pre-install (feed marker); installed (payload truth); marker/payload disagreement — payload wins (detection is authoritative, US-016.EC-2).
- **Artboard**: none — badge placement follows the existing badge cluster.

### S3. Extension badges (settings list + extension detail)

- **Today**: `web/src/systems/extensions/components/extension-trust-badges.tsx` — five `Pill mono xs` variants (`dev`, `overrides published`, source, registry tier, digest/checksum), each with a hard "never merged" rule.
- **Change**: add `ExtensionFormatBadge` — copy `agent plugin`, default tone, rendered from `payload.format === "agent-plugin"`; native extensions render nothing (US-015.EC-2 — the badge marks the exception, not the rule).
- **States to design**: badge in the badge cluster alongside `dev` (dev-linked portable package) and trust badges — worst-case cluster density.
- **Artboard**: none — sixth variant of an existing pattern.

### S4. Extension kit inventory panel

- **Today**: `web/src/systems/extensions/components/extension-kit-inventory-panel.tsx:62-64,107` — groups kit items by `kind`, shows shipped-vs-live.
- **Change**: below the kit groups, a **Skipped** section rendering each recorded `warn` diagnostic as a row: severity pill + scope (`mcp: legacy-events`) + reason text. Sourced from the **inventory payload's** `diagnostics` field (the widened `GET /api/extensions/:name/inventory` contract — ingest skips code `extension_agent_plugin_component_skipped`, live runtime entries code `extension_mcp_server_unhealthy`, rendered in that order). Zero diagnostics → section absent. Fully-degraded install → explicit "0 resources ingested" line above the Skipped rows.
- **States to design**: healthy (no section); partial (items + skipped rows); fully degraded (zero items + all reasons); scale (30+ items with a handful of skips — section stays below the fold of the groups, no truncation of reasons).
- **Artboard**: `agent-plugins-kit-inventory.html`.

### S5. Install + trust dialogs

- **Today**: `web/src/systems/marketplace/components/extension-install-dialog.tsx:46-227` (fields: source/ref/version/asset/allow-unverified) and `extension-trust-dialog.tsx:35-132` (renders daemon `trust.warnings[]` as severity rows with optional suggested command).
- **Change**: **none**. No new fields, no format selector — detection is automatic. A drifted-upstream install failure (US-016.EC-1) surfaces through the dialog's existing error rendering, showing the deterministic layout message from `_dx.md` — verify legibility of that copy length, change nothing structurally.
- **States to design**: the failure state with the layout-error message (copy-length check only).
- **Artboard**: none.

## Component plan (design → production mapping)

### Rules

- Compose from existing primitives only; the badge follows the exact `Pill mono size="xs"` recipe of its five siblings.
- Diagnostics rows reuse the severity-pill + text row pattern the trust dialog already renders for `warnings[]` — one visual language for "daemon recorded a caveat".
- Payload truth beats feed marker wherever both exist.

### New `@compozy/ui` primitives

None — `Pill`, list rows, and section headings cover every state.

### New domain components

- `ExtensionFormatBadge` — `web/src/systems/extensions/components/extension-trust-badges.tsx` (sixth variant in the existing file); composed from `Pill`; used by S1, S2, S3.
- `ExtensionSkippedComponents` — `web/src/systems/extensions/components/` beside the kit inventory panel; composed from `Pill` (severity) + existing row primitives; used by S4.

### Signal & state mapping

| Design state            | Primitive + token                                      |
| ----------------------- | ------------------------------------------------------ |
| Format identity         | `Pill` default tone, mono, xs — neutral information    |
| Skipped component (warn)| `Pill` warning tone (`#D6A647`) + plain row text       |
| Fully degraded install  | Explicit zero-count line + warning rows — no danger tone (nothing failed at runtime; ingestion recorded skips) |
| Healthy install         | No format-specific signal beyond the identity badge    |
