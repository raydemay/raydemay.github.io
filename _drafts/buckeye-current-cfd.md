---
layout: single
title: "CFD Analysis of Buckeye Current RW-5"
header:
  teaser: assets/images/default-image.jpeg
categories:
  - Engineering
tags:
  - fluent
  - cfd
  - motorcycle
  - projects
  - cae
  - engineering
---

# Setup

The Ansys Workbench workflow is based on a process that Formula Buckeye has used for their CFD analysis with some changes to mesh settings and solution method.

SOLIDWORKS model saved as .STEP file for Ansys Workbench compatibility

Ansys 2022 R2 and 2023 R2 were both used at different stages in the process

Ansys SpaceClaim utilized to create fluid domain

Enclosure sized to be 5 times the length of bike in each direction with exceptions. The ground plane was set to touch its wheels. The domain behind bike was doubled to allow room for wake.

Geometry was split on the x-y plane to utilize symmetry.

Two smaller domains were created to allow for local sizing of mesh to be more refined closer to model and coarser mesh

# Mesh

Fluent meshing utilized to take advantage of poly-hexcore mosaic mesh

Local sizing was set to 5mm cells near model and 2m cells at the domain boundary

1.2 growth rate was used in all conditions

Boundary layer

4 layers

Aspect ratio of 5

3D mesh

nodes: 6742617

edges: 6516

faces: 16663508

cells: 5128389

Solution

k-omega SST solution method with curvature correction enabled

90 m/s inlet velocity

Ground was set to moving wall with 90 m/s velocity

Wheel zones were set to rotating wall with 315 rad/s speed (90m/s rotational speed with our diameter tires)

Side and top boundaries were set to 0 shear to mimic freestream

Second order coupled solver

# Results

Bike has drag coefficient around 0.2. 1m^2 and 101325 Pa static pressure were used as reference values to get lift and drag coefficient. Lift coefficient is slightly negative, meaning there is some downforce. This is a large improvement from the prior revision of the fairing, which was simulated in March, where it had a lift coefficient of 0.5. I believe part of this is due to the slimming down of the geometry behind the rider.

# Next Steps

Three simulations are planned. The first is to run the same settings but a transient version to see changes in the flow at different time steps. The second is to run the simulation at a lower velocity. The third is to run a simulation with a crosswind component to see the behavior of the air on the back half of the fairing.
