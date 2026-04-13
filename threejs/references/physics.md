# Physics Integration

## Engine Comparison

| Engine | Language | Size | API | Best For |
|--------|----------|------|-----|----------|
| **Rapier** | Rust/WASM | ~300KB | Modern, typed | New projects (recommended) |
| **cannon-es** | JS | ~150KB | Easiest | Prototypes, simple physics |
| **Ammo.js** | C++/WASM | ~500KB | Complex | Max features, Bullet physics port |

## Rapier Setup

```javascript
import RAPIER from '@dimforge/rapier3d-compat';

await RAPIER.init();
const world = new RAPIER.World({ x: 0, y: -9.81, z: 0 });

// Static ground
const groundBody = world.createRigidBody(RAPIER.RigidBodyDesc.fixed());
world.createCollider(
  RAPIER.ColliderDesc.cuboid(50, 0.1, 50),
  groundBody
);

// Dynamic box
const bodyDesc = RAPIER.RigidBodyDesc.dynamic().setTranslation(0, 5, 0);
const rigidBody = world.createRigidBody(bodyDesc);
world.createCollider(
  RAPIER.ColliderDesc.cuboid(0.5, 0.5, 0.5),
  rigidBody
);

// Three.js mesh
const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0xff0000 })
);
scene.add(mesh);
```

## Fixed Timestep Loop

```javascript
const fixedDT = 1 / 60;
let accumulator = 0;

function animate() {
  const delta = Math.min(clock.getDelta(), 0.1); // cap to prevent spiral of death
  accumulator += delta;

  while (accumulator >= fixedDT) {
    world.step();
    accumulator -= fixedDT;
  }

  // Sync Three.js meshes to physics bodies
  const pos = rigidBody.translation();
  const rot = rigidBody.rotation();
  mesh.position.set(pos.x, pos.y, pos.z);
  mesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);

  renderer.render(scene, camera);
}
```

## cannon-es Setup

```javascript
import * as CANNON from 'cannon-es';

const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -9.82, 0) });

const groundBody = new CANNON.Body({
  type: CANNON.Body.STATIC,
  shape: new CANNON.Plane(),
});
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

const boxBody = new CANNON.Body({
  mass: 1,
  shape: new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5)),
  position: new CANNON.Vec3(0, 5, 0),
});
world.addBody(boxBody);

// In loop:
// world.step(fixedDT);
// mesh.position.copy(boxBody.position);
// mesh.quaternion.copy(boxBody.quaternion);
```

## Body Types
- **Dynamic:** affected by forces and collisions (mass > 0)
- **Static:** never moves, infinite mass (floors, walls)
- **Kinematic:** moved by code, affects dynamic bodies but not affected by them (moving platforms)
