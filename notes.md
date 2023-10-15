# Summary and reference for the step of the WebGPU tutorial

https://codelabs.developers.google.com/your-first-webgpu-app#0

These notes cover the code in the script inside index.html.
Not all lines of code are repeated in this summary. To read in combination with index.html


## Adapter and Device

Adapter represents a specific piece of GPU hardware in WebGPU

```javascript
const adapter = await navigator.gpu.requestAdapter();
```

The device is the interface through which most of the interaction with the GPU happens.

```javascript
const device = await adapter.requestDevice();
```

<br>

## Configuring the canvas:

```javascript
const canvas = document.querySelector("canvas");
const context = canvas.getContext("webgpu");
```

Different devices are optimized for different texture formats. Fortunately,

```javascript
navigator.gpu.getPreferredCanvasFormat();
```

returns the prefered format for the device. We can pass that as the format prop to context.configure({})
We also pass device: device,

Unlike WebGL, canvas creation is separate from device creation. One device can render several canvasses.

<br>

## Giving the GPU Commands with render passes:

```javascript
const encoder = device.createCommandEncoder();

const pass = encoder.beginRenderPass({
  colorAttachments: [
    {
      view: context.getCurrentTexture().createView(),
      loadOp: "clear",
      clearValue: { r: 0, g: 0.5, b: 0.5, a: 1 },
      storeOp: "store",
    },
  ],
});
```

loadOp: "clear" clears the texture when the render pass starts
storeOp: "store" saves drawing done during the render pass to the texture

```javascript
pass.end();
```

ends the render pass. These calls don't make the GPU do anything, but they record commands for the GPU to execute later.

We need to create a GPUCommandBuffer, like so:

```javascript
const commandBuffer = encoder.finish();
```

And then submit it to the queue:

```javascript
device.queue.submit([commandBuffer]);
```

or in one step:

```javascript
device.queue.submit([encoder.finish()]);
```

The queue executes an array of commands in order, and takes care of synchronization. In this case we only have one command.

Control then returns to the browser. It sees the texture of the context has changed and updates the canvas. To update the canvas again, we record and submit a new command buffer. In that case we call

```javascript
context.getCurrentTexture();
```

again, for a new render pass.

<br>

## How GPUs draw

WebGPU has only a few basic shapes (prinitives): points lines and triangles.

Most things GPUs draw are split up into predicatble and easy to process TRIANGLES!

Triangles are defined by their corner points, or vertices. (singular vertex)

Small programs called vertex shaders transform the vertices into **clip space** (the screen from -1 to +1 on Y and X axes)
"For example, the shader may apply some animation or calculate the direction from the vertex to a light source."

Then the GPU decides which pixels to assign to each triangle.

Then a fragment shader determines the color properties, possibly containing complicated calculations.

For 3d content, a series of matrices is used to transform positions prior to drawing, to create the perception of depth and volume.

<br>

## Buffer with vertex data

To define our vertices, we use Typed Arrays. These are arrays of values of a select type and size. For example:

- **Uint8Array**: Values can range from 0 to 255.

- **Float32Array**: Values can be any 32-bit floating point number, ranginf gtom small fractions, large numbers.

```javascript
const vertices = new Float32Array([vertex, vertex, vertex]);
```

GPUs are highly optimized and don't work with JS Typed Arrays. We need to create a Buffer, a memory object easily read by the GPU.

```javascript
device.createBuffer({});
```

We pass various props, label, size and usage.

The label helps with error messages.
In our example we use our JS Typed Array 'vertices' in size.

```javascript
    size: vertices.byteLength,

```

byteLength is a method which calculates the byte size for us based on the amount of values in the array.

we also specify the usage prop, with one or more of the [GPUBufferUsage flags](https://gpuweb.github.io/gpuweb/#buffer-usage)

When the buffer is created, its memory initialized to zero. The easiest way to change the content is calling:

```javascript
dedevice.queue.writeBuffer();
```

<br>

## Define the vertex layout

To draw anything with the vertex data in the buffer, we need to tell WebGPU about the structure of the data.

```javascript
const vertexBufferLayout = {};
```

We pass an arrayStride prop (8), the number of bytes to skip.
Each vertex is made up of a x and y coordinate (32-bit floats) which are 4 bytes each.

There is also the attributes property, which is an array of attribute objects. 
The example has only one attribute but more advanced cases may contain more. 

The attribute has a format prop, an offset (whish you worry about when there are more attributes),
and a shaderLocation, a number between 0-15, which should be unique for each attribute defined. 

We set these values here, as we define our verticed, but we don't pass them to the WebGPU API yet. 

## Shaders 

Shaders are written in WGSL (WebGPU Shading Language), we pass this as the code prop to:

```javascript
device.createShaderModule({code, ``})
```

A vertex shader is a function, called once for each vertex, so 6 times in our example. 
Each time, the vertex shader function returns a corresponding position in clip space. 
They run parellel, many hundreds at a time, but they run independently of each other. 

    @vertex
    fn vertexMain() -> @builtin(position) vec4f {
    }

 - -> indicates this is what the function returns
 - @builtin(position) marks that the returned value is the required position 
 - vec4f is a 4 dimensional vector type


## Points to remember:

```javascript
if (!navigator.gpu) {
  throw new Error("WebGPU not supported on this browser.");
}

if (!adapter) {
  throw new Error("No appropriate GPUAdapter found.");
}
```
