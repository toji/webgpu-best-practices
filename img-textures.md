# WebGPU &lt;img&gt;, &lt;canvas&gt;, and &lt;video&gt; texture best practices


If you are familiar with WebGL and have ever loaded a texture from a JPG, PNG, or any other browser-native image format before, chances are very good that the steps you took were something like the following:

```js
function webGLTextureFromImageUrl(gl, url, loadedCallback) {
  const imgElement = new Image(); // Creates an HTMLImageElement
  imgElement.addEventListener('load', () => {
    const texture = gl.createTexture(); // Create a WebGL Texture
    gl.bindTexture(gl.TEXTURE_2D, texture);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, imgElement);
    gl.generateMipmap(gl.TEXTURE_2D);
    loadedCallback(texture);
  });
  imgElement.src = url;
}
```

But if you go searching through the [WebGPU spec](https://gpuweb.github.io/gpuweb/), you'll notice very quickly that there's no mention of `HTMLImageElement` anywhere, nor is there anything about how to generate mipmaps. So how exactly do we load images as textures with WebGPU, and preferrably how do we do it as efficiently as possible? That's what this doc is here to tell you!

## Prefer compressed formats when possible!

Before we begin, it is worth mentioning that, while not the focus of this document, you should **strongly** consider using compressed texture formats when doing graphics work on the web, and specifically a format that can be transcoded to support multiple different platforms like [Basis Universal](https://github.com/BinomialLLC/basis_universal). Compressed formats take up significantly less space on the GPU, upload faster, can help with caching performance, and frequently can be delivered in a file about the same size as a JPG. If you are in control of your application's assets there is very little reason not to use compressed textures.

The mechanics of how to load compressed textures is beyond the scope of this document (I'll probably write a separate doc about it), but you can use libraries such as my [Web Texture Tool library](https://github.com/toji/web-texture-tool) to handle both compressed and uncompressed textures seamlessly and efficently.

That being said, there are still plenty valid reasons for needing to load uncompessed images. For example, you may not have full control over where your assets are coming from. If you were displaying a gallery of images from another page, for example, you'll need to consume the images in whatever format they were originally posted. Similarly you may have a page that displays user-supplied 3D models which have embedded images in them for textures. In these cases there are still a variety of best practices to follow to handle the textures as efficiently as possible, which is the focus of this document.

## Comparing `<img>` formats

For this document we're focusing on loading browser-native image formats as textures. A browser-native image is anything that you can display using an `HTMLImageElement` (`<img>` tag), which includes JPG, PNG, WEBP, GIF, BMP, and possibly more depending on your browser.

Of these, it's only really practical to use JPG, PNG, and WEBP for graphics work.

 - **WEBP** Should be preferred for it's small size, it's ability to be lossy or lossless, and transparency support. Previously it wasn't as widely supported as the other image formats, but any browser that supports WebGPU will have WEBP support as well.
 - **PNG** Will frequently have bigger file sizes but is lossless, supports transparency, and has widespread support in tools.
 - **JPG** doesn't support transparency and is similar in size to WEBP files, but it's the most ubiquitous image format around.

**GIF**s may seem attractive for their animation support, but the quality is very poor, especially when compared to the file size. Also, there is no programatic way to control which GIF frame will be copied to the texture, so using it for an animated texture is not practical. Use video formats for this use case instead.

**BMP** has no redeeming qualities whatsoever. You technically _can_ load them as textures, but just... don't.

## Creating a texture from an image URL

So, you have an image in one of the above formats, how do you make a WebGPU texture out of it?

Rather than accepting an `HTMLImageElement` directly, WebGPU is designed to accept an [`ImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap) instead. This can help with the performance of your page, because creating an `ImageBitmap` causes the image data to be decoded into an GPU-friendly format, and it does so off the main thread.

Once you have an `ImageBitmap`, you can use it to set the contents of a WebGPU texture through the [`device.queue.copyExternalImageToTexture()`](https://gpuweb.github.io/gpuweb/#dom-gpuqueue-copyexternalimagetotexture) method.

With that in mind, if you are starting with a URL of the image that you want load, the best way to get an `ImageBitmap` is not actually to use an `HTMLImageElement` to load the image! Instead, it's more efficent to use the [`fetch()`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) API to download the image as a [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob)". `Blob`s are one of the options you can pass to [`createImageBitmap()`](https://developer.mozilla.org/en-US/docs/Web/API/createImageBitmap), which makes our replacement for the code above very straightforward:

```js
// Defining this as a separate function because we'll be re-using it a lot.
function webGPUTextureFromImageBitmapOrCanvas(gpuDevice, source) {
  const textureDescriptor = {
    // Unlike in WebGL, the size of our texture must be set at texture creation time.
    // This means we have to wait until the image is loaded to create the texture, since we won't
    // know the size until then.
    size: { width: source.width, height: source.height },
    format: 'rgba8unorm',
    usage: GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST
  };
  const texture = gpuDevice.createTexture(textureDescriptor);

  gpuDevice.queue.copyExternalImageToTexture({ source }, { texture }, textureDescriptor.size);

  return texture;
}

async function webGPUTextureFromImageUrl(gpuDevice, url) { // Note that this is an async function
  const response = await fetch(url);
  const blob = await response.blob();
  const imgBitmap = await createImageBitmap(blob);

  return webGPUTextureFromImageBitmapOrCanvas(gpuDevice, imgBitmap);
}
```

## Creating a texture from an `HTMLImageElement` (`<img>` tag)

If you _do_ have an image element, though, it's trivial to get the `ImageBitmap` using `createImageBitmap()` as discussed above. The biggest quirk is that you either need to be sure that the image has finished loading previously OR wait for it to finish, which is a bit clunky:

```js
// Assumes the 
async function webGPUTextureFromImageElement(gpuDevice, imgElement) {
  if (imgElement.complete) {
    const imgBitmap = await createImageBitmap(imgElement);
    return await webGPUTextureFromImageBitmapOrCanvas(gpuDevice, imgBitmap);
  } else {
    // If the image isn't loaded yet we'll wrap the load/error events in a promise to keep the
    // function interface consistent.
    return new Promise((resolve, reject) => {
      imgElement.addEventListener('load', async () => {
        const imgBitmap = await createImageBitmap(imgElement);
        return await webGPUTextureFromImageBitmapOrCanvas(gpuDevice, imgBitmap);
      });
      imgElement.addEventListener('error', reject);
    });
  });
}
```

One caveat to be aware of, though, is that images used by an `HTMLImageElement` may make different assumptions about how they are going to be used which can cause the decode step done by `createImageBitmap()` to be performed synchronously. (This is the current behavior in Chrome.) As a result, **if you can load the image via a URL/`Blob` you should always prefer that.**

## Real world application: glTF

The [glTF 2.0 model format](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0) is quickly becoming the most common format used for displaying 3D models on the web. It's an open standard and based around web-native data formats such as JSON and TypedArrays, so it makes a great real-world use case to show how to apply the above principles.

glTF JSON contains an array of [Images](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#reference-image) that serve as the sources for textures. The core spec allows images to be either JPEG or PNG files, which of course the browser can easily load for us. The glTF spec also says that image source can be specified as:
 - An external URI
 - A [data URI](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs) (A base64 encoded image)
 - A buffer and a mime type

Which sounds like a lot of different edge cases to handle, until you realize that all of those different sources can fairly trivially be loaded as a `Blob`!

We can ignore the difference between external URIs and data URIs, because `fetch()` will handle them for us silently. `fetch()` will helpfully download the image if the URL points to another location, and decode the image if it's given as a data URI and you, as the developer, don't really have to care which was which. 

If, on the other hand, your are given a glTF [BufferView](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#reference-bufferview) and a MIME type, you can construct a `Blob` directly from that. All together it looks something like this:

```js
async function webGPUTextureFromGLTFImage(gpuDevice, gltfJson, imageIndex) {
  const gltfImage = gltfJson.images[imageIndex];
  let blob;
  if (gltfImage.uri) {
    // Image is given as a URI
    const response = await fetch(gltfImage.uri);
    blob = await response.blob();
  } else {
    // Image is given as a bufferView.
    const bufferView = gltfJson.bufferViews[gltfImage.bufferView];
    // The details of how glTF resolves buffers are beyond the scope of this page.
    const buffer = await resolveGLTFBuffer(gltfJson, bufferView.buffer);
    blob = new Blob(
      [new Uint8Array(buffer, bufferView.byteOffset, bufferView.byteLength)],
      { type: gltfImage.mimeType }
    );
  }
  const imgBitmap = await createImageBitmap(blob);
  return webGPUTextureFromImageBitmap(gpuDevice, imgBitmap);
}
```

## Creating a texture from a `HTMLCanvasElement` (`<canvas>` tag) or `OffscreenCanvas`

Another common place you may want to pull texture contents from is a Canvas (either a `HTMLCanvasElement` or `OffscreenCanvas`). Using the contents of a `2d` canvas as a texture can be a great way to generate textures programatically or render text, for example.

Canvases are already able to be passed to `createImageBitmap()`, so at first glance it may seem like the best way to create an image from a canvas is to once again get an `ImageBitmap` and use the same approach as above. We can actually take a shortcut in this case, however!

The [`GPUImageCopyExternalImage dictionary`](https://gpuweb.github.io/gpuweb/#dictdef-gpuimagecopyexternalimage) passed into `copyExternalImageToTexture()` can accept an `HTMLCanvasElement` or `OffscreenCanvas` directly in addition to an `ImageBitmap`, which allows the content of those canvases to be copied into the texture without the need to wait for `createImageBitmap()` to complete. In fact, we can use the `webGPUTextureFromImageBitmapOrCanvas()` we defined above completely unchanged!

```js
// Note: Unlike the other methods above this call doesn't need to be async.
const canvas = document.querySelector('canvas');
const texture = webGPUTextureFromImageBitmapOrCanvas(gpuDevice, canvas);
```

The browser can take a fast path copying content from the canvas because it almost certainly produced the canvas contents with the same GPU backend as is being used for WebGPU.

_Important note: this only applies to canvases with WebGL or Canvas2D contexts._

## Creating a texture from a `HTMLVideoElement` (`<video>` tag);

One final common source of textures is a video tag. As with Canvases, you _can_ pass an `<video>` element to `createImageBitmap()` and populate a texture with the resulting `ImageBitmap`. And in some cases, such as if you only want to capture a single frame of the video, it's your best option. Given that display video as a texture is such a common case, however, and by it's nature requires frequent updating, a fast path specifically for video was added to WebGPU in the form of the [`device.importExternalTexture()`](https://gpuweb.github.io/gpuweb/#dom-gpudevice-importexternaltexture) method.

```js
// This one is so simple it doesn't even warrant a helper function!
const texture = gpuDevice.importExternalTexture({ source: video });
```

`importExternalTexture()` returns a `GPUExternalTexture`, which can be sampled like a texture in your shaders (with some limits, more on that in a moment). It can't, however, have it contents changed, copied from, or otherwise manipulated. This restriction allows the browser to implement `importExternalTexture()` without creating a copy of the video frame, which can represent a big performance boost and memory savings!

Also, due to how browsers implement `GPUExternalTextures` they have some special needs regarding how they're used in shaders. When
declaring the binding the type needs to be `texture_external` and sampling needs to happen with the `textureSampleLevel()` function explicitly. (`GPUExternalTextures` hav no mipmaps to sample from.)

A simple fragment shader that uses an external texture looks like this:

```ts
[[group(0), binding(0)]] var externalTexture : texture_external;
[[group(0), binding(1)]] var textureSampler : sampler;

[[stage(fragment)]]
fn fragMain([[location(1)]] texCoord : vec2<f32>) -> [[location(0)]] vec4<f32> {
  return textureSampleLevel(externalTexture, textureSampler, texCoord);
}
```

## GPUExternalTexture lifetime

Finally, it's important to understand the lifetime of a `GPUExternalTexture` object: Because it attempts to not create a copy of the video frame in question, the contents of a `GPUExternalTexture` would dissapear when the video advances to the next frame. Because the exact timing of video frames is a non-trivial thing to keep track of, WebGPU simplifies things by tightly controlling the external texture's lifetime. The spec says:

```
External textures are destroyed automatically, as a microtask, instead of manually or upon garbage collection like other resources.
```

Which means that, from your program's point of view, all `GPUExternalTexture`s are destroyed _as soon as JavaScript returns to the browser_. Functionally, this means that you **must** do any rendering with a `GPUExternalTexture` in the same callback that you create it in. (FYI: This also technically means that you need to create a new `GPUBindGroup` with the texture every frame if you want to do anything useful with it.)

So this is good, because it uses the external texture in the same callback that creates it:

```js
function onFrame() {
  requestAnimationFrame(onFrame);

  const externalTexture = gpuDevice.importExternalTexture({ source: videoTag });

  // Assume we're using the fragment shader from above.
  const externalTextureBindGroup = gpuDevice.createBindGroup({
    layout: pipeline.getBindGroupLayout(0),
    entries: [{
      binding: 0,
      resource: externalTexture, // Note that you don't need to call .createView()
    }, {
      binding: 2,
      resource: sampler,
    }]
  });

  const commandEncoder = gpuDevice.createCommandEncoder({});
  const passEncoder = commandEncoder.beginRenderPass({
    colorAttachments: [{
      view: context.getCurrentTexture().createView(),
      loadValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
      storeOp: 'store'
    }]
  });

  passEncoder.setPipeline(pipeline);
  passEncoder.setBindGroup(0, externalTextureBindGroup);
  passEncoder.draw(4);
  passEncoder.endPass();

  gpuDevice.queue.submit([commandEncoder.finish()]);
}
requestAnimationFrame(onFrame);
```

But this will fail, because it creates the external texture in an event callback, returns control to the browser, and then tries to use the texture in a different callback:

```js
button.addEventListener('click', () => {
  // BAD! externalTexture will be destroyed as soon as this callback exits.
  externalTexture = gpuDevice.importExternalTexture({ source: videoTag });

  externalTextureBindGroup = gpuDevice.createBindGroup({
    layout: pipeline.getBindGroupLayout(0),
    entries: [{
      binding: 0,
      // Normally bind groups hold on to texture references, preventing them from being destroyed,
      // but that doesn't apply to externalTextures!
      resource: externalTexture, 
    }, {
      binding: 2,
      resource: sampler,
    }]
  });
});

function onFrame() {
  requestAnimationFrame(onFrame);

  const commandEncoder = gpuDevice.createCommandEncoder({});
  const passEncoder = commandEncoder.beginRenderPass({
    colorAttachments: [{
      view: context.getCurrentTexture().createView(),
      loadValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
      storeOp: 'store'
    }]
  });

  passEncoder.setPipeline(pipeline);
  passEncoder.setBindGroup(0, externalTextureBindGroup);
  passEncoder.draw(4); // ERROR! Tried drawing with the destroyed externalTexture.
  passEncoder.endPass();

  gpuDevice.queue.submit([commandEncoder.finish()]);
}
requestAnimationFrame(onFrame);
```

Additionally, using something like `await` between creating the `GPUExternalTexture` and rendering with it will also cause failures, because `await` returns control to the browser and allows microtasks to execute.

```js
async function drawFrame() {
  const externalTexture = gpuDevice.importExternalTexture({ source: videoTag });

  // Oops! This 'await' destroys externalTexture!
  const pipeline = await ensureAsyncPipelineCreated();

  const externalTextureBindGroup = gpuDevice.createBindGroup({
    layout: pipeline.getBindGroupLayout(0),
    entries: [{
      binding: 0,
      resource: externalTexture,
    }, {
      binding: 2,
      resource: sampler,
    }]
  });

  const commandEncoder = gpuDevice.createCommandEncoder({});
  const passEncoder = commandEncoder.beginRenderPass({
    colorAttachments: [{
      view: context.getCurrentTexture().createView(),
      loadValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
      storeOp: 'store'
    }]
  });

  passEncoder.setPipeline(pipeline);
  passEncoder.setBindGroup(0, externalTextureBindGroup);
  passEncoder.draw(4); // ERROR! Tried drawing with the destroyed externalTexture.
  passEncoder.endPass();

  gpuDevice.queue.submit([commandEncoder.finish()]);
}
```

This may seem like a lot of hoops to jump through, but when used properly `GPUExternalTexture` can _significantly_ improve the performance and memory usage of rendering videos with WebGPU!

## Generating Mipmaps

You may have notcied that in all of the above examples one thing that's not addressed is handling mipmaps. In WebGL, once you loaded a texture if you wanted mipmaps (so your texture doesn't look like a shimmery mess from far away) you just called:

```js
gl.generateMipmap(GL.TEXTURE_2D);
```

And, done! Moving on! With WebGPU, on the other hand things are... significantly less simple.

WebGPU requires you to explicitly set every mipmap level of each texture. For some texture types, like compressed textures, it's common to store each mip level separately in the file itself, which makes it relatively easy to populate the full mip chain, but when your image source is a JPEG/PNG/Canvas/etc you're stuck either not using mipmapping at all or generating the mipmaps yourself at runtime.

There's a myriad of ways that you can choose to generate mipmaps, with some of the fancier native libraries going so far as to do single-pass, compute shader-based, custom filtered downsampling. Chances are, though, if you simply want your textures to not sparkle when you move around your scene you'll be just fine with a much simpler solution.

The most straightforward approach is to do a render pass for each mip level, starting at the largest, and render it into the next level down using a linear filter. It's not the absolute fastest or most precise way of going about it, but it's definitely Good Enough™ for many use cases.

If we ignore some basic optimizations like trying to re-use the pipelines and samplers, you end up with something like this:

```js
// TextureDescriptor should be the descriptor that the texture was created with.
// This version only works for basic 2D textures.
function webGPUGenerateMipmap(gpuDevice, texture, textureDescriptor) {
  // Create a simple shader that renders a fullscreen textured quad.
  const mipmapShaderModule = gpuDevice.createShaderModule({
    code: `
      var<private> pos : array<vec2<f32>, 4> = array<vec2<f32>, 4>(
        vec2<f32>(-1.0, 1.0), vec2<f32>(1.0, 1.0),
        vec2<f32>(-1.0, -1.0), vec2<f32>(1.0, -1.0));

      struct VertexOutput {
        [[builtin(position)]] position : vec4<f32>;
        [[location(0)]] texCoord : vec2<f32>;
      };

      [[stage(vertex)]]
      fn vertexMain([[builtin(vertex_index)]] vertexIndex : u32) -> VertexOutput {
        var output : VertexOutput;
        output.texCoord = pos[vertexIndex] * vec2<f32>(0.5, -0.5) + vec2<f32>(0.5);
        output.position = vec4<f32>(pos[vertexIndex], 0.0, 1.0);
        return output;
      }

      [[binding(0), group(0)]] var imgSampler : sampler;
      [[binding(1), group(0)]] var img : texture_2d<f32>;

      [[stage(fragment)]]
      fn fragmentMain([[location(0)]] texCoord : vec2<f32>) -> [[location(0)]] vec4<f32> {
        return textureSample(img, imgSampler, texCoord);
      }
    `
  });

  const pipeline = gpuDevice.createRenderPipeline({
    vertex: {
      module: mipmapShaderModule,
      entryPoint: 'vertexMain',
    },
    fragment: {
      module: mipmapShaderModule,
      entryPoint: 'fragmentMain',
      targets: [{
        format: textureDescriptor.format // Make sure to use the same format as the texture
      }],
    },
    primitive: {
      topology: 'triangle-strip',
      stripIndexFormat: 'uint32',
    },
  });

  // We'll ALWAYS be rendering minified here, so that's the only filter mode we need to set.
  const sampler = gpuDevice.createSampler({ minFilter: 'linear' });

  let srcView = texture.createView({
    baseMipLevel: 0,
    mipLevelCount: 1
  });

  // Loop through each mip level and renders the previous level's contents into it.
  const commandEncoder = gpuDevice.createCommandEncoder({});
  for (let i = 1; i < textureDescriptor.mipLevelCount; ++i) {
    const dstView = texture.createView({
      baseMipLevel: i,  // Make sure we're getting the right mip level...
      mipLevelCount: 1, // And only selecting one mip level
    });

    const passEncoder = commandEncoder.beginRenderPass({
      colorAttachments: [{
        view: dstView, // Render pass uses the next mip level as it's render attachment.
        loadValue: [0, 0, 0, 0],
        storeOp: 'store'
      }],
    });

    // Need a separate bind group for each level to ensure
    // we're only sampling from the previous level.
    const bindGroup = gpuDevice.createBindGroup({
      layout: pipeline.getBindGroupLayout(0),
      entries: [{
        binding: 0,
        resource: sampler,
      }, {
        binding: 1,
        resource: srcView,
      }],
    });

    // Render
    passEncoder.setPipeline(pipeline);
    passEncoder.setBindGroup(0, bindGroup);
    passEncoder.draw(4);
    passEncoder.endPass();

    // The source texture view for the next iteration of the loop is the
    // destination view for this one.
    srcView = dstView;
  }
  gpuDevice.queue.submit([commandEncoder.finish()]);
}
```

In order to use that method, the texture itself needs to be created with some additional options. It needs to be told the number of mip levels it should have at creation time, and it needs to be created with the `RENDER_ATTACHMENT` usage. Those requirements changes our previously defined `webGPUTextureFromImageBitmapOrCanvas()` method to look like this:

```js
function webGPUTextureFromImageBitmapOrCanvas(gpuDevice, source, generateMipmaps = true) {
  const textureDescriptor = {
    size: { width: source.width, height: source.height },
    format: 'rgba8unorm',
    usage: GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST
  };

  if (generateMipmaps) {
    // Compute how many mip levels are needed for a full chain.
    textureDescriptor.mipLevelCount = Math.floor(Math.log2(Math.max(source.width, source.height))) + 1;
    // Needed in order to use render passes to generate the mipmaps.
    textureDescriptor.usage |= GPUTextureUsage.RENDER_ATTACHMENT;
  }

  const texture = gpuDevice.createTexture(textureDescriptor);

  gpuDevice.queue.copyExternalImageToTexture({ source }, { texture }, textureDescriptor.size);

  if (generateMipmaps) {
    webGPUGenerateMipmap(gpuDevice, texture, textureDescriptor);
  }

  return texture;
}
```

I've packaged up [my solution for mipmap generation](https://github.com/toji/web-texture-tool/blob/main/src/webgpu-mipmap-generator.js) as part of my [Web Texture Tool library](https://github.com/toji/web-texture-tool). Feel free to use that library however you see fit! It's also worth noting that the mipmapping portion doesn't have any dependencies on the rest of the library, so you can use it standalone if that suits your needs better.

## Have fun, and make cool stuff!

Hopefully this document helps you get up and running with texture in WebGPU! Despite feeling a little intimidating at first WebGPU is an exciting and powerful API that can actually be fairly easy to use once you've learned some of the best practices. Good luck on whatever projects are ahead of you, I can't wait to see what the spectacularly creative web community builds!
