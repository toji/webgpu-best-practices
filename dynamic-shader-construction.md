# WebGPU dynamic shader construction best practices

Authoring shaders are a major part of working with any modern GPU API, and WebGPU is no different. If you are familiar with other shading languages such as GLSL or HLSL, however, you may find that some language features used to alter the shaders at compile time are not present in WebGPU's shading language, WGSL. This includes preprocessor defines and conditionals.

This doc will help guide you through some easy ways to replicate the behavior of the various preprocessor statements in other shading languages when using WGSL, as well as some generally helpful patterns for constructing WGSL shaders dynamically.

## Define WGSL code in JavaScript, rather than standalone files.

It's very common to see tutorials or samples of GPU APIs where the shader code is placed in separate files with names like `default.wgsl` or `basic.vertex.wgsl`. The WebGPU code will then load the contents of these files as plain text and pass it directly to `device.createShaderModule()`, like so:

```rs
// basic.fragment.wgsl
@fragment fn fragmentMain() -> @location(0) vec4<f32> {
  // Return a flat red
  return vec4<f32>(1.0, 0.0, 0.0, 1.0);
}
```

```js
// Application JavaScript
async function loadShaderModuleFromFile(device, url) {
  const code = await fetch(url).text();
  return device.createShaderModule({ code });
}

const basicFragment = await loadShaderModuleFromFile('./basic.fragment.wgsl');
```

Simple, right? This works well for basic cases, and carries with it the benefit of allowing syntax highlighting for the shader code. However, if the shader needs to be more responsive to the application content, it's much more difficult to manipulate the shader strings when they are loaded this way. (Also, to be clear, there's no need for shaders to come from a file with any particular extension or structure. They're just strings.)

Instead, having the shader defined in code offers a lot more flexibility with very little downside. You can even continue keeping them in separate files if it's more convenient for you!

```js
// basic_fragment.js
export const basicFragmentSrc = `
  @fragment fn fragmentMain() -> @location(0) vec4<f32> {
    // Return a flat red
    return vec4<f32>(1.0, 0.0, 0.0, 1.0);
  }
`;
```

```js
// Application JavaScript
import { basicFragmentSrc } from './basic_fragment.js';

const basicFragment = device.createShaderModule({
  code: basicFragmentSrc
});
```

And, best of all, this pattern doesn't require you to asynchronously wait on the code, as JavaScript's own `import` mechanism will handle the loading for you in way that appears synchronous to the code. 

## Use string interpolation in place of defines.

[Template Literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals) are a very powerful, convenient mechanism for string building in JavaScript that can be used to great effect here.

For one, template literals (delimited with backticks `\`` instead of single or double quotes) allow for multi-line strings, as seen in the example code above. This makes authoring shader strings in a JavaScript file much easier!

Second, template literals allow for [string interpolation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#string_interpolation), which performs substitutions in the string with JavaScript variables.

These can be used as a very easy replacement for shader code that may have otherwise relied on `#define` statements to easily inject constant values into the shader. For example, consider the following GLSL code:

```glsl

```

```js
// basic_fragment.js
export function basicFragment(alpha = 1.0) {
  return `
    @fragment
    fn fragmentMain(@location(0) color : vec4<f32>) -> @location(0) vec4<f32> {
      // Return a flat red
      return vec4<f32>(1.0, 0.0, 0.0, ${alpha});
    }
  `;
}
```

```js
// Application JavaScript
import { basicFragmentSrc } from './basic_fragment.js';

const basicFragment = device.createShaderModule({
  code: basicFragmentSrc()
});

const basicAlphaFragment = device.createShaderModule({
  code: basicFragmentSrc(0.5)
});

