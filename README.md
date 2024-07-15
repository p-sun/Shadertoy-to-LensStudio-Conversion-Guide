# Shadertoy to Lens Studio Code Node 5.0 Conversion Guide

Convert Shadertoy shaders to Lens Studio!

First, read the official [Shadertoy to Lens Studio Guide](https://docs.snap.com/lens-studio/4.55.1/references/guides/lens-features/graphics/materials/material-editor/tutorials/shadertoy-to-lens-studio) and watch [Michael Porter's video](https://www.youtube.com/watch?v=1CdKj_kqieY).

This repository includes all the shaders from Micheal Porter's video, recreated in Lens Studio 5.0. All code in this guide is is in this repo.

## [Shadertoy to Code Node (Improved!)](https://codepen.io/PaigeSun/full/ExBYPVQ)

Paste shader code from Shadertoy into the above link, and paste it into a Code Node in Lens Studio. I have forked and added upon Hart Woolery's [original conversion tool](https://codepen.io/2020cv/pen/YzaEBgy) to handle more cases.

# Shadertoy Conversions
## iResolution / fragCoord / uv

Shadertoy:
```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord/iResolution.xy;
```

Lens Studio:
```glsl
input_texture_2d inputTexture;

vec3 getResolution() { 
    vec2 size = inputTexture.textureSize();
    return (size.x == 0.0) ? vec3(640.0, 640.0, 1.0) : vec3(size, 1.0);
}

#define iResolution getResolution()

void main()
{
    vec2 uv = system.getSurfaceUVCoord0();
    vec2 fragCoord = uv * iResolution.xy;
```

#### (Option A) If your shader is a square:
- In the Inspector Panel for the Screen Image, set `Stretch Mode` to `Fill`.

#### (Option B) If your shader needs the screen resolution:
- In the Inspector Panel for the Screen Image, set `Stretch Mode` to `Stretch`.
- In the Shader Graph, add a `Texture 2D Object Parameter` Node, and make that the input of `Input Texture` declared above.
- In the Inspector Panel for the Material, drag `Device Camera Texture` from the Assets Panel as the `Input Texture`. ** 

##### (Option B **) Fix if you're not seeing `Input Texture` in the Material Inspector Panel
In Lens Studio, `textureSize()` is only correct when you sample the texture and it contributes to the final output. You can get the correct resolution without using the pixel colors in the output by adding this in `main()`.

Lens Studio:
```glsl
input_texture_2d inputTexture;

void main()
{
    // textureSize() only works when the texture contributes to the output.
    // This allows us to get the textureSize() without affecting the output.
    vec2 uv = system.getSurfaceUVCoord0();
    if (uv.x == 1.0 && uv.y == 1.0) {
        vec4 sampledCol = inputTexture.sample(vec2(0.0));
	    fragColor = mix(sampledCol, fragColor, 0.9999999);
    }
```

## texture, textureLod, textureFetch

### Pixel Coordinates

Pixel coordinates, fragCoord,, uv, and resolution work exactly the same in Shadertoy and Lens Studio.

Ranges are the same in Shadertoy and LS.
| Variable | Shadertoy | Lens Studio | Range |
| -------- | -------- | ------- | ------- |
| `vec2 pixelCoord` | `vec2(floor(uv * iResolution.xy))` |  `vec2(floor(uv * iResolution.xy))` | [0, resolution - 1]
| `vec2 uv` | `fragCoord/iResolution.xy` or `(pixelCoords + 0.5) / iResolution.xy` | `system.getSurfaceUVCoord0()` or `(pixelCoords + 0.5) / iResolution.xy` | [0, 1] |
| `vec2 fragCoord` | `fragCoord` | `uv * iResolution.xy` | [0.5, iResolution.x - 0.5] |
| `vec3 iResolution` | `iResolution` | `input_texture_2d inputTexture; vec3(inputTexture.textureSize(), 1.0)` | Constant whole number depending on screen size |

Examples:
```
resolution = 4                          resolution = 3
             _________________                       _____________
pixelCoord   | 0 | 1 | 2 | 3 |          pixelCoord   | 0 | 1 | 2 |
             ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾                       ‾‾‾‾‾‾‾‾‾‾‾‾‾
             ^   ^   ^   ^   ^                       ^   ^   ^   ^
position     0   1   2   3   4          position     0   1   2   3

               ^   ^   ^   ^                           ^   ^   ^  
fragCoord     0.5 1.5 2.5 3.5           fragCoord     0.5 1.5 2.5 

               ^   ^   ^   ^                           ^   ^   ^  
uv            1/8 3/8 5/8 7/8           uv            1/6 3/6 5/6
```

### Read and Write to Pixel Coordinates

In Shadertoy, we can pass data from one frame to the next by encoding data as a color on a specific pixel, and reading that pixel on the next frame.

See this [Shadertoy Example](https://www.shadertoy.com/view/XflczH) or "Read and write to pixel" in this repo for the Lens Studio equivalent.


Write to pixel in Shadertoy and Lens Studio:
```glsl
// Draw an green pixel at (30, 30)
vec2 pixelCoord = uvToPixelCoords(uv);	 
if (pixelCoord.x == 30.0 && pixelCoord.y == 30.0) {
    fragColor = vec4(0, 1, 0, 1);
}
```

Reading from a pixel Shadertoy uses `texture`, `textureLod` or `textureFetch`.
```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uvToRead = pixelCoordsToUV(vec2(30.0, 30.0));

    // Read the pixel at (30, 30). These are equivalent.
    vec4 pixelColor = texture(iChannel0, uvToRead);
    vec4 pixelColor = textureLod(iChannel0, uvToRead, 0.0);
    vec4 pixelColor = texelFetch(iChannel0, ivec2(30,30), 0);
```

Reading from a pixel in Lens Studio uses `sample` or `sampleLod`.
```glsl
input_texture_2d iChannel0;

void main() {
    vec2 uvToRead = pixelCoordsToUV(vec2(30.0, 30.0));

    // Read the pixel at (30, 30). These are equivalent.
    vec4 pixelColor = iChannel0.sample(uvToRead);
    vec4 pixelColor = iChannel0.sampleLod(uvToRead, 0.0);
```

### Important Differences for Reading/Writing to Pixels
* Shadertoy buffers can save 32 bit floats on 4 channels for a __total of 128 bits per pixel__. but Lens Studio can only save 8 bit on 4 channels for a __total of 32 bits per pixel__. In Lens Studio, each 8 bit channel only take on 256 different values, from 0.0/255.0, 1.0/255.0, 2.0/255.0 ... 255.0/255.0.
* If we need to precisely save all 32 bits of a float, in Lens Studio we can pack that into 4 pixels. Set min and max for values specific to your shader.
    ``` glsl
    // Pack a 32-bit signed float into a Lens Studio rgba color.
    vec4 system.pack32Bit( float value, float min, float max )

    // Unpack a Lens Studio rbga color into a 32-bit signed float.
    float system.unpack32Bit( vec4 value, float min, float max )	
    ```
* In order to save data on the alpha channel in the output color in Lens Studio, in the Asset Browser, click the Render Target that the shader outputs to, then in the Inspector Panel,
  * Set the `Clear Color Option` to `None`. If you don't do this, the alpha channel read with `sample` or `sampleLod` will always be 1.0.
  * Set `Filtering Mode` to `Nearest` so sampling will get the original color of a pixel instead of averaging colors from neighbouring pixels.

## iFrame

### (Option A) Faster. If you don't need the specific Frame.

In Lens Studio, add the following. This guarentees that iFrame is a value that increases with each new frame, and that pixels in the same frame get the same `iFrame` value.
```glsl
#define iFrame (int(system.getTimeElapsed()*32.))
```

### (Option B) Slower. If you DO need the specific frame.
1. In the code node of the shader, add a new parameter.
```glsl
input_int frameCount;

#define iFrame frameCount
```

2. In the Shader Graph, add a `Int Parameter` node, set the `Script Name` to `frameCount`, and hook it up to the code node.

3. In an object at the top of the scene hierachy, add the following Script Component. In the Script Component, set the material.
```js
// @input Asset.Material material

var frameCount = 0;

function onUpdate(eventData) {
    script.material.mainPass.frameCount = frameCount;
    frameCount++;
}

script.createEvent("UpdateEvent").bind(onUpdate);
``` 

## Debugging Tip - Comparing Values
We can compare whether `float testValue` is roughly the same in Lens Studio and Shadertoy by mapping that float to a color with `fragColor = floatToColor(testValue);`. This is a quick and useful method to find where Lens Studio shader differs from the original.

```glsl
vec3 hsv2rgb(vec3 c) {
    vec4 K = vec4(1.0, 2.0/3.0, 1.0/3.0, 3.0);
    vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
    return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}

vec4 floatToColor(float value) {
    // Normalize the value to the range [0, 1]
    float hue = log2(value + 1.0) / log2(1001.0);
    vec3 rgb = hsv2rgb(vec3(hue, 1.0, 1.0));
    return vec4(rgb, 1.0);
}
```