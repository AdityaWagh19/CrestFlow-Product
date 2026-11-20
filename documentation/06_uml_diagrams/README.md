# CrestFlow Mermaid Diagram Pack

**Status:** Current diagram source  
**Version:** 1.2  
**Date:** 20 November 2026  
**Rendered exports:** `png/` (19 PNGs, 2x scale, white background, strictly monochrome)  
**Authority:** `documentation/01_canonical/CANONICAL_DECISIONS.md` v1.2 and the canonical component documents

---

## 1. Purpose

This folder is the **separate, code-first diagram pack** for the CrestFlow backend report. It abstracts the architecture into 19 Mermaid diagrams that are short enough to read in a report and precise enough to trace back to canonical decisions.

The pack is self-contained: the `.mmd` files are the source of truth, and `png/` holds the rendered exports for the backend report. Do not reintroduce pre-remediation details such as a three-check policy, `KYC_PENDING`, `SimulationResult`, or the old sequence ordering.

**Rule:** if a diagram and a canonical document disagree, the canonical document wins. Change the decision record first, then the diagram.

---

## 2. Contents

| # | File | Diagram type | Scope | Primary decisions |
|---:|---|---|---|---|
| 01 | `01_component.mmd` | UML component (flowchart approximation) | 16 components in 6 groups | D1, D7, T11, T12 |
| 02 | `02_class_domain.mmd` | UML class | 13 domain classes | D1, D4, D6, D7, T8, T10, T13 |
| 03 | `03_sequence_onboarding_kyc.mmd` | UML sequence | Google login, Path A/B wallet, KYC | D1, D9, T8, T9 |
| 04 | `04_sequence_policy.mmd` | UML sequence | Nine-check policy evaluation | D6, T1–T3, T5, T13 |
| 05 | `05_sequence_execution.mmd` | UML sequence | Canonical execution pipeline | D7, T11, T15 |
| 06 | `06_sequence_cctp.mmd` | UML sequence | CCTP V2 USDC bridge | D3, D5 |
| 07 | `07_sequence_x402.mmd` | UML sequence | x402 V2 paid Copilot query | D4, T17 |
| 08 | `08_state_saga.mmd` | UML state machine | 10-state execution saga, including CCTP sub-path | D1, D3, D5, D7, T10 |
| 09 | `09_state_kyc.mmd` | UML state machine | KYC lifecycle | T8, T9 |
| 10 | `10_state_delegation.mmd` | UML state machine | Autopilot delegation lifecycle | D1, D6 |
| 11 | `11_deployment.mmd` | UML deployment (flowchart approximation) | Trust zones and provider boundaries | D10, T15 |
| 12 | `12_system_architecture.mmd` | Layered architecture view | Five layers and topology | D7, T11, T14 |
| 13 | `13_dfd_level0.mmd` | DFD level 0 | One process and external entities | T14 |
| 14 | `14_dfd_level1.mmd` | DFD level 1 | Six processes, two data stores | D7, T14 |
| 15 | `15_use_case.mmd` | UML use case (flowchart approximation) | 10 use cases, 4 actors | D1, T8, T17 |
| 16 | `16_activity_execution.mmd` | UML activity | Three swimlanes and the CCTP decision path | D5, D7 |
| 17 | `17_er_core.mmd` | ER diagram | 13 logical entities | T8, T10, T11, T13 |
| 18 | `18_gantt.mmd` | Gantt | Indicative MVP schedule and gates | D9, T15 |
| 19 | `19_sdlc.mmd` | SDLC view | Iterative-incremental lifecycle | T14 |

> All diagram sources are `.mmd` files. Each source has a matching rendered PNG at `png/<same-basename>.png`.

### Pack size

| Element | Count |
|---|---:|
| Component elements | 16 |
| Domain classes | 13 |
| Saga states | 10 |
| ER entities | 13 |
| Use cases | 10 |

The saga keeps the canonical state set because those names are the execution contract. It is still abstract: it does not model every retry or every compensation action.

---

## 3. Rendering

### Single diagram

```powershell
npx --yes @mermaid-js/mermaid-cli `
  -i documentation/06_uml_diagrams/01_component.mmd `
  -o documentation/06_uml_diagrams/png/01_component.png `
  -b white -w 2000 -s 2 `
  -p documentation/06_uml_diagrams/puppeteer-mono.json
```

The checked-in PNGs use a white background, a 2,000 px page width, and a 2x scale factor (most outputs are about 4,000 px wide). The `puppeteer-mono.json` launch config disables LCD subpixel text, so every pixel is a neutral gray (`R = G = B`); without it, Chrome writes colored fringes on glyph edges.

### Whole pack

