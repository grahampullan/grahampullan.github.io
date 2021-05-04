---
layout: post
title:  "Interactive Joukowski airfoil"
date:   2021-05-04 11:53:49 +0100
categories: viz
summary: "Interactive demonstration of the flow around an airfoil shape, using the Joukowski transform"
image: "/assets/images/Joukowski_Airfoil.png"
image_alt: "Image shows streamlines and grid around an airfoil at incidence"
---

The image below is an interactive demonstration of a Joukowski airfoil. Try moving the *flow angle* and *thickness* sliders, and you can apply the [Joukowski transform](https://en.wikipedia.org/wiki/Joukowsky_transform) by checking the box.


<div id="observablehq-viewof-gl-e7cb58e4"></div>
<div id="observablehq-viewof-sliders-e7cb58e4"></div>
<div id="observablehq-viewof-transform-e7cb58e4"></div>

<script type="module">
import {Runtime, Inspector} from "https://cdn.jsdelivr.net/npm/@observablehq/runtime@4/dist/runtime.js";
import define from "https://api.observablehq.com/@grahampullan/joukowski-airfoils.js?v=3";
new Runtime().module(define, name => {
  if (name === "viewof gl") return new Inspector(document.querySelector("#observablehq-viewof-gl-e7cb58e4"));
  if (name === "viewof sliders") return new Inspector(document.querySelector("#observablehq-viewof-sliders-e7cb58e4"));
  if (name === "viewof transform") return new Inspector(document.querySelector("#observablehq-viewof-transform-e7cb58e4"));
  return ["programInfo","render","c","alpha","Gamma","values","initialGrid","grid"].includes(name);
});
</script>

This demo is written in JavaScript and you can see the code in [this Observable notebook](https://observablehq.com/@grahampullan/joukowski-airfoils).


The flow is modelled, using complex potentials, as inviscid and incompressible. The solution for the flow around a cylinder (with imposed circulation) is obtained by summing the potentials for uniform flow at angle alpha, a doublet and a line vortex. The initial grid is a regular mesh in polar coordinates (with a little clustering at the cylinder surface). We then use the Joukowski transform to map the geometry from a cylinder to an airfoil.

In this implementation, the grid and flow solution are evaluated on the CPU and the rendering is done on the GPU using WebGL. Data at each mesh point (position of the point, pressure value, grid i-value and j-value) are sent as vertex attributes. The fragment shader handles the colour shading, and also the streamline and grid plotting using some helpful built-in GLSL functions (thank you [Ricky Reusser](https://observablehq.com/@rreusser/adaptive-domain-coloring)).

The dynamic visualization enables us to experiment with the cylinder and airfoil. Experimentation is a great way to learn. For example, from this interactive Joukowski airfoil we obtain an understanding of the squashing-stretching nature of the Joukowski transform, and also the linkage between streamline curvature and pressure gradient.



