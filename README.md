# WebGPU Best Practices

This repo contains several articles that describe best practices for working with various aspects of the WebGPU API.

It should be noted that these pages are NOT a tutorial for getting started with WebGPU, and instead are focused on the
most effective patterns for working with specific parts of the API, especially if you are a developer that's familiar
with WebGL or one of the native APIs. If you're interested in learning WebGPU, check out the following resources first
(and be sure to come back when you've figured out the basics!)

 - [Raw WebGPU](https://alain.xyz/blog/raw-webgpu) - A beautifully presented tutorial
 - [WebGPU Samples](http://austin-eng.com/webgpu-samples) - Good for those that learn by example
 - [WebGPU Spec](https://gpuweb.github.io/gpuweb/) - Heavy reading, but a good reference

## Best Practices

 - [WebGPU &lt;img&gt;, &lt;canvas&gt;, and &lt;video&gt; texture best practices](./img-textures.md) - Covers loading textures from images, canvases, and video elements.
 - [WebGPU buffer upload best practices](./buffer-uploads.md) - Covers pushing data to any type of WebGPU buffer.
 - [Bind Group best practices](./bind-groups.md) - Explains why Bind Groups are structured the way they are and how to make the best use of them.

I've also written a much, much longer article about efficiently displaying glTF files in WebGPU. It's not necessarily a "best practices" doc, but it contains many useful WebGPU tips and patterns nonetheless, as well as working samples!
 - [Efficently rendering glTF models: A WebGPU case study ](https://toji.github.io/webgpu-gltf-case-study/)