```powershell
New-Item -ItemType Directory -Force -Path documentation/06_uml_diagrams/png | Out-Null
Get-ChildItem documentation/06_uml_diagrams/*.mmd | ForEach-Object {
  npx --yes @mermaid-js/mermaid-cli `
    -i $_.FullName `
    -o "documentation/06_uml_diagrams/png/$($_.BaseName).png" `
    -b white -w 2000 -s 2 `
    -p documentation/06_uml_diagrams/puppeteer-mono.json
}
```

`17_er_core.mmd` sets `htmlLabels: false`. Mermaid mismeasures long ER entity headers when they are HTML labels, which clips the last characters; SVG text labels keep the full entity name visible.

### Editor preview

- VS Code: install `Markdown Preview Mermaid Support` and open a `.mmd` file in a Markdown preview, or paste the body into a Markdown fence.
- Mermaid Live Editor: paste a file body at <https://mermaid.live/>.
- GitHub renders fenced `mermaid` blocks directly in Markdown.

**Validation status:** all 19 `.mmd` files render with `@mermaid-js/mermaid-cli` 11.17.1, and all 19 PNGs in `png/` are present and non-empty. Re-render after any edit and keep the PNG exports in sync.

---

## 4. Notation conventions and Mermaid approximations

Mermaid is not a full UML tool. The pack uses the closest native diagram type and states the mapping explicitly.

| Diagram | Mermaid type | Convention used | Approximation |
|---|---|---|---|
| Component | `flowchart LR` | Subgraph = layer or grouping; box = component; cylinder = data store; labeled arrow = dependency or interface (required/provided or event flow) | Mermaid has no UML component or port notation. Interfaces and dependencies are edges, not lollipops. |
| Class | `classDiagram` | Box = class; `<<enumeration>>` = enum; `*--` = composition; `-->` = association; quoted labels = multiplicity | Full UML association-end notation is not available; cardinalities are labels. |
| Sequence | `sequenceDiagram` | Actor/lifeline = participant; solid arrow = call; dashed arrow = return; `alt`/`loop` = control flow; `note` = constraint | Asynchronous messages are shown as normal calls where the abstraction does not need a reply. |
| State | `stateDiagram-v2` | Rounded state = state; arrow = transition; label = event/guard; `note` = invariant | Entry/exit actions and internal transitions are omitted. |
| Deployment | `flowchart TB` | Subgraph = trust zone; box = execution environment or artifact; cylinder = data store; labeled arrow = protocol | Mermaid has no UML deployment node/artifact notation. Zones carry the trust boundary. |
| Use case | `flowchart LR` | Rectangle with `«actor»` = actor; stadium = use case; labeled arrow = participation | Mermaid has no native actor shape; the stereotype is written in the label. The system boundary is a dashed subgraph. |
| DFD | `flowchart` | Rectangle = external entity; circle = process; cylinder = data store; labeled arrow = data flow | DFD is not UML. Level 0 has no data stores; level 1 has two. |
| Activity | `flowchart TD` | Subgraph = swimlane; diamond = decision; stadium = start/end; arrow = control flow | Mermaid has no formal activity/action notation. |
| ER | `erDiagram` | Entity box = table; `PK`/`FK`/`UK` = keys; crow's foot = cardinality | Logical model only; physical indexes, partitions, and constraints live in `implementation_spec.md`. |
| Gantt | `gantt` | Section = workstream; bar = task; diamond = milestone | Dates are planning estimates, not commitments. |
| SDLC | `flowchart LR` | Box = phase; diamond = gate; arrow = progression or feedback | Not a UML diagram type; it communicates the chosen lifecycle. |

### Monochrome

Every diagram is strictly black and white. Fills are white, text and 1 px borders are black, and there are no gradients or gray semantic shades. `classDef` blocks are retained only to force white fills and black strokes; they do not encode meaning. Dashed strokes mark external, optional, or feedback relationships. Keep this treatment if you edit a diagram, and recheck that the exported PNG has no colored pixels.

---

## 5. Traceability matrix

