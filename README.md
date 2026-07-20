# Bob-s-Very-Important-Delivery
A Monte Carlo Random Walk for a Radiation Shielding Problem

**The question:** How much lead shielding is needed so that no more than 0.1% of gamma photons from a Caesium-137 source escape?

**The finding:** The standard textbook answer (the Beer-Lambert attenuation law) says 5.53 cm. A Monte Carlo simulation of one million photons says 8.73 cm, roughly 58% more. The discrepancy is not a bug; it is a real physical effect called **buildup**, and this project identifies it, explains it mechanistically, and confirms the explanation with a controlled falsification test. Trusting the textbook number would leak about 1.2% of photons, twelve times the safety target.

![Simulated leak fraction vs. Beer-Lambert prediction on a log scale](images/theory_vs_simulation.png)

This is a model validation story: a simulation disagrees with a trusted benchmark, and the analysis has to determine whether the model is broken or the benchmark is incomplete. That workflow (validate against a baseline, diagnose the structured disagreement, test the proposed explanation) is the same one used to sanity-check any predictive model, and it is the reason this project exists in a data science portfolio.

## The Analysis in Four Steps

### 1. Establish the theoretical baseline
The Beer-Lambert law predicts exponential attenuation, I(t) = I₀e^(−μt). Using a linear attenuation coefficient of μ = 1.25 cm⁻¹ for lead at 0.6617 MeV (derived in Appendix I from the NIST XCOM Photon Cross Sections Database, the peer-reviewed standard for radiation physics), the required thickness for 0.1% transmission is **5.53 cm**.

### 2. Build the simulation
Photons are modeled as individual 1D random walkers: exponentially distributed step lengths (the mean free path), random forward/backward scattering, and a per-collision absorption probability. Both parameters are derived from the same NIST data, not invented, which makes the simulation directly comparable to theory. The walk is fully vectorized in NumPy, advancing up to one million photons simultaneously across a grid of 100 shielding thicknesses. Built-in conservation checks assert that every photon ends the simulation either leaked, back-leaked, or absorbed.

### 3. Quantify the disagreement with regression
Rather than reading an answer off a plot, a linear regression is fit to the log of the leak fraction, where the Beer-Lambert law predicts a straight line with slope −μ. The result:

| Quantity | Theory | Simulation |
|---|---|---|
| Attenuation rate μ (cm⁻¹) | 1.25 | 0.787 ± 0.001 |
| Required thickness (cm) | 5.53 | 8.73 |
| Fit quality (R²) | n/a | 0.9999 |

The simulation follows a clean exponential (R² = 0.9999) but at a different rate, a gap hundreds of standard errors wide. A bug would be unlikely to produce such an orderly disagreement; the structure of the failure is itself a clue.

### 4. Explain the gap, then try to break the explanation
The Beer-Lambert law describes narrow-beam attenuation: it writes a photon off the moment it interacts in any way. But at this energy, only ~39% of collisions in lead absorb the photon; the rest merely redirect it, and some scattered photons still escape out the far side. The simulation counts them; the formula does not. Radiation shielding theory calls this **buildup**.

To falsify-test this explanation, the simulation is rerun with the absorption probability set to 1, eliminating scattering entirely. If buildup is the cause, the simulation should collapse back onto the narrow-beam law. It does: the fitted rate returns to 1.24 ± 0.01 cm⁻¹ against the theoretical 1.25 cm⁻¹, and the required thickness returns to the analytical 5.53 cm. The gap was physics, not a bug.

## What This Project Demonstrates

- **Model validation against a known baseline**, and, more importantly, what to do when validation fails in a structured way
- **Stochastic simulation at scale**: vectorized NumPy random walks, up to 10⁸ photon-thickness trials per run, with a fixed seed for reproducibility
- **Regression for inference**: extracting a physical rate constant and its standard error from log-transformed data, including an explicit discussion of heteroscedastic residuals and why an unweighted fit is acceptable here
- **Uncertainty quantification**: binomial standard errors and 95% confidence bands on every simulated point, used to establish that the theory-simulation gap is systematic rather than sampling noise
- **Grounded parameters**: all physical inputs derived from the NIST XCOM database rather than assumed
- **Honest scoping**: the model's simplifications (1D geometry, energy-independent cross sections) are stated up front, and Appendix III outlines how each could be relaxed

## Tools

Python (NumPy, pandas, SciPy, Matplotlib), Jupyter. No external simulation libraries; the Monte Carlo engine is built from scratch.

## Running the Notebook

```
pip install numpy pandas scipy matplotlib jupyter
jupyter notebook Random_Walk.ipynb
```

Run all cells top to bottom. The random seed is fixed (seed=42), but the generator's state advances with every cell that draws random numbers, so exact reproduction requires a fresh top-to-bottom run. The main million-photon sweep takes less than 1 minute on a typical laptop.

## Limitations and Extensions

The model is a deliberate simplification, chosen so results could be checked against the 1D analytical law:

- **1D geometry.** Photons move forward/backward only. Extending to 3D (cardinal directions or a fully isotropic walk) would be more physically realistic at the cost of the clean analytical comparison.
- **Energy-independent cross sections.** Each Compton scatter should reduce photon energy and shift both the mean free path and absorption probability; the model holds both fixed at the initial 0.6617 MeV values. Making them energy-dependent is the most physically meaningful upgrade.
- **Importance sampling.** Biasing the walk toward rare deep-penetration paths (with likelihood-ratio reweighting) would cut variance dramatically and let the deep tail be estimated with far fewer photons.
