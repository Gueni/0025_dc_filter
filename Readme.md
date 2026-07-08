# DC-Side EMI Filter for an OBC (Post Dual-Module LLC Stage)

An On-Board Charger (OBC) is the power-electronics box inside an EV that turns AC wall/charging-station power into the DC voltage needed to charge the traction battery (400 V or 800 V class packs are typical).

```mermaid
flowchart LR
    AC["AC Grid<br/>230 V / 50 Hz"] --> EMI["EMI Filter<br/>(AC side)"]
    EMI --> PFC["PFC<br/>Power Factor Correction<br/>(AC → ~400 V DC bus)"]
    PFC --> LLC["2× LLC Resonant Converter Modules<br/>Isolated DC-DC stage<br/>fsw = 150-500 kHz"]
    LLC --> DCF["DC Filter<br/>← this document"]
    DCF --> BAT["HV Battery Pack"]

    style DCF fill:#ffe28a,stroke:#333,stroke-width:2px
```
The two LLC resonant converter modules switch in the **150 kHz – 500 kHz** range. Even after the transformers and rectification, the DC bus carries:

- Ripple at the switching frequency and its harmonics (differential-mode, line-to-line)
- Common-mode (CM) noise — currents that flow through parasitic capacitances (transformer interwinding capacitance, heat-sink coupling, cable-to-chassis capacitance) equally on both output conductors, returning through the chassis

Both must be attenuated before the DC leaves the module and heads to the battery, to satisfy the automotive conducted-emissions standard CISPR 25.

```mermaid
flowchart LR
    subgraph SRC["Source"]
        V["V1 source<br/>Rser = 1 mΩ"]
    end

    subgraph IN["Input Capacitor Cluster (4 cells, X-type)"]
        direction TB
        IN2["C8-C11: 6 µF / 10 mΩ / 9 nH each"]
    end

    subgraph CMC["Common-Mode Choke"]
        direction TB
        W1["L17 (winding A): 3 mH / 3 mΩ"]
        W2["L18 (winding B): 3 mH / 3 mΩ"]
        C1["C17: 3 pF across winding A"]
        C2["C18: 3 pF across winding B"]
        K["k = 0.99625"]
    end

    subgraph OUT["Output Capacitor Cluster (3 cells, X-type)"]
        direction TB
        O1["C13: 3 µF / 50 mΩ / 2 nH"]
        O2["C14: 4.7 µF / 50 mΩ / 25 nH"]
        O3["C15: 4.7 µF / 50 mΩ / 25 nH"]
    end

    VM["V(n002)<br/>(ideal voltmeter)"]

    SRC -- "Bus A (top)" --> IN
    SRC -- "Bus A (bottom)" --> IN
    IN --> CMC
    CMC -- "Bus B (top)" --> OUT
    CMC -- "Bus B (bottom)" --> OUT
    OUT --> VM

    style CMC fill:#cdeaff,stroke:#333,stroke-width:2px
    style IN fill:#fff3b0,stroke:#333,stroke-width:1px
    style OUT fill:#e6d6ff,stroke:#333,stroke-width:1px
```

- Top conductor and bottom conductor together form a single floating differential pair (DC+ and DC−).
- Every RLC "cell" in both clusters is wired across the two conductors (a shunt / X-capacitor), not in series along one conductor.
- The only series element in the entire path is the CMC — one winding (L17) sits in the top conductor, the other (L18) sits in the bottom conductor, and they are magnetically coupled with k = 0.99625.

![DC Filter Schematic](DC_filter.png)

### Self-Resonant Frequency (SRF) — Input Cluster

For N identical cells in parallel, `L_eff = L/N` and `C_eff = C·N`, so the product `L_eff · C_eff = L·C` is unchanged. The SRF of a parallel bank of identical cells is therefore *exactly the same* as the SRF of one cell.

Input cluster (C8–C11: 6 µF, 9 nH each):

```
f_SRF = 1 / (2π√(9e-9 × 6e-6)) ≈ 685 kHz
```

