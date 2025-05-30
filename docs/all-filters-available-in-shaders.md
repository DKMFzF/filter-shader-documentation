<h1>Shader Filter System Documentation</h1>

***Some functions and formulas may be displayed incorrectly on the GitHub page. In order to fully open the documentation, it is better to clone this repository to your computer and open it in your IDE.***

<p>This document describes a GLSL-based image filter system that applies various post-processing effects to image data. Each effect is implemented as a color transformation function, operating in linear RGB space unless otherwise noted.</p>

<h2>Color Mixing Utility</h2>

<h3>make_color</h3>

```glsl
vec3 make_color(in vec3 base_color, in vec3 mix_color, in float amount)
{
  return mix(base_color, mix_color, amount);
}
```

<p><strong>Parameters:</strong></p>
<ul>
  <li><code>base_color</code>: The original color vector (RGB) ∈ [0,1]³</li>
  <li><code>mix_color</code>: The target color for blending ∈ [0,1]³</li>
  <li><code>amount</code>: Blend ratio ∈ [0,1] where 0=full base, 1=full mix</li>
</ul>

<p><strong>Mathematical Formulation:</strong></p>
<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msub><mi>C</mi><mi>out</mi></msub>
  <mo>=</mo>
  <mo>(</mo>
  <mn>1</mn>
  <mo>−</mo>
  <mi>α</mi>
  <mo>)</mo>
  <mo>⋅</mo>
  <msub><mi>C</mi><mi>base</mi></msub>
  <mo>+</mo>
  <mi>α</mi>
  <mo>⋅</mo>
  <msub><mi>C</mi><mi>mix</mi></msub>
</math>
</div>

<br>

<p>Where α is the <code>amount</code> parameter. This linear interpolation forms the foundation for many subsequent effects.</p>

<h2>Sepia Tone</h2>

<h3>sepia</h3>

```glsl
vec3 sepia(in vec3 color)
{
  vec3 sepia = vec3(1.2, 1.0, 0.8);
  float gray = dot(color, vec3(0.299, 0.587, 0.114));
  return make_color(color, vec3(gray) * sepia, 0.75);
}
```

<p><strong>Description:</strong> Simulates the warm tone of vintage photos through a two-stage process:</p>
<ol>
  <li>Luminance extraction using Rec. 709 coefficients</li>
  <li>Application of sepia tint with controlled blending</li>
</ol>

<p><strong>Mathematical Breakdown:</strong></p>
<p>1. Luminance calculation:</p>

<div align="center">
  <math xmlns="http://www.w3.org/1998/Math/MathML">
    <mi>L</mi>
    <mo>=</mo>
    <mn>0.299</mn>
    <msub><mi>C</mi><mi>R</mi></msub>
    <mo>+</mo>
    <mn>0.587</mn>
    <msub><mi>C</mi><mi>G</mi></msub>
    <mo>+</mo>
    <mn>0.114</mn>
    <msub><mi>C</mi><mi>B</mi></msub>
  </math>
</div>

<br>

<p>2. Tint application and blending:</p>

<div align="center">
  <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
    <msub><mi>C</mi><mi>sepia</mi></msub>
    <mo>=</mo>
    <mi>L</mi>
    <mo>⋅</mo>
    <mfenced open="(" close=")">
      <mtable>
        <mtr><mtd><mn>1.2</mn></mtd></mtr>
        <mtr><mtd><mn>1.0</mn></mtd></mtr>
        <mtr><mtd><mn>0.8</mn></mtd></mtr>
      </mtable>
    </mfenced>
  </math>
</div>

<br>

<p>3. Final blending:</p>

<div align="center">
  <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
    <msub><mi>C</mi><mi>out</mi></msub>
    <mo>=</mo>
    <mi>mix</mi>
    <mfenced>
      <msub><mi>C</mi><mi>in</mi></msub>
      <mo>,</mo>
      <msub><mi>C</mi><mi>sepia</mi></msub>
      <mo>,</mo>
      <mn>0.75</mn>
    </mfenced>
  </math>
