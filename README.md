# EFD Localization — Design Specification

> A design-level document. It explains the problem we are solving, what the data gives us, the principles our solution should follow, and what the final product should output. It deliberately does **not** prescribe code, modules, libraries, or implementation steps — those belong in a later technical spec.

---

## 1. Purpose

### What problem are we solving?
Airfield lighting circuits already know **that** an earth fault has occurred somewhere on a runway or taxiway loop. They cannot tell anyone **where** on the loop the fault is. Today, finding the location is a manual job: a technician walks the circuit with handheld test gear or isolates sections one at a time. On a long runway, this is slow, expensive, and disruptive.

We want to make this question — *"where is the fault?"* — answerable from data, in the cloud, the moment a fault is detected.

### Why EFD localization matters for airport maintenance
- **Runway downtime is expensive.** Every minute the affected circuit is offline reduces airport throughput.
- **Engineer time is scarce.** Walking a long circuit by hand is labour-intensive, especially out-of-hours or in poor weather.
- **Operational disruption compounds.** Slow fault response can force schedule changes, restricted operations, or diversions.
- **Safety depends on speed.** Faults need rapid isolation; uncertainty about their location complicates safe dispatch.

### Value to ADB Safegate
ADB Safegate already collects PLC (powerline communication) telemetry from every fixture as part of its EFD product. This solution turns that *existing* stream into a **fault triage capability** — a credible, defensible upgrade from the current "we know there is a fault, good luck finding it" state, deliverable through CORTEX Service without new hardware.

Even a result that **halves the search zone** delivers meaningful operational value.

---

## 2. Problem Domain

### Airfield lighting series circuits
Unlike the parallel wiring in a building, airfield lighting fixtures are connected in **series** along a single long loop. A Constant Current Regulator (CCR) at one end pushes a regulated AC current — typically 6.6 A — around the entire loop. Every fixture sees the same current. This guarantees identical brightness at every fixture even on multi-kilometre runs.

The trade-off is **diagnosis is harder**: because every fixture shares the same current path, you cannot read one fixture's electrical value and say "this one is at fault" the way you could in a parallel circuit.

### Earth faults
An earth fault is an unintended electrical path between a circuit conductor and ground — caused by damaged insulation, water ingress, a corroded fitting, or a damaged cable section. Faults are characterised by their impedance (resistance) to ground:
- **0 Ω** — a hard short (dead earth fault). Easy to detect, large electrical signature.
- **Tens of kΩ** — a high-impedance fault. Far more subtle: only a trickle of current leaks, but the circuit is degraded and at risk of worse failure.

### Detection vs. localization
These are two different problems. Detection asks *"is there a fault?"* — and the existing EFD system answers it well. Localization asks *"where on the loop is it?"* — and is currently unsolved in software. This document is about the second question only.

### Why PLC telemetry can carry location information
The same two conductors that carry power to each fixture also carry the data link between the CCR and each fixture (PLC — Powerline Communication). When a fault leaks current to ground, it also leaks **PLC carrier signal** to ground. Fixtures near or beyond the fault should therefore experience a weaker, noisier, or more delay-prone communication channel than fixtures upstream of the fault.

The shape of that degradation, distributed across the 90 fixtures of the loop, is in principle a **spatial fingerprint** of where the fault is. Extracting it is the core technical bet of this work.

Crucially, the fixtures are addressable in a known order (IDs 1001–1090) and that order mirrors the way they sit along the physical loop. Even though we are not given exact metres-from-CCR coordinates, **fixture ordering preserves circuit topology and acts as a spatial proxy for localization reasoning.** Any signal that a fault carries about its location must show up as a change *along that ordered chain* — not as an isolated anomaly at one randomly addressed fixture.

---

## 3. User / Stakeholder Need

### Who would use this solution?
- **Primary user:** the dispatched maintenance technician or field engineer responsible for finding and clearing the fault.
- **Secondary user:** the airport operations or maintenance manager who decides when, how, and with what urgency to dispatch.
- **Indirect stakeholder:** the airport itself, whose runway availability and operational continuity depend on fast resolution.

### What question are they trying to answer?
> *"There is an earth fault on this circuit. Where should the technician inspect first?"*

### Operational framing
The user does not need a perfect coordinate. They need the search zone narrowed enough that they can act on it. A confident "between fixtures 40 and 55" is far more useful than a vague "somewhere on the loop", and is honest about the precision the data actually supports.

---

## 4. Dataset Role

