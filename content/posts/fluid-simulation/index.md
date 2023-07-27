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
  #canvas {
    padding-left: 0;
    padding-right: 0;
    margin-left: auto;
    margin-right: auto;
    display: block;
    width: 480px;
    height: 480px;
    border: 4px solid #909092;
  }
</style>

<script src="main.js"></script>

<!--more-->


## Interactive demo

...

{{< admonition tip >}} Use right click to add pigment, and left click to alter the flow. <kbd>M</kbd> can be used to switch rendering mode. {{< /admonition >}}

<canvas id="canvas" width="480" height="480">
  Your browser does not support the HTML5 canvas tag.
</canvas>

<script>
  simulate(document.getElementById("canvas"));
</script>


## Concepts

This demo is based on NVidia's GPU Gem {{< cite "gpu-gems" >}}, chapter [38](https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-38-fast-fluid-dynamics-simulation-gpu).

...


$$ \frac{\partial \mathbf{u}}{\partial t} = -\left(\mathbf{u} \cdot \nabla\right) \mathbf{u} - \frac{1}{\rho} \nabla p + \nu \nabla^2 \mathbf{u} + \mathbf{F} $$

$$ \nabla \cdot \mathbf{u} = 0 $$

...


## References

{{< bibliography >}}
