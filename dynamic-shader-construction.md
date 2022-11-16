# WebGPU dynamic shader construction best practices

Authoring shaders are a major part of working with any modern GPU API, and WebGPU is no different. If you are familiar with other shading languages such as GLSL or HLSL, however, you may find that some language features used to alter the shaders at compile time are not present in WebGPU's shading language, WGSL. This includes preprocessor defines and conditionals.

This doc will help guide you through some easy ways to replicate the behavior of the various preprocessor statements in other shading languages when using WGSL, as well as some generally helpful patterns for constructing WGSL shaders dynamically.

## Define WGSL code in JavaScript, rather than standalone files.

It's very common to see tutorials or samples of GPU APIs where the shader code is placed in separate files with names like `default.wgsl` or `basic.vertex.wgsl`. The WebGPU code will then load the contents of these files as plain text and pass it directly to `device.createShaderModule()`, like so:

```rs
// basic_fragment.wgsl
@fragment
fn fragmentMain() -> @location(0) vec4<f32> {
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

const basicFragment = await loadShaderModuleFromFile('./basic_fragment.wgsl');
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

For one, template literals (delimited with backticks instead of single or double quotes) allow for multi-line strings, as seen in the example code above. This makes authoring shader strings in a JavaScript file much easier!

Second, template literals allow for [string interpolation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#string_interpolation), which performs substitutions in the string with JavaScript variables.

These can be used as a very easy replacement for shader code that may have otherwise relied on `#define` statements to easily inject constant values into the shader. For example, consider the following WebGL code:

```glsl
// basic_fragment.glsl
void main() {
  gl_FragColor = return vec4(1.0, 0.0, 0.0, ALPHA);
}
```

```js
// Application JavaScript
async function basicFragmentSrc(alpha = 1.0) {
  const code = await fetch('./basic_fragment.glsl').text();

  let defines = '#define ALPHA ' + alpha + '\n';
  return defines + code;
}

// A fragment shader which outputs opaque red values
const basicFragment = gl.createShader(gl.VERTEX_SHADER);
gl.shaderSource(basicFragment, await basicFragmentSrc());
gl.compileShader(basicFragment);

// A fragment shader which outputs partially transparent red values
const basicAlphaFragment = gl.createShader(gl.VERTEX_SHADER);
gl.shaderSource(basicAlphaFragment, await basicFragmentSrc(0.5));
gl.compileShader(basicAlphaFragment);
```

If you've done much with WebGL you've likely run into a similar pattern before. The shader makes use of a fuzzily defined `ALPHA` "constant" that is supplied at compile prepending one or several `#define` lines to the beginning of the shader source. This works, but it's a little awkward to use in practice. It creates a disconnect between the code using the define and the code that's supplying it.

By using template literals and string interpolation, though, we can create a much more direct and visible association between the compile-time "constants" used by the shader and the functions that supply them.

```js
// basic_fragment.js
export function basicFragmentSrc(alpha = 1.0) {
  return `
    @fragment
    fn fragmentMain() -> @location(0) vec4<f32> {
      return vec4<f32>(1.0, 0.0, 0.0, ${alpha});
    }
  `;
}
```

```js
// Application JavaScript
import { basicFragmentSrc } from './basic_fragment.js';

// A fragment shader which outputs opaque red values
const basicFragment = device.createShaderModule({
  code: basicFragmentSrc()
});

// A fragment shader which outputs partially transparent red values
const basicAlphaFragment = device.createShaderModule({
  code: basicFragmentSrc(0.5)
});
```

## Use JavaScript modules string interpolation to reuse shader fragments

Another common pattern for applications with large shaders is to define common functions, structs, or constants in one file and then include them in others. Most shading languages have no facility for this, but we can use JavaScript imports and string interpolation to build out a library of common shader functionality quite elegantly:

```js
// shader_common.js
export const mathDefines = `
  const PI : f32 = ${Math.PI};
  const DEG_TO_RAD : f32 = ${180.0 / Math.PI};
`;

export const srgbUtils = `
  const INV_GAMMA = vec3(${1.0 / 2.2});
  fn linearToSrgb(color : vec3<f32>) {
    return pow(linear, INV_GAMMA);
  }
`;

export const cameraStruct = `
  struct Camera {
    projection : mat4x4<f32>,
    view : mat4x4<f32>,
    position : vec3<f32>,
  };
`
```

