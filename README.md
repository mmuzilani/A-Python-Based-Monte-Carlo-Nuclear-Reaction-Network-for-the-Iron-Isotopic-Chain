# A Python-Based Monte-Carlo Nuclear Reaction Network for the Iron Isotopic Chain

Neutron-capture reactions are central to the production of heavy elements in stellar environments, particularly through the slow neutron-capture process (s-process). During this process, seed nuclei gradually capture neutrons and undergo β-decays, forming increasingly heavier isotopes. Accurate modeling of these nucleosynthesis pathways requires reliable nuclear physics inputs: energy-dependent neutron-capture cross sections, Maxwellian-Averaged Cross Sections (MACS), reaction rates , and β-decay constants. Because experimental cross-section data are limited for many isotopes, theoretical nuclear reaction codes such as **TALYS** are widely used to generate complete sets of, which can then be validated and refined using databases like **KADoNiS** or **EXFOR**.

The iron isotopic chain provides a particularly important test case due to its astrophysical relevance and combination of stable and radioactive isotopes. In this project, we model the full neutron-capture chain by integrating theoretical cross sections, experimental MACS, and decay information into a unified computational framework.

This project implements a complete, end-to-end workflow:

- Generation of TALYS neutron-capture cross sections 
- Computation of Maxwellian-Averaged Cross Sections through numerical integration
- Validation of TALYS MACS against experimental data
- Monte-Carlo propagation of cross-section uncertainties
- Numerical solution of the ODE system using matrix exponential (Schur decomposition)
- Abundance evolution and uncertainty quantification
  
Together, these steps produce realistic nucleosynthesis predictions and provide insight into how nuclear uncertainties affect the formation of heavy isotopes in stars. The workflow mirrors modern computational methods used in nuclear astrophysics research.

---
                         ┌─────────────────────────┐
                         │ TALYS σ(E) Cross Section│
                         │    (4 CSV files)        │
                         └────────────┬────────────┘
                                      │
                                      ▼
                     ┌────────────────────────────────┐
                     │ 1) Compute MACS(kT) from σ(E)  │
                     │    (Code: compute_macs)        │
                     └────────────────┬───────────────┘
                                      │
                                      ▼
                        ┌─────────────────────────────────┐
                        │ Compare TALYS MACS vs Experiment│
                        │   (Code: compare_macs)          │
                        └──────────────┬──────────────────┘
                                       │
                                       ▼
                     ┌────────────────────────────────────┐
                     │ Plot Maxwellian Integrand          │
                     │   (Code: plot_integrand)           │
                     └─────────────────┬──────────────────┘
                                       │
                                       ▼
                   ┌───────────────────────────────────────────┐
                   │ Monte-Carlo MACS Uncertainty (±10%)       │
                   │  (Code: mc_macs_uncertainty)              │
                   └─────────────────────┬─────────────────────┘
                                         │
                                         ▼
      ┌──────────────────────────────────────────────────────────────────┐
      │ Build Reaction Network (5 isotopes)                              │
      │ Compute λ = nₙ⟨σv⟩ and λβ                                        │
      │ Solve ODE via Schur-Decomposition (200 Monte-Carlo runs)         │
      │ Plot abundance evolution + uncertainty bands                     │
      │ (Code: full_network_solver)                                      │
      └──────────────────────────────────────────────────────────────────┘
---

## Nuclear Reaction Data

In stellar environments, heavy elements are built through a sequence of neutron-capture reactions and β-decays.  
To model such nucleosynthesis, we require reliable nuclear physics input.  
The foundation of all calculations is the neutron-capture reaction:

$$
(A,Z) + n \rightarrow (A+1,Z) + \gamma ,
$$

characterized by an energy-dependent probability called the **cross section**, $\sigma(E)$.

Since physical conditions inside stars vary with temperature, neutron energies follow statistical distributions.  
Therefore, raw $\sigma(E)$ must be transformed into astrophysical quantities such as **MACS** and **reaction rates**.

We focuses on preparing these essential nuclear inputs:

- Energy-dependent cross section σ(E)  
- Maxwellian-Averaged Cross Section (MACS)  
- β-decay half-life data  