Above this frequency, each of these input capacitors starts to look inductive rather than capacitive. As shown below, however, this cluster has negligible influence on the AC-sweep transfer function because it sits directly across the near-ideal source.

### The Common-Mode Choke

A common-mode choke is two windings, one per conductor, wound on a shared magnetic core so that:

| Current type | What happens in the core | Impedance seen |
|--------------|---------------------------|-----------------|
| Differential (equal & opposite currents on the two conductors — the useful DC/ripple current) | The two windings' fluxes largely cancel | Very low (only the *leakage* inductance) |
| Common-mode (equal, same-direction currents on both conductors, e.g. noise returning via chassis) | The two windings' fluxes add | Very high (full self-inductance) |

This is what lets one small component pass the DC charging current with negligible loss while presenting a large blocking impedance to CM noise.

```
L17, L18 (self-inductance)   = 3 mH each          →  L = 3 mH per winding
Series R on L17, L18         = 3 mΩ each          →  R = 3 mΩ per winding
K1 L17 L18                   = 0.99625             →  coupling coefficient k
M (mutual inductance)        = L · k = 2.989 mH
```

Differential-mode leakage inductance (what the "useful" signal sees, summed over both windings around the loop):

```
L_leak(DM) = 2 × L×(1 − k)   = 2 × 3 mH × (1 − 0.99625) = 2 × 11.25 µH ≈ 22.5 µH
R(DM)      = 2 × 3 mΩ        = 6 mΩ
```

A coupling coefficient of 0.99625 is a well-wound CMC — only 0.375% of the flux fails to cancel, leaving a small but non-zero series inductance in the DM path. This is intentional: filter designers often *want* a modest residual DM leakage inductance from the CMC, because it adds a "free" extra pole to the differential-mode LC filter without needing a separate DM choke.

Each winding (L17, L18) has a 3 pF capacitor (C17, C18) wired directly across its own two terminals. This is not a Y-safety-capacitor (those are nanofarad-scale, chassis-referenced, and safety-agency rated) — 3 pF is far too small and, critically, it is *not* connected to ground anywhere. It represents the inter-winding/turn-to-turn parasitic capacitance that every real wound inductor has. It is what makes a physical choke stop behaving like a choke at high frequency.

![ltsp](ltsp.png)

### The CMC's Own Self-Resonance

Because C17/C18 sit in parallel with each winding's own (R + jωL) branch, they form a parallel resonant tank with that winding, not a series one. A parallel LC tank has *maximum* impedance at resonance — the opposite of a series LC notch filter.

```
f_SRF(CMC winding, full L) = 1 / (2π√(3e-3 × 3e-12)) ≈ 1.68 MHz   (using full self-inductance)
```

But because the winding's *effective* series inductance in the DM loop is the much smaller leakage value, the tank that actually matters for the differential-mode signal resonates far higher:

```
f_SRF(CMC, DM leakage path) = 1 / (2π√(11.25e-6 × 3e-12)) ≈ 27.4 MHz
```

This is the real cause of the deepest notch in the Bode plot: around this frequency, the choke's own parasitic capacitance and its leakage inductance anti-resonate, momentarily presenting a *very large* series impedance to the differential path, which produces the sharpest attenuation feature in the whole curve. Above that frequency, the parasitic capacitance takes over and the choke's impedance *collapses*, which is why filtering performance falls apart above ~30–50 MHz.

### The Output Capacitor Cluster

| Cell | C | ESR | ESL | Individual SRF |
|------|---|-----|-----|-----------------|
| C13 | 3 µF | 50 mΩ | 2 nH | 2.05 MHz |
| C14 | 4.7 µF | 50 mΩ | 25 nH | — |
| C15 | 4.7 µF | 50 mΩ | 25 nH | — |



C14 and C15 are identical and effectively in parallel:

```
C_eff (C14∥C15) = 9.4 µF
L_eff           = 25 nH / 2   = 12.5 nH
R_eff           = 50 mΩ / 2   = 25 mΩ
f_res           = 1 / (2π√(12.5e-9 × 9.4e-6)) ≈ 417 kHz
```

