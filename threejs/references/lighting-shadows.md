# Lighting & Shadows

## Light Types

| Light | Shadows | Use Case |
|-------|---------|----------|
| `AmbientLight(color, intensity)` | No | Cheap fake ambient. Use 0.3-0.5 intensity. |
| `HemisphereLight(sky, ground, intensity)` | No | Sky/ground gradient. Better ambient than Ambient. |
| `DirectionalLight(color, intensity)` | Yes | Sun. Parallel rays. Position = direction. |
| `PointLight(color, intensity, distance, decay)` | Yes | Bulbs, torches. Omni-directional. |
| `SpotLight(color, intensity, distance, angle, penumbra, decay)` | Yes | Flashlights, stage. Cone beam. |
| `RectAreaLight(color, intensity, width, height)` | No | Panels, windows. Only Standard/Physical materials. Requires `RectAreaLightUniformsLib.init()`. |

## Standard Lighting Recipe

```javascript
// Ambient fill
const hemi = new THREE.HemisphereLight(0xffffff, 0x444444, 0.6);
scene.add(hemi);

// Key light (sun)
const dir = new THREE.DirectionalLight(0xffffff, 1);
dir.position.set(5, 10, 7.5);
dir.castShadow = true;
scene.add(dir);

// Fill light (opposite side, softer)
const fill = new THREE.DirectionalLight(0x8888ff, 0.3);
fill.position.set(-5, 5, -5);
scene.add(fill);
```

## Shadow Setup (3 Steps)

```javascript
// 1. Renderer
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// 2. Light
dirLight.castShadow = true;
dirLight.shadow.mapSize.set(2048, 2048);
dirLight.shadow.normalBias = 0.05;     // fixes acne without peter-panning
dirLight.shadow.camera.near = 0.5;
dirLight.shadow.camera.far = 50;
dirLight.shadow.camera.left = -10;
dirLight.shadow.camera.right = 10;
dirLight.shadow.camera.top = 10;
dirLight.shadow.camera.bottom = -10;

// 3. Objects
mesh.castShadow = true;
floor.receiveShadow = true;
```

## Shadow Map Types

| Type | Quality | Performance | Notes |
|------|---------|-------------|-------|
| `BasicShadowMap` | Hard edges | Fast | Pixelated |
| `PCFShadowMap` | Filtered | Default | Good balance |
| `PCFSoftShadowMap` | Soft | Slower | Best quality |
| `VSMShadowMap` | Smooth | Variable | Can cause light bleeding |

## Bias Tuning

- `shadow.normalBias = 0.05` — start here, adjust per scene
- `shadow.bias` — keep close to 0 (default). Only use if normalBias alone isn't enough
- **Shadow acne:** stripey artifacts = bias too low
- **Peter-panning:** shadows floating/detached = bias too high

## CSM (Cascaded Shadow Maps) for Large Scenes

```javascript
import { CSM } from 'three/addons/csm/CSM.js';

const csm = new CSM({
  maxFar: camera.far,
  cascades: 4,          // 2 for mobile, 4 for desktop
  shadowMapSize: 2048,
  lightDirection: new THREE.Vector3(-1, -1, -1).normalize(),
  camera: camera,
  parent: scene
});

// In loop: csm.update();
// On resize: csm.updateFrustums();
```

## Debug Helpers

```javascript
scene.add(new THREE.DirectionalLightHelper(dirLight));
scene.add(new THREE.CameraHelper(dirLight.shadow.camera));
scene.add(new THREE.PointLightHelper(pointLight));
scene.add(new THREE.SpotLightHelper(spotLight));
```