### What the dataset represents
A controlled lab recording made by ADB Safegate. A real CCR cabinet feeds a real series circuit of 90 Axon EQ inset fixtures. Earth faults are deliberately introduced at four physical locations along the loop, at several resistance values per location, and the resulting PLC telemetry is captured for roughly 15 minutes per scenario.

This is a **prototyping dataset**, not production field data. It is engineered to stress-test localization logic in a clean, repeatable environment.

### Structure
- **Each CSV = one recorded fault scenario** — about 15 minutes of telemetry from all 90 fixtures.
- **90 fixtures** in series, with airfield IDs 1001 through 1090. The ID order corresponds to the order in which fixtures sit along the loop, so the ID sequence functions as a **topological coordinate** — a spatial proxy that we can reason on even though exact physical positions are not provided.
- **Three independent test runs** — T0001, T0002, T0003 — recorded on three separate days using the same physical setup. Generalisation across runs is a meaningful credibility check.

### The four message families (in plain language)
Within each scenario, four kinds of telemetry message arrive interleaved, each fixture reporting all four:

- **COMM — communication success/reliability.** How well is the master reaching this fixture? Response rate, attempts, failures, average repeater hop depth, average received signal strength.
- **RATES — communication behaviour across repeater depths.** A breakdown of how often each fixture answers at wave depth 0 (direct), 1, 2, ... up to 7, in both directions. Tells you the *shape* of the channel, not just an average.
- **EXCOMM — physical-layer signal quality.** Lower-level receiver diagnostics: peak input level, bit error count, phase noise, CRC failure count. Closer to the raw radio.
- **CORR — signal versus noise detection behaviour.** From the modem's correlator stage: how strong are real-signal peaks vs. noise peaks? Effectively a signal-to-noise indicator.

### Filename metadata is for evaluation only
Each filename encodes the true `faultLocation` (0–4, with 0 = no fault) and `faultResistance` code. **These are ground truth for measuring how well our predictions did. They must never be used as inputs to the prediction logic itself.** Reading the answer from the filename defeats the entire exercise.

---

## 5. Design Principles

The solution should be guided by the following principles, even before any specific approach is chosen.

- **Preserve spatial structure.** Fixture ordering (1001 → 1090) is the topological backbone of the problem. Localization is fundamentally about identifying *where along that ordered chain* the telemetry behaviour changes, so features must be kept per-fixture and analysed as a sequence. Collapsing the chain to a single circuit-level number erases exactly the information that distinguishes one fault location from another.
- **Use physics-guided reasoning, not only black-box pattern matching.** A prediction we can defend from how PLC signals propagate through a series circuit and how earth faults distort them is more credible — and more generalisable — than one justified only by statistical correlation.
- **Compare fault scenarios against baseline / no-fault behaviour.** Each fixture has its own quirks (manufacturing variation, position effects). Interpreting fault scenarios *relative to* baseline removes those fixed biases and makes a real spatial change easier to see.
- **Cross-validate across telemetry families.** COMM, RATES, EXCOMM, and CORR are physically independent views of the same channel — aggregate comms, depth-resolved comms, raw physical-layer quality, and modem-level SNR. No single family should be trusted in isolation. The localization signal becomes credible when **independent telemetry families converge on the same spatial region**; an isolated anomaly in one family alone is more likely to be noise than localization evidence.
- **Prefer ranked fault zones over overconfident single answers.** A ranked list of plausible zones is more useful, more honest, and more robust than a single "the fault is here" claim.
- **Quantify uncertainty honestly.** A confidence score must reflect how well the data actually constrains the answer. Overclaiming precision is worse than admitting a wide zone.
- **Explain why a zone is suspicious.** Every output should carry the telemetry evidence that drove it — which signals changed, where, and how much.
- **Avoid using leaked filename labels as inputs.** The filename leaks the ground truth. It is for evaluation only, never for prediction.
- **Be consistent across the three test runs.** A method that works on T0001 but not on T0002 or T0003 has overfitted to one recording, and is not yet a real localization method.
- **Allow the system to say "I don't know."** A localization tool that refuses to answer when the data doesn't support an answer is more useful in production than one that always commits.

### Interpretability first, opaque modelling later
The dataset is deliberately small and lab-controlled: three runs, four fault locations, a handful of resistance values, 90 fixtures. That is enough to *prototype* a localization method, but nowhere near enough to safely train and trust an opaque end-to-end model — and an airport engineer dispatched on the result needs evidence they can read, not a score they have to take on faith.

