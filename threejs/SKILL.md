---
name: threejs
description: >
  Core Three.js skill for building 3D web experiences. USE when creating 3D scenes,
  loading GLTF/GLB models, setting up WebGL renderers, adding materials and lighting,
  writing GLSL shaders, post-processing effects, animation systems, physics integration,
  instancing, or debugging Three.js performance. Also use when the user says "make
  something 3D", "build a scene", mentions Three.js, WebGL, or asks about meshes,
  cameras, raycasting, or 3D rendering — even if they don't explicitly say "Three.js".
  For WebGPU/TSL shaders, see webgpu-threejs-tsl skill. For ECS game architecture,
  see threejs-ecs skill. For React Three Fiber, see r3f skill (or this skill if r3f
  is not installed).
metadata:
  author: community
  version: "1.0.0"
---

# Three.js — Core 3D Skill

Build 3D web experiences with Three.js. Covers scene setup through production deployment.

## Quick Start (Vanilla)

```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x222222);

const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 1000);
camera.position.set(2, 2, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));  // ALWAYS cap at 2
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// Lighting
scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 1));
const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
dirLight.position.set(5, 10, 7.5);
dirLight.castShadow = true;
dirLight.shadow.mapSize.set(2048, 2048);
dirLight.shadow.normalBias = 0.05;
scene.add(dirLight);

// Mesh
const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x00ff00 })
);
mesh.castShadow = true;
scene.add(mesh);

// Loop
const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  controls.update();
  renderer.render(scene, camera);
}
animate();

// Resize
addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});
```

## Top 10 Gotchas (Know These Cold)

### 1. Memory Leaks — The #1 Three.js Footgun
`scene.remove(mesh)` does NOT free GPU memory. You MUST dispose:
```javascript
mesh.geometry.dispose();
mesh.material.dispose();        // also disposes textures inside
mesh.material.map?.dispose();   // explicitly if needed
scene.remove(mesh);
```
Recursive helper: see `references/patterns.md` "Disposal Helper".

### 2. sRGB Color Space
Color textures (`map`, `emissiveMap`) → `texture.colorSpace = THREE.SRGBColorSpace`
Data textures (`normalMap`, `roughnessMap`, `metalnessMap`, `aoMap`) → leave as default (linear).
Renderer: `renderer.outputColorSpace = THREE.SRGBColorSpace` (default).
**Why:** Renderer does linear workflow — converts sRGB→linear for lighting math→sRGB for display. Without this, colors look washed out.

### 3. Pixel Ratio — ALWAYS Cap at 2
```javascript
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
```
**Why:** DPR 3 = 9x pixels vs DPR 1. Mobile devices report DPR 3-5. Rendering at full res kills performance with imperceptible quality gain.

### 4. Shadow Acne
Use `shadow.normalBias = 0.05` (preferred over `shadow.bias`). Tighten the shadow camera frustum to your scene bounds. Keep `shadow.bias` close to 0.

### 5. Clock.getDelta() — Call Exactly Once Per Frame
`getDelta()` returns time since its last call. Calling it twice = second call returns ~0. Never mix `getDelta()` and `getElapsedTime()` (the latter calls getDelta internally).

### 6. Z-Fighting
Keep near/far ratio tight (`near=0.1, far=100` > `near=0.001, far=100000`). Use `material.polygonOffset = true` for coplanar surfaces. Last resort: `logarithmicDepthBuffer: true` on renderer (performance cost).

### 7. Transparent Object Sorting
Three.js sorts transparent objects by center distance to camera — fails for overlapping/large objects. Fixes: `mesh.renderOrder`, use `alphaTest` instead of `transparent`, set `depthWrite = false` for particles.

### 8. Never Create Objects in the Render Loop
```javascript
// BAD — creates every frame, GC pressure
function animate() { const v = new THREE.Vector3(); ... }

// GOOD — create once, reuse
const v = new THREE.Vector3();
function animate() { v.set(x, y, z); ... }
```

### 9. Frustum Culling
Enabled by default (`object.frustumCulled = true`). If objects disappear when near screen edge, their bounding sphere is too small — call `geometry.computeBoundingSphere()` after modifying vertices.

