# Effective Potential Inference and First-Passage Statistics
MSci Mathematics project.
Two notebooks presented (I split my work across ~6 total and picked the two included in this repo).
Input for the project was noisy positional data of simulated particle trajectories. This data was used to infer a potential curve which was used for running highly parallel simulations in order to estimate long-time statistics (first-passage times, and autocorrelations).

> **Note on code quality:** This is research-grade code from my master’s project, lightly tidied for presentation and readability. Beware dead-ends, repeated function definitions in different sections, potential missing .csv, .npz, etc. files, and other issues. This code was never intended to be published since the analysis is all based off from a private set of initial data, but it may find use in acting as a small portfolio of mine.

> **Note on running the code:** Don’t. I haven’t written this code to be run by third parties, so save yourself the trouble and don’t attempt to run it, as it will otherwise require heavy adaptation. The input data is private and the notebook which was used for initial processing is not included. The only notable exception to this is the GPU simulation code, which will only require the respective “launcher” function to be adapted, and given a reasonable potential function with associated tolerances. <ins>*Since the notebooks will not execute E2E as published, they are committed with their outputs intact, so that the figures and results can be read on GitHub without running.*</ins>
>
> I ran the code on Google Colab, and switched between CPU and GPU runtimes as necessary. The GPU kernels rely on an NVIDIA GPU (I used Numba CUDA), and there is no CPU fallback.

## The problem
Input data is a set of centre-of-mass, wall-normal positional data, generated independently by my supervisor using lattice-Boltzmann (LBM) simulations of deformable red blood cell meshes under shear flow. Simulations were ran in an infinite parallel-plate geometry, where we are interested in “margination” events. Stiff blood constituents (in our case, stiff red blood cells, representing diseased cells [e.g. malaria, sickle cell anaemia, etc.]) have a tendency to stick close to the confining boundary(ies) of the flow they are a member of, and when in this state, they are called “marginated” - when away from the cell-dense core at the centre of the blood flow. In this vein (pardon the pun), we define a margination event as a particle transitioning from the cell-dense core of the flow (an unmarginated state) to a marginated state. In the case of our special geometry, we have two distinct marginated states - next to either one of the two parallel plates, and we are interested in the dynamics of how particles might switch from one marginated state to another - as such, we consider such occurrences to be margination events too. Generally, blood constituents in marginated states can be dangerous in the human body, as they interfere with the cell-free layer and are prone to causing blockages, which is why such margination phenomena are important to us.

Large and computationally expensive models of blood flow exist and are the industry standard, but we have a unique take on the problem. Since industry models are expensive, could we abstract the many hydrodynamic collisions, wall-cell and wall-wall interactions, etc. into a simpler “noise” term? LBM simulations are extremely expensive and when margination events are rare, computability within a reasonable time-frame becomes a real bottleneck - a cheap surrogate stochastic model can go a long way if done properly; I presented my initial ground-work towards such a goal. The following two notebooks are part of my investigation into the problem.

The input data is collected across a parameter grid of channel width $w$, and capillary number $Ca$:
* $w \in \\{2, 3, 4, 5, 6, 8, 10\\}$
* $Ca \in \\{ 0.05, 0.1, 0.2, 0.4, 0.8\\}$

for a total of 30 combinations. Each distribution of particles is roughly bimodal or trimodal (transitions between these states will be the margination events we are interested in), and we assume a sufficiently smooth steady state probability density function (pdf) exists, and due to symmetry in the channel geometry and forces present in the original LBM simulation, we also assume the pdf will be symmetric. The reason for the latter assumption is that the spatial domain is not fully explored by our particles; we simply have incomplete data, so we have to be a bit creative, and symmetry allows us to effectively double the number of observations we have.

## Approach
### 1. Histogram creation and symmetrisation (not shown)
This work is not found in these notebooks, as it is quite simple and required a good bit of experimentation (for instance, to determine how many bins we should use), and the associated Jupyter notebook is not fit for public presentation. I chose to use 4096 bins and used an initial round of Gaussian smoothing, only to discover it was not sufficient - thus I used a cubic spline approximation instead. The original data was collapsed halfway along the channel and re-expanded out into a full height channel again; the spline approximation was intentionally created on this object, as opposed to the simple collapsed data to ensure the the spline approximation was well-behaved about the channel centre-line.

### 2. Effective potential construction
We inverted the smooth pdf approximation via the Boltzmann relation

$$V(x) = -\log \left( p(x) \right)$$

and we need the derivative in order to integrate a path along it during the simulations (which is why we needed to smooth aggressively, since the logarithm and differentiation are roughening operations)

$$F(x) = -\frac{dV}{dx}.$$

