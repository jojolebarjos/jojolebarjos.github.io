---
title: "Fluid simulation"
date: 2023-07-27T15:28:39+02:00
tags:
- js
- opengl
- shader
description: A minimalistic implementation of Navier-Stokes equations on a 2D discrete grid, using WebGL.
---

<!--more-->


<style>
  .figure {
    width: 50%;
    margin-left: auto;
    margin-right: auto;
    padding-left: 0;
    padding-right: 0;
  }

  .checkbox {
    display: inline-block;
  }

  #canvas {
    display: block;
    width: 100%;
    height: 100%;
    border: 4px solid #909092;
    touch-action: none;
    background: repeating-linear-gradient(
      -45deg,
      transparent,
      transparent 10px,
      #f00 10px,
      #f00 20px
    );
  }
</style>

<script src="main.js"></script>

<!--more-->


## Context

A decade ago, as part of a computer graphics course, we investigated [General-Purpose Computing on Graphics Processing Units (GPGPU)](https://en.wikipedia.org/wiki/General-purpose_computing_on_graphics_processing_units), applied to fluid dynamics.
This assignment revisited the early days of GPGPU, where people utilized textures to store grid data and fragment shaders for parallel computation, before the introduction of dedicated compute shaders and buffer objects.

Our focus was a classic numerical analysis challenge: solving the [Navier-Stokes equations](https://en.wikipedia.org/wiki/Navier%E2%80%93Stokes_equations) iteratively using numerical approximations. These equations, fundamental in fluid dynamics, describe the motion of an incompressible fluid. In our 2D simulation, we visualized this fluid movement by advecting dye within it. Notably, simulating the boundaries of the fluid (a.k.a. [free surface](https://en.wikipedia.org/wiki/Free_surface)), which is a complex task, was outside the scope of our experiment.

In our exploration, we drew inspiration from NVidia's GPU Gem {{< cite "gpu-gems" >}}, specifically [chapter 38](https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-38-fast-fluid-dynamics-simulation-gpu), which discusses fast and stable fluid simulations entirely on the GPU.
This chapter, while focusing on the practical implementation of fluid dynamics on the GPU, also shares about related challenges.
For those interested by further details, the authors suggest Griebel et al. {{< cite "griebel" >}} book.

Interestingly, [WebGL](https://en.wikipedia.org/wiki/WebGL), the technology I used for this blog post, does not support compute shaders, which makes the approach of using textures for grid data and fragment shaders for parallel computation still relevant nowadays.


## Interactive demo

Without further ado, here is a minimalistic WebGL demo.
The code is available on [this GitHub repository](https://github.com/jojolebarjos/fluid-simulation).

{{< admonition tip >}} Use left click to add pigment, and right click to alter the flow. On touchscreen, use the checkbox to swap mode instead. {{< /admonition >}}

<div class="figure">
  <label class="checkbox">
    <input type="checkbox" id="render-checkbox">
    <span class="checkmark">Raw render mode</span>
  </label>
  <label class="checkbox">
    <input type="checkbox" id="control-checkbox">
    <span class="checkmark">Swap control mode</span>
  </label>
  <canvas id="canvas" width="480" height="480">
    Your browser does not support the HTML5 canvas tag.
  </canvas>
</div>

<script>
  const canvas = document.getElementById("canvas");
  const renderCheckbox = document.getElementById("render-checkbox");
  const controlCheckbox = document.getElementById("control-checkbox");
  const simulation = new Simulation(canvas);

  // Prevent context menu
  canvas.addEventListener("contextmenu", (e) => {
    e.preventDefault();
  });

  // Convert coordinates
  function getOffsetLocation(e) {
    return {
      x: e.offsetX * canvas.width / canvas.clientWidth,
      y: (canvas.clientHeight - e.offsetY) * canvas.height / canvas.clientHeight
    };
  }

  // Start stroke, with selected mode
  canvas.addEventListener("pointerdown", (e) => {
    e.preventDefault();
    simulation.cursorMode = -1;

    // Primary button
    if (event.pointerType == "touch" || e.button == 0) {
      simulation.cursorMode = 1 ^ controlCheckbox.checked;
    }

    // Secondary button
    else if (e.button == 2) {
      simulation.cursorMode = 0 ^ controlCheckbox.checked;
    }

    simulation.cursorCurrentLocation = getOffsetLocation(e);
  });

  // Update stroke
  canvas.addEventListener("pointermove", (e) => {
    e.preventDefault();
    if (simulation.cursorMode >= 0) {
      simulation.cursorCurrentLocation = getOffsetLocation(e);
    }
  });

  // End stroke
  canvas.addEventListener("pointerup", (e) => {
    e.preventDefault();
    simulation.cursorMode = -1;
    simulation.cursorCurrentLocation = null;
    simulation.cursorPreviousLocation = null;
  });

  // Toggle render mode
  renderCheckbox.addEventListener("click", (e) => {
      simulation.renderMode = renderCheckbox.checked ? 1 : 0;
  });

  // Simulation step
  // Note: using `requestAnimationFrame` does not guarantee a known fixed rate
  function update() {
    simulation.update();
    window.requestAnimationFrame(update);
  }

  // Start the simulation and rendering loop
  update();

  // Manually call this to break the WebGL context
  // simulation.gl.getExtension("WEBGL_lose_context").loseContext();
</script>


## Concepts

As mentioned above, the Navier-Stokes equations for incompressible flow are at the core of the problem.
The first equation describes how the fluid's velocity, $\mathbf{u}$, changes over time:

$$ \frac{\partial \mathbf{u}}{\partial t} = -\left(\mathbf{u} \cdot \nabla\right) \mathbf{u} - \frac{1}{\rho} \nabla p + \nu \nabla^2 \mathbf{u} + \mathbf{F} $$

This combines four components:

 1. [_Advection_](https://en.wikipedia.org/wiki/Advection) is the process by which the velocity of a fluid transports properties like heat, mass, or, in the case of this simulation, dye, along with the flow.
    Velocity itself is also advected, causing the fluid to carry and reshape its own velocity as it moves; $-\left(\mathbf{u} \cdot \nabla\right) \mathbf{u}$ is the self-advection term.
 2. [_Pressure_](https://en.wikipedia.org/wiki/Pressure) is the force per unit area exerted by the molecules within a fluid; $- \frac{1}{\rho} \nabla p$ is the pressure term.
 3. [_Viscosity_](https://en.wikipedia.org/wiki/Viscosity) is a measure of a fluid's resistance to flow, determining how easily it deforms under shear or tensile stress; $\nu \nabla^2 \mathbf{u}$ is the diffusion term.
 4. Additional _forces_ may also be applied to specific regions of the fluid; $\mathbf{F}$ is the external force term.

The second equation describes the incompressible nature of the fluid, where the density remains constant:

$$ \nabla \cdot \mathbf{u} = 0 $$

Together, these equations form the backbone of the simulation.
The techniques described in GPU Gems are based on the stable fluids method {{< cite "stam" >}}.
As the name implies, this method emphasizes numerical stability, ensuring some level of consistency over time.
Its efficiency and simplicity make it suitable for real-time applications.

A key realization presented in GPU Gems is that both equations can be handled separately, according to [Helmholtz's theorem](https://en.wikipedia.org/wiki/Helmholtz_decomposition):

 1. The first equation can be applied as-is, to update the velocity according to advection, pressure, viscosity, and external forces.
    The resulting velocity field likely to have nonzero divergence.
 2. This field can be corrected by subtracting the gradient of the pressure field, which can be computed by solving a [Poisson equation](https://en.wikipedia.org/wiki/Poisson%27s_equation); here an iterative approach is applied.

All that's left is to discretize the space as a 2D grid and implement each step as a shader.
For more details, please refer to the well-written chapter in GPU Gems.


## References

{{< bibliography >}}
