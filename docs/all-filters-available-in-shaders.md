<div>
  <h1>Shader Filter System Documentation</h1>
  
  <p>This document describes a GLSL-based image filter system that applies various post-processing effects to image data. Each effect is implemented as a color transformation function, operating in linear RGB space unless otherwise noted.</p>

  <h2>Color Mixing Utility</h2>

  <h3>make_color</h3>
  
  <pre><code>vec3 make_color(in vec3 base_color, in vec3 mix_color, in float amount)
{
    return mix(base_color, mix_color, amount);
}</code></pre>

  <p><strong>Parameters:</strong></p>
  <ul>
    <li><code>base_color</code>: The original color vector (RGB).</li>
    <li><code>mix_color</code>: The color to blend with.</li>
    <li><code>amount</code>: A float ∈ [0, 1] controlling the blend ratio.</li>
  </ul>
  
  <br>

  <math style="font-size: 35px;" xmlns="http://www.w3.org/1998/Math/MathML">
    <mi>C</mi>
    <mo>=</mo>
    <mo>(</mo>
    <mn>1</mn>
    <mo>−</mo>
    <mi>α</mi>
    <mo>)</mo>
    <mo>⋅</mo>
    <mi>C</mi>
    <msub><mi>base</mi></msub>
    <mo>+</mo>
    <mi>α</mi>
    <mo>⋅</mo>
    <mi>C</mi>
    <msub><mi>mix</mi></msub>
  </math>
  
  <br>

  <p>Where <code>alpha</code> is the amount parameter. Used as a generic tool for gradual color adjustments across different effects.</p>

  <h2>Sepia Tone</h2>

  <h3>sepia</h3>
  
  <pre><code>vec3 sepia(in vec3 color)
{
    vec3 sepia = vec3(1.2, 1.0, 0.8);
    float gray = dot(color, vec3(0.299, 0.587, 0.114));
    return make_color(color, vec3(gray) * sepia, 0.75);
}</code></pre>

  <p><strong>Description:</strong> Simulates the warm tone of vintage photos. First, converts the input color to grayscale using a luminance-weighted average. Then, multiplies by a sepia tint and blends with the original color.</p>
  
  <p><strong>Mathematically:</strong></p>
  <p>Grayscale conversion:</p>
  <p>\[ L = 0.299R + 0.587G + 0.114B \]</p>
  <p>Sepia output:</p>
  <p>\[ C_{\text{out}} = \text{mix}(C_{\text{in}}, (1.2L, 1.0L, 0.8L), 0.75) \]

  <h2>Grayscale</h2>

  <h3>Grayscale Mixing</h3>
  
  <pre><code>float average_color = (color.r + color.g + color.b) / 3.0;
