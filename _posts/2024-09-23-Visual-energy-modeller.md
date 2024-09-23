---
layout: post
title:  "Interactive visual modelling for energy systems"
date:   2024-09-23 00:00:49 +0100
categories: viz
summary: "Interactive visual modelling for energy systems"
image: "/assets/images/energy-modeller-1.png"
image_alt: "Image shows 3 panels: a system diagram (nodes and links); outputs from the system model (power in each link vs time); and sliders allowing node parameters to be changed."
usemathjax: false
custom_css: visualmodeller1
---

Even small systems, comprised of simple components, can be hard to reason about. I'm interested in interactive visualisation tools that can help us to understand the behaviour of such systems. 

### Energy systems


<video style="width: 100%; height: auto;" autoplay loop muted controls>
  <source src="{{site.baseurl}}/assets/images/energy-modeller-1.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>


### Nodes, sockets, links and logs

**Nodes** kkdjfkdf 

**Sockets** lsklksj

**Links** djldkjf

**Logs**

### Implementation

Abstraction

<div id="target1"></div>
<script type="module">
    import { Model } from 'https://cdn.jsdelivr.net/npm/visual-modeller-energy@0.1.4/+esm';
    const model = new Model();
    await model.loadFromUrl("{{ site.baseurl }}/assets/data/visualmodeller1/PV-battery.json");
    model.run();
    model.startVis({targetId:"target1", height:460})
</script>







