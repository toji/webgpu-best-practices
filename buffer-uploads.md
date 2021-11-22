# WebGPU Buffer upload best practices

In WebGPU, the `GPUBuffer` is one of the primary objects you'll be working with. It, along side `GPUTexture`s represent the majority of the data that your application pushes to the GPU for rendering. In WebGPU buffers are used for Vertex and Index data, uniforms, generic storage for compute and fragment shaders, and as a staging area for texture data.

// TODO

This doc is focused primariy on how to get data into these buffers, regardless of their ultimate use.

## Buffer data flow

// TODO. Talk about how data gets into and out of buffers, and what mapping and unmapping mean.

## When in doubt, `writeBuffer()`!

The very first thing to establish is that if you ever have a question about what the best way is to get data into a partiuclar buffer, the `writeBuffer()` method is always a safe fallback that doesn't have many downsides.

`writeBuffer()` is a convenience method on `GPUQueue` that copies values from an `ArrayBuffer` into a `GPUBuffer` in whatever way the user agent deems best. Generally this will be a fairly efficeint path, and in some cases it can be the _most_ efficient path!

Specifically, if you are using WebGPU from WASM code, `writeBuffer()` is the preferred path. This is because WASM apps would need to perform an additional copy from the WASM heap when you use a mapped buffer.

In summary, the advantages of using `writeBuffer()` are:
 - Preferred route for WASM apps.
 - Lowest overall code complexity.
 - Sets the buffer data immediately.
 - If your data is already in an `ArrayBuffer`, avoids allocation/copy of a mapped `ArrayBuffer`.
 - Avoids the need to set the contents of a mapped buffer's array to zero before returning it.
 - Allows the user agent to pick an (presumably optimal) pattern for uploading the data to the GPU.

The (fairly minor) disadvantages to this path are:
 - Requires that the buffer is created with `COPY_DST` usage.

Here's an example of using `writeBuffer()`. You can see the code is very brief:

```js
// At some point during the app startup...
const projectionMatrixBuffer = gpuDevice.createBuffer({
  size: 16 * Float32Array.BYTES_PER_ELEMENT, // Large enough for a 4x4 matrix
  usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST, // COPY_DST is required
});

// Whenever the projection matrix changes (ie: window is resized)...
function updateProjectionMatrixBuffer(projectionMatrix) {
  const projectionMatrixArray = projectionMatrix.getAsFloat32Array();
  gpuDevice.queue.writeBuffer(projectionMatrixBuffer, 0, projectionMatrixArray, 0, 16);
}
```

## Buffers that are written once and never change

There's many cases where you will create a buffer who's contents need to be set once on creation and then never changed again. A simple example would be the vertex and index buffers for a static mesh: The buffer itself needs to be filled with the mesh data immediately after the buffer is created, after which any changes to the mesh during the render loop will be done with a transform matrix or maybe mesh skinning the the vertex shader. The only time the buffer contents will change after it's initially set is when it's eventually destroyed.

In this case, one of the best ways to set the buffer's data is using the `mappedAtCreation` flag when calling `createBuffer()`. This creates the buffer in a mapped state, so that `getMappedRange()` can be called immediately after creation. This provides an `ArrayBuffer` to be filled, after which `unmap()` is called and the buffer data is set! In practice the browser will almost certainly need to do a copy of the array buffer contents to the GPU in the background after `unmap()` is called, but you can generally be assured that it's done in an efficient manner.

One advantage of this approach is that if your buffer data is being created dynamically, you can save on at least one CPU-side copy by generating the data directly into the mapped buffer.

Another perk of `mappedAtCreation` is that it can be used even on buffers that don't have `COPY_DST` usage or some other way to set the data. This allows the buffer to act almost like a `const` variable that's set once at creation and then becomes immutable. (Whether or not that has potential performance benefits is unclear.)

// TODO: IS there any benefit?

Advantages to this approach:
 - Sets the buffer data immediately.
 - No specific usage flags are required.
 - Data can be written directly into the mapped buffer to avoid a CPU-side copy.

