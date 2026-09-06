# Effective Potential Inference and First-Passage Statistics
MSci Mathematics project.
Two notebooks presented (I split my work across ~6 total and picked these).
Input for the project was noisy positional data of simulated particle trajectories. This data was used to infer a potential curve which was used for running highly parallel simulations in order to estimate long-time statistics (such as first-passage times, and autocorrelations).

> **Note on code quality:** This is research-grade code from an active project, lightly tidied for presentability and readability. Beware dead-ends, repeated function definitions in different sections, etc. This code was never intended to be published since the analysis is all based off from a private set of initial data, but it may find use in acting as a small portfolio of mine.

## The problem
Input data is a set of centre-of-mass, wall-normal positional data, generated independently by my supervisor using lattice-Boltzmann (LBM) simulations of deformable red blood cell meshes under shear flow. Simulations were ran in an infinite parallel-plate geometry, where we are interested in “margination” events. Stiff blood constituents (in our case, stiff red blood cells, representing diseased cells [e.g. malaria, sickle cell anaemia, etc.]) have a tendency to stick close to the confining boundary(ies) of the flow they are a member of, and when in this state, they are called “marginated” - when away from the cell-dense core at the centre of the blood flow. In this vein (pardon the pun), we define a margination event as a particle transitioning from the cell-dense core of the flow (an unmarginated state) to a marginated state. In the case of our special geometry, we have two distinct marginated states - next to either one of the two parallel plates, and we are interested in the dynamics of how particles might switch from one marginated state to another - as such, we consider such occurrences to be margination events too. Generally, blood constituents in marginated states can be dangerous in the human body, as they interfere with the cell-free layer and are prone to causing blockages, which is why such margination phenomena are important to us.

Large and computationally expensive models of blood flow exist and are the industry standard, but we have a unique take on the problem. Since industry models are expensive, could we abstract the many hydrodynamic collisions, wall-cell and wall-wall interactions, etc. into a simpler “noise” term? These two notebooks are part of my investigation into the problem.

The input data is collected across a parameter grid of channel width $w$, and capillary number $Ca$:
* $w \in {2, 3, 4, 5, 6, 8, 10}$
* $Ca \in $
