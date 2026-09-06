# SoC Estimation of a Li-ion NMC (LG) Cell using an Extended Kalman Filter

Software-in-the-Loop (SIL) implementation of an **Extended Kalman Filter (EKF)** that estimates the State of Charge (SoC) of a lithium-ion NMC cell under **dynamic load**, validated against a Batemo high-fidelity battery plant model.

---

## 1. Why an EKF? (The Core Idea)

Every practical SoC method fails on its own:

| Method | Principle | Weakness |
|---|---|---|
| **OCV look-up** | Map rest voltage → SoC via the OCV–SoC curve | Only valid after a long rest; useless while current flows |
| **Coulomb counting (CC)** | Integrate current over time | Open-loop. Drifts forever, because sensor bias and the unknown initial SoC are never corrected |
| **EKF (this work)** | Fuse a battery **model** (prediction) with a **voltage measurement** (correction) | Needs a decent model and honest noise covariances |

The EKF is not "a better formula". It is an **arbiter**: at every time step it decides *how much to trust the model versus the measurement*, through the Kalman gain. Coulomb counting supplies the prediction; terminal-voltage feedback through the OCV–SoC relation supplies the correction. That is the whole project in one sentence.

**TL;DR** — Build a 2nd-order Thevenin (2-RC) equivalent circuit model of the cell, extract its parameters as functions of SoC from HPPC tests, discretise the model with Zero-Order Hold, linearise the output equation (`dVoc/dSoC`), and run an EKF in Simulink against a Batemo plant. Accuracy is reported as RMSE on a randomised current profile.

---

## 2. Method Overview

The work was carried out in three stages:

1. **Model study** — derive the state-space representation of the cell and identify what the EKF actually needs from it.
2. **Parameter extraction** — run HPPC (Hybrid Pulse Power Characterisation) tests, fit the ECM parameters, and build the OCV–SoC curve.
3. **Performance test** — run the SIL loop and evaluate against the plant model.

---

## 3. Battery Plant and State-Space Model

The cell is represented by a **2-RC Thevenin equivalent circuit model (ECM)**. States are the SoC and the two RC-branch voltages:

$$x = \begin{bmatrix} SoC \\ V_1 \\ V_2 \end{bmatrix}$$

Discretised with **Zero-Order Hold** over a sampling period $\Delta t$:

$$
\begin{bmatrix} SoC_{k+1} \\ V_{1,k+1} \\ V_{2,k+1} \end{bmatrix}
=
\begin{bmatrix}
1 & 0 & 0 \\
0 & e^{-\Delta t / \tau_1} & 0 \\
0 & 0 & e^{-\Delta t / \tau_2}
\end{bmatrix}
\begin{bmatrix} SoC_k \\ V_{1,k} \\ V_{2,k} \end{bmatrix}
+
\begin{bmatrix}
-\dfrac{\eta \Delta t}{C_n} \\
R_1\left(1 - e^{-\Delta t / \tau_1}\right) \\
R_2\left(1 - e^{-\Delta t / \tau_2}\right)
\end{bmatrix} I_k
$$

with $\tau_i = R_i C_i$. The measurement (output) equation is:

$$V_{t,k} = OCV(SoC_k) - V_{1,k} - V_{2,k} - R_0 I_k$$

Because $OCV(SoC)$ is **non-linear**, the output matrix must be linearised at each step — this is precisely what makes the filter *extended* rather than plain linear Kalman:

$$C_k = \frac{\partial V_t}{\partial x}\bigg|_{x = \hat{x}_k} = \begin{bmatrix} \dfrac{\partial OCV}{\partial SoC}\bigg|_{\hat{SoC}_k} & -1 & -1 \end{bmatrix}$$

> The state-space formulation and the EKF tuning strategy follow Farhad et al., *Scientific Reports* (2024) — see references.

📷 *State-space and EKF equations:* `docs/state_space_equations.png`

---

## 4. HPPC Parameter Extraction

**HPPC (Hybrid Pulse Power Characterisation)** applies discharge/charge current pulses at fixed SoC break-points and observes the voltage response. The response is then decomposed:

- The **instantaneous** voltage jump at pulse onset → $R_0$ (ohmic resistance)
- The **exponential relaxation** afterwards → $R_1, C_1$ (fast, charge-transfer) and $R_2, C_2$ (slow, diffusion)

ECM parameters are genuinely functions of **both SoC and temperature**. In principle the HPPC campaign must therefore be repeated across a temperature grid.

> ⚠️ **Scope limitation of this project:** only the **SoC dependence** was characterised. Temperature was held constant, so the parameter set is *not* valid outside the tested thermal condition. Extending the look-up tables to a 2-D (SoC, T) grid is the most obvious next step.