</div>

<br>

<h2>Grayscale Conversion</h2>

<h3>grayscale</h3>

```glsl
vec3 grayscale(in vec3 color, in float intensity)
{
  float average = (color.r + color.g + color.b) / 3.0;
  return make_color(color, vec3(average), intensity);
}
```

<p><strong>Description:</strong> Converts to grayscale using arithmetic mean (unweighted average) with controllable intensity.</p>

<p><strong>Mathematical Formulation:</strong></p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msub><mi>L</mi><mi>avg</mi></msub>
  <mo>=</mo>
  <mfrac>
    <mrow>
      <msub><mi>C</mi><mi>R</mi></msub>
      <mo>+</mo>
      <msub><mi>C</mi><mi>G</mi></msub>
      <mo>+</mo>
      <msub><mi>C</mi><mi>B</mi></msub>
    </mrow>
    <mn>3</mn>
  </mfrac>
</math>
</div>

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msub><mi>C</mi><mi>out</mi></msub>
  <mo>=</mo>
  <mfenced>
    <mrow>
      <mn>1</mn>
      <mo>−</mo>
      <mi>α</mi>
    </mrow>
  </mfenced>
  <msub><mi>C</mi><mi>in</mi></msub>
  <mo>+</mo>
  <mi>α</mi>
  <mfenced>
    <mtable>
      <mtr><mtd><msub><mi>L</mi><mi>avg</mi></msub></mtd></mtr>
      <mtr><mtd><msub><mi>L</mi><mi>avg</mi></msub></mtd></mtr>
      <mtr><mtd><msub><mi>L</mi><mi>avg</mi></msub></mtd></mtr>
    </mtable>
  </mfenced>
</math>

<h2>Color Inversion</h2>

<h3>invert</h3>

```glsl
vec3 invert(in vec3 color)
{
  return vec3(1.0) - color;
}
```

<p><strong>Description:</strong> Computes the photographic negative of the input color through component-wise subtraction.</p>

<p><strong>Mathematical Definition:</strong></p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msub><mi>C</mi><mi>out</mi></msub>
  <mo>=</mo>
  <mfenced open="(" close=")">
    <mtable>
      <mtr><mtd><mn>1</mn><mo>−</mo><msub><mi>C</mi><mi>R</mi></msub></mtd></mtr>
      <mtr><mtd><mn>1</mn><mo>−</mo><msub><mi>C</mi><mi>G</mi></msub></mtd></mtr>
      <mtr><mtd><mn>1</mn><mo>−</mo><msub><mi>C</mi><mi>B</mi></msub></mtd></mtr>
    </mtable>
  </mfenced>
</math>
</div>

<h2>Film Grain & Vignette</h2>

<h3>oldFilm</h3>

```glsl
vec3 oldFilm(vec3 color, vec2 uv)
{
  // Grain generation
  float grain = fract(sin(dot(uv, vec2(13, 78.2))) * 44000);
  color += grain * 0.25;

  // Vignette effect
  vec2 center = uv - 0.5;
  float vignette = 1.0 - dot(center, center);
  color *= vignette * 1.25;

  return color;
}
```

<p><strong>Description:</strong> Combines procedural noise generation with radial darkening to simulate vintage film characteristics.</p>

<p><strong>Component 1: Grain Generation</strong></p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi>N</mi>
  <mo>=</mo>
  <mi>fract</mi>
  <mfenced>
    <mrow>
      <mn>44000</mn>
      <mo>⋅</mo>
      <mi>sin</mi>
      <mfenced>
        <mrow>
          <mn>13</mn>
          <msub><mi>u</mi><mi>tex</mi></msub>
          <mo>+</mo>
          <mn>78.2</mn>
          <msub><mi>v</mi><mi>tex</mi></msub>
        </mrow>
      </mfenced>
    </mrow>
  </mfenced>
