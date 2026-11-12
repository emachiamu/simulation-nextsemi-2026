# Does a 1×10⁻¹⁵ @ 1 s chip-scale atomic clock matter, and where?

Technical-validation simulation for the TVA Labs submission to the NextSEMI Hackathon (SEMICON Europa × TUM Venture Labs, Munich, 10–12 Nov 2026), imec ventures challenge *"Build the next timing unicorn"*.

**The challenge assumption.** A chip-scale atomic clock (CSAC) with every published spec of the Microchip SA65 (17 cm³, 120 mW, ±3×10⁻¹⁰ over −40…+80 °C, ageing 3×10⁻¹¹/day) **except** short-term stability, which is assumed to be **ADEV < 1×10⁻¹⁵ at τ = 1 s** (the SA65 is 3×10⁻¹⁰).

**The question this code answers.** For which applications does that one number change what is physically possible, and for which does it not (because another noise source, or a cheaper clock, already suffices)?

Everything here is Python (NumPy + Matplotlib). One command regenerates every number and figure in about 25 seconds, and a self-check compares the headline numbers with this README.

---

## 1. Result in one table

All values are simulation outputs (`[SIM]`) under the assumptions in §6. "Chip" is the challenge clock. "HSO" is the best quartz oscillator we modelled (6×10⁻¹⁴ @ 1 s). "Rb" is a lab rubidium reference (2×10⁻¹¹ @ 1 s). Two independent, free-running clocks are synchronised once at t = 0, then left alone.

| Metric | Chip | HSO (best quartz) | Rb | Produced by |
|---|---|---|---|---|
| Time two free-running clocks stay within 10° at **300 GHz** (lab environment) | **31.6 s** | 1 s | < 0.1 s | `scenarios.py` (B) |
| Radar image, 300 GHz, 60 s aperture, no shared reference: peak loss | **−0.01 dB** | −4.3 dB (blurred) | −18 dB | `poc_phases.py` (P1) |
| Same at 150 GHz | −0.00 dB | −1.3 dB | −18 dB | `poc_phases.py` (P1) |
| 16-node coherent radar, sync link cut: longest outage keeping ≥ 90 % coherent gain, X-band (10 GHz) | **100 s** | 30 s | < 0.1 s | `poc_phases.py` (P2) |
| Same at W-band (94 GHz) | **30 s** | 3 s | < 0.1 s | `poc_phases.py` (P2) |
| One-way Doppler range-rate error (spacecraft clock vs ground maser, 60 s) | 0.004 mm/s | 0.019 mm/s | n/a | `scenarios.py` (C) |

**Where the clock is the limiting factor:** high carrier frequency × long coherence window × no shared reference. That is sub-THz coherent measurement/links (140–300 GHz), Ka-band distributed SAR, and radar networks whose sync link can be denied.

**Minimum clock performance needed** (`out/H_min_clock.csv`, ADEV @ 1 s, clock-only requirement):

| Application | Needs ADEV(1 s) ≤ |
|---|---|
| Ka-band distributed InSAR, 60 s, 1° | 1.8×10⁻¹⁵ |
| Sub-THz sounding at 300 GHz, re-reference every 60 s, 10° | 2.1×10⁻¹⁵ |
| C-band (Harmony-class) InSAR, 120 s, 1° | 5.7×10⁻¹⁵ |
| Sub-THz sounding at 140 GHz, 60 s, 10° | 4.5×10⁻¹⁵ |
| Sub-THz inter-satellite link at 300 GHz, sync every 10 s, 0.1 rad | 7.3×10⁻¹⁵ |
| Sub-THz inter-satellite link at 140 GHz, sync every 10 s, 0.1 rad | 1.6×10⁻¹⁴ |
| One-way Doppler parity with two-way tracking (0.1 mm/s @ 60 s) | **3.3×10⁻¹³: a USO-class clock suffices; the chip adds only size/power** |

The requirements for the first six rows lie within a factor of about 2–16 of the assumed 10⁻¹⁵, so the margin is thin. That is why the unknown noise floor above 1 s matters (§7).

