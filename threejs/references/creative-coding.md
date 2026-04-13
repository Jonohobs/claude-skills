# Creative Coding Patterns

> Reference for creative/generative Three.js work. Based on patterns from vipin018/curllmooha's 99 Three.js repos.

## Table of Contents
- [Sketch Workflow with Vite](#sketch-workflow-with-vite)
- [MeshPhysicalMaterial vs ShaderMaterial Decision Tree](#meshphysicalmaterial-vs-shadermaterial-decision-tree)
- [Holographic / Iridescent Effects](#holographic--iridescent-effects)
- [Procedural Growth & Organic Animation](#procedural-growth--organic-animation)
- [Audio-Reactive Visualization](#audio-reactive-visualization)
- [Scroll-Driven 3D Animation](#scroll-driven-3d-animation)
- [Scientific / Data Visualization](#scientific--data-visualization)
- [Cloth & Soft Body Simulation](#cloth--soft-body-simulation)

---

## Sketch Workflow with Vite

### vite-plugin-glsl setup
```bash
npm create vite@latest sketch -- --template vanilla && cd sketch
npm i three vite-plugin-glsl
```
```js
// vite.config.js
import glsl from 'vite-plugin-glsl';
export default { plugins: [glsl()] };
```
Now import shaders directly:
```js
import vertexShader from './shaders/blob.vert';
import fragmentShader from './shaders/blob.frag';
```

### Minimal sketch boilerplate (~25 lines)
```js
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth / innerHeight, 0.1, 100);
camera.position.z = 3;

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

const clock = new THREE.Clock();

// --- your sketch here ---
const mesh = new THREE.Mesh(
  new THREE.IcosahedronGeometry(1, 64),
  new THREE.MeshStandardMaterial({ color: 0x44aaff })
);
scene.add(mesh, new THREE.AmbientLight(0xffffff, 0.5));

(function loop() {
  const t = clock.getElapsedTime();
  mesh.rotation.y = t * 0.3;
  renderer.render(scene, camera);
  requestAnimationFrame(loop);
})();

window.addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});
```

### Portfolio / gallery architecture
```
sketches/
  001-blob/        ← each sketch is its own Vite entry
  002-hologram/
  003-cloth/
  shared/          ← common utils, base scene setup
    createScene.js
    gui.js         ← lil-gui wrapper
  index.html       ← gallery grid linking to each sketch
```
Multi-page Vite config:
```js
// vite.config.js
import { resolve } from 'path';
import glsl from 'vite-plugin-glsl';
export default {
  plugins: [glsl()],
  build: {
    rollupOptions: {
      input: {
        main: resolve(__dirname, 'index.html'),
        blob: resolve(__dirname, 'sketches/001-blob/index.html'),
        hologram: resolve(__dirname, 'sketches/002-hologram/index.html'),
      }
    }
  }
};
```

---

## MeshPhysicalMaterial vs ShaderMaterial Decision Tree

| Need | Use |
|------|-----|
| PBR reflections, clearcoat, iridescence, transmission (glass) | `MeshPhysicalMaterial` |
| PBR + custom vertex deformation | `MeshPhysicalMaterial` + `onBeforeCompile` |
| Full vertex/fragment control, no PBR | `ShaderMaterial` |
| Porting from ShaderToy, no Three.js helpers | `RawShaderMaterial` |

### Blob-mixer: MeshPhysicalMaterial + custom vertex via onBeforeCompile
```js
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js';

const mat = new THREE.MeshPhysicalMaterial({
  roughness: 0.1, metalness: 0.0, transmission: 1.0,
  thickness: 1.5, ior: 1.5, iridescence: 1.0,
  iridescenceIOR: 1.3, clearcoat: 1.0,
});

// Inject custom vertex displacement
mat.onBeforeCompile = (shader) => {
  shader.uniforms.uTime = { value: 0 };
  shader.vertexShader = 'uniform float uTime;\n' + shader.vertexShader;
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    `#include <begin_vertex>
     float displacement = sin(position.x * 4.0 + uTime) *
                          sin(position.y * 4.0 + uTime * 0.7) *
                          sin(position.z * 4.0 + uTime * 1.3) * 0.15;
     transformed += normal * displacement;`
  );
  mat.userData.shader = shader; // store ref for animation
};

// In loop:
if (mat.userData.shader) mat.userData.shader.uniforms.uTime.value = elapsed;
```

Load an HDR environment for reflections:
```js
new RGBELoader().load('/env.hdr', (tex) => {
  tex.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = tex;
});
```

---

## Holographic / Iridescent Effects

### Fresnel hologram shader
```glsl
// hologram.vert
varying vec3 vNormal;
varying vec3 vViewDir;
varying vec2 vUv;
uniform float uTime;

void main() {
  vUv = uv;
  vec4 mvPos = modelViewMatrix * vec4(position, 1.0);
  vNormal = normalize(normalMatrix * normal);
  vViewDir = normalize(-mvPos.xyz);
  gl_Position = projectionMatrix * mvPos;
}
```
```glsl
// hologram.frag
varying vec3 vNormal;
varying vec3 vViewDir;
varying vec2 vUv;
uniform float uTime;

void main() {
  // Fresnel rim
  float fresnel = pow(1.0 - dot(vViewDir, vNormal), 3.0);

  // Scanlines
  float scanline = sin(vUv.y * 200.0 + uTime * 5.0) * 0.5 + 0.5;
  scanline = smoothstep(0.4, 0.6, scanline);

  // Chromatic shift
  vec3 color;
  color.r = fresnel * sin(uTime * 0.5) * 0.5 + 0.5;
  color.g = fresnel * sin(uTime * 0.5 + 2.094) * 0.5 + 0.5;
  color.b = fresnel * sin(uTime * 0.5 + 4.189) * 0.5 + 0.5;

  float alpha = fresnel * 0.8 + scanline * 0.2;
  gl_FragColor = vec4(color * (0.6 + scanline * 0.4), alpha);
}
```
```js
const holoMat = new THREE.ShaderMaterial({
  vertexShader, fragmentShader,
  uniforms: { uTime: { value: 0 } },
  transparent: true, side: THREE.DoubleSide,
  blending: THREE.AdditiveBlending, depthWrite: false,
});
```

---

## Procedural Growth & Organic Animation

### Vertex displacement — stacked sine waves
```glsl
// In vertex shader
uniform float uTime;
uniform float uGrowth; // 0→1 over time

vec3 displace(vec3 p) {
  float wave1 = sin(p.y * 3.0 + uTime) * 0.1;
  float wave2 = sin(p.x * 5.0 + uTime * 1.3) * 0.05;
  float wave3 = cos(p.z * 4.0 + uTime * 0.7) * 0.08;
  return p + normal * (wave1 + wave2 + wave3) * uGrowth;
}
```

### Time-based growth animation
```js
const uniforms = { uTime: { value: 0 }, uGrowth: { value: 0 } };

function loop() {
  const t = clock.getElapsedTime();
  uniforms.uTime.value = t;
  uniforms.uGrowth.value = Math.min(t / 5.0, 1.0); // grow over 5 seconds
  // Scale geometry from center
  mesh.scale.setScalar(THREE.MathUtils.lerp(0.01, 1.0, uniforms.uGrowth.value));
}
```

### Particle trails for growth paths
```js
const trailCount = 500;
const positions = new Float32Array(trailCount * 3);
const trailGeo = new THREE.BufferGeometry();
trailGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));

// Each frame: shift positions back, add new head position
function updateTrail(headPos) {
  for (let i = trailCount - 1; i > 0; i--) {
    positions[i * 3] = positions[(i - 1) * 3];
    positions[i * 3 + 1] = positions[(i - 1) * 3 + 1];
    positions[i * 3 + 2] = positions[(i - 1) * 3 + 2];
  }
  positions[0] = headPos.x;
  positions[1] = headPos.y;
  positions[2] = headPos.z;
  trailGeo.attributes.position.needsUpdate = true;
}
const trail = new THREE.Points(trailGeo, new THREE.PointsMaterial({ size: 0.02 }));
```

---

## Audio-Reactive Visualization

### Web Audio API setup
```js
const listener = new THREE.AudioListener();
camera.add(listener);
const audio = new THREE.Audio(listener);
const analyser = new THREE.AudioAnalyser(audio, 256); // fftSize

new THREE.AudioLoader().load('/track.mp3', (buffer) => {
  audio.setBuffer(buffer);
  audio.play();
});
```

### Map frequency to uniforms
```js
function loop() {
  const data = analyser.getFrequencyData(); // Uint8Array 0-255
  const bass = avg(data, 0, 8) / 255;       // low freq
  const mid = avg(data, 8, 32) / 255;
  const treble = avg(data, 32, 128) / 255;

  uniforms.uBass.value = bass;
  uniforms.uMid.value = mid;
  uniforms.uTreble.value = treble;
  mesh.scale.setScalar(1.0 + bass * 0.3);
}

function avg(arr, start, end) {
  let sum = 0;
  for (let i = start; i < end; i++) sum += arr[i];
  return sum / (end - start);
}
```

### Beat detection
```js
let prevBass = 0;
function detectBeat(data) {
  const bass = avg(data, 0, 8) / 255;
  const delta = bass - prevBass;
  prevBass = bass;
  return delta > 0.15; // threshold — tune per track
}
```

### Vertex shader displacement from audio
```glsl
uniform float uBass;
uniform float uMid;
void main() {
  vec3 pos = position;
  pos += normal * uBass * sin(position.y * 10.0) * 0.3;
  pos += normal * uMid * cos(position.x * 8.0) * 0.1;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

---

## Scroll-Driven 3D Animation

### GSAP ScrollTrigger integration
```js
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

// Map scroll to camera movement
gsap.to(camera.position, {
  z: 0.5, y: 2,
  scrollTrigger: {
    trigger: '#section-2',
    start: 'top bottom', end: 'top top',
    scrub: 1,
  }
});

// Map scroll to shader uniform
const tl = gsap.timeline({
  scrollTrigger: { trigger: '#hero', start: 'top top', end: 'bottom top', scrub: true }
});
tl.to(uniforms.uProgress, { value: 1, duration: 1 });
```

### Scroll progress → material uniforms (vanilla)
```js
let scrollProgress = 0;
window.addEventListener('scroll', () => {
  scrollProgress = window.scrollY / (document.body.scrollHeight - window.innerHeight);
}, { passive: true });

// In render loop — bridge to rAF for smoothness
function loop() {
  uniforms.uScroll.value = THREE.MathUtils.lerp(
    uniforms.uScroll.value, scrollProgress, 0.1
  );
  renderer.render(scene, camera);
  requestAnimationFrame(loop);
}
```

### Intersection Observer for triggering
```js
const observer = new IntersectionObserver((entries) => {
  entries.forEach(e => {
    if (e.isIntersecting) startAnimation(e.target.dataset.scene);
  });
}, { threshold: 0.3 });
document.querySelectorAll('[data-scene]').forEach(el => observer.observe(el));
```

---

## Scientific / Data Visualization

### Data → particle positions
```js
function dataToParticles(dataPoints) {
  const count = dataPoints.length;
  const positions = new Float32Array(count * 3);
  const colors = new Float32Array(count * 3);
  const color = new THREE.Color();

  dataPoints.forEach((d, i) => {
    positions[i * 3] = d.x;
    positions[i * 3 + 1] = d.y;
    positions[i * 3 + 2] = d.z;

    // Color ramp: blue (low) → red (high)
    color.setHSL(0.66 - d.value * 0.66, 1.0, 0.5);
    colors[i * 3] = color.r;
    colors[i * 3 + 1] = color.g;
    colors[i * 3 + 2] = color.b;
  });

  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
  return new THREE.Points(geo, new THREE.PointsMaterial({
    size: 0.05, vertexColors: true, sizeAttenuation: true
  }));
}
```

### Animated transitions between data states
```js
const posA = currentPositions; // Float32Array
const posB = nextPositions;    // Float32Array
let morphT = 0;

function loop() {
  if (morphT < 1) {
    morphT += 0.01;
    const pos = geo.attributes.position.array;
    for (let i = 0; i < pos.length; i++) {
      pos[i] = THREE.MathUtils.lerp(posA[i], posB[i], morphT);
    }
    geo.attributes.position.needsUpdate = true;
  }
}
```

---

## Cloth & Soft Body Simulation

### Verlet integration cloth
```js
class Particle {
  constructor(x, y, z, pinned = false) {
    this.pos = new THREE.Vector3(x, y, z);
    this.prev = this.pos.clone();
    this.pinned = pinned;
  }
  update(gravity, damping) {
    if (this.pinned) return;
    const vel = this.pos.clone().sub(this.prev).multiplyScalar(damping);
    this.prev.copy(this.pos);
    this.pos.add(vel).add(gravity);
  }
}

class Constraint {
  constructor(a, b) {
    this.a = a; this.b = b;
    this.rest = a.pos.distanceTo(b.pos);
  }
  solve() {
    const diff = this.b.pos.clone().sub(this.a.pos);
    const dist = diff.length();
    const correction = diff.multiplyScalar((dist - this.rest) / dist * 0.5);
    if (!this.a.pinned) this.a.pos.add(correction);
    if (!this.b.pinned) this.b.pos.sub(correction);
  }
}
```

### Cloth setup (grid of particles + constraints)
```js
const W = 20, H = 20, spacing = 0.2;
const particles = [];
const constraints = [];

for (let y = 0; y < H; y++) {
  for (let x = 0; x < W; x++) {
    const pinned = y === 0 && (x === 0 || x === W - 1); // pin top corners
    particles.push(new Particle(x * spacing, -y * spacing, 0, pinned));
    if (x > 0) constraints.push(new Constraint(particles[y * W + x], particles[y * W + x - 1]));
    if (y > 0) constraints.push(new Constraint(particles[y * W + x], particles[(y - 1) * W + x]));
  }
}
```

### Wind force
```js
function applyWind(particles, time) {
  particles.forEach(p => {
    if (p.pinned) return;
    const wind = new THREE.Vector3(
      Math.sin(time + p.pos.y * 3.0) * 0.002,
      0,
      Math.cos(time * 0.7 + p.pos.x * 2.0) * 0.001
    );
    p.pos.add(wind);
  });
}
```

### Sync to Three.js mesh
```js
const clothGeo = new THREE.PlaneGeometry(
  (W - 1) * spacing, (H - 1) * spacing, W - 1, H - 1
);

function syncCloth() {
  const pos = clothGeo.attributes.position;
  particles.forEach((p, i) => {
    pos.setXYZ(i, p.pos.x, p.pos.y, p.pos.z);
  });
  pos.needsUpdate = true;
  clothGeo.computeVertexNormals();
}

// Simulation step (in loop)
const gravity = new THREE.Vector3(0, -0.001, 0);
function simulate() {
  applyWind(particles, clock.getElapsedTime());
  particles.forEach(p => p.update(gravity, 0.99));
  for (let i = 0; i < 5; i++) constraints.forEach(c => c.solve()); // 5 iterations
  syncCloth();
}
```
