# Instancing, Batching & LOD

## InstancedMesh (100+ Identical Objects → 1 Draw Call)

```javascript
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0xff0000 });
const count = 1000;
const mesh = new THREE.InstancedMesh(geometry, material, count);

const dummy = new THREE.Object3D();
for (let i = 0; i < count; i++) {
  dummy.position.set(Math.random() * 50 - 25, 0, Math.random() * 50 - 25);
  dummy.rotation.set(0, Math.random() * Math.PI, 0);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
mesh.instanceMatrix.needsUpdate = true;
scene.add(mesh);
```

### Per-Instance Color
```javascript
const color = new THREE.Color();
for (let i = 0; i < count; i++) {
  color.setHSL(Math.random(), 0.8, 0.5);
  mesh.setColorAt(i, color);
}
mesh.instanceColor.needsUpdate = true;
```

### Update Instances at Runtime
```javascript
// Only update what changed, then:
mesh.instanceMatrix.needsUpdate = true;
```

## BatchedMesh (r160+ — Different Geometries, Same Material, 1 Draw Call)

```javascript
const batchedMesh = new THREE.BatchedMesh(maxGeometryCount, maxVertexCount, maxIndexCount, material);

const geoId1 = batchedMesh.addGeometry(boxGeometry);
const geoId2 = batchedMesh.addGeometry(sphereGeometry);

const instId1 = batchedMesh.addInstance(geoId1);
const instId2 = batchedMesh.addInstance(geoId2);

batchedMesh.setMatrixAt(instId1, matrix1);
batchedMesh.setMatrixAt(instId2, matrix2);

scene.add(batchedMesh);
```

Per-object frustum culling is built-in (`perObjectFrustumCulled: true` by default).

## Merge Static Geometry

```javascript
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

const merged = mergeGeometries([geo1, geo2, geo3]);
const mesh = new THREE.Mesh(merged, sharedMaterial);
// 1 draw call instead of 3. Cannot transform individually.
```

## LOD (Level of Detail)

```javascript
const lod = new THREE.LOD();
lod.addLevel(highDetailMesh, 0);    // < 10 units from camera
lod.addLevel(medDetailMesh, 10);    // 10-50 units
lod.addLevel(lowDetailMesh, 50);    // > 50 units
scene.add(lod);

// In loop: lod.update(camera);
```

## Decision Matrix

| Scenario | Pattern |
|----------|---------|
| 1 unique object | Regular Mesh |
| Few static objects, same material | mergeGeometries |
| 100+ identical objects | InstancedMesh |
| Many different geometries, same material | BatchedMesh |
| Objects at varying distances | LOD |
| Dynamic spawn/despawn | InstancedMesh + object pooling |
| Forests, crowds, cities | InstancedMesh + LOD combo |

## Instance Count Guidelines
- Mobile: 100-1,000
- Desktop: 1,000-10,000
- High-end: 10,000-100,000+