**Where the clock is *not* the limit** (also in the outputs, not hidden):
- One-way deep-space Doppler needs only 3.3×10⁻¹³ (above).
- At 150 GHz and a 10 s aperture, even the HSO loses only 0.04 dB.
- Indoor TDOA positioning (`P3a`): going from 1 ns to 100 ps of anchor sync improves a 100 MHz-class ranging system only from 0.31 m to 0.23 m median error. Ranging noise, not the clock, dominates. Better sync only helps once ranging is ~1 GHz-class (0.23 m → 0.03 m).
- At 300 GHz, antenna-position error (10 µm → 3.6°, 50 µm → 18°) and cable/extender drift (0.1–5°) are the same size as the chip's own clock term (`out/R_redteam_checks.json`, R2).
- If the real noise floor is ≥ 10⁻¹³ (100× the assumed spec) nothing survives (§7).

---

## 2. Run it yourself

Requirements: Python ≥ 3.11 (tested on 3.12; developed on 3.14), NumPy ≥ 2.0, Matplotlib ≥ 3.8. No network, GPU, or data download needed.

```bash
git clone https://github.com/emachiamu/simulation-nextsemi-2026.git && cd simulation-nextsemi-2026
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
PYTHON=.venv/bin/python ./run_all.sh
```

`run_all.sh` runs every script (≈ 25 s in total on a laptop), rewrites everything in `out/`, and ends with `check_results.py`, which prints 18 `OK`/`FAIL` lines comparing the headline numbers with this README and exits non-zero if any is off. Expected last line: `18/18 checks passed`.

Run one script alone with `.venv/bin/python scenarios.py` (**from the repo root**: scripts write to `./out/`). The committed `out/` already contains the results, so you can read them without running anything.

**Reproducibility.** Every random generator is seeded, so reruns are deterministic. On the author's machine (macOS arm64, Python 3.12.15, NumPy 2.5.4, Matplotlib 3.11.2) a rerun reproduces the committed outputs, including the PNGs, byte for byte. Other platforms or library versions may differ in the last digits. Coherence times are reported on a logarithmic grid with steps of about 1.47×, so one grid step either way is not a discrepancy, and `check_results.py` allows for that.

**Try a what-if (≈ 5 s):** edit `phi_B` or the carrier tuple in the "B" block of `scenarios.py`, or change a clock's numbers in `clock_noise.py` (`CLOCKS`), then run `python floor_envelope.py`.

---

## 3. What is in the repo

| File | What it does | Writes to `out/` |
|---|---|---|
| `clock_noise.py` | **Clock noise generator** and catalogue of clocks. Generates fractional-frequency time series (white FM, flicker FM, random-walk FM, ageing, temperature, magnetic terms) and **validates itself** by measuring the Allan deviation of what it generated. `python clock_noise.py` prints the validation. | `validation_adev.csv` (when called from `scenarios.py`) |
| `coherence.py` | **The physics core.** Phase error between two independent clocks after a sync event (§4). Imported by the others. | none |
| `scenarios.py` | Scenarios A–E and H, and the two overview plots (below). | `A_…`, `B_…`, `C_…`, `D_…`, `E_…`, `H_…csv`, `F_fT_map.png`, `G_coherence_vs_carrier.png`, `summary.json` |
| `hybrid_gnss.py` | Tests whether GNSS carrier-phase sync (515 fs) plus the chip as a smoothing "flywheel" beats GNSS alone at Ka-band. | `I_hybrid_gnss_ka.txt` |
| `poc_phases.py` | Three application proofs-of-concept: P1 radar image blur, P2 sync-outage tolerance of a 16-node radar plus fibre-vs-module economics, P3 indoor positioning / anchor holdover / relay-attack detection. | `P1_…`, `P2_…`, `P3a/b/c_…`, `poc_summary.json` |
| `redteam_checks.py` | Adversarial checks: **R1** P10/P50/P90 bands on the P1 result, **R2** the non-clock phase budget at 300 GHz, **R3** a bottom-up revenue model (all inputs assumed). | `R_redteam_checks.json` |
| `floor_envelope.py` | **J:** what the 1 s spec does *not* fix. Bounds the damage from unknown noise above 1 s (random-walk FM, unestimated drift, poor thermal compensation, and spec-violating floors of 10⁻¹⁴ and 10⁻¹³). | `J_floor_envelope.csv` |
| `navigator.py` | Scores five opportunities on seven criteria, with evidence text. **Judgement scores, not a simulation** (§8). | `navigator.csv` |
| `check_results.py`, `run_all.sh` | Self-check and one-command runner. | none |

