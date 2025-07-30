# Underwater Effect

The "underwater" effect is a visual phenomenon that simulates being submerged in water. It's achieved by combining visual distortions, volumetric fog, and adjusted lighting, creating a distinct environmental feel. This effect typically involves a combination of vertex and fragment shaders, along with specialized textures.

---

### 1. Determining "Underwater" State

The initial step is to determine if the camera is currently submerged. This can be handled by the game engine (CPU) or within the shader pipeline by checking the camera's position relative to the water level.

* **CPU-side / Engine:**
    * Compare the camera's Y-position (`cameraPosition.y`) with a known water level height.
    * Pass a boolean uniform (`isUnderwater`) to the shaders.

* **Shader-side (less common for a definitive "underwater" state, more for water surface detection):**
    * If a specific water surface is rendered, its normals (`gnormal`) can identify water pixels.
    * However, to ascertain if the *camera* is under the water, a direct check against `cameraPosition.y` is more robust.
    * **Reference from your `main.fsh` (`isWater`):**
        ```glsl
        bool isWater = (length(normal) > 0.94 && length(normal) < 0.96); // This detects water surface pixels, not camera underwater state.
        ```
        *To determine camera's underwater state, you would typically use a uniform indicating water height, e.g., `uniform float waterLevel;` and then `if (cameraPosition.y < waterLevel) { /* underwater effects */ }`.*

---

### 2. Underwater Fog and Color Tint

When submerged, visibility should decrease, and colors should shift to a characteristic hue (typically blue or green), simulating light absorption by water.

* **Implementation Location:** A **post-processing fragment shader** (one that takes the final rendered scene as input).
* **Required Data:**
    * **`depthtex0` (Depth Map):** Used to calculate the distance of each scene pixel from the camera.
    * **`cameraPosition` (Uniform):** The camera's world space position.
    * **`far` and `near` (Uniforms):** Far and near clipping planes for accurate depth reconstruction.
    * **`underwaterFogColor` (Uniform):** The desired color of the underwater fog (e.g., `vec3(0.1, 0.3, 0.4)` for a blue tint).
    * **`fogDensity` (Uniform):** Controls how quickly the fog intensifies with distance.

* **GLSL Snippet (Conceptual - in a separate post-processing fragment shader):**

    ```glsl
    #version 120

    uniform sampler2D gcolor; // Main scene color texture
    uniform sampler2D depthtex0; // Depth texture

    uniform vec3 cameraPosition;
    uniform float near;
    uniform float far;
    uniform float waterLevel; // Assuming water level is passed
    uniform bool isUnderwater; // Assuming this is set by the engine
    uniform vec3 underwaterFogColor;
    uniform float fogDensity;

    varying vec2 texcoord; // Screen-space texture coordinates

    // Function to reconstruct world position from depth
    vec3 getWorldPos(vec2 uv, float depth) {
        // This requires projection and inverse view matrices,
        // similar to what's done in main.fsh with gbufferProjectionInverse
        // For simplicity, direct depth-based calculation here:
        float z = depth * 2.0 - 1.0; // Convert to NDC space [-1, 1]
        vec4 projectedPos = vec4(uv * 2.0 - 1.0, z, 1.0);
        vec4 unprojectedPos = gbufferProjectionInverse * projectedPos; // Requires gbufferProjectionInverse
        return unprojectedPos.xyz / unprojectedPos.w;
    }

    void main() {
        vec4 finalColor = texture2D(gcolor, texcoord);

        if (isUnderwater) {
            float depth = texture2D(depthtex0, texcoord).x;
            // Assuming simplified depth conversion for example:
            float linearDepth = (near * far) / (far + depth * (near - far));
            float distance = linearDepth; // Or more complex calculation using worldPos

            // Apply exponential fog
            float fogFactor = exp(-distance * fogDensity);
            fogFactor = clamp(fogFactor, 0.0, 1.0);

            finalColor.rgb = mix(underwaterFogColor, finalColor.rgb, fogFactor);
        }

        gl_FragColor = finalColor;
    }
    ```

---

### 3. Underwater Distortion/Refraction

This effect mimics light refraction through water, making objects viewed through the water appear distorted or wavy.

* **Implementation Location:** A **post-processing fragment shader** or a dedicated water surface fragment shader.
* **Required Data:**
    * **`noisetex` (Noise Texture):** Perlin noise or similar, used to generate a wavy pattern.
    * **`frameTimeCounter` (Uniform):** For animating the distortion over time.
    * **`distortionStrength` (Uniform):** Controls the intensity of the distortion.
    * **Scene Texture (e.g., `gcolor`):** The rendered scene to be distorted.