The design philosophy therefore prioritises **physically interpretable reasoning and explainable spatial evidence** before any opaque approach is even considered. Every claim the system makes about a zone should be traceable back to specific telemetry behaviours along the ordered fixture chain — patterns an engineer could in principle eyeball and agree with. More opaque techniques may earn a place later, once there is enough production data to validate them and once interpretability tooling can keep up; they are not the starting point.

---

## 6. Expected Solution Behavior

### Exploratory analysis as a foundation
Before any localization logic is fixed, the solution depends on a deliberate phase of **exploratory analysis** of the dataset itself. The goal of that phase is conceptual, not algorithmic: to understand what "healthy" looks like before trying to recognise "faulted". Specifically, it should:
- Characterise **baseline / no-fault behaviour** per fixture, so the system knows the natural operating range of each telemetry field.
- Quantify **natural fixture-to-fixture variation** under no-fault conditions — some fixtures will simply be quieter or noisier than others, and that variation must not be misread as a fault signature.
- Identify which **telemetry families exhibit stable, smooth spatial structure** along the fixture chain in the no-fault case — these are the families whose deviations are most informative under fault.
- Build an intuition for **how fault scenarios differ from baseline** — where, by how much, and across which families — at both the easy end (hard shorts) and the hard end (high-impedance faults).

This phase is the empirical grounding for every later design choice. Without it, any localization rule risks being a guess dressed up in equations.

### What the solution should ultimately do
At the conceptual level, the solution should:

1. **Accept one telemetry scenario** as input — the equivalent of one CSV: roughly 15 minutes of interleaved telemetry from all 90 fixtures.
2. **Summarise communication health by fixture** across all four message families, producing a per-fixture view of how the circuit looked during the scenario.
3. **Look for spatial changes across the 90-fixture chain** — discontinuities, slopes, sustained degraded plateaus, or clustered anomalies that are unlikely under a healthy baseline.
4. **Identify likely fault zones** — contiguous ranges of fixtures that show coordinated degradation.
5. **Provide ranked predictions with confidence** — most likely zone first, with second- and third-most-likely alternatives and their confidence levels.
6. **Provide supporting evidence** — for each ranked zone, the telemetry patterns that drove it (e.g., "RATES show a step in upstream depth-0 success rate around fixture X; EXCOMM peakInputLevel drops in the same region").

---

## 7. Localization Logic at Design Level

This section describes the *kind* of reasoning the solution should use, not how to implement it.

- **A fault weakens or distorts the PLC carrier where it leaks to ground.** This is the underlying physical mechanism the localization signal relies on.
- **Fixtures near or beyond the fault are most likely to show the symptoms.** Upstream fixtures (between the CCR and the fault) often look healthier; downstream and adjacent fixtures absorb most of the degradation.
- **Useful symptoms span all four message families:**
  - Lower per-fixture **response rate** (COMM).
  - Greater reliance on **repeaters** — i.e., higher wave depths (COMM, RATES).
  - Weaker **received signal strength** (COMM).
  - More **bit errors** and **CRC failures** (EXCOMM).
  - Worse **signal-to-noise** behaviour at the modem — correlated peaks weaker, uncorrelated peaks stronger (CORR).
- **A likely fault zone is a region of the chain where the spatial behaviour *changes* meaningfully.** Localization is, at heart, a search for **change-points and spatial discontinuities** along the ordered fixture sequence. Several shapes of change are physically plausible:
  - **Step / change-point** — a sharp transition between healthier upstream fixtures and degraded downstream fixtures, consistent with a fault that splits the loop into a "before" and "after" region.
  - **Gradient / slope** — a gradual roll-off of signal quality across a span of fixtures, consistent with a softer or higher-impedance fault whose influence fades with distance.
  - **Clustered degradation region** — a bounded group of adjacent fixtures whose telemetry departs together from baseline, separated on both sides by healthier neighbours.
  Wherever one of these patterns is consistent across telemetry families and stable across the recording window, it is a candidate fault zone.
- **Multiple degradation modes pointing to the same zone are a stronger signal than any one mode alone.** COMM, RATES, EXCOMM, and CORR are physically independent windows on the channel; **agreement between them is a form of cross-validation** between independent signal sources. Convergence across families raises confidence; disagreement lowers it; an apparent anomaly visible in only one family is more likely to be local noise than localization evidence.

---

## 8. Confidence Concept

Confidence is not a decoration — it is a first-class output. It tells the engineer how much weight to give the prediction, and whether the dispatch decision should depend on it.

The core mechanism that drives confidence here is **cross-validation between independent telemetry families.** COMM, RATES, EXCOMM, and CORR each look at the channel through a different physical lens; when several of them flag the same spatial region for the same reason, that agreement is engineering evidence that the signal is real and not an artefact of one stream.