C13's own SRF:

```
f_SRF(C13) = 1 / (2π√(2e-9 × 3e-6)) ≈ 2.05 MHz
```

Total output-side capacitance across all three cells: `3 + 4.7 + 4.7 = 12.4 µF`.

**Why real capacitors are modeled as RLC, not ideal C:** An ideal capacitor has impedance `Z = 1/(jωC)`, which falls forever as frequency rises. A real capacitor has parasitic series resistance (ESR) and parasitic series inductance (ESL):

```
Z_real(f) = R + jωL + 1/(jωC)
```

Below its SRF: capacitive (impedance falls with frequency). At its SRF: purely resistive, minimum impedance = ESR. Above its SRF: inductive (impedance rises with frequency, stops filtering). Modeling every capacitor bank as RLC cells is what allows this simulation to correctly predict the notches, the anti-resonance peak, and the eventual loss of high-frequency performance.

### The Transfer Function

Because the input capacitor cluster sits directly across the (nearly ideal) voltage source, with only the negligible 1 mΩ Rser in between, it draws current but cannot change the voltage that the CMC "sees." The entire shape of `H(f)` is therefore governed by just two things:

1. The CMC's differential-mode series impedance, `Z_CMC(f)` — combining leakage inductance, winding resistance, and the parasitic self-capacitance's anti-resonance
2. The output capacitor cluster's shunt impedance, `Z_out(f)` — the parallel combination of C13, C14, C15

To first order:

```
H(f) ≈ Z_out(f) / (Z_CMC(f) + Z_out(f))
```

## Bode Plot, Zone by Zone

![Plot](Plot.png)


#### Zone 1 — 10 Hz to a few kHz: Passband (0 dB, 0°)
At these frequencies every capacitor's impedance is enormous and every inductor's is negligible, so the whole network is transparent — required, since the actual DC charging current and low-frequency ripple must pass essentially undisturbed.

#### Zone 2 — ~8–15 kHz: The +45 dB Anti-Resonance Peak
```
f_peak = 1 / (2π√(L_leak(DM) × C_out,total)) = 1 / (2π√(22.5 µH × 12.4 µF)) ≈ 9.5 kHz
```
![alt text](image.png)

This is a series LC resonance between the CMC's DM leakage inductance and the total output capacitance. At this frequency, `Z_CMC + Z_out` collapses toward the small series resistances, so `H(f)` spikes well above 0 dB — the filter *amplifies* rather than attenuates.

**Critically, this peak sits well below the LLC's 150–500 kHz switching band**, so it is not directly excited by the fundamental switching frequency. It could still be a risk if burst-mode/light-load operation, sub-harmonics, or interleaving artifacts push energy down into the 8–15 kHz range — this should be checked against actual burst-mode behavior of the two LLC modules.

Typical mitigations: series/parallel damping resistor across part of the output bank, deliberately raising ESR or leakage inductance to lower Q, a dedicated RC damping leg, or verifying via simulation/measurement that no significant energy exists near 9–15 kHz.

#### Zone 3 — ~15 kHz to ~1 MHz: First Deep Notch (~417 kHz)
Caused by C14∥C15 (4.7 µF, 25 nH each) resonating together at ~417 kHz — the minimum of `Z_out(f)`, now bottomed out at **25 mΩ** (not 2.35 mΩ as in the earlier incorrect model). This means the notch is **roughly 20 dB shallower** than the earlier readme suggested.

**This notch lands almost exactly inside the LLC's 150–500 kHz fundamental switching range** — this is the filter's primary working region for the fundamental switching ripple and its low-order harmonics, and its placement here looks like deliberate design intent rather than coincidence.

If the two LLC modules are interleaved (phase-shifted operation), the fundamental ripple component may partially cancel and push dominant ripple energy toward the first harmonic (300 kHz–1 MHz) instead of the fundamental — in that case the 417 kHz notch is even better positioned, since it sits inside that harmonic band too. This should be confirmed against the actual interleaving/phase relationship between the two LLC modules.