color = make_color(color, vec3(average_color), GRAY_AMOUNT);</code></pre>

  <p><strong>Description:</strong> Reduces saturation by blending original color toward an average of RGB values.</p>
  
  <p>Contrast to luminance-based grayscale (used in sepia), this method treats all RGB channels equally:</p>
  <p>\[ C_{\text{out}} = \text{mix}\left(C_{\text{in}}, \left(\frac{R+G+B}{3}, \frac{R+G+B}{3}, \frac{R+G+B}{3}\right), \alpha\right) \]

  <h2>Invert</h2>

  <h3>invert</h3>
  
  <pre><code>vec3 invert(in vec3 color)
{
    return vec3(1.0) - color;
}</code></pre>

  <p><strong>Description:</strong> Inverts the RGB values. Mathematically:</p>
  <p>\[ C_{\text{out}} = (1 - R, 1 - G, 1 - B) \]</p>
  <p>This operation simulates a photographic negative or "color inverse" filter.</p>

  <h2>Grain & Vignette (Old Film Look)</h2>

  <h3>oldFilm</h3>
  
  <pre><code>vec3 oldFilm(vec3 color, vec2 uv)
{
    float grain = fract(sin(dot(uv, vec2(13, 78.2))) * 44000);
    color += grain * 0.25;

    vec2 center = uv - 0.5;
    float vignette = 1.0 - dot(center, center);
    color *= vignette * 1.25;

    return color;
}</code></pre>

  <p><strong>Description:</strong> Adds procedural grain (pseudo-random noise) and vignette darkening based on distance from the center.</p>
  
  <p>Grain uses a simple hash function via <code>fract(sin(dot(...)))</code> to generate noise:</p>
  <p>\[ N = \text{fract}(\sin(\mathbf{uv} \cdot (13,78.2)) \times 44000) \]
  <p>\[ C_{\text{out}} = C_{\text{in}} + 0.25N \]</p>
  
  <p>Vignette effect fades toward the corners by evaluating:</p>
  <p>\[ V = 1 - \|\mathbf{uv} - 0.5\|^2 \]
  <p>\[ C_{\text{out}} = C_{\text{in}} \times 1.25V \]</p>
  
  <p>The result gives a radial gradient centered on (0.5, 0.5), enhancing the vintage feel.</p>

  <h2>Contrast Adjustment</h2>

  <h3>effectContrast</h3>
  
  <pre><code>vec3 effectContrast(in vec3 color, in float value)
{
    return 0.5 + (1.0 + value) * (color - 0.5);
}</code></pre>

  <p><strong>Description:</strong> Linearly scales the deviation of each channel from 0.5 (mid-gray), effectively increasing or decreasing contrast.</p>
  
  <p><strong>Mathematically:</strong></p>
  <p>\[ C_{\text{out}} = 0.5 + (1 + k)(C_{\text{in}} - 0.5) \]</p>
  <ul>
    <li><code>value > 0</code>: boosts contrast (k > 0)</li>
    <li><code>value = 0</code>: no change (k = 0)</li>
    <li><code>value < 0</code>: reduces contrast (k < 0)</li>
  </ul>

  <h2>Brightness Adjustment</h2>

  <h3>effectBrightness</h3>
  
  <pre><code>vec3 effectBrightness(in vec3 color, in float value)
{
    return color + value;
}</code></pre>

  <p><strong>Description:</strong> Adds a constant value to each RGB component, shifting brightness uniformly.</p>
  <p>\[ C_{\text{out}} = C_{\text{in}} + \Delta \]</p>
  <p>where \(\Delta\) is the brightness adjustment value.</p>
  
  <p><strong>Note:</strong> This operation can push values beyond [0, 1], which may require clamping or tone-mapping.</p>

  <h2>Saturation Control</h2>

  <h3>effectSaturation</h3>
  
  <pre><code>vec3 effectSaturation(vec3 color, float value)
{
    float gray = dot(color, vec3(0.299, 0.587, 0.114));
    return mix(vec3(gray), color, 1.0 + value);
}</code></pre>

  <p><strong>Description:</strong> Interpolates between a grayscale value and the original color. The grayscale is derived from luminance.</p>
  <p>\[ L = 0.299R + 0.587G + 0.114B \]</p>
  <p>\[ C_{\text{out}} = (1 + s)C_{\text{in}} - sL \]</p>
  
  <ul>
    <li><code>value = 0</code>: fully desaturated (s = 0)</li>
    <li><code>value > 0</code>: oversaturated (s > 0)</li>
    <li><code>value < 0</code>: muted (s < 0)</li>
  </ul>

  <h2>Selective Blur</h2>

  <h3>effectSelectiveBlur</h3>
  
  <pre><code>vec3 effectSelectiveBlur(
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
  
  <p><strong>Mathematically:</strong></p>
  <p>\[ d = \|\mathbf{uv} - 0.5\| \]</p>
  <p>\[ \alpha = \text{smoothstep}(r, r + t, d) \]</p>
  <p>\[ C_{\text{out}} = \begin{cases} 
  C_{\text{in}} & \text{if } \alpha = 0 \\
  \frac{1}{N}\sum_{i=-n}^{n}\sum_{j=-n}^{n} T(\mathbf{uv} + \alpha r(i,j)) & \text{otherwise}
  \end{cases} \]</p>
  
  <p>where:</p>
  <ul>
    <li>\( r \) is centerRadius</li>
    <li>\( t \) is transition width</li>
    <li>\( n = \lfloor 4\alpha \rfloor \)</li>
    <li>\( T \) is the texture sampling function</li>
  </ul>

  <h2>Output</h2>
  
  <pre><code>gl_FragData[0] = vec4(color, 1.0);</code></pre>

  <p>After applying selected effects, the result is stored in the output framebuffer.</p>

  <h2>Notes</h2>
  
  <ul>
    <li><strong>Gamma Correction</strong> is not handled here. To render correctly on screen, apply gamma (e.g., <code>pow(color, vec3(1.0/2.2))</code>) in post.</li>
    <li>Most functions assume input in linear RGB.</li>
    <li>Precision and clamping may be required for HDR pipelines.</li>
  </ul>
</div>