| Diagram | Canonical source of truth | Coverage note |
|---|---|---|
| `01_component.mmd` | `02_architecture/system_architecture.md`; `implementation_spec.md` §1–2 | 16 components in 6 groups; interfaces shown as edges |
| `02_class_domain.mmd` | `01_canonical/canonical_schemas.md` §4–15 | 13 domain classes; no transient `SimulationResult` |
| `03_sequence_onboarding_kyc.mmd` | `02_architecture/user_onboarding.md`; `identity_kyc.md` | Canonical Google login, Path A/B, and KYC states |
| `04_sequence_policy.mmd` | `02_architecture/policy_engine.md` | All nine checks; no short-circuit |
| `05_sequence_execution.mmd` | `02_architecture/orchestrator_planner.md`; `implementation_spec.md` §7 | Canonical Engine 6 → Router → Planner → Simulation → Policy → CCE → PSM order |
| `06_sequence_cctp.mmd` | `02_architecture/cross_chain_ecosystem.md`; `protocol_layer.md` | CCTP V2 calldata; Standard and Fast paths |
| `07_sequence_x402.mmd` | `02_architecture/audit_layer_x402.md` §x402 | V2 headers, Base USDC, settle only after 2xx |
| `08_state_saga.mmd` | `02_architecture/implementation_spec.md` §7.1 | Canonical saga states plus the CCTP sub-path |
| `09_state_kyc.mmd` | `02_architecture/identity_kyc.md` §2–4 | Six canonical KYC statuses |
| `10_state_delegation.mmd` | `02_architecture/user_onboarding.md`; `policy_engine.md` check 3 | Path A autopilot only; Path B manual |
| `11_deployment.mmd` | `02_architecture/implementation_spec.md` §2, §8–10 | Trust zones and provider boundaries |
| `12_system_architecture.mmd` | `02_architecture/system_architecture.md` | Five layers and the canonical execution graph |
| `13_dfd_level0.mmd` | `02_architecture/system_architecture.md` | One process and external entities |
| `14_dfd_level1.mmd` | `02_architecture/system_architecture.md`; component docs | Six processes and two data stores |
| `15_use_case.mmd` | `02_architecture/user_onboarding.md`; `audit_layer_x402.md` | 10 use cases and 4 actors |
| `16_activity_execution.mmd` | `02_architecture/implementation_spec.md` §7 | Three swimlanes and the CCTP decision path |
| `17_er_core.mmd` | `01_canonical/canonical_schemas.md`; `implementation_spec.md` §3 | 13 logical entities; composite audit key |
| `18_gantt.mmd` | `01_canonical/CANONICAL_DECISIONS.md`; `implementation_spec.md` | Indicative schedule and release gates |
| `19_sdlc.mmd` | `01_canonical/CANONICAL_DECISIONS.md` T14 | Iterative-incremental lifecycle |

### Decision anchors

| Decision | Diagram impact |
|---|---|
| D1 Path B read-only + manual | 01, 02, 03, 05, 08, 10, 15 |
| D2 GMX manual-only | 01, 12, 15 |
| D3 USDC/CCTP only | 01, 06, 08, 12 |
| D4 x402 V2 | 01, 07, 15 |
| D5 CCTP Standard/Fast | 06, 08, 16 |
| D6 policy ownership | 02, 04, 05, 10 |
| D7 execution order | 01, 05, 12, 14, 16 |
| D8 Gemini models | 01, 12 |
| D9 HyperVerge conditional | 03, 09, 18 |
| D10 TypeScript/NestJS/testnets | 11, 12 |

---

## 6. Known limitations and review notes

1. **Mermaid is an approximation.** Component, deployment, use-case, and DFD views use flowchart shapes. The README mapping is part of the diagram contract.
2. **The ER diagram is logical.** It shows 13 entities, not the full physical schema. `audit_entries` has a composite primary key; `payments` and `audit_entries` allow a null user for anonymous agents or system events.
3. **x402 settlement order.** The sequence follows the canonical charging rule: settle only after a successful 2xx query. The middleware call order in `audit_layer_x402.md` is the implementation reference.
4. **KYC vendor.** HyperVerge remains conditional on the D9 two-week sandbox/webhook spike. If the spike fails, the KYC diagrams keep the same canonical states and only the adapter changes.
5. **Gantt dates are indicative.** They start after the planning freeze and are not delivery commitments.
6. **No mainnet approval.** The deployment diagram includes testnet and mainnet-capable nodes, but the current release target is testnet. Mainnet remains a separate gate.
7. **`.mmd` is the source of truth.** `png/` is a generated export. If a diagram and a canonical document disagree, change the decision record first, then update and re-render the diagram.

---

## 7. Change control

- Add or change a diagram only after the corresponding decision or canonical document is updated.
- Keep each `.mmd` file focused on one view. Do not turn a diagram into a second specification.
- Re-render and validate the PNG after every edit; keep source and export in sync.
- Record material diagram changes in the repository change log or the remediation report.

---

## 8. Revision history

| Version | Date | Change |
|---|---|---|
| 1.2 | 20 Nov 2026 | Reduced all 19 diagrams by one abstraction level; made the pack strictly monochrome; simplified the use-case view; documented the monochrome render config and ER label workaround; refreshed counts. |
| 1.1 | 19 Nov 2026 | Removed superseded diagram content; added 19 PNG exports; refreshed render commands and coverage notes. |
| 1.0 | 19 Nov 2026 | Initial 19-diagram Mermaid pack. |