#### Zone 4 — ~1–5 MHz: Partial Recovery
Between the C14/C15 resonance (~417 kHz) and C13's own resonance (~2.05 MHz), the two branches of the output cluster present an anti-resonance to each other, so combined `Z_out` rises again and `H(f)` recovers partway back up.

#### Zone 5 — ~2–15 MHz: Secondary Dip
As C13 (3 µF / 2 nH) crosses its own ~2.05 MHz SRF and starts going inductive, `Z_out` dips again, producing a secondary notch region.

#### Zone 6 — ~15–50 MHz: The Deepest Notch (~27.4 MHz)
Caused by the CMC itself: its 3 pF winding self-capacitance anti-resonates with its own leakage inductance around 27.4 MHz, briefly presenting a very large series impedance in the differential path. This drives `H(f)` toward zero — the deepest, sharpest notch in the whole sweep.

#### Zone 7 — Above ~50 MHz: Rising Floor, Filter Degrades
Past its own self-resonance, the CMC's parasitic capacitance takes over and its impedance falls with frequency instead of rising. Simultaneously, every output capacitor is now well above its own SRF and looks inductive. With both the series-L and shunt-C elements behaving backwards from their intended roles, the network loses its low-pass character entirely, and `H(f)` climbs back toward (and potentially above) 0 dB.

**Relevance to CISPR 25:** the strictest automotive class (Class 5) requires conducted-emission compliance up to 108 MHz. Filter performance visibly degrading above ~50 MHz means the 76–108 MHz FM broadcast band sits in the region where this filter is least effective — supplementary measures (ferrite beads, shielded enclosure, feedthrough capacitors) are the normal way this gap gets closed.

### Summary Table

| Frequency | Feature | Magnitude | Root Cause (verified) | Relation to LLC (150–500 kHz) |
|-----------|---------|-----------|------------------------|--------------------------------|
| 10 Hz – 5 kHz | Passband | 0 dB | Filter transparent | Below switching band |
| ~9.5 kHz | Anti-resonance peak | +45 dB | CMC leakage inductance (22.5 µH) resonating with total output capacitance (12.4 µF) | Below fundamental; risk only if burst-mode energy dips this low |
| ~417 kHz | 1st deep notch (shallower, ~25 mΩ floor) | Primary attenuation | C14∥C15 (4.7 µF/25 nH) minimum-impedance resonance | **Inside fundamental switching band — primary suppression region** |
| ~1–2 MHz | Partial recovery | rising | Anti-resonance between C13 and C14∥C15 branches | Above fundamental, in low harmonics |
| ~2 MHz | 2nd dip onset | falling | C13 (3 µF/2 nH) crossing its own SRF | 4th harmonic region |
| ~27.4 MHz | Deepest notch | Strongest attenuation | CMC's own winding self-resonance (leakage L vs. 3 pF parasitic C) | Well above fundamental and its early harmonics |
| > 50 MHz | Rising floor | increasing | Full parasitic (ESL/self-capacitance) dominance everywhere | FM band (76–108 MHz) underserved |

The input capacitor cluster (C8–C11) has essentially zero effect on the plotted curve. Why: it sits directly across the source, separated only by a 1 mΩ series resistor. An ideal voltage source holds its terminal voltage constant regardless of what's connected across it, so `V_in` doesn't change and everything downstream of the CMC neither knows nor cares that those 4 capacitors are there.

This doesn't mean the input cluster is pointless in real hardware — it does real work there: bulk energy storage, ripple-current sharing at the LLC's actual output impedance, and providing a low-impedance path for the LLC's switching ripple *current* right at the source. This AC-sweep configuration answers "given an ideal disturbance voltage at the input, how much reaches the output?" — a fair proxy for conducted emissions reaching the battery pack. It does not answer "how much does the input cluster reduce the ripple current the LLC stage itself must supply, or the voltage stress the CMC sees?" — that requires a current-source excitation with realistic source impedance.

### EMC Compliance Context — CISPR 25