* **GLSL Snippet (Conceptual - in a separate post-processing fragment shader):**

    ```glsl
    #version 120

    uniform sampler2D gcolor; // Main scene color texture
    uniform sampler2D noisetex; // Noise texture for distortion
    uniform float frameTimeCounter;
    uniform float distortionStrength;
    uniform bool isUnderwater;

    varying vec2 texcoord;

    void main() {
        vec2 distortedTexcoord = texcoord;

        if (isUnderwater) {
            // Sample noise texture, often with animation
            vec2 noise = texture2D(noisetex, texcoord * 0.5 + frameTimeCounter * 0.001).xy;
            noise = noise * 2.0 - 1.0; // Remap noise to [-1, 1]

            // Apply distortion by offsetting texture coordinates
            distortedTexcoord += noise * distortionStrength;
        }

        gl_FragColor = texture2D(gcolor, distortedTexcoord);
    }
    ```

---

### 4. Underwater Caustics

Caustics are patterns of light and dark on the seafloor created by light focusing through the wavy water surface.

* **Implementation Location:** Can be added to the terrain/object fragment shaders or a dedicated post-processing pass that overlays the caustics.
* **Required Data:**
    * **`causticTexture` (Uniform Sampler2D):** A pre-rendered or animated texture of caustic patterns.
    * **`worldPosition` (Varying/Reconstructed):** The world-space position of the fragment, needed to project the caustics.
    * **`causticIntensity` (Uniform):** Strength of the caustics.
    * **`causticSpeed` (Uniform):** Animation speed for the caustics.

* **GLSL Snippet (Conceptual - in a terrain/gbuffer fragment shader or post-process):**

    ```glsl
    #version 120

    uniform sampler2D causticTexture;
    uniform float frameTimeCounter;
    uniform float causticIntensity;
    uniform vec3 waterLevel; // Or specific water plane

    varying vec3 worldPosition; // Assuming this is available

    void main() {
        // ... previous color calculations ...

        // Project caustics onto the XZ plane
        vec2 causticUV = worldPosition.xz * 0.01 + vec2(frameTimeCounter * causticSpeed, 0.0);
        vec3 caustics = texture2D(causticTexture, causticUV).rgb;

        // Only apply if below water and above ground
        if (worldPosition.y < waterLevel.y && worldPosition.y > (waterLevel.y - 100.0)) { // Example depth limit
            color.rgb += caustics * causticIntensity;
        }

        gl_FragData[0] = color;
    }
    ```

---

### 5. Adjusting Underwater Lighting

Light under water is scattered and absorbed, leading to a dimmer and more diffuse illumination.

* **Implementation Location:** Fragment shaders responsible for calculating direct and ambient lighting (e.g., `gbuffers_terrain.fsh` from your example, or a dedicated lighting pass).
* **Required Data:**
    * **`isUnderwater` (Uniform):** To conditionally apply lighting changes.
    * **`underwaterLightMultiplier` (Uniform):** A scalar to reduce light intensity.
    * **`underwaterAmbientBoost` (Uniform):** Potentially to increase ambient light slightly for better visibility.

* **GLSL Snippet (Modifying `gbuffers_terrain.fsh` example):**

    ```glsl
    #version 120

    // ... (existing uniforms and varyings) ...
    uniform bool isUnderwater; // New uniform
    uniform float underwaterLightMultiplier; // New uniform
    uniform float underwaterAmbientBoost; // New uniform

    void main() {
        // ... (existing color and lightmap setup) ...

        // ... (existing LIGHTING_STYLES 0 and 1) ...

        // Apply underwater lighting adjustments
        if (isUnderwater) {
            // Reduce direct light contribution
            color.rgb *= underwaterLightMultiplier;

            // Potentially boost ambient light to prevent pitch black areas
            color.rgb += sky_color * underwaterAmbientBoost; // Using sky_color as an ambient source example
            // Or directly manipulate lm.y (ambient lightmap component)
            // lm.y *= underwaterAmbientMultiplier;
        }

        // Final lighting calculation
        color.rgb = color.rgb * (
            fire_color * lm.x
            + texture2D(lightmap, lm).y * lm.y
            + lightDot
        );

        // ... (BERSERK_MOD) ...

        // Final lightmap application
        color *= texture2D(lightmap, lm);

        /* DRAWBUFFERS:0 */
        gl_FragData[0] = color;
    }
    ```
