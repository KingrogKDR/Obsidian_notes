Your instinct is right, and I'd hold that line firmly: **nothing that produces or joins a number goes through a model.** The useful way to think about it isn't "AI vs. deterministic" as a single choice — it's drawing a hard boundary between a deterministic _core_ that owns every value that must reconcile, and an optional AI-assisted _periphery_ that never touches those values at runtime.

Here's where the boundary sits:

|Task|Who does it|Why|
|---|---|---|
|Rating (hours × rate), aggregation, currency conversion|Deterministic only|Must reproduce exactly and tie to an invoice. A model doing arithmetic is both wrong-prone and non-reproducible.|
|Joining cost rows to utilization by resource ID|Deterministic only|This is the exact thing you're worried about. A fuzzy/AI join that's 99% right means 1% of rows silently attribute one VM's GPU usage to another's spend — undetectable and corrosive to trust.|
|Reconciliation against the real invoice|Deterministic only|Auditable math, or it's not reconciliation.|
|Schema mapping (provider field → FOCUS column)|AI _may propose_, human approves, frozen into config|The discovery is fuzzy; the _application_ must not be. See below.|
|Classifying ambiguous line items into a taxonomy (service category, ChargeCategory)|AI may propose a lookup table, reviewed, then applied deterministically|Genuinely fuzzy input, but you want a frozen table, not a per-row model decision.|
|Narrative / "why did cost change" summaries|AI, on top of already-correct numbers|Explanation layer, never the computation layer.|

The principle that keeps you safe: **AI proposes, deterministic code disposes, a human approves — at design time, not run time.** When you onboard a new provider (Hetzner, OTC), it's completely reasonable to point a model at their field docs and say "propose a mapping to FOCUS 1.2." That saves real work. But its output is a _draft config_ a person reviews and commits. From then on, every row flows through that reviewed, version-controlled mapping deterministically. The model is a compile-time assistant, not a runtime component. Same for classification: use it to _build_ the crosswalk table once, review it, then apply the table by exact lookup forever after.

The specific trap to name and avoid: the **fuzzy-join temptation.** On these sovereign clouds, the resource ID in the cost data and the resource ID in the metrics API often don't match formats (different prefixes, region-qualified vs not, UUID vs slug). The lazy fix is an AI/fuzzy matcher. Don't. Build a deterministic ID-normalization crosswalk, and where an ID genuinely can't be matched, **emit it as unmatched rather than guessing.** A visible null is a data-quality signal you can chase; a confident wrong match is a silent lie in a financial report. "Accurate to the tee" means you'd rather show a gap than paper over it.

Three reasons this matters beyond correctness, given everything upstream in this project:

- **Auditability / GDPR:** you've been building toward a defensible processor posture. If a customer disputes a cost or utilization figure, you need to show the exact deterministic computation — inputs, rate, formula, join key. "The model inferred it" is not a defensible answer for a financial or personal-data-adjacent figure, and it breaks the reproducibility an audit expects.
- **The rating estimates are already approximations.** For Hetzner and OTC you're synthesizing cost from inventory × published price, which already won't perfectly match the invoice. Layering probabilistic enrichment on top of an already-modeled number compounds two error sources and makes the gap un-diagnosable. Keep the modeled-cost path boringly deterministic so the _only_ discrepancy is the known rating gap, which you can reconcile monthly.
- **Testability:** a deterministic core is unit-testable and regression-testable — you can assert "this fixture produces exactly these numbers" in CI. Anything with a model in the value path can't be pinned like that, so you lose your safety net exactly where you most need one.

So: yes to AI, but scoped to the fuzzy edges — mapping discovery, classification table-building, and post-hoc narrative — with every numeric value, join, and reconciliation staying in deterministic, testable, auditable code. That gives you the productivity where ambiguity is real, without ever putting a probabilistic component between the customer and a number that has to be exactly right.