---

## Neutron-Capture Cross Section σ(E)

The cross section $\sigma(E)$ measures the probability that a nucleus will capture a neutron of energy **E**.

- **Unit:** barn (1 b = 10⁻²⁸ m²)  
- **Behavior:** varies strongly with energy and shows resonance peaks  
- **Importance:** determines how fast isotopes transform during nucleosynthesis  

For each Fe isotope in the chain (56→57→58→59→60), TALYS provides:

```
Energy(MeV), CrossSection(b)
```

This σ(E) dataset is the **most fundamental input**, because *all other astrophysical quantities are derived from it*.

---

## Maxwellian-Averaged Cross Section (MACS)

The **MACS** is the average of σ(E) over a Maxwell–Boltzmann distribution:

$$
MACS(kT)=
\frac{2}{\sqrt{\pi} (kT)^2}
\int_{0}^{\infty}
\sigma(E)\, E\, e^{-E/kT}\, dE .
$$

### Meaning of MACS

- It is the **astrophysical cross section**, relevant for stellar interiors  
- It smooths out resonance peaks of σ(E)  
- It depends on temperature  

**Unit:** barn  
MACS is essential because stars are thermal environments — σ(E) alone cannot describe reaction rates.

---

## Experimental MACS (KADoNiS / EXFOR)

TALYS gives **theoretical** σ(E). But stars follow **real nuclear physics**, so we need experimental MACS from:

- **KADoNiS** (s-process database)  
- **EXFOR** (global reaction data library)

I prepare **four experimental MACS files**, one for each neutron-capture step:

- ⁵⁶Fe(n,γ) → ⁵⁷Fe  
- ⁵⁷Fe(n,γ) → ⁵⁸Fe  
- ⁵⁸Fe(n,γ) → ⁵⁹Fe  
- ⁵⁹Fe(n,γ) → 60Fe 

These experimental values allow **validation of TALYS output**.
---

## Relation between MACS and ⟨σv⟩

For neutron captures, we use a special identity:

$$
\langle \sigma v \rangle = v_T \, MACS(kT),
$$

where
- **MACS** captures the energy dependence (σ(E) and Maxwellian weight)
- **v_T** adds the thermal velocity scale

Together, they produce a proper stellar reaction rate.

**Units of ⟨σv⟩:**  m³ s⁻¹

---

## From Reaction Rate to Capture Rate λ

Stars contain an enormous number of free neutrons.  
If neutron density is \( n_n \), the **capture rate** per nucleus is:

$$
\lambda_{\text{cap}} = n_n \, \langle \sigma v \rangle .
$$

This λ is the **probability per second** that a nucleus captures a neutron.

Interpretation:

- λ = 1 s⁻¹ → reaction happens once per second  
- λ = 10⁻⁶ s⁻¹ → reaction takes ≈ 10⁶ s to occur  

In my project, λ is computed for four Fe captures:

- ⁵⁶Fe(n,γ) → ⁵⁷Fe  
- ⁵⁷Fe(n,γ) → ⁵⁸Fe  
- ⁵⁸Fe(n,γ) → ⁵⁹Fe  
- ⁵⁹Fe(n,γ) → 60Fe  

These λ values form the backbone of the reaction network.

---

## Constructing the 5-Isotope Nuclear Reaction Network

The isotopes form a sequential chain:

$$
^{56}Fe \rightarrow {}^{57}Fe \rightarrow {}^{58}Fe \rightarrow {}^{59}Fe \rightarrow {}^{60}Fe.
$$

## Matrix Form of the Network (for Fast Numerical Solving)

We can rewrite the system in **matrix form**:

$$
\frac{d\mathbf{Y}}{dt} = A \mathbf{Y},
$$

where **A** is the 5×5 rate matrix containing λ values.

Example structure:

$$
A =
\begin{pmatrix}
-\lambda_0 & 0 & 0 & 0 & 0 \\
\lambda_0  & -\lambda_1 & 0 & 0 & 0 \\
0          & \lambda_1  & -\lambda_2 & 0 & 0 \\
0          & 0          & \lambda_2  & -(\lambda_3 + \lambda_{\beta,3}) & 0 \\
0          & 0          & 0          & \lambda_3                        & -\lambda_{\beta,4}
\end{pmatrix}
$$

