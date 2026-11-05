# Superdense Coding on NISQ Hardware

**Cross-platform benchmarking of the superdense coding (SDC) protocol on a SpinQ Gemini NMR processor and an IBM superconducting processor (`ibm_kingston`).**

This repository holds the analysis notebook behind the paper *"Experimental Analysis of Superdense Coding on NISQ Hardware: Cross-Platform Benchmarking on NMR and Superconducting Quantum Processors"* (Abilash, Abeywaradana, Jayawardhene, Dissanayake, Mahasinghe). The notebook builds the four SDC circuits in Qiskit, simulates them with and without noise, analyzes real hardware counts from both platforms, and regenerates every figure used in the paper.

> **Status:** research code accompanying a manuscript in preparation (target journal: CLEI Electronic Journal; results to be presented at ICCMM 2026).

---

## Contents

- [Background](#background)
- [Headline results](#headline-results)
- [Repository layout](#repository-layout)
- [Quick start](#quick-start)
- [Notebook walkthrough](#notebook-walkthrough)
- [Methods in brief](#methods-in-brief)
- [Outputs and figure map](#outputs-and-figure-map)
- [Running on real IBM Quantum hardware](#running-on-real-ibm-quantum-hardware)
- [Reproducibility notes and caveats](#reproducibility-notes-and-caveats)
- [Security: API keys](#security-api-keys)
- [Citation](#citation)
- [Authors and acknowledgments](#authors-and-acknowledgments)

---

## Background

Superdense coding sends **two classical bits using one transmitted qubit**, provided the sender (Alice) and receiver (Bob) share a Bell pair in advance. Alice applies one of four Pauli operations to her half of the pair; Bob undoes the Bell-state preparation (CNOT, then Hadamard) and measures both qubits.

| Message `b1b2` | Alice's gate | Bell state |
|:---:|:---:|:---:|
| `00` | I | Φ⁺ |
| `01` | X | Ψ⁺ |
| `10` | Z | Φ⁻ |
| `11` | X then Z (ZX) | Ψ⁻ |

On noiseless hardware the channel carries 2 bits per use, double the classical limit of 1 bit. The question this project answers is how much of that advantage survives on real, noisy devices, and where exactly it disappears. The notebook derives that break-even point from the channel's mutual information instead of assuming a rule of thumb.

## Headline results

Every number below is taken from the stored outputs of the notebook, at **N = 8192 shots per message**.

| Configuration | Mean fidelity | MI capacity (bits/use) |
|---|:---:|:---:|
| Ideal simulation | 1.0000 | 2.0000 |
| Generic noisy simulation (depolarizing + readout) | 0.8659 | 1.2287 |
| **SpinQ Gemini, real hardware (NMR)** | **0.9430** | **1.6305** |
| SpinQ + readout error correction (simulation-informed) | 0.9680 | 1.7885 |
| IBM `ibm_kingston`, counts bundled with this notebook | 0.9365 | 1.6033 |

- **Break-even fidelity:** `F* = 0.8107` (mutual-information based). The older per-bit BER rule gives 0.8900 and is reported only for comparison.
- **SpinQ noise fingerprint** (differential-evolution fit to the measured confusion matrix): single-qubit error 0.45 %, CNOT error 2.72 %, readout error 0.51 %, Frobenius residual 0.103. Two-qubit gate error dominates.
- **Gate asymmetry on SpinQ:** the X-encoded messages (`01`, `11`) are about nine points worse than the Z-encoded one, consistent with Z rotations being near-lossless virtual frame updates in NMR.

> ### ⚠️ IBM numbers: notebook vs. paper
> The IBM counts stored in `results/ibm_hardware_counts.json` (and printed by this notebook) give **93.65 % mean fidelity and 1.603 bits/use**. The **final paper reports a different IBM run: 86.0 % mean fidelity and 1.28 bits/use** (per-message fidelities 87.7 / 83.4 / 86.1 / 86.9 %), which the paper attributes to calibration drift between the calibration snapshot and the measurement. The SpinQ numbers match between the notebook and the paper.
>
> To reproduce the paper's IBM figures and tables, replace `results/ibm_hardware_counts.json` with the raw counts from the run reported in the paper (same `{message: {bitstring: count}}` format) and re-run the [Cross-Platform Analysis](#notebook-walkthrough) cells. Do not quote the 93.65 % / 1.603 values as the paper's result.

## Repository layout

```
.
├── SDC_SpinQ_Notebook_FIXED.ipynb   # the full analysis (this is the whole pipeline)
├── README.md
└── results/                         # created automatically on first run
    ├── spinq_hw_results.json        # real SpinQ counts (from cell 8)
    ├── ibm_hardware_counts.json     # real IBM counts (live run, or supply your own)
    ├── ideal_results.json
    ├── noisy_generic.json
    ├── noisy_spinq_sim.json
    ├── rec_results.json
    ├── zne_results.json
    ├── extracted_noise_params.json  # noise-fingerprint fit
    └── plots/                       # all figures, .png and .pdf
```

## Quick start

**Requirements:** Python 3.10+, Jupyter, and the packages below. The notebook does not pin versions, so run `pip freeze` after a successful run if you need a locked environment.

```bash
python -m venv .venv && source .venv/bin/activate
pip install qiskit qiskit-aer numpy scipy matplotlib jupyter
# Only needed for the optional live IBM section:
pip install qiskit-ibm-runtime

jupyter notebook SDC_SpinQ_Notebook_FIXED.ipynb
```

**Offline run (no accounts, no network):** run the cells from the top down through *Final summary report* (cell 20). This covers simulation, the entered SpinQ hardware counts, all metrics, REC, ZNE, noise fingerprinting, and the summary. Then skip the live IBM block and run the *Cross-Platform Analysis* cells at the end, which load IBM counts from `results/ibm_hardware_counts.json`.

**Runtime:** everything is fast except the noise-fingerprinting cell, which runs a differential-evolution optimization (with reduced `maxiter`/`popsize` for practicality). Expect that single cell to dominate the total time.

**Determinism:** simulator runs use `SEED = 42`. Hardware results are fixed inputs, so the analysis is reproducible end to end.

## Notebook walkthrough

| Cell | Purpose |
|---|---|
| 1 | Install dependencies |
| 2 | **Configuration**: shots, messages, seed, SpinQ device parameters (T1/T2, gate times), noise-model parameters, ZNE scale factors |
| 3 | Imports; creates `results/` and `results/plots/` |
| 4 | Builds the four SDC circuits (entangle → encode → decode → measure) |
| 4b | Publication-quality circuit diagrams (`circuit_z`, `circuits_all`) |
| 5 | Ideal (noiseless) statevector simulation |
| 6 | Noise models: generic depolarizing + readout, and a SpinQ-calibrated T1/T2 thermal-relaxation model |
| 7 | Noisy simulations under both models |
| 8 | **Real SpinQ hardware results** (entered as measured probabilities, converted to counts) |
| 9 | Metric functions: fidelity, BER, BER-based capacity, mutual-information capacity |
| 10 | Metrics for every configuration |
| 11 | **Break-even fidelity F\*** (MI-based, primary; BER-based, legacy) |
| 12 | Readout error correction (calibration-matrix inversion) |
| 13 | Zero-noise extrapolation |
| 14 | **Noise fingerprinting** from the 4×4 confusion matrix |
| 15–19 | Figures (SpinQ-only versions are superseded; see below) |
| 20 | Final summary report |
| Optional | Live execution of the same circuits on IBM Quantum hardware |
| Cross-Platform | Loads IBM counts, computes the same metrics, and regenerates the final figures with IBM merged in |

> The final saved figures come from the **Cross-Platform Analysis** cells, which overwrite the SpinQ-only versions. Cells 15–19 now keep only the data they compute and point to the later cells for plotting, so each figure is drawn exactly once, with both platforms.

## Methods in brief

- **Fidelity** of a message is the fraction of shots that land on the correct bitstring, computed directly from raw counts. The same `get_fidelity`, `get_confusion_matrix`, and `get_mi_capacity` functions are used for SpinQ and IBM, so the two platforms are never compared with different code.
- **Channel capacity** is the mutual information between sent and received messages under a uniform prior, computed from the 4×4 confusion matrix. This is the right measure because SDC sends one of four symbols rather than two independent bits. The BER-based capacity is kept only as a legacy comparison.
- **Break-even fidelity F\*** is the fidelity at which the mutual-information capacity of a symmetric 4-ary depolarizing channel equals 1 bit/use (solved with `scipy.optimize.brentq`). It corresponds to a critical depolarizing parameter μ\* ≈ 0.252.
- **Noise models:** a generic depolarizing + readout model, and a SpinQ model built from published T1/T2 and gate times (Hou et al., *EPJ Quantum Technol.* 8, 20, 2021): ¹H T1 = 4 s, T2 = 0.3 s; ³¹P T1 = 7.2 s, T2 = 0.5 s; single-qubit gate about 20 µs; CNOT about 800 µs.
- **Readout error correction:** the 4×4 calibration matrix is built from a *simulated* noise model, so the corrected capacity is a simulation-informed estimate, not a hardware-verified one.
- **Zero-noise extrapolation:** three-point Richardson extrapolation, `F_ZNE = 3F(1×) − 3F(2×) + F(3×)`. The scaling is applied to the **noise-model error probabilities** (1×, 2×, 3×) rather than by physically folding gates, so this is a simulated feasibility demonstration, not an error-mitigated hardware result.
- **Noise fingerprinting:** differential evolution fits a three-parameter model (single-qubit, two-qubit, and readout error) to the measured confusion matrix, using only the four circuits already run.

## Outputs and figure map

Figures are saved to `results/plots/` as both `.png` and `.pdf`.

| File | Content | Paper figure |
|---|---|:---:|
| `circuit_z`, `circuits_all` | Circuit diagrams | Figs. 1–2 |
| `fig1_fidelity` | Per-symbol fidelity, all configurations incl. IBM | Fig. 3 |
| `fig2_confusion` | Confusion matrices (5 panels) | Fig. 4 |
| `fig3_capacity` | Channel capacity, all configurations | Fig. 5 |
| `fig4_breakeven` | Capacity vs. fidelity with F\* | Fig. 6 |
| `fig5_noise_asymmetry` | Fidelity loss and per-symbol trend | Fig. 7 |
| `fig_crossplatform_timescales` | T1/T2/gate-time comparison, log scale | Fig. 8 |

The file numbering inside the notebook (`fig1`–`fig5`) predates the paper's final figure order; use the table above to map between them.

## Running on real IBM Quantum hardware

This block is **optional** and needs your own IBM Quantum Platform account, an API key, and network access. The notebook was run against `ibm_kingston` (156-qubit superconducting processor) on the open plan.

1. Create an API key on the IBM Quantum Platform dashboard.
2. Save it locally once, **without committing it** (see the security note below):
   ```python
   from qiskit_ibm_runtime import QiskitRuntimeService
   QiskitRuntimeService.save_account(channel="ibm_quantum_platform", token="<YOUR_API_KEY>", overwrite=True)
   ```
3. Run the backend-selection cell. It picks the operational, non-simulator backend with the shortest queue. This replaces `least_busy()`, which fails on some open-plan setups.
4. Run the transpile and submit cells. Circuits are transpiled at optimization level 1 to the backend's native gates (RZ, SX, X, CZ) and executed with the `SamplerV2` primitive at 8192 shots per message, with no error mitigation beyond the Sampler's defaults.
5. Read the counts from the `.c` attribute of the result data (the classical register is named `c`). The notebook saves them to `results/ibm_hardware_counts.json`.

Hardware results depend on the day's calibration and queue, so a new run will not reproduce the earlier counts exactly.

## Reproducibility notes and caveats

- **SpinQ counts are transcribed.** The SpinQ interface does not export shot-level counts, so cell 8 holds the measured projection probabilities read from the interface and converts them to counts. This introduces a small transcription uncertainty.
- **Simulation versus hardware are kept separate.** REC and ZNE are simulation-based and are labeled that way throughout; only the raw SpinQ and IBM counts are hardware measurements.
- **IBM results vary run to run.** The paper's IBM run (86.0 %) and the counts bundled here (93.65 %) differ markedly, and the paper's own analysis points to calibration drift. Quoted calibration figures are a loose, time-limited lower bound on error, not a stable predictor of circuit fidelity.
- **IBM qubit choice.** The transpiler picks the physical qubits, so the two circuits may not run on a fixed, matched pair. Sampler defaults such as dynamical decoupling were not separately characterized.
- **No physical channel.** Both qubits stay in the same NMR sample (or on the same chip), so the fidelities exclude the cost of physically transmitting a qubit.
- **TVD is not computed here.** The paper reports a total variation distance of 0.038 between SpinQ and ideal simulation; that metric is not part of this notebook's output.
- **Depolarizing fit.** The scalar fit μ_eff ≈ 0.076 reproduces SpinQ's mean fidelity but cannot capture the X/Z asymmetry; the fingerprint model shares that limitation, as its residual shows.

## Security: API keys

**Never commit an IBM Quantum API key.** Before pushing this repository or sharing the notebook, check that the `save_account(...)` cell contains a placeholder and not a real token. A key that has appeared in a shared file, a chat, or a public repository should be treated as compromised: revoke it on the IBM Quantum Platform dashboard and generate a new one. Prefer saving credentials once through `save_account` on your own machine, or reading the token from an environment variable, and keep any key files out of version control (for example, add `apikey.json` to `.gitignore`).

## Citation

If you use this code or data, please cite the paper (full venue details to be added on publication):

```bibtex
@unpublished{abilash2026sdc,
  title  = {Experimental Analysis of Superdense Coding on {NISQ} Hardware:
            Cross-Platform Benchmarking on {NMR} and Superconducting Quantum Processors},
  author = {Abilash, Sundaresan and Abeywaradana, Udara and Jayawardhene, Vidura
            and Dissanayake, Shakya and Mahasinghe, Anuradha},
  year   = {2026},
  note   = {Manuscript in preparation}
}
```

Key references used by the analysis: Bennett and Wiesner (1992); Mattle et al. (1996); Shadman et al. (2010); Temme, Bravyi and Gambetta (2017); Hou et al. (2021) for SpinQ Gemini; Javadi-Abhari et al. (2024) for Qiskit; Storn and Price (1997) for differential evolution.

## Authors and acknowledgments

- Sundaresan Abilash, Department of Chemistry, University of Colombo
- Udara Abeywaradana, Department of Mathematics, University of Colombo
- Vidura Jayawardhene, Department of Transport Management and Logistics Engineering, University of Moratuwa
- Shakya Dissanayake, Department of Computer Engineering, General Sir John Kotelawala Defence University
- Anuradha Mahasinghe, Department of Mathematics, University of Colombo

The authors thank the Centre for Mathematical Modelling at the University of Colombo for access to the SpinQ Gemini processor and for supporting this work through the Certificate Course in Quantum Computing. They also acknowledge Qiskit and Qiskit Aer, and access to IBM Quantum Platform for the superconducting-hardware results.

## License

No license has been chosen yet. Add a `LICENSE` file (for example MIT for code, CC BY 4.0 for figures and text) before making the repository public.
