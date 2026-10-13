# Firefox 150.0.3 / ANGLE bug: shader fails to compile when a `varying`/`out` is declared after `main` (which is valid and used in this case, thanks to forward declaration of routine using this varying)

## This is solved in Firefox >= 154

**This repository is not very interesting anymore -- the bug is fixed in recent Firefox versions.**

Tested on Linux/x86_64 with:
- Firefox 154.0 (from packages, package version `154.0-1`, from PikaOS repository, based on Debian unstable)
- Firefox 160.0a1 (latest nightly from https://www.firefox.com/en-US/channel/desktop/ )

Tested on Windows/x86_64 with:
- Firefox 157.0.1 (official Firefox downloads)
- Firefox 160.0a1 (as above: latest nightly from https://www.firefox.com/en-US/channel/desktop/ )

In all 4 above cases,
- the testcase https://github.com/michaliskambi/firefox-webgl-angle-varying-bug is fixed.
- also Castle Game Engine examples, build with FPC to WebAssembly, with the workaround _disabled_ (`WebGLMoveMainBlockToEnd` call commented out in https://github.com/castle-engine/castle-engine/blob/0c21dcbc42ba9d9d2b605f518cc962068bcf2030/src/base_rendering/castleglshaders.pas#L1835 ) -> works fine (fog shader, that previously didn't compile, now works OK).

Judging from the version numbers (the fix was _not_ in nightly 153.0b13 from 2026-07-16, but is _in_ 154.0) -> the fix landed at some point during 153.x development, before 154.0.

Screenshots showing it is OK in [fixed_screenshots_new_firefox](fixed_screenshots_new_firefox).

## Running

Run the server, like `python3 -m http.server 8000`, to enable `fetch` from JS.

Then open `http://localhost:8000` in Firefox. You should see shader compile error in console, and the page will be blank. (Tested with Firefox 150.0.3 on Linux/x86-64, on NVidia GPU.)

In Vivaldi (which uses Chromium), the shader compiles and you see a colored triangle. Vivaldi also uses ANGLE and processes the shader under the hood -- but it seems a different (newer?) version of ANGLE has this fixed.

## Overview of the bug

_Castle Game Engine_ generated shaders triggered a bug in ANGLE (version used by Firefox 150.0.3).

The bug happens when you declare a varying (as `out float ...` or `varying float ...` in vertex shader) after the `main` function.

While it seems strange, it's actually valid and the varying _is used in real code_, as `shader.vs` shows. The varying `castle_ClipDistance` used there is accessed by routine `PLUG_vertex_eye_space`. This routine is called from `main`, but it's body is defined after `main`, only the forward declaration is before `main`.

So it's valid, code is used, and everything is _declared_ before use.

Still, the shader compilation in Firefox 150.0.3 fails. ANGLE processes the shader in a way that assumes that all varyings must be declared before `main`. See below for dump proving it.

Reason for ANGLE processing is https://bugzilla.mozilla.org/show_bug.cgi?id=1910306 . They want to preinitialize all variables, to avoid undefined behavior when accessing uninitialized variables. Makes sense, but the code assumes all varyings are declared before `main`, which is not required by GLSL spec, as far as I know.

## Shader dump from Firefox 150.0.3 - showing the bug

Dump of the shader code thanks to `WEBGL_debug_shaders`:

```glsl
#version 150
#extension GL_ARB_explicit_attrib_location : require
uniform mat4 webgl_91ad3fd88a7c1b43;
uniform mat4 webgl_c3999341c978d032;
in vec4 webgl_9862d5949fcdd346;
in vec4 webgl_cfbdf7eaa0ae115a;
out vec4 webgl_6759ee1d414e354d;
out vec4 webgl_9650b14c1178d97e;
out vec3 webgl_6387d1bb3dddf8b3;
void webgl_429bd4f81a9a58ae(const vec4 webgl_d753c1fcedc1b503, const vec3 webgl_9740f06e1c98da8e);
void main(){
  (webgl_c326dd01ea9f6428 = 0.0);
  (gl_Position = vec4(0.0, 0.0, 0.0, 0.0));
  (webgl_6387d1bb3dddf8b3 = vec3(0.0, 0.0, 0.0));
  (webgl_9650b14c1178d97e = vec4(0.0, 0.0, 0.0, 0.0));
  (webgl_6759ee1d414e354d = vec4(0.0, 0.0, 0.0, 0.0));
  vec4 webgl_a0c7ba6d398a0c1b = webgl_9862d5949fcdd346;
  (webgl_9650b14c1178d97e = (webgl_91ad3fd88a7c1b43 * webgl_a0c7ba6d398a0c1b));
  webgl_429bd4f81a9a58ae(webgl_9650b14c1178d97e, webgl_6387d1bb3dddf8b3);
  (webgl_6387d1bb3dddf8b3 = vec3(0.0, 0.0, 1.0));
  (webgl_6759ee1d414e354d = webgl_cfbdf7eaa0ae115a);
  (gl_Position = (webgl_c3999341c978d032 * webgl_9650b14c1178d97e));
}
out float webgl_c326dd01ea9f6428;
uniform vec4 webgl_b3bff7aede8b012e;
void webgl_429bd4f81a9a58ae(const vec4 webgl_d753c1fcedc1b503, const vec3 webgl_9740f06e1c98da8e){
  (webgl_c326dd01ea9f6428 = dot(webgl_b3bff7aede8b012e, webgl_d753c1fcedc1b503));
}
```

vertex shader compile status

FAILED

vertex shader info log

```
0(12) : error C1503: undefined variable "webgl_c326dd01ea9f6428"
```

Fatal error

```
Error: vertex shader compile failed: 0(12) : error C1503: undefined variable "webgl_c326dd01ea9f6428"
```

## Shader dump from Vivaldi 7.9.3970.64 - no bug

ANGLE translated source — vertex shader

```
#version 450
uniform mat4 _ucastle_ModelViewMatrix;
uniform mat4 _ucastle_ProjectionMatrix;
uniform vec4 _ucastle_ClipPlane0;
in vec4 _ucastle_Vertex;
in vec4 _ucastle_MultiTexCoord0;
out vec4 _ucastle_TexCoord0;
out vec4 _ucastle_vertex_eye;
out vec3 _ucastle_normal_eye;
void _uPLUG_vertex_eye_space(const vec4 _uvertex_eye, const vec3 _unormal_eye);
out float _ucastle_ClipDistance;
void _uPLUG_vertex_eye_space(const vec4 _uvertex_eye, const vec3 _unormal_eye){
  (_ucastle_ClipDistance = dot(_ucastle_ClipPlane0, _uvertex_eye));
}
void main(){
  (gl_Position = vec4(0.0, 0.0, 0.0, 0.0));
  (_ucastle_ClipDistance = 0.0);
  (_ucastle_normal_eye = vec3(0.0, 0.0, 0.0));
  (_ucastle_vertex_eye = vec4(0.0, 0.0, 0.0, 0.0));
  (_ucastle_TexCoord0 = vec4(0.0, 0.0, 0.0, 0.0));
  vec4 _uvertex_object = _ucastle_Vertex;
  (_ucastle_vertex_eye = (_ucastle_ModelViewMatrix * _uvertex_object));
  _uPLUG_vertex_eye_space(_ucastle_vertex_eye, _ucastle_normal_eye);
  (_ucastle_normal_eye = vec3(0.0, 0.0, 1.0));
  (_ucastle_TexCoord0 = _ucastle_MultiTexCoord0);
  (gl_Position = (_ucastle_ProjectionMatrix * _ucastle_vertex_eye));
}
```

vertex shader compile status

```
OK
```
vertex shader info log

```
(empty info log)
```

## ANGLE versions

ANGLE version in Vivaldi seems newer:
- emits `#version 450`
- emits easier to read variable names (like `_ucastle_ClipDistance` instead of `webgl_c326dd01ea9f6428`)
- moves (hoists) the declaration of `out float _ucastle_ClipDistance` before `main`, which is valid and fixes the bug

## Notes

The HTML and JS inside `index.html` are mostly Claude-generated, throwaway test code.

The `shader.vs` comes from real _Castle Game Engine_ generated code.