- **High confidence** — multiple telemetry families independently point to the same zone, the spatial pattern is sharp (a clean change-point or tight cluster), and the result is consistent with how the fault would physically behave. Suitable for direct dispatch guidance.
- **Medium confidence** — only some families show the pattern, or families agree on the region but the spatial transition is soft (a shallow gradient rather than a step). The zone is meaningful, but the engineer should treat it as a starting point rather than a verdict.
- **Low confidence** — signals are flat, faint, or partially contradict each other across families. Common with high-impedance faults (tens of kΩ), where the comm channel is barely disturbed and any one family may not move at all.
- **Uncertain.** The system must be allowed to say *"I cannot localize this fault from the available data."* That is a valuable answer in itself — far better than fabricated precision.

---

## 9. Evaluation Concept

At the design level, evaluation answers a single question: **how well do our predicted zones align with the truth, across the dataset?**

- **Ground truth comes from the filename**, which encodes `faultLocation` and `faultResistance`. It is read **only after** prediction, never inside the prediction logic.
- **Easy cases first.** Strong, low-resistance faults (0 Ω, 5 kΩ) should produce the cleanest spatial signature and the highest accuracy. If the method cannot localize a hard short, it cannot localize anything subtler.
- **Hard cases second.** High-impedance faults (30 kΩ, 50 kΩ) are genuinely hard; partial credit (a wider but correct zone) and honest "uncertain" outputs are acceptable.
- **Cross-run consistency.** A credible method gives consistent results across T0001, T0002, and T0003 with the same parameters. Big swings between runs suggest overfitting to one recording.
- **No-fault scenarios matter too.** Baseline files (`...0-0.csv`) test whether the method correctly says "no localized fault" rather than hallucinating one.

---

## 10. Limitations and Gaps

A clear-eyed view of what this dataset and approach **cannot** do is part of the deliverable.

- **Fault locations 1–4 are codes, not coordinates.** We know there are four distinct physical locations on the loop, but their actual positions are deliberately confidential. We can localize *to a zone* but cannot map predictions to physical metres without further information.
- **Lab data only.** No real airport variability — no weather, no ageing field cable, no installation noise, no concurrent operational disturbances. Production behaviour will differ.
- **No fine-grained fixture position map.** Fixture order is preserved (IDs 1001–1090), but per-fixture distances and exact circuit geometry beyond that order are not provided.
- **High-resistance faults may be undetectable from comms alone.** The PLC channel can stay healthy through a real fault. Absence of degradation is **not** absence of fault.
- **No electrical telemetry from the CCR.** Direct CCR readings (insulation resistance, phase angles, leakage currents) are not part of this dataset, and would likely lift localization precision substantially.
- **Sample size is modest.** Three runs across a handful of locations and resistances is enough to prototype, not enough to fully characterise edge cases.
- **Additional production data would meaningfully improve confidence.** CCR electrical readings, impedance measurements, fixture position metadata, and finer-grained sensing are all credible upgrades.

---

## 11. Final Product Vision

What a service engineer should see when they open the result of one fault scenario, in human terms:

- **Most likely inspection zone.** A clear range on the circuit, e.g., *"Inspect fixtures 40 to 55 first."*
- **Ranked alternatives.** Second- and third-most-likely zones, each with their own confidence — so if the first zone turns up nothing, the engineer already knows where to look next.
- **Confidence level.** High / Medium / Low / Uncertain — explicit, plain language.
- **Reason and evidence.** A short plain-language summary of which telemetry patterns drove the call (e.g., *"Response rate drops sharply at fixture 42; phase noise rises through fixture 50; correlated signal weakens beyond fixture 45"*).
- **Plots / visual explanation.** Per-fixture telemetry plots across the ordered 90-fixture chain are not a "nice to have" — they are central to the product. Visual explainability lets a human engineer **see** the spatial degradation pattern the system reasoned from, judge for themselves whether it looks like a credible fault signature, and spot when the system has been fooled by something benign. The output should make it easy to compare a faulted scenario against the baseline along the same fixture axis, with the suspected zone clearly marked.
- **Notes for technicians and engineers.** What to inspect, what to bring, caveats about the confidence level, and an honest *"the data does not support a confident answer"* when that is the truth.

The product's job is **not to replace the engineer's judgement, but to support it.** It should give an engineer a defensible starting point, the spatial evidence behind it, and an honest read on how much to trust it — so that the human dispatched to fix the fault walks toward the right part of the runway first, with the reasoning visible enough to challenge if something on site doesn't match the prediction.
