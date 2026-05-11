# Case Study 3: EFD Localization

## Case Introduction
Airport lighting circuits run hundreds of fixtures in series along runways and taxiways. When an earth fault occurs anywhere on a circuit, the system knows a fault exists — but not *where*. Identifying the precise location quickly is operationally critical: a fault on an active runway must be isolated and resolved with minimum disruption to aircraft operations, and the longer it takes to localize, the larger the impact.

Today, fault localization is largely a manual process — engineers walk the circuit or use specialised handheld equipment to track down the failure. The company wants to make this **data-driven from the cloud**: surface the most likely fault location automatically, so dispatched engineers know where to start looking.

The signal source you will work with is the **powerline communication (PLC) telemetry** the CCR exchanges with each fixture. ADB Safegate's series airfield circuits use PLC over the same lighting power conductors the fixtures run on, so when an earth fault occurs, signal energy bleeds to ground and degrades the communication path between the CCR and fixtures near or beyond the fault. The pattern of that degradation — distributed across the fixtures in the circuit — encodes spatial information about where the fault is. Your job is to extract it.

The dataset is from a controlled lab test: 90 Axon EQ fixtures in series on a CCR, with deliberately injected earth faults at four physical locations, across multiple fault resistance values from a 0 Ω dead short up to a high-impedance 50 kΩ path, repeated across three independent test runs. For each scenario you receive ~15 minutes of telemetry across four message types per fixture:

* **COMM** — communication-attempt statistics: response rate, attempts, failures, repeater hop depth, average receive signal strength
* **RATES** — downstream and upstream raw response rates at 8 different wave depths
* **EXCOMM** — extended physical-layer metrics: peak input level, bit error count, phase noise, CRC failure count
* **CORR** — additional comm-related metric (schema pending) ➤ Simply put: the system already knows *when* something is wrong, and the comm telemetry quietly carries hints about *where*. Your job is to find them.

Fault location ground truth is deliberately withheld from the student-facing dataset — that is what your algorithm will be measured against.

## The Challenge
Your mission: Given PLC telemetry from a 90-fixture series circuit under an earth fault of unknown location, narrow down where on the circuit the fault has occurred — and convey how confident you are in that estimate.

You will work with telemetry from the CCR and the Axon EQ fixtures and must build a solution that can:

✅ Produce a fault triage output — given a fault scenario (one CSV's worth of telemetry), return a ranked list of probable fault locations or zones, with confidence levels that honestly reflect how well the available signals constrain the answer
✅ Justify your localization logic from physics and engineering reasoning — how PLC signals propagate through a series airfield circuit, how earth faults distort those signals, and how those distortions manifest across fixture-level telemetry — not just statistical correlation
✅ Make meaningful use of the spatial pattern across fixtures and the structural differences between message types, rather than collapsing the data to a single feature
✅ Provide a clear gap analysis — what existing telemetry is genuinely useful, what is unreliable, and what additional data (CCR electrical readings, impedance measurements, fixture position metadata, finer-grained sensing) would be needed to lift localization precision toward production-grade
✅ Demonstrate or sketch how the localization output would surface to a service engineer dispatched to fix the fault

**Out of scope:** Pinpointing faults to absolute precision from PLC telemetry alone. This data was not engineered for localization — it is a comm-health stream that happens to encode spatial fault signatures as a side-effect. Even an estimate that halves the area an engineer needs to inspect delivers real operational value — your goal is useful narrowing, not perfect location.

## Bonus Track: From Algorithm to Product
*(Optional — additional points)*

Once you have built your algorithm, step back and ask: does it fit the kind of product CORTEX Service is, and does it deliver what an airport customer would actually pay for? Sketch a commercial roadmap for it — anchored in how you think customers would want to use this capability.

This case study (EFD Localization) and Case Study 1 (Structural Integrity Monitoring) are intended to ship as part of the same algorithm package within CORTEX Service. You may consider this algorithm **together with** the other algorithm in the package, or treat it as a **standalone offering** — whichever framing produces a more credible commercial story.

Address the following dimensions:

* **Commercialization strategy** — Where does this product live? Inside CORTEX Service as a built-in capability, alongside CORTEX Service as a separate entity airports buy on its own terms, or as some hybrid? And within whichever choice you make, is it sold as one bundle with the other algorithm or as separate modules? Anchor both decisions in how you think airport customers would actually want to use this.
* **Pricing model & roadmap** — How do you charge for this: flat fee, per-fixture, per-airport, tiered, value-based per incident? And how does pricing evolve as the package matures from v1 (current data, current capability) through v2 and v3 (additional data streams, additional algorithms in the suite)?
* **Value articulation** — In concrete terms, what is the customer actually buying? Runway minutes saved per fault event, engineer hours avoided, faults isolated faster, downstream operational disruption reduced?
* **Algorithm-package lifecycle** — What ships at launch with the data and capability you have today? What improves as data quality and coverage grow? What additional algorithms (e.g. LED degradation, once production-ready) might join the package over time?

## Why It Matters
Slow fault localization at airport scale carries real cost:

* **Runway and taxiway downtime** — every minute a fault remains unlocalized is a minute the affected circuit is unavailable, with knock-on effects on aircraft movement
* **Engineer time** — manually walking circuits to track down faults is labour-intensive and slow, especially across long runways
* **Operational disruption** — fault response on active runways can require schedule changes, restricted operations, or diversions
* **Safety** — faults need rapid isolation, and uncertainty about fault location complicates safe maintenance dispatch

The bar for value is genuinely low here: even a rough zone estimate that halves the area an engineer needs to inspect translates directly into faster fault response, less runway downtime, and better dispatch decisions. Software-driven localization on PLC telemetry — even if imperfect — is a meaningful step up from the manual process used today.

## Judging Focus
* **Quality of localization logic** — is the link between PLC telemetry patterns and fault location defensible from engineering and physics reasoning, or speculative? Does it make sense given how powerline communication propagates through a series airfield circuit?
* **Use of the message-type structure** — does the team make meaningful use of the four telemetry message types and the per-fixture spatial pattern, or does the analysis collapse the data to a single signal?
* **Quality of confidence and uncertainty handling** — does the team honestly communicate how certain their estimate is, and how that certainty depends on available signal quality? Overclaiming precision is worse than admitting a wide zone.
* **Generalization across the three test runs** — does the team's approach work across all three independent test runs, or has it been over-fit to one? Cross-run consistency is a meaningful credibility signal.
* **Specificity of the gap analysis** — does the team clearly distinguish what their localization can and cannot tell from this dataset, and articulate what additional measurements would unlock higher precision, concretely?
* **Usefulness of the engineer-facing output** — would a dispatched engineer be able to act on the result and find the fault faster, or does it still require interpretation that has not been done?
* **Bonus — Commercial coherence** *(stretch)* — does the team's commercialization strategy, pricing, and value articulation hang together as a credible CORTEX Service offering, driven by how customers would actually want to use the product?
