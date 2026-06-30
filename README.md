# ARDC 2024 — Final Report & Artifacts

Public, citable bundle of the **non-code, AlterMundi-authored** deliverables produced under the
**ARDC 2024 grant**, covering the four fronts of the proposal: MW-CA-RF, Event Machine, Data
Collection, and 44Mesh.

> Third-party code (the Shared State Rust rewrite, the vwifi/HIL testbeds) is **linked, not bundled** —
> it lives in its authors' repositories under its own licence. See Annex D in `report/ARDC-2024-final-report.md`.

## Contents

| Path | What | Licence |
|---|---|---|
| `report/ARDC-2024-final-report.md` | The full final report — sections 0–6 and annexes A–F (incl. PR #1200 as Annex A and the §1.5 reference specification as Annex B) | CC-BY-SA-4.0 |
| `report/figures/out/` | Report figures (8 PNGs); per-file attribution in `.reuse/dep5` (third-party items under `LicenseRef-third-party-figure`) | CC-BY-SA-4.0 + third-party |
| `research/` | Four AlterMundi research lines: medium-access coordination on ESP32 (`esp32-medium-coordination`, the §1.5 reference algorithm, 25 docs), shared-state merge strategy (§1.4), distributed sensing architecture (§3), and LibreMesh mesh optimization (§3.4) | CC-BY-SA-4.0 |
| `data/montenet-2026-02-05/` | Five sanitized `lime-report` field captures (5× LibreRouter v1) | CC-BY-4.0 |

Licensing follows [REUSE](https://reuse.software/) (`.reuse/dep5`), matching the convention adopted in
`AlterMundi/44mesh`.

## Data sanitization

The MonteNet field captures in `data/montenet-2026-02-05/` are **published as sanitized versions**, safe for
public release. Device **MAC addresses** — low-sensitivity identifiers (they are broadcast over the air), but
the one item worth removing — are replaced with deterministic `02:*` pseudonyms in the locally-administered
space, so neighbour/topology relationships are preserved while the real hardware identifier is removed. SSIDs
are public community-network names, and no keys or secrets are present (the captures contained none). The
structured shared-state JSON is preserved intact. The production sanitizer and its salt are **not part of this
published artifact** — publication is the git-tracked tree only, and the tooling lives outside it. See
`data/montenet-2026-02-05/README.md` for the per-file detail.

## How to cite

See `CITATION.cff` for citation metadata. Cite the repository at the submitted commit (or its git tag) for
the exact version.
