# Three.js Common Patterns

## Table of Contents
- Transparency & Z-Fighting (TESTED)
- Grouping Animated Objects (TESTED)
- Loading GLTF/GLB Models
- Disposal Helper
- Animation System
- Raycasting (Mouse Picking)
- Scroll-Driven 3D (GSAP)
- Responsive Canvas
- Scene Transitions

## Transparency & Z-Fighting (TESTED)

**Status:** Battle-tested 2026-03-17 on agentic-taxonomy viz. This is the #1 cause of mysterious "pop-in" during orbit.

### The Problem
Transparent objects write to the depth buffer by default. As the camera orbits, Three.js re-sorts transparent objects by distance-to-camera. When sort order changes, objects suddenly block/unblock each other → visual popping.

### The Fix: 4 Rules

```javascript
// 1. ALWAYS set depthWrite: false on transparent materials
const mat = new THREE.MeshPhysicalMaterial({
  transparent: true,
  opacity: 0.85,
  depthWrite: false,  // ← THIS FIXES POP-IN
});

// 2. Use renderOrder to control draw sequence
mesh.renderOrder = 1;       // solid-ish nodes
wireframe.renderOrder = 0;  // behind nodes
glow.renderOrder = 2;       // in front
label.renderOrder = 3;      // always on top

// 3. Avoid large transparent planes — they z-fight with everything
// BAD: PlaneGeometry(30, 30) with opacity 0.02
// GOOD: RingGeometry or wireframe mesh instead
const ring = new THREE.Mesh(
  new THREE.RingGeometry(14, 14.05, 64),
  new THREE.MeshBasicMaterial({
    transparent: true, opacity: 0.08,
    side: THREE.DoubleSide, depthWrite: false,
  })
);

// 4. Transparent spheres wrapping other objects = guaranteed flicker
// Switch to wireframe: true to avoid it
const shell = new THREE.MeshBasicMaterial({
  transparent: true, opacity: 0.03,
  wireframe: true,    // ← no solid faces to z-fight
  depthWrite: false,
});
```

