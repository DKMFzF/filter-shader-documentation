# Sun Glare (Lens Flare) Documentation

Sun glare, often referred to as **lens flare**, is a visual effect that simulates the scattering of light within a camera lens when a bright light source (like the sun) is in or near the frame. It typically appears as a series of circles, halos, or streaks of light. In real-time rendering, this effect enhances realism and the perception of a bright sun.

---

### 1. Core Principles of Lens Flare

A lens flare is usually composed of several elements, each with its own texture and properties:

* **Ghost elements (Ghosts):** These are the distinct, often circular or polygonal, reflections of the light source that appear along a line opposite the light source's position on the screen.
* **Halo/Glow:** A soft, diffused light surrounding the light source.
* **Streaks:** Linear light artifacts, sometimes extending from the center.
* **Chromatic aberration:** Color fringing around bright areas.

The key to creating a convincing lens flare is to correctly position and scale these elements based on the light source's screen position and visibility.

---

### 2. Required Data and Setup

To render sun glare effectively, you'll need the following data and setup:

* **Sun's Screen Position:** The 2D coordinates of the sun on the screen. This is crucial for positioning the flare elements.
* **Sun's Visibility (Occlusion):** You need to determine if the sun is obscured by geometry (e.g., mountains, clouds, terrain). This is often done using a depth buffer lookup or a dedicated occlusion query.
* **Lens Flare Textures:** Individual textures for each flare element (e.g., a simple white circle, a soft halo texture, etc.).
* **Screen Resolution:** To normalize screen coordinates.
* **Time (Optional):** For subtle animations or variations in the flare.

---

### 3. Implementation Steps (Post-Processing Fragment Shader)

Lens flare is almost always a **post-processing effect**. This means it runs after the entire scene has been rendered to a texture.

#### 3.1. Calculating Sun's Screen Position

First, transform the sun's 3D world position into 2D screen coordinates. Your provided `main.fsh` already does this:

```glsl
// Calculate sun's screen position
vec4 tpos = vec4(sunPosition, 1.0) * gbufferProjection;
tpos = vec4(tpos.xyz / tpos.w, 1.0);
vec2 pos1 = tpos.xy / tpos.z; // This gives NDC (-1 to 1)
vec2 lightPos = pos1 * 0.5 + 0.5; // This remaps to [0, 1] screen coordinates
```

- sunPosition: A vec3 uniform representing the sun's world-space position.

- gbufferProjection: A mat4 uniform for the projection matrix.

- lightPos: The sun's screen position, where (0,0) is bottom-left and (1,1) is top-right.

#### 3.2. Determining Sun Occlusion

If an object is blocking the sun, the flare should fade out or disappear. You can do this by sampling the depth buffer at the sun's screen position and comparing it to the sun's actual depth.

```glsl

// Conceptual: In your main.fsh, getDepth0 is the depth of the current pixel.
// You'd need a separate lookup at lightPos for sun occlusion.
uniform sampler2D depthtex0; // Depth texture of the main scene
uniform float sunDepth; // Pre-calculated depth of the sun (in the same depth space as depthtex0)

// ...
float sunOcclusion = 1.0; // Assume fully visible initially
vec2 sunScreenPos = lightPos; // From 3.1

// Sample depth at the sun's screen position
float actualDepthAtSunPos = texture2D(depthtex0, sunScreenPos).x;

// Compare sampled depth with the sun's true depth.
// This comparison is simplified; complex occlusion might involve sampling a larger area
// or using dedicated occlusion queries on the CPU.
if (actualDepthAtSunPos < sunDepth - 0.001) { // Add a small epsilon to avoid Z-fighting/shadow acne issues
    sunOcclusion = 0.0; // Sun is occluded
} else {
    // Optionally, calculate partial occlusion if the sun is near an edge
    // or use a separate occlusion texture/query result.
}

// Calculate an overall intensity factor for the flare. It also fades as the sun moves to screen edges.
float intensityFactor = 1.0 - length(sunScreenPos * 2.0 - 1.0); // Fades as sun goes towards corners
intensityFactor = clamp(intensityFactor, 0.0, 1.0) * sunOcclusion;

```

- sunDepth: This uniform needs to be passed to the shader. It represents the depth value of the sun's position, transformed into the same depth space as depthtex0.

#### 3.3. Rendering Flare Elements

Each flare element is essentially a textured quad drawn at a specific position and scaled. Ghost elements are typically positioned along a line extending from the screen center, through the sun's screen position.

Let screenCenter be vec2(0.5, 0.5).
Let lightToCenterVector be screenCenter - sunScreenPos.

For each flare element (e.g., numGhosts elements):

```glsl

uniform sampler2D flareTexture; // Texture for this specific flare element
uniform float flareSize; // Size of this element
uniform float flareOffset; // Offset along the line (e.g., 0.0 for the sun itself, 1.0 for the first ghost, etc.)

// In the main function of your post-processing shader:
void main() {
    vec4 finalColor = texture2D(gcolor, texcoord); // Your current scene rendered to gcolor

    // ... (calculate sunScreenPos and intensityFactor as above) ...

    if (intensityFactor > 0.0) {
        // Calculate the screen position for this specific flare element
        vec2 elementPos = sunScreenPos + lightToCenterVector * flareOffset;

        // Calculate UVs for drawing a quad around elementPos.
        // Check if the current pixel's texcoord falls within the bounds of the flare element.
        vec2 diff = texcoord - elementPos;
        float distFromCenter = length(diff) / (flareSize * 0.5); // Normalized distance from the element's center

        if (distFromCenter < 1.0) { // If the current pixel is within the flare element's radius
            vec4 flareSample = texture2D(flareTexture, diff / (flareSize * 0.5) * 0.5 + 0.5); // Sample the flare texture
            // The scaling (flareSize * 0.5) and remapping (+0.5) depends on your flare texture's UVs.

            // Mix the flare color with the scene color, weighted by the flare texture's alpha and intensityFactor
            finalColor.rgb = mix(finalColor.rgb, flareSample.rgb, flareSample.a * intensityFactor);
        }
    }

    gl_FragColor = finalColor;
}

```

- **Iterative Drawing**: To draw multiple ghost elements, you would loop this process for each ghost, adjusting flareOffset and flareSize accordingly. Often, a single texture atlas containing all flare elements is used, with flareTexture being sampled from a sub-region.

- **Render Pass**: For more complex flares, each element might be rendered as a separate transparent quad onto the screen, rather than drawing them directly within a single post-processing shader with multiple texture lookups (which can be inefficient for many elements). This approach involves drawing a quad for each ghost to the screen using its calculated position.


#### 3.4. Fade out at Edges

The flare should smoothly fade as the sun approaches the screen edges, as demonstrated in the intensityFactor calculation in section 3.2.

### 4. Advanced Considerations
Bloom Integration: Lens flare often complements a bloom effect. Bloom handles the general glow of bright areas, while lens flare adds specific optical artifacts. The bloom pass typically runs before or after the lens flare pass.

- Anamorphic Flares: For a more cinematic appearance, lens flares can be horizontally stretched (anamorphic), especially for very bright light sources.

- Performance: Lens flares can be computationally expensive if many elements are drawn or if occlusion checks are complex. Optimize by reducing the number of elements or using simpler occlusion methods.

