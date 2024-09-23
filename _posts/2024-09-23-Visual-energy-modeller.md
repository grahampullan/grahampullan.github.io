---
layout: post
title:  "Interactive visual modelling for energy systems"
date:   2024-09-23 00:00:49 +0100
categories: viz
summary: "Interactive Gauss and Stokes theorem demonstrations"
image: "/assets/images/stokes-and-box.png"
image_alt: "Image shows a swirling vector field with a red square where Stokes theorem is applied"
usemathjax: false
custom_css: visualmodeller1
---

Text here
<video style="width: 100%; height: auto;" autoplay loop muted>
  <source src="{{site.baseurl}}/assets/images/energy-modeller-1.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>


<div id="target1"></div>
<script type="module">
    import { Model } from 'https://cdn.jsdelivr.net/npm/visual-modeller-energy@0.1.4/+esm';
    const model = new Model();
    await model.loadFromUrl("{{ site.baseurl }}/assets/data/visualmodeller1/PV-battery.json");
    model.run();
    model.startVis({targetId:"target1", height:460})
</script>