### 10. renderer.info for Debugging
```javascript
console.log(renderer.info.render.calls);     // draw calls
console.log(renderer.info.render.triangles); // triangle count
console.log(renderer.info.memory.geometries); // geometry count
console.log(renderer.info.memory.textures);   // texture count
```
If geometry/texture counts grow over time → you have a memory leak.

## Materials — When to Use What

| Material | Lighting | PBR | Use Case |
|----------|----------|-----|----------|
| `MeshBasicMaterial` | No | No | Unlit, wireframes, debug, skyboxes |
| `MeshLambertMaterial` | Yes | No | Cheap diffuse (mobile perf) |
| `MeshPhongMaterial` | Yes | No | Shiny surfaces (legacy) |
| `MeshStandardMaterial` | Yes | Yes | **Default choice.** Metalness/roughness PBR |
| `MeshPhysicalMaterial` | Yes | Yes | Glass, car paint, fabric, iridescence (GPU cost per feature) |
| `ShaderMaterial` | Custom | Custom | Custom GLSL with built-in uniforms prepended |
| `RawShaderMaterial` | Custom | Custom | Full GLSL control, nothing prepended |

Start with `MeshStandardMaterial`. Upgrade to Physical only for specific effects.

## Performance Budgets

| Metric | Mobile | Desktop |
|--------|--------|---------|
| FPS | 30+ | 60 |
| Frame time | <33ms | <16ms |
| Draw calls | <100 | <500 |
| Triangles | <100K | <1M |
| Textures (total) | <100MB | <500MB |
| Unique materials | <50 | <200 |

## Key Architecture Patterns

- **Share geometries and materials** across meshes (reduces draw calls)
- **InstancedMesh** for 100+ identical objects (single draw call)
- **BatchedMesh** (r160+) for different geometries + same material (single draw call)
- **LOD** for 2-3 detail levels based on camera distance
- **Object pooling** for frequently spawned/despawned objects
- **Fixed timestep** for physics (accumulator pattern, `fixedDT = 1/60`)
- **Merge static geometry** with `BufferGeometryUtils.mergeGeometries()`

## Latest Three.js State (March 2026)

- **Current release:** r183 (Feb 2025)
- **WebGPU:** Production since r171. All major browsers. Up to 5x over WebGL. r183 adds reversed depth, per-attachment MRT blending, shared BindGroup optimization.
- **TSL:** Introduced r166, stable r171+. r183 adds `clipSpace`, `exponentialHeightFogFactor()`, `retroPass`, StackTrace debugging. Scriptable node removed. Use `webgpu-threejs-tsl` skill for this.
- **BatchedMesh:** r160+. Different geometries, same material, one draw call. r183 adds per-instance opacity + wireframe support.
- **Clock:** Deprecated in r183 → use `Timer` instead.
- **GLTFLoader:** r183 adds `KHR_meshopt_compression` support.
- **USDLoader:** Major expansion in r183 — USDC format, animation, OpenPBR Surface.
- **OrbitControls:** r183 exposes `pan()`/`rotate()`/`dolly()` methods, adds `cursorStyle`.
- **Lambert/Phong:** r183 adds `scene.environment` IBL support (was Physical-only).

## Reference Files

| File | When to Load |
|------|-------------|
| `references/patterns.md` | Loading models, disposal helpers, animation, scroll-driven 3D, raycasting |
| `references/lighting-shadows.md` | Setting up lights, shadow configuration, CSM |
| `references/shaders.md` | Custom GLSL, ShaderMaterial, onBeforeCompile, uniforms |
| `references/post-processing.md` | EffectComposer, bloom, SSAO, FXAA/SMAA, custom passes |
| `references/physics.md` | Rapier, cannon-es integration, fixed timestep |
| `references/instancing.md` | InstancedMesh, BatchedMesh, LOD, merge geometries |
| `references/environment.md` | HDR/EXR loading, PMREMGenerator, scene.environment |

## Related Skills

- `webgpu-threejs-tsl` — WebGPU renderer, TSL node materials, compute shaders
- `threejs-ecs` — ECS game architecture, game systems
- `r3f` — React Three Fiber (if installed)
- `blender` — 3D modeling, export to GLTF/GLB