CISPR 25 is the IEC/CISPR standard governing conducted and radiated emissions from vehicle components, chosen so they don't interfere with the vehicle's own radio receivers.

| Class | Typical application | Strictness |
|-------|----------------------|------------|
| Class 1 | Commercial vehicles | Least strict |
| Class 3 | Standard passenger cars | Mid-range |
| Class 5 | Premium passenger vehicles | Strictest |

Conducted emissions are measured (via a LISN) across 150 kHz – 108 MHz — **note this range starts exactly at the LLC's minimum switching frequency**, meaning the fundamental switching ripple sits right at the bottom edge of the measured window.

| Band | Frequency | Service |
|------|-----------|---------|
| LW | 150–300 kHz | AM Long Wave |
| MW | 530 kHz – 1.8 MHz | AM Medium Wave |
| SW | 5.9–6.2 MHz | AM Short Wave |
| FM | 76–108 MHz | FM Radio |

| Band | Coverage in this filter | Rough attenuation |
|------|---------------------------|---------------------|
| LW (150–300 kHz) | Approaching the 417 kHz notch — also directly overlaps LLC fundamental | Improving, tens of dB |
| MW (530 kHz–1.8 MHz) | Just past the notch, into recovery | Strong, but recovering |
| SW (5.9–6.2 MHz) | Near the secondary dip (past C13 SRF) | Strong |
| FM (76–108 MHz) | Above the CMC's own self-resonance | Degraded — needs supplementary filtering |

### Practical Design Takeaways

1. **The +45 dB peak at ~9.5 kHz sits below the LLC's 150–500 kHz operating band** — not a direct fundamental-frequency risk, but should still be checked against burst-mode/light-load sub-harmonic behavior of both LLC modules.

2. **The primary notch (~417 kHz) now falls almost exactly inside the LLC fundamental switching range** — good design placement, but it is **~20 dB shallower than previously modeled** due to the corrected 50 mΩ ESR (vs. the earlier incorrect 4.7 mΩ). This reduced attenuation margin at the fundamental should be re-verified against the actual CISPR 25 limit line for your target class.

3. **Interleaving matters:** with two LLC modules, confirm whether they're phase-interleaved. If so, the fundamental component may cancel and dominant ripple could shift toward the first harmonic (300 kHz–1 MHz) — still reasonably well covered by the notch/recovery region, but worth explicit verification.

4. A CMC's parasitic winding capacitance is a double-edged sword: it produces the deepest notch (~27.4 MHz) but also destroys high-frequency performance above ~50 MHz.

5. RLC (not ideal-C) modeling of every capacitor is what makes this simulation trustworthy — an ideal-capacitor model would predict ever-improving attenuation with frequency, which is physically impossible and dangerously optimistic for EMC sign-off.

6. An AC-sweep with a near-ideal voltage source is blind to the input-side capacitor bank's contribution to ripple-current handling; a current-source excitation is needed to evaluate that separately.

7. The filter's weakest region (FM band, 76–108 MHz) is unrelated to the LLC's fundamental switching frequency and will need supplementary measures (ferrite beads, shielding, feedthrough caps) regardless of switching-frequency tuning.

## Complete Component Value Tables (Corrected, from LTspice Schematic)

### Input capacitor cluster

| Group | # cells | C/cell | ESR/cell | ESL/cell | Individual SRF |
|-------|---------|--------|----------|----------|-----------------|
| Input filter stage (C8–C11) | 4 | 6 µF | 10 mΩ | 9 nH | 685 kHz |

### Common-Mode Choke (L17/L18)

| Parameter | Value |
|-----------|-------|
| Self-inductance per winding | 3 mH |
| Winding resistance per winding | 3 mΩ |
| Coupling coefficient (k) | 0.99625 |
| Mutual inductance (M = L·k) | 2.989 mH |
| DM leakage inductance (total, both windings) | 22.5 µH |
| DM series resistance (total) | 6 mΩ |
| Parasitic self-capacitance per winding (C17, C18) | 3 pF |
| CMC self-resonance (DM leakage vs. parasitic C) | ~27.4 MHz |