### Gotchas
- `frustumCulled = false` does NOT fix transparency pop-in (different problem)
- Large transparent planes (layer separators, ground planes) are the worst offenders
- Sprite glows need `depthWrite: false` too (they're transparent)
- The more transparent objects overlap, the worse it gets — minimize overlapping transparent geometry

## Grouping Animated Objects (TESTED)

**Status:** Battle-tested 2026-03-17. Fixes "detached glow/label" artifacts.

### The Problem
If you animate a mesh's position but its wireframe, glow sprite, and label are siblings (not children), they stay at the original position → visual tearing.

### The Fix: Group per entity

```javascript
// Group everything that should move together
const group = new THREE.Group();
group.position.set(x, y, z);  // world position on the group

const mesh = new THREE.Mesh(geo, mat);       // at origin (0,0,0) within group
const wire = new THREE.Mesh(geo, wireMat);   // same
const glow = new THREE.Sprite(glowMat);      // same
const label = new THREE.Sprite(labelMat);
label.position.set(0, size + 0.7, 0);        // offset within group

group.add(mesh, wire, glow, label);
scene.add(group);

// Animate the GROUP position, rotate the MESH
function animate(t) {
  group.position.y = baseY + Math.sin(t * 0.4) * 0.25;  // bob
  mesh.rotation.y = t * 0.12;  // spin just the shape
}
```

### Key Insight
- Position animation → group level
- Rotation/scale animation → mesh level (so labels don't spin)
- Raycasting still works on the mesh directly (Three.js resolves world coords)

## Loading GLTF/GLB Models

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/draco/');

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

let mixer;

gltfLoader.load('/models/scene.glb', (gltf) => {
  const model = gltf.scene;
  model.traverse((child) => {
    if (child.isMesh) {
      child.castShadow = true;
      child.receiveShadow = true;
    }
  });
  scene.add(model);

  if (gltf.animations.length > 0) {
    mixer = new THREE.AnimationMixer(model);
    mixer.clipAction(gltf.animations[0]).play();
  }
});

// In animation loop: if (mixer) mixer.update(delta);
```

**Draco compression:** 70-90% smaller files. Decoder WASM files must be served from `/draco/`.
**KTX2 textures:** GPU-compressed. Use `KTX2Loader` with `MeshoptDecoder`.
**Optimization CLI:** `npx gltf-transform optimize input.glb output.glb --compress draco`

## Disposal Helper

```javascript
function disposeObject(object) {
  object.traverse((child) => {
    if (child.isMesh) {
      child.geometry.dispose();
      const materials = Array.isArray(child.material) ? child.material : [child.material];
      materials.forEach((mat) => {
        Object.values(mat).forEach((value) => {
          if (value && typeof value.dispose === 'function') value.dispose();
        });
        mat.dispose();
      });
    }
  });
  object.parent?.remove(object);
}

// Usage
disposeObject(model);

// Full cleanup on unmount
renderer.dispose();
cancelAnimationFrame(rafId);
window.removeEventListener('resize', onResize);
```

## Animation System

```javascript
// AnimationMixer — drives clips from GLTF
const mixer = new THREE.AnimationMixer(model);
const idleAction = mixer.clipAction(gltf.animations[0]);
const walkAction = mixer.clipAction(gltf.animations[1]);

idleAction.play();

// Crossfade
function crossFade(from, to, duration = 0.3) {
  to.reset().setEffectiveWeight(1).play();
  from.crossFadeTo(to, duration, true);
}

// In loop: mixer.update(delta);

// Action controls
action.timeScale = 2;             // speed
action.setLoop(THREE.LoopOnce);   // play once
action.clampWhenFinished = true;  // hold last frame
action.weight = 0.5;              // blend weight
```

## Raycasting (Mouse Picking)

```javascript
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

window.addEventListener('pointermove', (e) => {
  pointer.x = (e.clientX / innerWidth) * 2 - 1;
  pointer.y = -(e.clientY / innerHeight) * 2 + 1;
});

window.addEventListener('click', () => {
  raycaster.setFromCamera(pointer, camera);
  const hits = raycaster.intersectObjects(interactiveObjects, true);
  if (hits.length > 0) {
    const hit = hits[0];
    console.log(hit.object.name, hit.point, hit.face, hit.uv);
  }
});
```

**Performance:** Pass specific array instead of `scene.children`. Set `raycaster.far` to limit distance. For complex meshes, use `three-mesh-bvh`.

## Scroll-Driven 3D (GSAP)

```javascript
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

// Canvas is position:fixed behind scrollable HTML sections
gsap.to(camera.position, {
  x: 5, y: 3, z: 2,
  scrollTrigger: {
    trigger: '#section-two',
    start: 'top bottom',
    end: 'top top',
    scrub: true,
  }
});

// Bind GLTF animation to scroll progress
ScrollTrigger.create({
  trigger: '#section-three',
  start: 'top top',
  end: 'bottom bottom',
  onUpdate: (self) => {
    if (mixer) mixer.setTime(clip.duration * self.progress);
  }
});
```

## Responsive Canvas

```javascript
function onResize() {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
  renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
}
window.addEventListener('resize', onResize);
```

## Scene Transitions

```javascript
// Camera tween
function cameraTween(target, lookAt, duration = 2) {
  gsap.to(camera.position, {
    ...target,
    duration,
    ease: 'power2.inOut',
    onUpdate: () => camera.lookAt(lookAt.x, lookAt.y, lookAt.z)
  });
}

// Fade transition
function transitionTo(newSetup) {
  gsap.to(renderer.domElement, {
    opacity: 0, duration: 0.5,
    onComplete: () => {
      disposeObject(scene); // clean up old
      newSetup(scene, camera);
      gsap.to(renderer.domElement, { opacity: 1, duration: 0.5 });
    }
  });
}
```