</math>
</div>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msub><mi>C</mi><mi>grain</mi></msub>
  <mo>=</mo>
  <msub><mi>C</mi><mi>in</mi></msub>
  <mo>+</mo>
  <mn>0.25</mn>
  <mi>N</mi>
</math>
</div>

<br>

<p><strong>Component 2: Vignette Effect</strong></p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>V</mi>
  <mo>=</mo>
  <mn>1</mn>
  <mo>−</mo>
  <msup>
    <mfenced>
      <mrow>
        <msup><mi>u</mi><mn>2</mn></msup>
        <mo>+</mo>
        <msup><mi>v</mi><mn>2</mn></msup>
      </mrow>
    </mfenced>
    <mn>1</mn>
  </msup>
</math>
</div>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msub><mi>C</mi><mi>out</mi></msub>
  <mo>=</mo>
  <mn>1.25</mn>
  <mo>⋅</mo>
  <mi>V</mi>
  <mo>⋅</mo>
  <msub><mi>C</mi><mi>grain</mi></msub>
</math>
</div>

<h2>Contrast Adjustment</h2>

<h3>effectContrast</h3>

```glsl
vec3 effectContrast(in vec3 color, in float value)
{
  return 0.5 + (1.0 + value) * (color - 0.5);
}
```

<p><strong>Description:</strong> Performs linear contrast adjustment centered at mid-gray (0.5).</p>

<p><strong>Mathematical Operation:</strong></p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msub><mi>C</mi><mi>out</mi></msub>
  <mo>=</mo>
  <mn>0.5</mn>
  <mo>+</mo>
  <mfenced>
    <mrow>
      <mn>1</mn>
      <mo>+</mo>
      <mi>k</mi>
    </mrow>
  </mfenced>
  <mfenced>
    <mrow>
      <msub><mi>C</mi><mi>in</mi></msub>
      <mo>−</mo>
      <mn>0.5</mn>
    </mrow>
  </mfenced>
</math>
</div>

<br>

<p>Where <code>k</code> is the contrast parameter:</p>
<ul>
  <li><code>k > 0</code>: Increases contrast (steeper slope)</li>
  <li><code>k = 0</code>: Identity operation (no change)</li>
  <li><code>-1 < k < 0</code>: Reduces contrast (shallower slope)</li>
  <li><code>k ≤ -1</code>: Collapses all colors to 0.5 (undefined behavior)</li>
</ul>

<h2>Brightness Adjustment</h2>

<h3>effectBrightness</h3>

```glsl
vec3 effectBrightness(in vec3 color, in float value)
{
  return color + value;
}
```

<p><strong>Description:</strong> Simple additive brightness control.</p>

<p><strong>Mathematical Definition:</strong></p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msub><mi>C</mi><mi>out</mi></msub>
  <mo>=</mo>
  <msub><mi>C</mi><mi>in</mi></msub>
  <mo>+</mo>
  <mi>Δ</mi>
</math>
</div>

<br>

<p>Where Δ is the brightness offset. Note that values may exceed [0,1] range.</p>

<h2>Saturation Control</h2>

<h3>effectSaturation</h3>

```glsl
vec3 effectSaturation(vec3 color, float value)
{
  float gray = dot(color, vec3(0.299, 0.587, 0.114));
  return mix(vec3(gray), color, 1.0 + value);
}
```

<p><strong>Description:</strong> Adjusts color saturation using luminance-preserving interpolation.</p>

<p><strong>Mathematical Process:</strong></p>
<p>1. Luminance extraction (same as sepia):</p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>L</mi>
  <mo>=</mo>
  <mn>0.299</mn>
  <msub><mi>C</mi><mi>R</mi></msub>
  <mo>+</mo>
  <mn>0.587</mn>
  <msub><mi>C</mi><mi>G</mi></msub>
  <mo>+</mo>
  <mn>0.114</mn>
  <msub><mi>C</mi><mi>B</mi></msub>
</math>
</div>

<br>

