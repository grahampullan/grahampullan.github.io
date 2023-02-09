---
layout: post
title:  "Visualising vector calculus"
date:   2023-02-09 09:00:49 +0100
categories: viz
summary: "Interactive Gauss and Stokes theorem demonstrations"
image: "/assets/images/stokes-and-box.png"
image_alt: "Image shows a swirling vector field with a red square where Stokes theorem is applied"
usemathjax: true
---
I teach vector calculus to the second year undergraduate engineering students here in [Cambridge](http://www.eng.cam.ac.uk). There are over 300 students in the class so one way I have tried to more deeply connect those attending with a subject that I find both fascinating and elegant is by using interactive demonstrations. Using [Observable](https://www.observablehq.com) I can write notebooks in JavaScript that students can immediately access on the phones. Not only does it break up the lecture, but it also creates a tactile, responsive connection between the course material and the students. I've done this for a few years now and the feedback from students has been really positive - most just enjoy playing with vector calculus but they can also, if they wish, edit and adapt the code themselves.

Here are two vector calculus theorems, from Gauss and Stokes, that I think work nicely as interactive demos.

### Gauss's divergence theorem

Gauss's theorem says that the integral of the divergence of a vector field over a volume *V* is equal to the sum of the flux of that vector field through the enclosing surface *A*:

$$ 
\int_V (\nabla\cdot\mathbf{F})\:dv = \oint_A \mathbf{F}\cdot d\mathbf{A} 
$$

In lectures, we discuss why this is the case, but it is difficult to visualise for all but the very simplest of vector fields. With the interactive demo, the flux through each side of the red box is shown (North, South, East and West) as well as the sum of the fluxes, divided by the area of the red box (so that the sum bar indicates the average divergence of the field over the area enclosed by the box). As you move the box around, the fluxes through each side can change a lot but for a constant divergence, or zero divergence, the sum stays the same. The vectors indicate the strength and direction of the field, so it is easy to see why the fluxes are different on each side of the box. The select dropdown allows different vector fields to be investigated (the 2D position vector field has a divergence of 2 everywhere, "Field 1" has varying divergence and the other two fields both have zero divergence). By editing the [Observable notebook](https://observablehq.com/@grahampullan/gausss-divergence-theorem) additional fields can easily be added. The feedback from students is that the interactivity helps us feel the diveregence theorem in action!


<div id="observablehq-845596d0">
  <div class="observablehq-chart"></div>
  <div class="observablehq-viewof-boxSize"></div>
  <div class="observablehq-viewof-field"></div>
  <div style="overflow: hidden;"><a style="display: block; float:right;" href="https://observablehq.com/@grahampullan/gausss-divergence-theorem"><object type="image/svg+xml" style="pointer-events: none;" width=180 height=22 data="https://static.observableusercontent.com/files/c3fab254a006f1a3a1f9f63aba8ab1460db4752529036b9962950bde0ec195bab823daa6b278b1c3401e545b3bd640ddfdcad805cf9859af218cb2b9fed4ddf0"></object></a></div>
</div>
<script type="module">
  import {Runtime, Inspector} from "https://cdn.jsdelivr.net/npm/@observablehq/runtime@4/dist/runtime.js";
  import define from "https://api.observablehq.com/@grahampullan/gausss-divergence-theorem.js?v=3";
  const runtime = new Runtime();
  const main = runtime.module(define, name => {
    if (name === "chart") return Inspector.into("#observablehq-845596d0 .observablehq-chart")();
    if (name === "viewof boxSize") return Inspector.into("#observablehq-845596d0 .observablehq-viewof-boxSize")();
    if (name === "viewof field") return Inspector.into("#observablehq-845596d0 .observablehq-viewof-field")();
  });
  main.redefine("width",600);
</script>


### Stokes's theorem

Stokes's theorem says that the flux of the curl of a vector field through a surface *S* is equal to the line integral of the vector field around the perimeter *C* of the surface:

$$
\int_S (\nabla\times\mathbf{F})\:d\mathbf{A} = \oint_C \mathbf{F}\cdot d\mathbf{l}   
$$

A similar kind of interactive visualisation can be used. Now, as the box is moved, the components of the line integral along the edges of the box are shown, as well as their sum (again, the bars in the chart are all divided by the area of the box so that the sum shows the average curl). The solid body rotation field has a constant curl; the Rankine vortex has a solid body core (constant curl) but a free vortex outer (zero curl, even though the vectors show the field is swirling) - this works really well as you move the box around; and the final field is irrotational (zero curl) throughout. It's easy to add new fields directly into the [notebook](https://observablehq.com/@grahampullan/stokess-theorem).


<div id="observablehq-62f8dee9">
  <div class="observablehq-chart2"></div>
  <div class="observablehq-viewof-field"></div>
  <div style="overflow: hidden;"><a style="display: block; float:right;" href="https://observablehq.com/@grahampullan/stokess-theorem"><object type="image/svg+xml" style="pointer-events: none;" width=180 height=22 data="https://static.observableusercontent.com/files/c3fab254a006f1a3a1f9f63aba8ab1460db4752529036b9962950bde0ec195bab823daa6b278b1c3401e545b3bd640ddfdcad805cf9859af218cb2b9fed4ddf0"></object></a></div>
</div>
<script type="module">
  import {Runtime, Inspector} from "https://cdn.jsdelivr.net/npm/@observablehq/runtime@4/dist/runtime.js";
  import define from "https://api.observablehq.com/@grahampullan/stokess-theorem.js?v=3";
  const runtime = new Runtime();
  const main = runtime.module(define, name => {
    if (name === "chart") return Inspector.into("#observablehq-62f8dee9 .observablehq-chart2")();
    if (name === "viewof field") return Inspector.into("#observablehq-62f8dee9 .observablehq-viewof-field")();
  });
  main.redefine("width",600);
</script>

### Implementation
Using [Observable](https://observablehq.com) is a neat way to write reactive JavaScript. When the value of a cell changes, all cells depenendent on that cell will re-run automatically (wherever they are in the notebook). So dragging the box causes the box coordinates to change, which triggers a re-evaluation of the data for the bar chart (fluxes or line-integrals), and hence a re-draw of the plots.

