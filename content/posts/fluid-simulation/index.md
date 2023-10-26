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


## Interactive demo

...

{{< admonition tip >}} Use left click to add pigment, and right click to alter the flow. {{< /admonition >}}

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

This demo is based on NVidia's GPU Gem {{< cite "gpu-gems" >}}, chapter [38](https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-38-fast-fluid-dynamics-simulation-gpu).

...


$$ \frac{\partial \mathbf{u}}{\partial t} = -\left(\mathbf{u} \cdot \nabla\right) \mathbf{u} - \frac{1}{\rho} \nabla p + \nu \nabla^2 \mathbf{u} + \mathbf{F} $$

$$ \nabla \cdot \mathbf{u} = 0 $$

...


## References

{{< bibliography >}}