Scenarios in `scenarios.py`: **A** baseline sub-THz inter-satellite links; **B** untethered sub-THz channel sounding; **C** one-way deep-space Doppler; **D** distributed SAR interferometry; **E** sensitivity of the chip result to each unknown (flicker floor, thermal compensation, ageing estimation, magnetic shielding); **H** minimum clock each application needs; **F** and **G** are plots (required stability vs window; coherence time vs carrier frequency).

---

## 4. How it works

### 4.1 The governing law
A coherent system tolerates a phase error φ at carrier frequency f over a window T only if the clock's relative stability satisfies roughly

> σ_y ≳ φ / (2π · f · T)

For 0.1 rad at 300 GHz over 60 s that is about 9×10⁻¹⁶, which is why the assumed 10⁻¹⁵ is interesting at sub-THz, and why the same number is irrelevant at 10 MHz or 1 s windows. Plot `F_fT_map.png` shows this against the clocks. The simulation exists to replace this order-of-magnitude rule with a calculation that includes estimation, drift and environment.

### 4.2 Clock model (`clock_noise.py`)
Each clock has an Allan deviation

σ_y(τ) = √( a² / τ + f² + b² · τ )

with `a` the white-FM level (1 s intercept), `f` a flat flicker-FM floor, `b` random-walk FM (default 0). On top come deterministic terms: linear ageing (per day), temperature α_T·(1−κ_T)·ΔT(t) and magnetic α_B·(1−κ_B)·ΔB(t), where κ is the fraction removed by compensation or shielding.

The challenge fixes ADEV only at **τ = 1 s**, so the behaviour at other τ is *our* parameterisation, never given. The chip is modelled as white FM 7×10⁻¹⁶ plus a flicker floor 7×10⁻¹⁶, which gives 9.9×10⁻¹⁶ at 1 s (meets the spec). The flicker floor is the most sensitive unknown and is swept in E and J.

Time series are generated in the time domain (Gaussian white noise; 1/f noise by spectral shaping; random walk by cumulative sum). The amplitudes are **calibrated by measuring the overlapping Allan deviation of the generated noise**, so the generator is checked rather than trusted (§5).

### 4.3 Coherence between two independent clocks (`coherence.py`)
1. Two clocks, each with independent noise, start in phase at t = 0 (sync error is added separately where relevant).
2. The relative frequency is **estimated causally** from the preceding window of equal length T: (phase difference over that window) / T.
3. Over the next window T the residual time error r(t) = x(t) − x(0) − ŷ·t is accumulated.
4. The metric is the RMS of 2π·f·r(t) in radians or degrees. It is averaged (as RMS) over 20–60 Monte-Carlo realisations.
5. **Coherence time** = the longest T on a logarithmic grid (0.1 s … ~30 000 s, 34 points) for which that RMS stays below the tolerance.

Environment profiles (all `[ASM]`, `coherence.ENVS`): `lab` (0.5 K / 30 min thermal cycle, 1 mG, magnetic shielding ×100, κ_T = 0.99), `leo` (1 K per 95-min orbit, 0.15 G, shielding ×1000, κ_T = 0.99), `deep` (slow thermal, negligible field), `ideal` (no environment, perfect ageing estimation). Residual ageing after estimation is 1 % of the SA65 value unless a scenario overrides it.

### 4.4 Application metrics
- **Radar image blur (P1):** the phase error r(t)·2πf is applied across the aperture; the image is the FFT of exp(jφ). *Peak loss* compares the image peak with the ideal one; *ISLR* is the energy outside ±1 resolution cell over the energy inside.
- **Network coherent gain (P2):** N nodes each hold their own clock during a sync outage T; gain = |Σ exp(jφᵢ)|² / N². The outage tolerated is the longest T with mean gain ≥ 0.9.
- **Doppler (C):** relative ADEV at 60 s between spacecraft clock and ground maser, times c.
- **Minimum clock (H):** bisection on a clock level L (white = floor = L/√2) until the RMS phase just meets the tolerance, with environment terms switched off, so it is the *clock-only* requirement.
- **GNSS + flywheel (I):** GNSS sync noise of 515 fs (white), smoothed over a window W with the clock as the interpolator.
- **TDOA positioning (P3a):** least-squares solution with simulated ranging noise plus anchor-sync error.

---

## 5. Validation