The solution is:

$$
Y(t) = e^{At} \, Y(0).
$$

SciPy computes this using:

- Schur decomposition  
- Matrix exponential expm()  
- Smooth time sampling  

This yields the abundance curves.
---

# Outcome Of this project.

## Maxwellian Integrand Plot — Checking the MACS Calculation

To compute MACS, the weighted integrand is:

$$
I(E) = \sigma(E)\,E\, e^{-E/kT}.
$$

Plotting this function shows:

- Which neutron energies contribute most to the MACS  
- Whether your interpolation and integration are stable  
- Whether the σ(E) dataset is realistic  

### Interpretation of the Plot

- **X-axis:** neutron energy (keV)  
- **Y-axis:** integrand value \( I(E) \) (arbitrary units)  
- Peak location → “effective energy window”  
- Width → energy spread contributing to the MACS  

For s-process temperatures (kT = 30 keV), the peak usually lies around 20–60 keV.

This plot is a **scientific validation** of MACS.

![Image](https://github.com/user-attachments/assets/13bb9409-720a-4d55-b43b-ae79f2a828be)


---

## MACS Comparison — TALYS vs Experimental

To assess accuracy, we compare:

- **TALYS MACS** (theory)  
- **KADoNiS/EXFOR MACS** (experiment)  

These values are plotted together as:

- Points for TALYS  
- Points for experiment  
- Text labels indicating percentage difference  

![Image](https://github.com/user-attachments/assets/c9a519ac-e283-43ce-b31c-1caeae30a12e)

## Interpretation of Monte-Carlo MACS Distribution Plots

These plots show the Monte-Carlo probability distributions of the Maxwellian-Averaged Cross Section (MACS) for different Fe isotopes at stellar temperatures kT = 10, 30, and 60 keV.
Each plot corresponds to one isotope (⁵⁹Fe, ⁵⁷Fe, ⁵⁸Fe) and one temperature.

1. Histogram (Green Bars) Shows how the MACS value changes when random ±10% uncertainties are applied to TALYS cross sections σ(E).
This represents the probability density of getting a certain MACS value.

2. A smoothed probability curve that shows the underlying distribution.

3. Red Dashed Line: The mean MACS computed from all Monte-Carlo samples.

4. Pink Shaded Region (±1σ Band): The 68% confidence interval, also called the 16th–84th percentile range.

### Results
- MACS Distributions Are Approximately Gaussian
- All histograms are smooth and slightly symmetric → the uncertainty propagation is stable.
- Higher Temperature = Lower MACS. This is physically correct because at higher temperature, neutrons are faster → lower probability of capture.
- Uncertainty (Width of Histogram) becomes smaller at high kT
- MACS becomes less sensitive to σ(E) uncertainties at high temperature.
- Each isotope responds differently\
  ⁵⁷Fe(n,γ) shows the largest spread → largest uncertainty.\
  ⁵⁸Fe(n,γ) and ⁵⁹Fe(n,γ) show tighter distributions → lower uncertainty.

![Image](https://github.com/user-attachments/assets/66106c57-217e-44f2-91d8-3c771a36c879)

![Image](https://github.com/user-attachments/assets/f171e8b0-c086-4d0c-ac74-40f48cc16118)

![Image](https://github.com/user-attachments/assets/ed4bc87d-0669-4772-b6ad-292b25cbe3ae)

![Image](https://github.com/user-attachments/assets/ec58ccb3-9fab-4a9a-8be6-0340cdeb6f5f)

## Monte-Carlo Uncertainty Bands — (16th–84th Percentiles)

Nuclear data has uncertainties (MACS measurement error, model error, etc.).  
To quantify their effect, we perform Monte-Carlo sampling:

$$
MACS_{\text{random}} =
MACS_{\text{mean}} \cdot (1 + \epsilon),
$$

where:

- \(\epsilon\) ~ Gaussian(0, σ)  

Repeating the network simulation many times gives many abundance curves. From all Monte-Carlo solutions:

- **50th percentile** → median curve  
- **16th percentile** → lower uncertainty limit  
- **84th percentile** → upper uncertainty limit  

- **Narrow band:** prediction is robust  
- **Wide band:** uncertainties strongly affect nucleosynthesis  
- **Shifted median:** TALYS may deviate from experiment  

Uncertainty quantification is essential in modern nuclear astrophysics.

![Image](https://github.com/user-attachments/assets/0e34008c-b62a-4f4f-95b6-ba042b566116)
---

# Conclusion

This project successfully demonstrates a complete, end-to-end workflow for constructing a realistic neutron-capture reaction model using modern nuclear-physics tools. Beginning with TALYS-generated cross sections, we computed Maxwellian-Averaged Cross Sections (MACS), validated them against experimental KADoNiS data, quantified uncertainties through Monte-Carlo simulations, and finally propagated those uncertainties into a full 5-isotope Fe reaction network. The combined analysis provides both deterministic and statistical insight into neutron-capture nucleosynthesis.

## Major Outcomes of the Project
```
1. Accurate MACS Computation From TALYS.
   - Developed a robust numerical integrator for Maxwellian folding.
   - Successfully generated MACS values across 10–100 keV for: ⁵⁶Fe(n,γ), ⁵⁷Fe(n,γ), ⁵⁸Fe(n,γ), ⁵⁹Fe(n,γ)
   - Verified integrand behavior to ensure correct Maxwellian weighting.

2. TALYS vs Experimental Validation
  - Compared TALYS MACS against KADoNiS data.
  - Found good agreement for ⁵⁷Fe, ⁵⁸Fe, and ⁵⁹Fe, with reasonable percent deviations.
  - Identified ⁵⁶Fe MACS mismatch, discussed later.

3. Monte-Carlo Uncertainty Quantification
  - Implemented a ±10% σ(E) perturbation model.
  - Generated Gaussian-like distributions for MACS.
  - Extracted mean, standard deviation, and 1σ confidence intervals.
  - Determined that uncertainty decreases at higher stellar temperatures (kT).

4. Reaction Network Solution With Uncertainty Propagation
 - Constructed a realistic 5-isotope chain: ⁵⁶Fe → ⁵⁷Fe → ⁵⁸Fe → ⁵⁹Fe → ⁶⁰Fe
 - Included both neutron-capture rates and β–decay rates.
 - Solved using the numerical matrix exponential (Schur method).
 - Computed median, 16th, and 84th percentiles for each isotope abundance.
 - Produced uncertainty bands showing how MACS variation affects nucleosynthesis.
```

## Key Observations
```
### Temperature Dependence
- MACS decreases as kT increases, as expected from nuclear statistical physics.
- Reaction rates λ ∝ ⟨σv⟩ follow consistent scaling with neutron density.

### Isotope Behavior in the Network
- ⁵⁶Fe decreases monotonically (seed consumed).
- ⁵⁷Fe & ⁵⁸Fe rise and fall as intermediate isotopes.
- ⁵⁹Fe rises but is partially removed by β–decay.
- ⁶⁰Fe accumulates as the final product.

### Monte-Carlo Distributions

- All isotopes show stable Monte-Carlo convergence.
- Broadest uncertainty at low kT (10 keV), narrowest at 60 keV.
- ⁵⁷Fe(n,γ) shows the largest sensitivity to σ(E) variations.
```

### Note on the Inconsistency of ⁵⁶Fe MACS

```
Among all isotopes, 56Fe showed an inconsistent or irregular MACS distribution. Possible reasons include:
- 56Fe has strong resonance peaks at low energies.
- If σ(E) grid spacing is coarse, the Maxwellian integral becomes inaccurate.
- 56Fe is a doubly magic-like nucleus → lower capture probability → small variations in σ(E) dramatically change MACS.
```

Overall, this project successfully reproduces the essential components of an astrophysical neutron-capture study, from raw nuclear data to reaction-network evolution with full uncertainty quantification. The workflow demonstrates how theoretical cross sections, experimental MACS, statistical sampling, and ODE-based nucleosynthesis models all come together to produce physically meaningful predictions.







