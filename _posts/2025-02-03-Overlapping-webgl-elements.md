---
layout: post
title:  "Multiple overlapping threejs webgl elements"
date:   2025-02-03 00:00:49 +0100
categories: viz
summary: "How to manage multiple overlapping threejs webgl elements"
image: "/assets/images/overlapping-webgl.png"
image_alt: "Image shows four square cards, each containing a cube. The cards are arranged at random and overlap each other."
usemathjax: false
---

## The problem with multiple WebGL contexts
In web-based data visualisation applications, a common UX design is to have multiple cards (each card is a html `<div>` element) each containing a graphical representation of a different aspect of the data. If the plot is a view of a 3D surface, or involves thousands of data points, it is natural to use WebGL. This is most easily achieved by adding a `<canvas>` element as a child of the card `<div>` and making this a WebGL context. Unfortunately, browsers limit the number of available WebGL contexts (a limit of 16 is common), so, if you want more cards than that, another strategy is required.


## Drag these spinning cubes
<select id="stencil">
    <option value="with">Use stencil</option>
    <option value="show">Show stencil</option>
    <option value="without">Don't use stencil</option>
</select>

<div id="target"></div>

### Using only one WebGL context with scissor
A solution is to use just one WebGL context, with a fixed `<canvas>` that covers the whole window. Each card then renders to only a portion of that canvas using WebGL **scissor**. There is a nice `threejs` [demo of this here](https://threejs.org/examples/webgl_multiple_elements.html).

The key `threejs` renderer methods that are used are `setViewport` and `setScissor`:
```js
renderer.setViewport(left, bottom, width, height);
renderer.setScissor(left, bottom, width, height);
```
`setViewport` specifies how the WebGL clip-space maps to the screen, and `setScissor` prevents any rendering outside of a specified box. In our case, we obtain `left`, `bottom`, `width` and `height` using the `getBoundingClientRect()` method of the `<div>` element (remember that `bottom` is measured from the bottom of the WebGL `<canvas>` whereas the element bounding rect is measured from the top of the window). The arguments of `setViewport` and `setScissor` can be the same (as above) or different if you want to cut off a portion of the `scene` that is outside of an enclosing container `<div>` (but still inside the `<canvas>`) - this approach is used in the demo at the start of this post.

### Use a stencil as well
If we only use the **scissor** approach, a remaining problem is that a WebGL render that should be obscured by an overlapping `<div>` card will still be visible (as long as it is not covered by another overlapping WebGL viewport). This can be addressed using WebGL **stencil**. There is a good tutorial on how to use stencil buffers in `threejs` [here](https://www.youtube.com/watch?v=X93GxW84t84).

For our application, every time we want to render a card we need to:

* find which `<div>` elements are overlapping the current one;
* for each overlap, add a stencil rectangle to the current threejs scene that covers the overlapping area.

The stencil rectangles are not normally shown (`colorWrite=false`) but they can be made visible using the the dropdown menu in the above demo. The material used for the stencil rectangles needs to have the following settings:

``` js
const stenMat = new THREE.MeshBasicMaterial({color: "yellow"});
stenMat.colorWrite = false; // don't write to the colour buffer
stenMat.depthWrite = false; // don't write to the depth buffer
stenMat.stencilWrite = true; // write to the stencil buffer
stenMat.stencilRef = 1; // reference value for the stencil test
stenMat.stencilZPass = THREE.ReplaceStencilOp; // set stencil value to 1
```

These settings mean that there is a `1` in the stencil buffer for each pixel of the rectangle stencil.  

We also need to make sure that the material used for the boxes performs the stencil test by setting the following:

```js
const boxMat = new THREE.MeshPhongMaterial({color: cube.colour});
boxMat.stencilWrite = true; // Perform a stencil check
boxMat.stencilRef = 1; // set stencilRef
boxMat.stencilFunc = THREE.NotEqualStencilFunc; // render if stencil value not equal to 1
```

These settings mean that the boxes will not be rendered in any part of the screen covered by a stencil rectangle, which is what we want!

### Best of both worlds!
Using `<div>` elements and WebGL (one fixed `<canvas>` element) works well because we can use the DOM to manage the `<div>` locations and their depth (position in the DOM tree), and still use WebGL for speedy rendering of the data!

<script type="module">
    import * as d3 from 'https://cdn.skypack.dev/d3@7.0.0'; // I like to use d3 to manage the DOM
    import * as THREE from 'https://cdn.skypack.dev/three@0.132.2';
    import { OrbitControls } from 'https://cdn.skypack.dev/three@0.132.2/examples/jsm/controls/OrbitControls.js';

    // array of objects, each with the position and colour of a div
    const cubes = [
        {idx:0, x: 10, y: 10, colour: 'red', stencilRects:[]},
        {idx:1, x: 350, y: 150, colour: 'blue', stencilRects:[]},
        {idx:2, x: 200, y: 250, colour: 'green', stencilRects:[]},
        {idx:3, x: 400, y: 50, colour: 'yellow', stencilRects:[]}
    ];

    const target = d3.select('#target');

    //set width of and height of container div
    target.style('width', '100%')
        .style('height', '500px')
        .style('border', '1px solid black')
        .style('border-radius', '5px')
        .style('overflow', 'hidden')
        .style('position', 'relative');

    // create a div (class outer-div) for each object in the cube array
    target
        .selectAll('.outer-div')
        .data(cubes)
        .enter().append('div')
            .attr('class', 'outer-div')
            .style('position', 'absolute')
            .style('left', d => d.x + 'px')
            .style('top', d => d.y + 'px')
            .style('width', '200px')
            .style('height', '200px')
            .style('background-color', 'white')
            .style('border', '2px solid lightgrey')
            .call(d3.drag()
                .on('start', function(event, d) {
                    d3.select(this).raise();
                })
                .on('drag', function(event, d) {
                    d3.select(this)
                        .style('left', `${d.x = event.x}px`)
                        .style('top', `${d.y = event.y}px`);
                })
            )
            // append an inner-div which is 20px smaller all round than the outer-div
            .append('div')
                .attr('class', 'inner-div')
                .style('position', 'absolute')
                .style('left', '20px')
                .style('top', '20px')   
                .style('width', '160px')
                .style('height', '160px')
                .on('pointerdown', function(event, d) { // stop the div from being dragged when the mouse is over the inner-div
                    event.stopPropagation();
                    event.preventDefault();
            });
            
    // one renderer for whole screen
    const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, stencil: true });
    document.body.appendChild(renderer.domElement);
    d3.select("canvas")
        .style("position", "fixed")
        .style("left", "0px")
        .style("top", "0px")
        .style("pointer-events", "none");

    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setScissorTest(true);
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setAnimationLoop(renderCubes);

    // set up scene for each cube
    cubes.forEach( cube => {
        const scene = new THREE.Scene();
        const backgroundGeometry = new THREE.PlaneGeometry(1000, 1000);
        const backgroundMaterial = new THREE.MeshBasicMaterial({color: 0xd3d3d3});
        backgroundMaterial.depthWrite = false;
        backgroundMaterial.colorWrite = true;
        backgroundMaterial.stencilWrite = true;
        backgroundMaterial.stencilRef = 1;
        backgroundMaterial.stencilFunc = THREE.NotEqualStencilFunc;
        const background = new THREE.Mesh(backgroundGeometry, backgroundMaterial);
        background.material.onBeforeCompile = function( shader ){
            shader.vertexShader = shader.vertexShader.replace( `#include <project_vertex>` , 
                `gl_Position = vec4( position , 1.0 );`
            );
        }
        background.renderOrder = 9;
        scene.add(background);

        const camera = new THREE.PerspectiveCamera(45, 1, 0.1, 1000);
        camera.position.set(0, 0, 3);

        const box = new THREE.BoxGeometry(1, 1, 1);
        const boxMat = new THREE.MeshPhongMaterial({color: cube.colour});
        boxMat.stencilWrite = true;
        boxMat.stencilRef = 1;
        boxMat.stencilFunc = THREE.NotEqualStencilFunc;

        const cubeMesh = new THREE.Mesh(box, boxMat)
        cubeMesh.renderOrder = 10;
        scene.add(cubeMesh);

        // light
        const light = new THREE.DirectionalLight(0xffffff, 1);
        light.position.set(0, 0, 1);
        scene.add(light);

        //controls
        const element = d3.selectAll(".inner-div").filter(d => d.idx == cube.idx).node();
        const controls = new OrbitControls(camera, element);
        controls.enabled = true;
        controls.update();
        controls.addEventListener('change', function() {
            light.position.copy(camera.position);
        });
        controls.enableZoom = true;

        cube.scene = scene;
        cube.camera = camera;
    });

    function renderCubes() {
        const containerRect = target.node().getBoundingClientRect();
        const outerDivs = d3.selectAll(".outer-div").nodes().map(d => d.getBoundingClientRect());

        const stencilMenu = d3.select('#stencil').node().value;
        let stencilColorWrite = false;
        let stencilWrite = true;
        if (stencilMenu == 'show') {
            stencilColorWrite = true;
        } else if (stencilMenu == 'without') {
            stencilWrite = false;
        }
   
        d3.selectAll('.inner-div').each(function(d, i) { // i is the child index of the div in the parent
            const idx = d.idx;
            const cubeData = cubes.find(d => d.idx==idx); // get this cube's data
            const scene = cubeData.scene;
            const camera = cubeData.camera;
            scene.children[ 1 ].rotation.y = Date.now() * 0.001; // spin the cube

            const canvasHeight = renderer.domElement.clientHeight;

            const rect = this.getBoundingClientRect();
            const width = rect.width;
            const height = rect.height
        
            const scissorLeft = Math.max(rect.left,containerRect.left);
            const scissorRight = Math.min(rect.right,containerRect.right);
            const scissorTop = Math.max(rect.top, containerRect.top);
            const scissorBottom = Math.min(rect.bottom, containerRect.bottom);


            renderer.setScissor(scissorLeft, canvasHeight - scissorBottom, scissorRight-scissorLeft, scissorBottom - scissorTop); // set scissor to match the div
            renderer.setViewport(rect.left, canvasHeight - rect.bottom, width, height); // set viewport to match the div

            const nearerDivs = outerDivs.filter((d, j) => i < j); // get the divs that are later in the array
            const nearerDivsClipSpace = nearerDivs.map(d => { // convert to clip space
                const left = (d.left - rect.left) / width * 2 - 1;
                const right = (d.right - rect.left) / width * 2 - 1;
                const top = (rect.top + rect.height - d.top) / height * 2 - 1;
                const bottom = (rect.top + rect.height - d.bottom) / height * 2 - 1;
                let overlap = false;
        
                if (left < 1 && right > -1 && bottom < 1 && top > -1) {
                    overlap = true;
                }
                return {left, right, top, bottom, overlap};
            });
         
            const overlappingDivsClipSpace = nearerDivsClipSpace.filter(d => d.overlap);

            // remove old stencil rectangles from the scene
            cubeData.stencilRects.forEach( uuid => {
                const oldRect = scene.getObjectByProperty('uuid', uuid);
                oldRect.geometry.dispose();
                oldRect.material.dispose();
                scene.remove(oldRect);
            })
            cubeData.stencilRects = [];

            // create new stencil rectangles where there is overlap
            overlappingDivsClipSpace.forEach(d => {
                const rectangleBufferGeometryForMesh = new THREE.BufferGeometry();
                const vertices = new Float32Array([
                    d.left, d.top, 0,
                    d.right, d.top, 0,
                    d.right, d.bottom, 0,
                    d.left, d.bottom, 0
                ]);
                const indices = new Uint32Array([
                    0, 2, 1,
                    0, 3, 2
                ]);
                rectangleBufferGeometryForMesh.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
                rectangleBufferGeometryForMesh.setIndex(new THREE.BufferAttribute(indices, 1));
                const stenMat = new THREE.MeshBasicMaterial({color: "yellow"});
                stenMat.colorWrite = stencilColorWrite; // default = false (don't show the colour)
                stenMat.depthWrite = false; // don't write to the depth buffer
                stenMat.stencilWrite = stencilWrite; // default is true (write to the stencil buffer)
                stenMat.stencilRef = 1; // reference value for the stencil test
                stenMat.stencilZPass = THREE.ReplaceStencilOp; // set stencil value to stencilRef if rectangle is drawn
                const rectangle = new THREE.Mesh(rectangleBufferGeometryForMesh, stenMat);
                rectangle.material.onBeforeCompile = function( shader ){ // keep rectangle fixed in clip space
                    shader.vertexShader = shader.vertexShader.replace( `#include <project_vertex>` , 
                        `gl_Position = vec4( position , 1.0 );`
                    );
                }
                rectangle.renderOrder = 0; // draw first

                cubeData.stencilRects.push(rectangle.uuid)
                scene.add(rectangle);  
                
            });

            renderer.render(scene, camera);

        });
    }

</script>