### Output capacitor cluster (C13, C14, C15)

| Cell | C | ESR | ESL | Notes |
|------|---|-----|-----|-------|
| C13 | 3 µF | 50 mΩ | 2 nH | f_res ≈ 2.05 MHz |
| C14 | 4.7 µF | 50 mΩ | 25 nH | Paired with C15 |
| C15 | 4.7 µF | 50 mΩ | 25 nH | Identical to C14 |
| C14 ∥ C15 combined | 9.4 µF | 25 mΩ | 12.5 nH | f_res ≈ 417 kHz |

**LLC Operating Context**

| Parameter | Value |
|-----------|-------|
| Number of LLC modules | 2 |
| LLC switching frequency range | 150 kHz – 500 kHz |
| CISPR 25 conducted-emissions range | 150 kHz – 108 MHz |
| Filter primary notch vs. LLC band | 417 kHz notch overlaps upper half of LLC range |
| Filter weak point vs. LLC band | FM band (76–108 MHz) unrelated to LLC fundamental; needs supplementary filtering |


---
# Proposed Redesigned Component Values
---
| Component | Parameter | Current Value | **Proposed Value** | Rationale |
|---|---|---|---|---|
| **Input cluster (C8–C11)** | C / cell | 6 µF | **6 µF** (unchanged) | Bulk storage already adequate; no effect on H(f) due to ideal-source shielding |
| | ESR / cell | 10 mΩ | **10 mΩ** (unchanged) | Not performance-limiting here |
| | ESL / cell | 9 nH | **9 nH** (unchanged) | — |
| **Damping network (NEW)** | Topology | — | **RC snubber, added in shunt across CMC input side** | Needed to tame the +45 dB peak |
| | Rd | — | **1.3 Ω** | ≈ √(L_leak / C_total), critical damping of the 9.5 kHz LC resonance |
| | Cd | — | **10 µF** | Large enough to pass DC/low-freq without loading the bus; blocks Rd's DC dissipation |
| **CMC (L17/L18)** | Self-inductance / winding | 3 mH | **3 mH** (unchanged) | Already gives good leakage inductance and CM blocking |
| | Winding R | 3 mΩ | **3 mΩ** (unchanged) | Low-loss, fine as-is |
| | Coupling coefficient k | 0.99625 | **0.99625** (unchanged) | DM leakage of 22.5 µH is already well-placed |
| | Parasitic C / winding | 3 pF | **1 pF** | Tighter winding/potting pushes CMC's own SRF from ~27 MHz → **~47.5 MHz**, extending useful attenuation further into the CISPR band before rollover |
| **Output cluster** | C13 | 3 µF / 50 mΩ / 2 nH | **3 µF / 20 mΩ / 2 nH** | Lower ESR deepens the ~2 MHz secondary dip without changing its frequency |
| | C14 | 4.7 µF / 50 mΩ / 25 nH | **10 µF / 5 mΩ / 25 nH** | Larger C + much lower ESR → deeper, better-placed primary notch |
| | C15 | 4.7 µF / 50 mΩ / 25 nH | **10 µF / 5 mΩ / 25 nH** | Identical to C14, paired |
| | C14 ∥ C15 combined | 9.4 µF / 25 mΩ / 12.5 nH, f_res ≈ 417 kHz | **20 µF / 2.5 mΩ / 12.5 nH, f_res ≈ 318 kHz** | Centers the primary notch inside the 150–500 kHz LLC band (was at the edge); ESR drop turns a shallow ~−88 dB notch into a much deeper one |

**Net effect of these changes:**
- The 9.5 kHz peak is critically damped instead of spiking +45 dB.
- The primary notch moves from ~417 kHz (shallow, off-center) to ~318 kHz (deep, centered in the 150–500 kHz LLC band).
- CMC self-resonance moves from ~27 MHz to ~47.5 MHz, buying ~20 MHz more useful attenuation.
- The remaining 47.5–108 MHz gap (FM band) is closed with a conventional supplementary ferrite bead + feedthrough cap rather than trying to solve it purely with lumped LC values.
