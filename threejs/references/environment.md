# Environment Maps

## HDR Environment (Most Common)

```javascript
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

const rgbeLoader = new RGBELoader();
rgbeLoader.load('/envmaps/studio.hdr', (hdrTexture) => {
  hdrTexture.mapping = THREE.EquirectangularReflectionMapping;

  scene.environment = hdrTexture;     // PBR reflections on all materials
  scene.background = hdrTexture;       // visible background (optional)
  scene.backgroundBlurriness = 0.5;    // blur background (optional)
});
```

## EXR Environment

```javascript
import { EXRLoader } from 'three/addons/loaders/EXRLoader.js';

new EXRLoader().load('/envmaps/studio.exr', (exrTexture) => {
  exrTexture.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = exrTexture;
});
```

## PMREMGenerator (Pre-filtered Mipmapped Radiance Environment Map)

For custom or dynamically generated environments:

```javascript
const pmremGenerator = new THREE.PMREMGenerator(renderer);
pmremGenerator.compileEquirectangularShader();

rgbeLoader.load('/envmaps/studio.hdr', (texture) => {
  const envMap = pmremGenerator.fromEquirectangular(texture).texture;
  scene.environment = envMap;
  texture.dispose();
  pmremGenerator.dispose();
});
```

## CubeTexture (Six-Face Cubemap)

```javascript
const cubeLoader = new THREE.CubeTextureLoader();
const cubeTexture = cubeLoader.load([
  'px.jpg', 'nx.jpg',  // +X, -X
  'py.jpg', 'ny.jpg',  // +Y, -Y
  'pz.jpg', 'nz.jpg',  // +Z, -Z
]);
scene.environment = cubeTexture;
scene.background = cubeTexture;
```

## Per-Material vs Scene-Wide

```javascript
// Scene-wide (all PBR materials)
scene.environment = envMap;

// Per-material override
material.envMap = customEnvMap;
material.envMapIntensity = 1.5;    // boost reflections

// Background only (no reflections)
scene.background = envMap;
scene.environment = null;
```

## Free HDR Sources
- **Poly Haven** (polyhaven.com/hdris) — CC0, 600+ HDRIs
- Accessible via Blender MCP: `mcp__blender__search_polyhaven_assets`
