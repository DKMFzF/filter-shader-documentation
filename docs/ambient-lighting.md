<h1>Shader Lighting Normal Documentation</h1>

```version change v1.0 - v.1.2```

<p>This document provides an overview of the shader for lighting in Minecraft <code>gbuffers_terrain.vsh</code> and <code>gbuffers_terrain.fsh</code> shader files. The Minecraft computes face normals in the vertex shader and utilizes them in the fragment shader for directional lighting calculations.</p>

<h2>Theoretical base</h2>

**Lambertian reflection** — is a property that defines an ideal "matte" (diffusely reflective) surface. From whatever angle you look at it, the apparent brightness will be the same for the observer.

[Explanation of lighting](https://www.youtube.com/watch?v=HPNW0we-ft0&t=24s) — very cool video, I recommend watching it. The minimal theory is explained.

<h2 id="varying-normals_face">Varying Variable: <code>normals_face</code></h2>

```glsl
...
varying vec3 normals_face;
...
```

<p>This varying vector represents the normalized normal of the vertex, transformed into eye space. It is passed from the vertex shader to the fragment shader for use in lighting computations.</p>

<h2 id="vertex-shader">Vertex Shader: <code>gbuffers_terrain.vsh</code></h2>

```glsl
...
void main() {
...
  normals_face = normalize(gl_NormalMatrix * gl_Normal);
...
}
```

<p><strong>Purpose:</strong> This line calculates the transformed and normalized normal vector for the vertex using the OpenGL-provided <code>gl_NormalMatrix</code> and <code>gl_Normal</code>.</p>

<ul>
  <li><code>gl_Normal</code>: The vertex normal in object space.</li>
  <li><code>gl_NormalMatrix</code>: The inverse transpose of the model-view matrix (used to properly transform normals).</li>
</ul>

<p><strong>Explanation:</strong> This transformation converts the normal from object space into eye space, which is suitable for use in lighting calculations in the fragment shader.</p>

<h2 id="fragment-shader">Fragment Shader: <code>gbuffers_terrain.fsh</code></h2>

```glsl
...
void main() {
  ...
  float lightDot = clamp(dot(normalize(shadowLightPosition), normals_face), 0., 1.);
  ...
}
...
```

<p>This line computes the directional lighting intensity based on the angle between the light direction and the surface normal.</p>

<ul>
  <li><code>shadowLightPosition</code>: A vector pointing from the fragment to the light source.</li>
  <li><code>normals_face</code>: The interpolated and normalized surface normal from the vertex shader.</li>
  <li><code>dot(...)</code>: Computes the cosine of the angle between the light vector and the normal (Lambertian reflectance).</li>
  <li><code>clamp(..., 0., 1.)</code>: Ensures the resulting intensity stays in the visible range.</li>
</ul>

<p><strong>Lighting Formula:</strong></p>

```glsl
...
void main() {
  ...
  color.rgb = color.rgb * (
      fire_color * lm.x
      + texture2D(lightmap, lm).y * lm.y
      + lightDot
  );
  ...
}
...
```

<p>This formula blends three lighting contributions:</p>
<ul>
  <li><code>fire_color * lm.x</code>: Dynamic lighting (e.g., torchlight) scaled by a lightmap component.</li>
  <li><code>texture2D(lightmap, lm).y * lm.y</code>: Ambient or indirect light based on the lightmap texture.</li>
  <li><code>lightDot</code>: Direct light contribution based on the angle to the light source.</li>
</ul>

<p><strong>Final Modulation:</strong></p>

```glsl
...
void main() {
  ...
  color *= texture2D(lightmap, lm);
  ...
}
...
```

<p>The color is further modulated by the full lightmap texture, providing a baked-in lighting effect from the world’s environment.</p>

<h2 id="implementation-notes">Implementation Notes</h2>

<ul>
  <li><strong>Lighting Model:</strong> Basic Lambertian reflectance using a per-fragment normal.</li>
  <li><strong>Coordinate Space:</strong> Normals are transformed into eye space for proper light interaction.</li>
  <li><strong>Performance:</strong> This model is efficient and suitable for real-time rendering of terrain surfaces.</li>
  <li><strong>Compatibility:</strong> Relies on fixed-function pipeline uniforms like <code>gl_Normal</code> and <code>gl_NormalMatrix</code>.</li>
</ul>