1. **Generator vs target.** `out/validation_adev.csv` (37 rows): generated ADEV / target ADEV lies between **0.927 and 1.066** for every clock, τ = 1…1000 s, and record lengths 256…8192 samples.
2. **External validation against measured data.** The model with a CSAC-class spacecraft clock gives a one-way Doppler range-rate error of **11.5 mm/s**; JPL measured **11 mm/s** with the Iris radio and an SA.45s CSAC (NAVIGATION 71(1); see §6). This test checks the model structure, but it exercises the CSAC's *poor* short-term noise, not the 10⁻¹⁵ regime.
3. **Analytic cross-check** (rms time error over T ≈ σ_y·T·√2 for two clocks) at 300 GHz, 60 s:
   - HSO: 6×10⁻¹⁴ × 60 s × √2 = 5.1 ps → 9.6 rad. Simulation: 487° = 8.5 rad. Same order.
   - Chip: 7×10⁻¹⁶ × 60 s × √2 = 59 fs → 6.4°. Simulation with no environment (`ideal`): **5.6°**. With the lab environment the simulation gives **19.6°**, so in the lab case the temperature residual, not the clock noise, dominates the chip's error.
4. **Random-walk-FM generator** is checked separately in `floor_envelope.py` (ratio to target printed on run).
5. **Uncertainty bands** (`redteam_checks.py`, R1, 200 runs, 300 GHz): chip peak loss P50 −0.01 dB (60 s) and −0.74 dB (300 s, P10 −4.3 dB); HSO P50 −4.1 dB (60 s) and −9.4 dB (300 s).

**Bug log (kept deliberately).**
- *12 Nov 2026, 04:20.* The flicker-noise amplitude scaled with record length (≈ 1/n) but was calibrated only at n = 4096, which was also the only length validated. Scenarios using 512–2048 samples had 2–8× too much flicker noise. Fixed by calibrating per record length (`clock_noise._flicker_cal`); validation now sweeps n = 256…8192. Effect: chip results nearly unchanged (environment-limited); quartz results got *better* (HSO at 300 GHz 0.46 s → 1 s; X-band outage 3 s → 30 s). The Doppler validation (n = 4096) was unaffected.
- *12 Nov 2026, repo preparation.* `H_min_clock.csv` and `J_floor_envelope.csv` were written without CSV quoting, so labels containing commas split into extra columns. Fixed with the `csv` module. No number changed.

---

## 6. Inputs by provenance

`[CHAL]` = given by the challenge. `[SRC]` = from a public source. `[ASM]` = our engineering assumption. `[INT]` = from an interview (not publicly checkable).

| Input | Value | Tag |
|---|---|---|
| Challenge chip ADEV @ 1 s | 1×10⁻¹⁵ (modelled as white 7×10⁻¹⁶ + flicker floor 7×10⁻¹⁶) | `[CHAL]`; the split and floor are `[ASM]`, swept in E and J |
| Chip ageing, temperature, magnetic sensitivity | 3×10⁻¹¹/day; 5×10⁻¹²/K (from ±3×10⁻¹⁰ over 120 K); 9×10⁻¹¹/G | `[CHAL]` = SA65 sell sheet |
| SA65 ADEV | 3×10⁻¹⁰/√τ, τ = 1…1000 s | `[SRC]` SA65 sell sheet |
| Two-way Doppler reference | 0.1 mm/s @ 60 s | `[SRC]` Iris/CSAC paper, NAVIGATION 71(1) |
| One-way Doppler with SA.45s (validation target) | 11 mm/s | `[SRC]` same paper |
| GNSS-based SAR phase sync | 515 fs (1σ) ≈ 1° @ 5.4 GHz | `[SRC]` DLR, IEEE TGRS 2025 |
| "HSO" quartz oscillator | 6×10⁻¹⁴ @ 1 s, 50 ppt/day, flat floor | `[INT]` manufacturer quote; **not independently verifiable, treat as `[ASM]`** |
| Lab rubidium | 2×10⁻¹¹ @ 1 s, floor 10⁻¹³ | `[ASM]` (PRS10 class; verify against a datasheet) |
| Space USO | ≈ 10⁻¹³ flat | `[ASM]` (typical deep-space USO) |
| Ground H-maser | 10⁻¹³/√τ, floor 7×10⁻¹⁶ | `[ASM]` |
| TCXO, standard OCXO | 10⁻¹⁰ flat; 3×10⁻¹³ / 10⁻¹² | `[ASM]` |
| Environments (lab, LEO, deep), κ_T, shielding | see §4.3 | `[ASM]` |
| Tolerances | 0.1 rad (beam combining), 10° (virtual array), 1° (InSAR) | `[ASM]` (InSAR "few degrees": TanDEM-X) |
| Positioner, cable drift, air-path terms (R2) | 10–50 µm; 0.1–5°; 3×10⁻⁷ per unit | `[ASM]` |

