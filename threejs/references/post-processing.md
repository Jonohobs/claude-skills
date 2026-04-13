# Post-Processing (WebGL Path)

For WebGPU/TSL post-processing, use `webgpu-threejs-tsl` skill.

## EffectComposer Setup

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

const composer = new EffectComposer(renderer);
composer.setPixelRatio(Math.min(devicePixelRatio, 2));
composer.addPass(new RenderPass(scene, camera));   // FIRST
// ... effect passes ...
composer.addPass(new OutputPass());                 // LAST (sRGB + tone mapping)

// In loop: composer.render() instead of renderer.render()
// On resize: composer.setSize(width, height)
```

**OutputPass must be last** — it handles sRGB conversion and tone mapping.
**Anti-aliasing must be re-added** — EffectComposer disables default AA. Use SMAA or FXAA pass.

## Common Passes

| Pass | Import | Purpose |
|------|--------|---------|
| `UnrealBloomPass(resolution, strength, radius, threshold)` | `postprocessing/UnrealBloomPass.js` | Glow/bloom effect |
| `SSAOPass(scene, camera, width, height)` | `postprocessing/SSAOPass.js` | Ambient occlusion |
| `OutlinePass(resolution, scene, camera)` | `postprocessing/OutlinePass.js` | Object outlines |
| `SMAAPass(width, height)` | `postprocessing/SMAAPass.js` | Anti-aliasing (better quality) |
| `FXAAPass()` | `postprocessing/ShaderPass.js` + `FXAAShader` | Anti-aliasing (faster) |
| `FilmPass(intensity, grayscale)` | `postprocessing/FilmPass.js` | Film grain |

## Bloom Example

```javascript
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

const bloom = new UnrealBloomPass(
  new THREE.Vector2(innerWidth, innerHeight),
  0.5,    // strength
  0.4,    // radius
  0.85    // threshold
);
composer.addPass(bloom);
```

## Selective Bloom

Bloom only emissive objects:
```javascript
// Technique: render emissive to separate layer, bloom only that
const bloomLayer = new THREE.Layers();
bloomLayer.set(1);

// Objects that should glow
glowMesh.layers.enable(1);

// In render:
// 1. Hide non-bloom objects, render bloom pass
// 2. Show all objects, render normal pass
// 3. Composite
```

## Custom ShaderPass

```javascript
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';

const customPass = new ShaderPass({
  uniforms: {
    tDiffuse: { value: null },  // auto-filled with previous pass output
    uIntensity: { value: 0.5 },
  },
  vertexShader: `
    varying vec2 vUv;
    void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
  `,
  fragmentShader: `
    uniform sampler2D tDiffuse;
    uniform float uIntensity;
    varying vec2 vUv;
    void main() {
      vec4 color = texture2D(tDiffuse, vUv);
      color.rgb = mix(color.rgb, vec3(dot(color.rgb, vec3(0.299, 0.587, 0.114))), uIntensity);
      gl_FragColor = color;
    }
  `,
});
composer.addPass(customPass);
```

## Performance Tips
- Lower resolution for expensive effects: `composer.setSize(width/2, height/2)`
- Disable passes dynamically: `pass.enabled = false`
- Mobile: 2-3 passes max. Desktop: 4-6.
- Skip post-processing on very low-end devices