### 3. Fitting the noise amplitude
The inversion only determines the potential $V(x)$ up to a scale factor which determines the barrier height between different basins. It turns out that by requiring the pdf be constant (this requirement is a must-have since the model is informed by the distributions), we inherently lock $\varepsilon = 1$, but we allow the fitting of $\varepsilon$ anyway, because as it allows us to recover the sharpness of data which was lost during smoothing operations!

Measuring the recovery of peaks was attempted with multiple loss functions; $L^1, L^2, L^\infty, \log L^2$, as well as Wasserstein-1 distance, but the usual $L^2$ norm did the best job in recovering sharpness; for instance Wasserstein prioritised the tails of the distribution too heavily.

Fitted splines are written out per $(w, Ca)$, with (x, density, potential, force, eps).

### 4. Kramers' time
Basins (wells) and barriers are located from the spline, with associated basin boundaries which are geometrically motivated by the curvature of the spline. Kramers' estimates were mostly used as a reference point, and were compared against transitions times directly extracted from the data and/or GPU simulated data later in the notebooks.

Some jargon was adopted as the project developed. For a 3-basin potential (or equivalently, a 3-peak pdf) the terminology "outer-to-outer" (OtO), "outer-to-inner" (OtI) and "inner-to-outer" (ItO) was used to refer to transitions between wells. ItO transitions refer to a transition from the central basin to either of the left or right basins, and vice-versa with OtI transitions. OtO originally was intended to count transitions between either outer region to either outer region - that is - $\\{ L, R \\} \to \\{ L, R\\}$, but later rebranded it to mean "outer-to-other outer", or "one outer-to-other outer" to record $L\to R$ and $R\to L$ transitions, and Re-margination (ReMarg) to record $L\to C\to L$ and $R\to C\to R$ transition cycles.

### 5. Simulation of stochastic surrogate model
This is the part of the notebooks that is the most runnable. The remainder of the project heavily relies on private data and is mostly for shown as a sort of personal portfolio, rather than functional code.

An Euler-Maruyama integrator (equivalent to Milstein in the ver1, ver2, cloud integrators) was written as a numba.cuda kernel, which evolves thousands of independent walkers in the fitted potential, with reflective boundaries, and linear interpolations between a tabulated force grid (derived from the spline approximations). I believe rewriting the respective launcher function will be enough to run highly parallel simulations on native potentials, provided they are well-behaved enough. Do note, however, that ver1 is completely obsolete and was only left in for reproducibility purposes, as I'm not sure how random number generation is handled when host and device communicate. Similarly for when devices are changed, so I've left in which GPUs were used and when.

The ”general” integrator assumes a model of the form:

$$
dX_t = -A(X_t)V\\, \prime (X_t)dt + \left(2\varepsilon B(X_t)\right)^{1/2}dB_t
$$

where $A(x)$ is fully determined by $B(x)$. This is because we require the original pdf $p(x)$ to remain unchanged under this new framework - a very natural requirement for our data-informed model. For this integrator, we also need to parse a sufficiently smooth function $B(x)$ if we wish to run it. The next steps for the project were functional calibration, but this sadly ended up outside of the project scope. For those interested, the relation is:

$A(x) = B(x) - \frac{\varepsilon}{V\\,\prime(x)}B\\,\prime(x)$.

The notebooks also include some functions which I used to calculate autocorrelations for the particle trajectories via the Wiener-Khinchin fast Fourier transform (FFT) method. Originally I intended to run this on GPU too, but Numba does not have a native FFT method/function, and the closest equivalent was CuPy, but the two libraries are not (at least were not, at the time) compatible. I also had a brief timestep stability study, since the timesteps used in simulation varied by orders of magnitude.

## Data
The raw LBM simulation data is not included. It is large, and not mine to redistribute. The related particle distributions are likewise not included. A dedicated party may be able to mine some useful information, but much meaningful structure has been destroyed, and the only immediately recognisable structure is the metastability of the system, which is widely accessible information.

What *is* included in data/ are small tables which the notebooks depend on and would otherwise have to be pasted inline. They are the fitted noise amplitudes, integration timesteps, well positions and barrier tolerances, and the resulting mean first-passage times (MFPTs), all keyed by $(w, Ca)$. Scalar tables are stored as .csv, and tables with several fields per key are stored as .npz.

## Known Issues
Aside from the notebooks not generally being fit for running.
* A few helper functions are defined more than once across the notebooks
* The original GPU simulation function (ver1) still exists, despite being obsolete [left in for reproducibility]
* $w = 2$ may be present in some results, but not others. It was excluded from much of the analysis due to it lacking a well-resolved distribution.
* Some cells have commented out approaches, which were abandoned
* When uploading the .ipynb files, some of the markdown/inline math used in the graph creation seems to have corrupted in an unusual way, and some symbols have been replaced with their LaTeX equivalents; e.g. “\infty” -> [LaTeX infinity symbol].

## Author
Kalin Mihaylov - MSci Mathematics, UCL, 2026.

Supervised by Dr Freya Bull.
