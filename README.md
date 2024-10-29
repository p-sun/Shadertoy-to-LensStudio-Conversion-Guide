# Shadertoy to Lens Studio Code Node 5.0 Conversion Guide

Convert Shadertoy shaders to Lens Studio!

First, read the official [Shadertoy to Lens Studio Guide](https://docs.snap.com/lens-studio/4.55.1/references/guides/lens-features/graphics/materials/material-editor/tutorials/shadertoy-to-lens-studio) and watch [Michael Porter's video](https://www.youtube.com/watch?v=1CdKj_kqieY).

This repository includes all the shaders from Micheal Porter's video, recreated in Lens Studio 5.0. All code in this guide is is in this repo.

## [Shadertoy to Code Node (Improved!)](https://codepen.io/PaigeSun/full/ExBYPVQ)

Paste shader code from Shadertoy into the above link, and paste it into a Code Node in Lens Studio. I have forked and added upon Hart Woolery's [original conversion tool](https://codepen.io/2020cv/pen/YzaEBgy) to handle more cases.

# Shadertoy Conversions
## Basic Setup - iResolution / fragCoord / uv

In Shadertoy, our main function looks like this.
```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord/iResolution.xy;
    fragColor = vec4(uv, 0, 1); // Draw red-green gradient

    // Blue circle of radius 1, at 0,0.
    if (abs(length(uv) - 1.0) < 0.04) { 
     fragColor = vec4(0, 0, 1, 1);
    }
}
```

To set that up for Lens Studio, see "1.1 - Circle - Using resolution" in this project.
1. Make a new Screen Image. In the Inspector Panel for the Screen Image, set `Stretch Mode` to `Stretch`.
   * Optionally, for performance and if your shader don't need screen resolution and is a square, set `Stretch Mode` to `Fill` and skip setup from step 4 and on.
2. Select the Screen Image > Select Material > Select the Shader Graph. Setup the Shader Graph like so.

<img src="https://github.com/user-attachments/assets/9ca50187-c3bd-465b-9860-1877520ae126" width="600">

3. Convert the Shadertoy code into [Shadertoy to Code Node (Improved!)](https://codepen.io/PaigeSun/full/ExBYPVQ). In the Shader Graph in Lens Studio, paste that code into the Code Node.

```glsl
// The shader output color
output_vec4 fragColor;
// The shader resolution
input_vec2 resolution;
vec3 getResolution() { if (resolution.x == 0.) return vec3(640,640,1); return vec3(resolution,1);}

#define iResolution getResolution()

void main()
{
    vec2 fragCoord = system.getSurfaceUVCoord0() * iResolution.xy;   
    // Nomalized to [0, 1] on the x-axis. 
    vec2 uv = (2.*fragCoord - iResolution.xy)/iResolution.x;

    // Red-green gradient
    fragColor = vec4(uv, 0, 1);
    
    // Blue circle of radius 1, at 0,0.
    if (abs(length(uv) - 1.0) < 0.04) { 
     fragColor = vec4(0, 0, 1, 1);
    }
}
```
4. In the Inspector Panel for the Material, drag `Device Camera Texture` from the Assets Panel as the `Resolution Texture`. Set `Filtering Mode` to `Nearest`.

<img src="https://github.com/user-attachments/assets/49af9e3d-d0b5-4200-801a-ce5ff6dd8c96" width="300">

### iResolution Tip - Get the resolution without sampling texture
In Lens Studio, `textureSize()` only works when the texture sample contributes to the output. If we don't need to sample the texture, we can add the following in the main() function of our Code Node.

This allows us to simplify the Shader Graph from step 2 above this.

<img src="https://github.com/user-attachments/assets/c68634df-05c1-4cb9-8295-b721ed8daae9" width="600">

```glsl
// Texture used to get the resolution
input_texture_2d resolutionTexture;

void main() {
    //...
    // Add this to only sample one pixel on the first frame.
    if (system.getTimeElapsed() == 0.0 && system.getSurfaceUVCoord0().y == 1.0 && resolutionTexture.textureSize().x == 0.0) {
        fragColor = mix(resolutionTexture.sample(vec2(0)), fragColor, 0.9999999);
    }
}
```

See "1.3 - Circle - Using resolution (fewest nodes)" example in this project.

<img src="https://github.com/user-attachments/assets/5dcc807f-6290-4703-8457-aba5d237e591" width="300">

## texture, textureLod, textureFetch, iChannel

### Pixel Coordinates

Pixel coordinates, fragCoord,, uv, and resolution work exactly the same in Shadertoy and Lens Studio.

Ranges are the same in Shadertoy and LS.
| Variable | Shadertoy | Lens Studio | Range |
| -------- | -------- | ------- | ------- |
| `vec2 pixelCoord` | `vec2(floor(uv * iResolution.xy))` |  `vec2(floor(uv * iResolution.xy))` | [0, resolution - 1]
| `vec2 uv` | `fragCoord/iResolution.xy` or `(pixelCoords + 0.5) / iResolution.xy` | `system.getSurfaceUVCoord0()` or `(pixelCoords + 0.5) / iResolution.xy` | [0, 1] |
| `vec2 fragCoord` | `fragCoord` | `uv * iResolution.xy` | [0.5, iResolution.x - 0.5] |
| `vec3 iResolution` | `iResolution` | `input_texture_2d inputTexture; vec3(inputTexture.textureSize(), 1.0)` | Constant whole number depending on screen size |

Examples for the x-axis for resolution of 4 and 3. The value below the arrow `^` is the position on the horizonal number line.
```
resolution = 4                     |     resolution = 3
             _________________     |                  _____________
pixelCoord   | 0 | 1 | 2 | 3 |     |     pixelCoord   | 0 | 1 | 2 |
             ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾     |                  ‾‾‾‾‾‾‾‾‾‾‾‾‾
             ^   ^   ^   ^   ^     |                  ^   ^   ^   ^
position     0   1   2   3   4     |     position     0   1   2   3
                                   |
               ^   ^   ^   ^       |                    ^   ^   ^  
fragCoord     0.5 1.5 2.5 3.5      |     fragCoord     0.5 1.5 2.5 
                                   |
               ^   ^   ^   ^       |                    ^   ^   ^  
uv            1/8 3/8 5/8 7/8      |     uv            1/6 3/6 5/6
```

### Read and Write to Pixel Coordinates

In Shadertoy, we use `iChannel0`/`iChannel1`/`iChannel2`/`iChannel3` to insert 2D textures, such as images, videos, or 2D buffers. We use `sample` or `sampleLod` as above to read the color at a position. In Shadertoy, we can pass data from one frame to the next by encoding data as a color on a specific pixel, and reading that pixel on the next frame.

See this repo, see [Shadertoy Example](https://www.shadertoy.com/view/XflczH) or "Read and write to pixel" in this repo for the Lens Studio equivalent.

Write to pixel in Shadertoy and Lens Studio:
```glsl
// Draw an green pixel at (30, 30)
vec2 pixelCoord = uvToPixelCoords(uv);	 
if (pixelCoord.x == 30.0 && pixelCoord.y == 30.0) {
    fragColor = vec4(0, 1, 0, 1);
}
```

To read from a pixel in Shadertoy we use `texture`, `textureLod` or `textureFetch`.
```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uvToRead = pixelCoordsToUV(vec2(30.0, 30.0));

    // Read the pixel at (30, 30). These are equivalent.
    vec4 pixelColor = texture(iChannel0, uvToRead);
    vec4 pixelColor = textureLod(iChannel0, uvToRead, 0.0);
    vec4 pixelColor = texelFetch(iChannel0, ivec2(30,30), 0);
```

To read from a pixel in Lens Studio we use `sample` or `sampleLod`.
```glsl
input_texture_2d iChannel0;

void main() {
    vec2 uvToRead = pixelCoordsToUV(vec2(30.0, 30.0));

    // Read the pixel at (30, 30). These are equivalent.
    vec4 pixelColor = iChannel0.sample(uvToRead);
    vec4 pixelColor = iChannel0.sampleLod(uvToRead, 0.0);
```

To set it up in Lens Studio:
1. Create a new Render Target, rename it `Feedback Render Target`. Create a new Orthographic Camera, set it to another layer. Set the camera to `Feedback Render Target`.

![image](https://github.com/user-attachments/assets/702d5840-6640-4a15-b080-c8c7593a85d4)

3. In the Code Node, we declare `iChannel0`.
```glsl
input_texture_2d iChannel0;
```
4. In the Shader Graph, we add a `Texture 2D Object Parameter` node, and set it as the input of `iChannel0` in the Code Node, and set the Title to `Feedback Buffer`.

![image](https://github.com/user-attachments/assets/a00f4b3d-31b6-482a-91d3-0d2fd35e4249)

5. In the Material for the shader, we set `Feedback Buffer` to `Feedback Render Target`.

![image](https://github.com/user-attachments/assets/eab91d64-0d54-48f8-ae54-f90564e82941)

### Important Differences when Reading/Writing to Pixels

- Important buffer size difference:
  1) Shadertoy buffers has  __four channels of 32-bit floats, for a total of 128 bits per pixel__.
  2) Lens Studio has only __four channels of 8-bit floats between 0 and 1, for a total of 32 bits per pixel__. i.e. In Lens Studio, each 8 bit channel only take on 256 different values, from 0.0/255.0, 1.0/255.0, 2.0/255.0 ... 255.0/255.0.
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
We can compare whether some variable `float testValue` is roughly the same in Lens Studio and Shadertoy by mapping that float to a color with `fragColor = floatToColor(testValue);`. This is a useful strategy to see where our Lens Studio shader differs from the original.

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

# Included Shaders

<img src="https://github.com/user-attachments/assets/31b8da85-cbb9-4577-8185-c035225a7ebf" width="300">

https://github.com/user-attachments/assets/66ce250f-952f-4315-bcd9-cd2f77ef542a

https://github.com/user-attachments/assets/7dcf4999-4970-4dde-b9c2-c623cbd28f45

https://github.com/user-attachments/assets/d59f247e-4728-40be-a329-8a4a9d92beb9

# Not Included due to license
Using these strategies, we can even convert [Iq's infamous Rainforest Shader](https://www.shadertoy.com/view/4ttSWf) (top) to run in Lens Studio (bottom).

https://github.com/user-attachments/assets/f157b615-cbd7-41c0-8650-d312fcaca875