Parameters were fitted by comparing the analytical pulse-response curve against the measured HPPC response and selecting the parameter set that minimised the mismatch. The extraction procedure and the fitting equations follow Tran et al., *Batteries* (2021).

📷 *HPPC current/voltage profile:* `docs/hppc_profile.png`
📷 *Extracted parameters vs. SoC:* `docs/ecm_parameters.png`

---

## 5. OCV–SoC Characterisation

The EKF needs not only $OCV(SoC)$ for the output equation but also its **derivative** $\partial OCV / \partial SoC$ for the linearised $C_k$ matrix. The curve was obtained from low-rate charge/discharge data and fitted so that the derivative remains smooth — a noisy derivative directly corrupts the Kalman gain.

📷 *OCV–SoC curve:* `docs/ocv_soc_curve.png`

---

## 6. Filter Tuning (Q and R)

Plugging in the model is **not sufficient**. The filter only works once the covariances express an honest belief about uncertainty:

- **Q** — process noise covariance: how much the model is distrusted (current-sensor bias, ECM simplification, ageing).
- **R** — measurement noise covariance: how much the voltage sensor is distrusted.
- **P₀** — initial state covariance: how wrong the initial SoC guess may be.

Practical intuition: a **large Q / small R** makes the filter chase the voltage measurement (fast convergence, noisy estimate); the reverse makes it lean on Coulomb counting (smooth, slow to recover from a wrong initial SoC). Tuning is the trade-off between these two failure modes.

---

## 7. Software-in-the-Loop Setup

```
                ┌──────────────────────┐
   Random       │  Batemo Battery      │  V_terminal (measured)
   current ────▶│  Plant Model         │────────────┐
   profile      └──────────────────────┘            │
        │                                            ▼
        │                                  ┌──────────────────┐
        └─────────────────────────────────▶│  EKF Estimator   │──▶ SoC_est
                                           │  (2-RC ECM)      │
                                           └──────────────────┘
                                                    │
   SoC_plant ───────────────────────────────────────┴──▶ error ──▶ RMSE
```

The estimated SoC is compared **directly** against the plant's internal SoC, which is the advantage of SIL: ground truth is available, which is never the case on real hardware.

📷 *Simulink SIL block diagram:* `docs/sil_diagram.png`

---

## 8. Test Profile and Results

Validation uses a **randomised dynamic current profile** — deliberately not a constant-current discharge, since the point of the EKF is to survive load transients.

Performance metric:

$$RMSE = \sqrt{\frac{1}{N}\sum_{k=1}^{N}\left(SoC_{plant,k} - \hat{SoC}_k\right)^2}$$

📷 *Random current profile:* `docs/current_profile.png`
📷 *SoC estimation vs. plant + error:* `docs/results_rmse.png`

---

## 9. Repository

```
SOC-ESTIMATION-BY-EKF/
├── docs/          # figures used in this README
├── data/          # HPPC and OCV datasets
├── models/        # Simulink SIL model
└── scripts/       # parameter extraction & post-processing
```

**Repository:** https://github.com/Fadhillahusein/SOC-ESTIMATION-BY-EKF

---

## 10. Known Limitations / Roadmap

- [ ] Parameters characterised at a single temperature — extend to a 2-D (SoC, T) look-up.
- [ ] No hysteresis state in the model; NMC hysteresis is mild but not zero.
- [ ] No ageing / capacity-fade adaptation — a **joint** or **dual** EKF could co-estimate $C_n$.
- [ ] Move from SIL to Hardware-in-the-Loop, then to the embedded target.

---

## References

1. Ospina Agudelo, B. et al. *Advancing state estimation for lithium-ion batteries with hysteresis through systematic extended Kalman filter tuning.* **Scientific Reports**, 2024. https://www.nature.com/articles/s41598-024-61596-0
2. Tran, M.-K. et al. *Comparative Study of Equivalent Circuit Models Performance in Four Common Lithium-Ion Batteries: LFP, NMC, LMO, NCA.* **Batteries** 7(3):51, 2021. https://www.mdpi.com/2313-0105/7/3/51
3. Plett, G. L. *Battery Management Systems, Volume II: Equivalent-Circuit Methods.* Artech House, 2015. — the standard reference for ECM-based EKF SoC estimation.
4. Batemo Cell Library — plant model source. https://www.batemo.com/products/batemo-cell-library/
5. Idaho National Laboratory. *Battery Test Manual for Electric Vehicles (Rev. 3)*, 2015 — the origin of the HPPC procedure. https://inldigitallibrary.inl.gov/sites/sti/sti/6492291.pdf