Every output of this repo is `[SIM]`. Anything that depends mainly on an `[ASM]` row should be read as "if this assumption holds", and E and J show how fast each conclusion degrades when it does not.

---

## 7. What would change the conclusion

`out/J_floor_envelope.csv` (300 GHz, 10°, lab; coherence time and 60 s image loss):

| Variant | Coherence time | Image loss, 60 s |
|---|---|---|
| V0 current model (flicker 7×10⁻¹⁶) | 31.6 s | −0.01 dB |
| V1 worst spec-consistent: random-walk FM so that ADEV(1 s) is still 10⁻¹⁵ but rises as √τ | 21.5 s | −0.02 dB |
| V2 = V1 and ageing not estimated | 14.7 s | −0.07 dB |
| V3 = V2 and poor thermal compensation (κ_T = 0.9) | 10 s | −0.15 dB |
| V4 flicker floor 10⁻¹⁴ (**violates the spec by 10×**) | 6.8 s | −0.16 dB |
| V5 floor 10⁻¹³ (**violates the spec by 100×**) | 0.68 s | −5.8 dB |
| HSO (reference) | 1.5 s | −3.8 dB |

Reading: the worst spec-consistent noise above 1 s (V1) costs a factor 1.5 in coherence time; adding unestimated ageing and poor thermal compensation (V2, V3) costs up to 3×, and the advantage over quartz survives (10 s vs 1.5 s). A 10× spec violation (V4) still leaves a lead of about 4.6× at 300 GHz, while at 100× (V5) the advantage is gone. A flat flicker floor is bounded by the 1 s spec itself; random-walk FM and drift are not, which is why V1–V3 exist. Scenario E adds: without thermal compensation coherence drops 10× (31.6 → 3.2 s), without lab magnetic shielding 3×, and with a 10⁻¹⁴ floor about 5×. These are engineering requirements for the product, not physics limits.

---

## 8. Limitations (read before relying on a number)

- **Validity range.** The model is valid for T ≥ 0.1 s. A real CSAC's internal VCXO and the phase noise added by a frequency-multiplier chain (+20·log₁₀ N; roughly +89.5 dB to reach 300 GHz) are not modelled.
- **Two image/phase metrics, different strictness.** The coherence time uses the full RMS phase error. The image *peak loss* does not penalise a linear phase ramp (a residual frequency offset only shifts the image). For the chip at 300 GHz / 60 s in the lab, 18.6° of RMS phase error is mostly such a ramp (1.6° remains after removing it), while the HSO retains 89°. So the image-blur results are the more lenient view; the 31.6 s coherence time is the stricter one.
- **Coarse grid.** Coherence times step by ≈ 1.47×.
- **Assumed environments.** Sinusoidal temperature and magnetic profiles with assumed amplitudes. No vibration or acceleration sensitivity, no relativistic or orbit-determination error, no multipath.
- **Noise above 1 s is a parameterisation, not a measurement** (see §7).
- **Not physics:** `navigator.py` scores, the revenue model (`redteam_checks.py` R3) and the fibre-vs-module cost table (`P2_scale_economics.csv`) are judgement- and assumption-driven. They are there to be inspected and changed, not validated by this simulation. The 1B€-outcome question in R3 depends entirely on its adoption assumptions.
- **Not modelled:** continuous clock disciplining, hardware calibration drift, and GNSS noise other than white (hybrid result assumes white GNSS noise; multipath would set a floor).
- **Baseline inter-satellite links** (navigator opportunity 0) are clock-feasible (7×10⁻¹⁵ needed at 300 GHz) but need ~16 µm line-of-sight knowledge, far below the mm-class orbit determination we assume (`[ASM]`), so the non-clock term dominates there.

---

## 9. Context and licence

This repository is the simulation part of a venture submission; the pitch, interview record and market research are not included here. The code is self-contained and does not read any external data. Code and outputs: MIT (see `LICENSE`).
