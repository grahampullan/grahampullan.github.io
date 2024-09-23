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

Even small systems, comprised of simple components, can be hard to reason about. I'm interested in interactive visualisation tools that can help us to understand the behaviour of such systems. This post shows the current status of a prototyping tool that I am working on.

### Energy systems

The video below shows a conceptual domestic solar energy system. There are three panels: a model structure panel; a log viewer panel; and a node inspector panel. Each box (node) in the model structure is either an electricity supply (Solar PV, Grid Supply), demand (Load, Grid Export), storage (Battery) or Controller. The load demand and solar supply variations throughout the day are prescribed on a minute-by-minute basis. In the video, the shape of these variations is seen by mousing-over the links connected to the Load or Solar PV nodes - this highlights the corresponding curves in the log viewer. By clicking on the nodes, the user can modify the configuration using sliders in the node inspector panel (the 'socketMultiplier' is a factor applied to the prescribed load and supply curves); the model analysis is then re-run and the curves updated. The bar chart shows the time-integration (i.e. total energy in a day) of each of the power vs time curves.

<video style="width: 100%; height: auto;" autoplay loop muted controls>
  <source src="{{site.baseurl}}/assets/images/energy-modeller-1.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>


### Nodes, sockets, links and logs

The model has 4 classes of object:

**Nodes** Each node represents an entity in the real system. For example, the *Load* node represents all the demands for electricity captured by the model, aggregated into one object. A node can have its own state (properties) - a battery has its current charge level, for example.

**Sockets** Sockets (shown as small circles on the border of the nodes) are points where a node can be connected to another node. Sockets contain information about the power going into or out of a node, and whether this is allowed to be set by another node (such as the power supplied from the grid) or is a constrained value (such as the power supplied from solar PV). In the current implementation, a socket on the left side of a node is an input to the node, whereas a socket on the right side is an output (this is why the links between the Controller and the Battery nodes cross each other!).

**Links** Two sockets, each on a different node, are connected by a link. The link has its own state: the current power that is flowing along the link.

**Logs** The model can be configured with a number of logs. Each log records the current state of a specified node, socket or link. The logs are then plotted in the log viewer, where mouse-over events illustrate the associated object in the model structure (and *vice versa*).

### Implementation
There are aspects of the above approach that are generic to different types of system and others that are specific to the energy model. Generic aspects include parent classes for the model, node, socket, link and log objects; all of the visualisation and interactivity is currently within the generic framework [visual-modeller-core](https://github.com/grahampullan/visual-modeller-core). Specific aspects include classes for the nodes of the energy model and the logic for how the model should be solved (a series of bespoke operations over the collections of nodes, links and sockets held by the model), [visual-modeller-energy](https://github.com/grahampullan/visual-modeller-energy).


### Have a play with the model!

Here is an embedded version of the energy model shown in the video. By moving the sliders in the node inspector, it's easy to end up with complex power curves. By interacting with the model, the user can see the influence of the number of solar panels or the size of the battery.
<div id="target1"></div>
<script type="module">
    import { Model } from 'https://cdn.jsdelivr.net/npm/visual-modeller-energy@0.1.4/+esm';
    const model = new Model();
    await model.loadFromUrl("{{ site.baseurl }}/assets/data/visualmodeller1/PV-battery.json");
    model.run();
    model.startVis({targetId:"target1", height:460})
</script>