```js
// basic_shader.js
import { srgbUtils, cameraStruct } from './shader_common.js';

export const basicShaderSrc = `
    ${cameraStruct}
    @group(0) @binding(0) var<uniform> camera : Camera;

    @fragment
    fn fragmentMain(@location(0) position : vec4<f32>) -> @builtin(position) vec4<f32> {
      return camera.projection * camera.view * position;
    }

    ${srgbUtils}

    @fragment
    fn fragmentMain() -> @location(0) vec4<f32> {
      let srgbColor = linearToSrgb(vec3(1.0, 0.0, 0.0));
      return vec4<f32>(srgbColor, 1.0);
    }
  `;
}
```

## Use tagged template literals for more advanced processing

Another common use for `#define` statements in WebGL is to use them to select from different branches of shader code, like so:

```glsl
// basic_fragment.glsl
void main() {
  #ifdef BLUE
    gl_FragColor = return vec4(0.0, 0.0, 1.0, 1.0);
  #else
    gl_FragColor = return vec4(1.0, 0.0, 0.0, 1.0);
  #endif
}
```

```js
// Application JavaScript
async function basicFragmentSrc(blue = false) {
  const code = await fetch('./basic_fragment.glsl').text();

  let defines;
  if (blue) {
    defines += '#define BLUE 1\n';
  }
  return defines + code;
}

// A fragment shader which outputs red
const basicFragment = gl.createShader(gl.VERTEX_SHADER);
gl.shaderSource(basicFragment, await basicFragmentSrc());
gl.compileShader(basicFragment);

// A fragment shader which outputs blue
const basicAlphaFragment = gl.createShader(gl.VERTEX_SHADER);
gl.shaderSource(basicAlphaFragment, await basicFragmentSrc(true));
gl.compileShader(basicAlphaFragment);
```

It's a little awkward to read, but it gets the job done and allows for more complex "ubershaders" to be adjusted situationally, often to account for things like missing vertex attributes or taking a cheaper path on mobile devices, etc.

Unfortunately for us, JavaScript doesn't offer the same type of easy replacement for this behavior as it does for using defines as contants. The obvious patterns all tend to make the code harder to read:

```js
// basic_fragment.js
export function basicFragmentSrc(blue = false) {
  let code = `
    @fragment
    fn fragmentMain() -> @location(0) vec4<f32> {
  `;
  if (blue) {
    code += `return vec4<f32>(0.0, 0.0, 1.0, 1.0);\n`;
  } else {
    code += `return vec4<f32>(1.0, 0.0, 0.0, 1.0);\n`;
  }
  code += '}';
  return code;
}
```

Ugh.

Fortunately JavaScript does offer the tools for us to make this better, even if it requires a bit more code. By using [Tagged Template Literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#tagged_templates) we can write code that performs custom processing on a template literal string. Using this I was able to create a very small (~90 line) [wgsl preprocessor library](https://github.com/toji/wgsl-preprocessor) that offers GLSL-like preprocessor conditionals simply by adding `wgsl` to the front of the template literal string:

```js
// basic_fragment.js
import { wgsl } from 'https://cdn.jsdelivr.net/npm/wgsl-preprocessor@1.0/wgsl-preprocessor.js';

export function basicFragmentSrc(blue = false) {
  return wgsl`
    @fragment
    fn fragmentMain() -> @location(0) vec4<f32> {
      #ifdef ${blue}
        return vec4<f32>(0.0, 0.0, 1.0, 1.0);
      #else
        return vec4<f32>(1.0, 0.0, 0.0, 1.0);
      #endif
    }
  `;
}
```

That's far easier to read! You're more than welcome to use that library in your own code, or reference it in order to build your own custom tags that suit your needs better. Tagged template literals are powerful tools that can do a lot more than just this sort of simple conditional handling.

## Have fun, and make cool stuff!

This was a shorter doc, but only because JavaScript already offers so many powerful ways to work with strings out of the box. It makes it easy to build out complex WGSL shader libraries without relying on features of the WGSL language to accomodate it. These patterns are really easy to make use of in your own application, and it usually takes just a little nudge to be able to make the leap from a more `#define`-heavy GLSL approach.

And it's worth mentioning that **all of the above techniques ALSO apply to building GLSL shaders too**! That should make the transition easier when building apps that have both a WebGL and WebGPU rendering path.

Good luck on whatever projects are ahead of you, I can't wait to see what the spectacularly creative web community builds!
