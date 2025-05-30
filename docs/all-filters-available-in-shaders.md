<div class="shader-docs">
  <h1>Shader Filter System Documentation</h1>
  
  <p>This document describes a GLSL-based image filter system that applies various post-processing effects to image data. Each effect is implemented as a color transformation function, operating in linear RGB space unless otherwise noted.</p>

  <h2>Color Mixing Utility</h2>

  <h3>make_color</h3>
  
  <pre><code class="language-glsl">vec3 make_color(in vec3 base_color, in vec3 mix_color, in float amount)
{
    return mix(base_color, mix_color, amount);
}</code></pre>

  <p><strong>Parameters:</strong></p>
  <ul>
    <li><code>base_color</code>: The original color vector (RGB).</li>
    <li><code>mix_color</code>: The color to blend with.</li>
    <li><code>amount</code>: A float ∈ [0, 1] controlling the blend ratio.</li>
  </ul>
  
  <p><strong>Description:</strong> Performs a linear interpolation between two color vectors:</p>
  <p>Used as a generic tool for gradual color adjustments across different effects.</p>

  <h2>Sepia Tone</h2>

  <h3>sepia</h3>
  
  <pre><code class="language-glsl">vec3 sepia(in vec3 color)
{
    vec3 sepia = vec3(1.2, 1.0, 0.8);
    float gray = dot(color, vec3(0.299, 0.587, 0.114));
    return make_color(color, vec3(gray) * sepia, 0.75);
}</code></pre>

  <p><strong>Description:</strong> Simulates the warm tone of vintage photos. First, converts the input color to grayscale using a luminance-weighted average. Then, multiplies by a sepia tint and blends with the original color.</p>
  
  <p><strong>Mathematically:</strong></p>
  <p>Grayscale conversion:</p>
  <p>Sepia output:</p>

  <h2>Grayscale</h2>

  <h3>Grayscale Mixing</h3>
  
  <pre><code class="language-glsl">float average_color = (color.r + color.g + color.b) / 3.0;
