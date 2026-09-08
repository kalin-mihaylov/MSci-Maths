# Effective Potential Inference and First-Passage Statistics
MSci Mathematics project.
Two notebooks presented (I split my work across ~6 total and picked these).
Input for the project was noisy positional data of simulated particle trajectories. This data was used to infer a potential curve which was used for running highly parallel simulations in order to estimate long-time statistics (such as first-passage times, and autocorrelations).

> **Note on code quality:** This is research-grade code from an active project, lightly tidied for presentability and readability. Beware dead-ends, repeated function definitions in different sections, potential missing .csv, .npz, etc. files, and other issues. This code was never intended to be published since the analysis is all based off from a private set of initial data, but it may find use in acting as a small portfolio of mine.

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

$V(x) = -\log \left( p(x) \right)$

and we need the derivative in order to integrate a path along it during the simulations (which is why we needed to smooth aggressively, since the logarithm and differentiation are roughening operations)

$F(x) = -\frac{dV}{dx}.$

### 3. Fitting the noise amplitude
The inversion only determines the potential $V(x)$ up to a scale factor which determines the barrier height between different basins. It turns out that by requiring the pdf be constant (this requirement is a must-have since the model is informed by the distributions), we inherently lock $\varepsilon = 1$, but we allow the fitting of $\varepsilon$ anyway, because as it allows us to recover the sharpness of data which was lost during smoothing operations!

Measuring the recovery of peaks was attempted with multiple loss functions; $L^1, L^2, L^\infty, \log L^2$, as well as Wasserstein-1 distance, but the usual $L^2$ norm did the best job in recovering sharpness; for instance Wasserstein prioritised the tails of the distribution too heavily.

Fitted splines are written out per $(w, Ca)$, with (x, density, potential, force, eps).

### Kramers' time
Basins (wells) and barriers are located from the spline, with associated basin boundaries which are geometrically motivated by the curvature of the spline. Kramers' estimates were mostly used as a reference point, and were compared against transitions times directly extracted from the data and/or GPU simulated data later in the notebooks.
