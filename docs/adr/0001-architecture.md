# ADR-0001: WiringDeviceAdvisor ⊣ Wiring Devices Plant Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-2733` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-2733` publishes an OSS blueprint for the ISIC 2733
class ("Manufacture of wiring devices") **plant operations
coordination** (production-batch product-type/contact-resistance/
quantity/defect-rate data logging, molding/assembly/test-line-equipment
maintenance scheduling, safety-concern flagging, and outbound
wiring-device shipment coordination). Like every actor in this fleet,
the blueprint alone is not an implementation: this ADR records the
governed-actor architecture that promotes it to real, tested code,
following the same langgraph StateGraph + independent Governor + Phase
0->3 rollout pattern established across the cloud-itonami fleet.

The closest domain analogs are `cloud-itonami-isic-2790` (Manufacture
of other electrical equipment) and `cloud-itonami-isic-2710`
(Manufacture of electric motors, generators, transformers and
electricity distribution and control apparatus): all three are
back-office coordination actors for a fixed electrical-equipment-
manufacturing PLANT with heavy molding/assembly/test equipment and a
real physical safety dimension, and all three share the same four-op
shape (`:log-production-batch`/`:schedule-maintenance`/
`:flag-safety-concern`/`:coordinate-shipment`) and the same two-entity
verified/registered gate structure (equipment for maintenance
scheduling, batch for shipment coordination). This build mirrors those
two siblings' architecture closely but adapts the product/hazard
vocabulary to ISIC 2733's own scope: switches, socket-outlets
(receptacles), plugs, and junction boxes -- distinct from sibling ISIC
2732 ("Manufacture of other electronic and electric wires and
cables"), which covers wire/cable products rather than the fixed
devices that terminate, switch, or distribute a circuit. This
vertical's production-batch record declares a `:product-type` (closed
set spanning switch/socket-outlet/plug/junction-box) and a
`:contact-resistance-milliohm` (a routine micro-ohmmeter contact-
resistance test reading, plausibility-checked 0-20,000 mΩ against the
working range of standard production micro-ohmmeters per IEC 60669-1 /
IEC 60884-1 acceptance testing) in addition to a
`:defect-rate-percent`, rather than 2790's `:insulation-resistance-
mohm` (a lower-current megohmmeter insulation-resistance reading) or
2710's `:dielectric-test-kv` (a high-voltage hipot withstand-test
reading in kV) -- ISIC 2733's wiring devices are typically QC-tested
with a contact-resistance (micro-ohmmeter) test on their switching/
terminal contacts after a temperature-rise cycle, reflecting this
domain's focus on the physical switching/connection contact itself
rather than bulk winding insulation or high-voltage withstand. Its
shipment quantity is tracked in finished-unit UNITS (`:units`/
`:quantity-units`/`:shipped-units`), the same shape both siblings use
for finished units (counted, not weighed, for freight coordination).

Like both siblings, this vertical is subject to electrical-safety
certification regimes (e.g. UL 498/UL 20, IEC 60669, IEC 60884, CE
marking under the EU Low Voltage Directive) for its finished wiring-
device products. This actor is never the certification authority --
any proposal (regardless of op) that declares `:issue-certification?
true` is a HARD, PERMANENT, unconditional block
(`wiringdevmfg.governor/certification-authority-blocked-violations`),
the same "no phase, no human override" posture as the equipment-
actuation block.

This vertical has NO pre-existing `kotoba-lang/wiringdevmfg`-style
capability library to wrap (verified: no such repo exists). This build
therefore uses self-contained domain logic -- pure functions in
`wiringdevmfg.registry` (equipment/batch verification, shipment-
quantity recompute, product-type validation, contact-resistance
plausibility validation, defect-rate plausibility validation) are
re-verified independently by the governor, the same "ground truth, not
self-report" discipline established across prior actors (most directly
`cloud-itonami-isic-2790`'s `otherelecmfg.registry` and
`cloud-itonami-isic-2710`'s `elecequipmfg.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:wiring-devices-plant-operations-governor`, is grep-verified UNIQUE
fleet-wide (`gh search code "wiring-devices-plant-operations-governor"
--owner cloud-itonami`, zero hits before this repo was created).

## Decision

### Decision 1: Self-contained domain logic (no external wiring-device-manufacturing capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
ISIC 2733 vertical has NO pre-existing capability library to wrap. The
equipment/batch-verification / shipment-quantity / product-type /
contact-resistance / defect-rate validation functions live as pure
functions in `wiringdevmfg.registry` and are re-verified independently
by `wiringdevmfg.governor` -- the same "ground truth, not self-report"
discipline established across prior actors (most directly
`cloud-itonami-isic-2790`'s `otherelecmfg.registry`).

### Decision 2: Coordination, not control — scope boundary at the back-office

This actor is **strictly back-office coordination** of ISIC 2733
wiring-device plant operations. It does NOT:
- Control molding, assembly, or test-line equipment directly
- Make plant-safety or certification decisions (exclusive to the human plant supervisor / accredited certification body)
- Actuate molding/assembly/test-line equipment
- Self-issue an electrical-safety certification mark (e.g. UL/CE/IEC)

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human plant-supervisor
approval. This is not a replacement for the supervisor's authority or
the certification body's authority — it is a proposal-screening and
documentation layer.

**CRITICAL SAFETY BOUNDARY**: wiring-device manufacturing (switches,
socket-outlets, plugs, junction boxes) is a safety-critical domain
(electrical-safety certification, downstream product-safety and
worker-safety consequence). Safety-concern flagging NEVER
auto-commits. All safety concerns escalate immediately to human
review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (contact-overheating concern, electrical-safety
concern, dielectric-withstand-test-hazard concern, equipment-safety
concern) ALWAYS escalates, never auto-commits. This is not a
"low-stakes proposal" — it is a circuit-breaker that must reach human
authority.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Like both siblings, this vertical has TWO entity kinds each gating a
different op: `:schedule-maintenance` independently verifies the
referenced **equipment** unit's own `:verified?`/`:registered?`
fields; `:coordinate-shipment` independently verifies the referenced
**batch**'s own `:verified?`/`:registered?` fields. Both are the same
"plant/batch record must be independently verified/registered before
any action" HARD invariant applied to the two distinct record kinds
this domain actually has. `:coordinate-shipment` additionally
independently recomputes whether a batch's own recorded shipped-to-
date unit quantity plus the proposal's own claimed unit quantity would
exceed the batch's own recorded production quantity — never taken on
the advisor's self-report.

### Decision 5: HARD invariants (no override)

Four HARD governor invariants (elaborated into twelve concrete checks
in `wiringdevmfg.governor`, mirroring `cloud-itonami-isic-2790`'s own
elaboration of its HARD invariants into concrete checks) block
proposals and cannot be overridden by human approval:
1. Plant/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's quantity must independently recompute within the batch's own logged production quantity
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Direct molding/assembly/test-line-equipment control, equipment actuation, or self-issued electrical-safety certification is permanently blocked
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) ISIC 2733 wiring-device plant operations back-office now has a
documented, governed, auditable coordination layer that funnels all
decisions through independent validation before human approval.

(+) The "coordination, not control" boundary is explicit in code: all
`:effect :propose`, all real-world actuation requires human plant-
supervisor sign-off, and no certification mark can ever be
self-issued.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into twelve concrete governor checks) protect against scope creep into
unauthorized equipment operation, equipment actuation, or
certification self-issuance. Safety concerns are a circuit-breaker,
not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.
Human review is mandatory.

(-) Still a simulation/proposal layer, not a real plant-operations
control system. Equipment actuation, line operation, and certification
issuance remain human-/institution-controlled via external channels.

(-) No integration with real plant-management databases (equipment
telemetry, batch tracking, freight dispatch, certification-body APIs)
— this is a standalone coordinator blueprint.

## Verification

- `cloud-itonami-isic-2733`: `kbb -M:test` green (see the
  superproject ADR and `kotoba-lang/industry` registry entry for the
  exact fresh-clone re-verified output), `kbb -M:lint` clean,
  `kbb -M:dev:run` demo narrative exercises proposal submission,
  escalation, and every HARD-hold scenario directly (not-propose-
  effect, unknown-op, equipment-not-verified, batch-not-verified,
  shipment-quantity-exceeded, equipment-actuate-blocked,
  certification-authority-blocked, already-scheduled, invalid-
  product-type, invalid-contact-resistance-milliohm, invalid-defect-
  rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `kbb -M:test` resolves offline inside the monorepo checkout.