<p>2. Saturation adjustment:</p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msub><mi>C</mi><mi>out</mi></msub>
  <mo>=</mo>
  <mfenced>
    <mrow>
      <mn>1</mn>
      <mo>+</mo>
      <mi>s</mi>
    </mrow>
  </mfenced>
  <msub><mi>C</mi><mi>in</mi></msub>
  <mo>−</mo>
  <mi>s</mi>
  <mi>L</mi>
</math>
</div>

<br>

<p>Where <code>s</code> is the saturation parameter:</p>
<ul>
  <li><code>s = 0</code>: Full grayscale</li>
  <li><code>s = 1</code>: Original saturation</li>
  <li><code>s > 1</code>: Supersaturated</li>
  <li><code>s < 0</code>: Inverted luminance relationships (artistic effect)</li>
</ul>

<h2>Selective Blur</h2>

<h3>effectSelectiveBlur</h3>

```glsl
vec3 effectSelectiveBlur(
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
}
```

<p><strong>Description:</strong> Implements radial-variable box blur with smooth transition.</p>

<p><strong>Mathematical Components:</strong></p>
<p>1. Radial distance calculation:</p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>d</mi>
  <mo>=</mo>
  <msqrt>
    <msup><mfenced><mrow><mi>u</mi><mo>−</mo><mn>0.5</mn></mrow></mfenced><mn>2</mn></msup>
    <mo>+</mo>
    <msup><mfenced><mrow><mi>v</mi><mo>−</mo><mn>0.5</mn></mrow></mfenced><mn>2</mn></msup>
  </msqrt>
</math>
</div>

<br>

<p>2. Blur strength mapping:</p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>α</mi>
  <mo>=</mo>
  <mfenced>
    <mrow>
      <mfrac>
        <mrow>
          <mi>min</mi>
          <mfenced>
            <mrow>
              <mi>max</mi>
              <mfenced>
                <mrow>
                  <mi>d</mi>
                  <mo>−</mo>
                  <msub><mi>r</mi><mn>0</mn></msub>
                </mrow>
                <mn>0</mn>
              </mfenced>
            </mrow>
            <mi>t</mi>
          </mfenced>
        </mrow>
        <mi>t</mi>
      </mfrac>
    </mrow>
  </mfenced>
</math>
</div>

<br>

<p>3. Sample accumulation:</p>

<div align="center">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <msub><mi>C</mi><mi>out</mi></msub>
  <mo>=</mo>
  <mfrac>
    <mn>1</mn>
    <mi>N</mi>
  </mfrac>
  <munderover>
    <mo>∑</mo>
    <mrow><mi>i</mi><mo>=</mo><mo>−</mo><mi>n</mi></mrow>
    <mi>n</mi>
  </munderover>
  <munderover>
    <mo>∑</mo>
    <mrow><mi>j</mi><mo>=</mo><mo>−</mo><mi>n</mi></mrow>
    <mi>n</mi>
  </munderover>
  <mi>T</mi>
  <mfenced>
    <mrow>
      <mi>u</mi>
      <mo>+</mo>
      <mi>i</mi>
      <mi>α</mi>
      <mi>r</mi>
      <mo>,</mo>
      <mi>v</mi>
      <mo>+</mo>
      <mi>j</mi>
      <mi>α</mi>
      <mi>r</mi>
    </mrow>
  </mfenced>
</math>
</div>

<p>Where:</p>
<ul>
  <li><code>r₀</code> = centerRadius (protected area)</li>
  <li><code>t</code> = transition width</li>
  <li><code>n</code> = ⌊4α⌋ (sample count)</li>
  <li><code>T</code> = texture sampling function</li>
</ul>

<h2>Implementation Notes</h2>

<ul>
  <li><strong>Color Space:</strong> All operations assume linear RGB input</li>
  <li><strong>Precision:</strong> FP32 recommended for HDR workflows</li>
  <li><strong>Gamma Correction:</strong> Apply sRGB conversion post-processing</li>
  <li><strong>Performance:</strong> Selective blur is the most computationally intensive operation</li>
</ul>