Disadvantages:
 - Only works for newly created buffers.
 - User agent must zero out the buffer before it's mapped.
 - If data is already in an `ArrayBuffer`, requires another CPU-side copy.

Here's an example of using `mappedAtCreation` to set some static vertex data:

```js
// Creates a grid of vertices on the X, Y plane
function createXYPlaneVertexBuffer(width, height) {
  const vertexSize = 3 * Float32Array.BYTES_PER_ELEMENT; // Each vertex is 3 floats (X,Y,Z position)

  const vertexBuffer = gpuDevice.createBuffer({
    size: width * height * vertexSize, // Allocate enough space for all the vertices
    usage: GPUBufferUsage.VERTEX, // COPY_DST is not required!
    mappedAtCreation: true,
  });

  const vertexPositions = new Float32Array(vertexBuffer.getMappedRange()),

  // Build the vertex grid
  for (let y = 0; y < height; ++y) {
    for (let x = 0; x < width; ++x) {
      const vertexIndex = y * width + x;
      const offset = vertexIndex * 3;

      vertexPositions[offset + 0] = x;
      vertexPositions[offset + 1] = y;
      vertexPositions[offset + 2] = 0;
    }
  }

  // Commit the buffer contents to the GPU
  vertexBuffer.unmap();

  return vertexBuffer;
}
```

# Buffers that are written to frequently

If you have buffers that change frequently (such as once per frame) then updaing them efficiently is slightly more complicated. Though before we go any further it should be noted that in most cases using `writeBuffer()` will be a perfectly acceptable path to take from a performance perspective!



Here's an example of how a staging buffer ring could work setting vertex data:

```js
const waveGridSize = 1024;
const waveGridBufferSize = waveGridSize * waveGridSize * 3 * Float32Array.BYTES_PER_ELEMENT;
const waveGridVertexBuffer = gpuDevice.createBuffer({
  waveGridBufferSize,
  usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
});
const waveGridStagingBuffers = [];

// Updates a grid of vertices on the X, Y plane with wave-like motion
function updateWaveGrid(time) {
  // Get a new or re-used staging buffer that's already mapped.
  let stagingBuffer;
  if (readyStagingBuffers.length) {
    stagingBuffer = waveGridStagingBuffers.pop();
  } else {
    stagingBuffer = gpuDevice.createBuffer({
      size: waveGridBufferSize,
      usage: GPUBufferUsage.MAP_WRITE | GPUBufferUsage.COPY_SRC,
      mappedAtCreation: true,
    });
  }

  // Fill in the vertex grid values.
  const vertexPositions = new Float32Array(stagingBuffer.getMappedRange()),
  for (let y = 0; y < height; ++y) {
    for (let x = 0; x < width; ++x) {
      const vertexIndex = y * width + x;
      const offset = vertexIndex * 3;

      vertexPositions[offset + 0] = x;
      vertexPositions[offset + 1] = y;
      vertexPositions[offset + 2] = Math.sin(time + (x + y) * 0.1);
    }
  }
  stagingBuffer.unmap();

  // Copy the staging buffer contents to the vertex buffer.
  const commandEncoder = gpuDevice.createCommandEncoder({});
  commandEncoder.copyBufferToBuffer(stagingBuffer, 0, waveGridVertexBuffer, 0, waveGridBufferSize);
  gpuDevice.queue.submit([commandEncoder.finish()]);

  // Immediately after copying, re-map the buffer. Push onto the list of staging buffers when the
  // mapping completes.
  stagingBuffer.mapAsync(GPUMapMode.WRITE).then(() => {
    waveGridStagingBuffers.push(stagingBuffer);
  });
}
```

Advantages to this approach:
 - Sets the buffer data immediately.
 - Staging buffer re-use means initialization costs are bounded.
 - Data can be written directly into the mapped buffer to avoid a CPU-side copy.

Disadvantages:
 - Higher complexity than other methods.
 - Higher ongoing memory usage.
 - User agent must zero out the staging buffers the first time they are mapped.
 - If data is already in an `ArrayBuffer`, requires another CPU-side copy.

// TODO