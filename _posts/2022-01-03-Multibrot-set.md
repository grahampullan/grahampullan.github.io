---
layout: post
title:  "Multibrot sets"
date:   2022-01-03 09:00:49 +0100
categories: viz
summary: "Interactive Multibrot sets demonstration, with WebGL"
image: "/assets/images/Multibrot.png"
image_alt: "Image shows the Multibrot set when d=4"
---

My last blog post was on May 4, 2021 - so this post either establishes a regular interval of 8 months, or demonstrates a new year's resolution for more frequent posts. Time will tell!

The image below is an interactive demonstration of [Multibrot Sets](https://en.wikipedia.org/wiki/Multibrot_set). The image initialises to the familiar [Mandelbrot Set](https://en.wikipedia.org/wiki/Mandelbrot_set) - the special case when the exponent *d* is 2. You can change *d*, zoom, pan, and change the maximum number of iterations used.


<div id="observablehq-canvas-e800ed8a"></div>
<div id="observablehq-viewof-d-e800ed8a"></div>
<div id="observablehq-viewof-maxIters-e800ed8a"></div>
<p>Credit: <a href="https://observablehq.com/@grahampullan/multibrot-sets">Multibrot Sets by grahampullan</a></p>

<script type="module">
import {Runtime, Inspector} from "https://cdn.jsdelivr.net/npm/@observablehq/runtime@4/dist/runtime.js";
import define from "https://api.observablehq.com/@grahampullan/multibrot-sets.js?v=3";
const runtime = new Runtime();
const main = runtime.module(define, name => {
  if (name === "canvas") return new Inspector(document.querySelector("#observablehq-canvas-e800ed8a"));
  if (name === "viewof d") return new Inspector(document.querySelector("#observablehq-viewof-d-e800ed8a"));
  if (name === "viewof maxIters") return new Inspector(document.querySelector("#observablehq-viewof-maxIters-e800ed8a"));
  return ["gl","programInfo","render","zoom","fragShader"].includes(name);
});
main.redefine("width",600);
</script>

In the [Joukowski airfoil]({% post_url 2021-04-15-Joukowski-airfoils %}) post, WebGL was used to render the coloured pressure field and draw the streamlines, but the computation to obtain the flowfield was done in JavaScript. For the Multibrot Set demo, all the hard work is done by WebGL. We normally think of the WebGL graphics pipeline as a *vertex shader* that maps our wireframe of triangles in 3D space to the 2D screen, and then a *fragment shader* that works out how to colour the pixels that lie within each transformed triangle. But for our current demo, we just ask the *vertex shader* to draw two triangles that fill the required image area (and are always fixed) and then write a *fragment shader* that uses the coordinates of each pixel to generate the required colour for the Multibrot set. This in the principle of the [Shadertoy](https://www.shadertoy.com) website where there are endless examples of amazing shaders!

I want to show two parts of our fragment shader here. The first is the `mandelbrot` function that takes in coordinates and returns the number of iterations (normalised by the maximum number of iterations limit). There is nothing too "shadery" about this function (apart from the vec2 data type):

```glsl
float mandelbrot(float x, float y) {
  vec2 z, c;
  float d = float(u_d); // exponent in the multibrot equation

  c.x = x;
  c.y = y;

  z = c; // set the initial value of z to the current point
  int iterCount = 0;
  for (int i=0; i<maxIters; i++) {
    // evaluate z^d using de Moivre's theorem
    float r = sqrt(z.x*z.x + z.y*z.y);
    float theta = atan(z.y, z.x);
    float rNew = pow(r,d);         
    float thetaNew = d*theta;
    float x = rNew*cos(thetaNew) + c.x;
    float y = rNew*sin(thetaNew) + c.y;
    // check the magnitude of z and break if too high
    if ( (x * x + y * y) > 4.0) break;
    // otherwise update z and repeat the loop
    z.x = x;
    z.y = y;
    iterCount += 1;
  }
  return float(iterCount)/float(maxIters);
}
```

It's in the `main` function where we link the pixel coordinate to the colour we want to see. The pixel coordinate (i.e. in number of pixels in the x and y directions) is stored in `gl_FragCoord` (a vec2). We manipulate this to end up with the real x and y ordinates (`xpt` and `ypt`) that we need to send to `mandelbrot`. `u_translate` and `u_scale` come directly from the user's input as processed by [d3-zoom](https://github.com/d3/d3-zoom) and `u_resolution` is our image size in pixels. We then use the output of `mandelbrot` to set `glFragColor` - a vec4 that stores the colour of the pixel at the location given by `gl_FragCoord`.


```glsl
void main() {
  float xpt = (gl_FragCoord.x - u_translate.x) / u_resolution.x / u_scale;
  float ypt = (u_resolution.y - gl_FragCoord.y - u_translate.y) / u_resolution.y / u_scale;
  float t = mandelbrot( xpt, ypt);
  gl_FragColor = vec4(colorScale(fract(sqrt(t)+0.5)),1);
}
```

Of course, the advantage of using WebGL in this way is that evaluation of each pixel is now done in parallel on dedicated hardware. It means our demo is both fast and portable.

A final note is that the original [Observable notebook](https://observablehq.com/@grahampullan/multibrot-sets) for this demo did not contain the selector for the maximum number of iterations - this was a really nice suggestion by [Prof Mark McClure](https://marksmath.org) - thanks Mark!

I hope you enjoy exploring the Multibrot sets!