color = make_color(color, vec3(average_color), GRAY_AMOUNT);</code></pre>

  <p><strong>Description:</strong> Reduces saturation by blending original color toward an average of RGB values.</p>
  
  <p>Contrast to luminance-based grayscale (used in sepia), this method treats all RGB channels equally:</p>

  <h2>Invert</h2>

  <h3>invert</h3>
  
  <pre><code class="language-glsl">vec3 invert(in vec3 color)
{
    return vec3(1.0) - color;
}</code></pre>

  <p><strong>Description:</strong> Inverts the RGB values. Mathematically:</p>
  
  <p>This operation simulates a photographic negative or "color inverse" filter.</p>

  <h2>Grain & Vignette (Old Film Look)</h2>

  <h3>oldFilm</h3>
  
  <pre><code class="language-glsl">vec3 oldFilm(vec3 color, vec2 uv)
{
    float grain = fract(sin(dot(uv, vec2(13, 78.2))) * 44000);
    color += grain * 0.25;

    vec2 center = uv - 0.5;
    float vignette = 1.0 - dot(center, center);
    color *= vignette * 1.25;

    return color;
}</code></pre>

  <p><strong>Description:</strong> Adds procedural grain (pseudo-random noise) and vignette darkening based on distance from the center.</p>
  
  <p>Grain uses a simple hash function via <code>fract(sin(dot(...)))</code> to generate noise.</p>
  
  <p>Vignette effect fades toward the corners by evaluating:</p>
  
  <p>The result gives a radial gradient centered on (0.5, 0.5), enhancing the vintage feel.</p>

  <h2>Contrast Adjustment</h2>

  <h3>effectContrast</h3>
  
  <pre><code class="language-glsl">vec3 effectContrast(in vec3 color, in float value)
{
    return 0.5 + (1.0 + value) * (color - 0.5);
}</code></pre>

  <p><strong>Description:</strong> Linearly scales the deviation of each channel from 0.5 (mid-gray), effectively increasing or decreasing contrast.</p>
  
  <p><strong>Mathematically:</strong></p>
  
  <ul>
    <li><code>value > 0</code>: boosts contrast.</li>
    <li><code>value < 0</code>: reduces contrast.</li>
  </ul>

  <h2>Brightness Adjustment</h2>

  <h3>effectBrightness</h3>
  
  <pre><code class="language-glsl">vec3 effectBrightness(in vec3 color, in float value)
{
    return color + value;
}</code></pre>

  <p><strong>Description:</strong> Adds a constant value to each RGB component, shifting brightness uniformly.</p>
  
  <p><strong>Note:</strong> This operation can push values beyond [0, 1], which may require clamping or tone-mapping.</p>

  <h2>Saturation Control</h2>

  <h3>effectSaturation</h3>
  
  <pre><code class="language-glsl">vec3 effectSaturation(vec3 color, float value)
{
    float gray = dot(color, vec3(0.299, 0.587, 0.114));
    return mix(vec3(gray), color, 1.0 + value);
}</code></pre>

  <p><strong>Description:</strong> Interpolates between a grayscale value and the original color. The grayscale is derived from luminance.</p>
  
  <ul>
    <li><code>value = 0</code>: fully desaturated</li>
    <li><code>value > 0</code>: oversaturated</li>
    <li><code>value < 0</code>: muted</li>
  </ul>
  
  <p><strong>Formula:</strong></p>

  <h2>Selective Blur</h2>

  <h3>effectSelectiveBlur</h3>
  
  <pre><code class="language-glsl">vec3 effectSelectiveBlur(
    sampler2D tex,
    vec2 uv,
    float radius,
    float centerRadius,
    float transition
) {
    vec2 center = vec2(0.5, 0.5);
    float distanceToCenter = length(uv - center);

    float blurStrength = smoothstep(centerRadius, centerRadius + transition, distanceToCenter);
    if (blurStrength <= 0.0) return texture2D(tex, uv).rgb;

    vec3 sum = vec3(0.0);
    int samples = 0;

    int maxSamples = int(mix(1, 4, blurStrength));
    for(int i = -maxSamples; i <= maxSamples; i++) {
        for(int j = -maxSamples; j <= maxSamples; j++) {
            sum += texture2D(tex, uv + vec2(i, j) * radius * blurStrength).rgb;
            samples++;
        }
    }

    return sum / float(samples);
}</code></pre>

  <p><strong>Description:</strong> Applies a radial blur effect centered at the screen midpoint (0.5, 0.5). Blur strength increases smoothly from centerRadius to centerRadius + transition.</p>
  
  <ul>
    <li><code>blurStrength</code> is modulated by <code>smoothstep</code>, producing a smooth radial transition.</li>
    <li>Number of samples for blur increases proportionally with <code>blurStrength</code>.</li>
  </ul>
  
  <p><strong>Mathematically:</strong></p>
  
  <p>where</p>

  <h2>Output</h2>
  
  <pre><code class="language-glsl">gl_FragData[0] = vec4(color, 1.0);</code></pre>

  <p>After applying selected effects, the result is stored in the output framebuffer.</p>

  <h2>Notes</h2>
  
  <ul>
    <li><strong>Gamma Correction</strong> is not handled here. To render correctly on screen, apply gamma (e.g., <code>pow(color, vec3(1.0/2.2))</code>) in post.</li>
    <li>Most functions assume input in linear RGB.</li>
    <li>Precision and clamping may be required for HDR pipelines.</li>
  </ul>
</div>

<style>
.shader-docs {
  font-family: Arial, sans-serif;
  line-height: 1.6;
  color: #333;
}

.shader-docs h1 {
  color: #2c3e50;
  border-bottom: 2px solid #eee;
  padding-bottom: 10px;
}

.shader-docs h2 {
  color: #34495e;
  margin-top: 30px;
  border-bottom: 1px solid #eee;
  padding-bottom: 5px;
}

.shader-docs h3 {
  color: #7f8c8d;
  margin-top: 20px;
}

.shader-docs pre {
  background-color: #f8f8f8;
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 10px;
  overflow-x: auto;
}

.shader-docs code {
  font-family: 'Courier New', monospace;
  background-color: #f8f8f8;
  padding: 2px 4px;
  border-radius: 3px;
  font-size: 0.9em;
}

.shader-docs ul {
  padding-left: 20px;
}

.shader-docs li {
  margin-bottom: 5px;
}
</style>
