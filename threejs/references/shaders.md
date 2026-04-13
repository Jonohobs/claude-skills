# Custom Shaders (GLSL)

For WebGPU/TSL shaders, use the `webgpu-threejs-tsl` skill instead.

## ShaderMaterial vs RawShaderMaterial

| Feature | ShaderMaterial | RawShaderMaterial |
|---------|---------------|-------------------|
| Built-in uniforms | Yes (projectionMatrix, modelViewMatrix, etc.) | No — you declare everything |
| Built-in attributes | Yes (position, normal, uv) | No |
| When to use | Custom effects with convenience | Full control, porting from other engines |

## ShaderMaterial Template

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    uTime: { value: 0 },
    uColor: { value: new THREE.Color(0xff0000) },
    uTexture: { value: texture },
  },
  vertexShader: `
    uniform float uTime;
    varying vec2 vUv;
    varying vec3 vPosition;

    void main() {
      vUv = uv;
      vPosition = position;
      vec3 pos = position;
      pos.y += sin(pos.x * 2.0 + uTime) * 0.1;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3 uColor;
    uniform sampler2D uTexture;
    varying vec2 vUv;

    void main() {
      vec4 texColor = texture2D(uTexture, vUv);
      gl_FragColor = vec4(texColor.rgb * uColor, 1.0);
    }
  `,
});

// Update in loop
material.uniforms.uTime.value = clock.getElapsedTime();
```

## onBeforeCompile — Extending Standard Materials

Inject custom GLSL into MeshStandardMaterial without rewriting the entire shader:

```javascript
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });

material.onBeforeCompile = (shader) => {
  shader.uniforms.uTime = { value: 0 };

  // Inject uniform declaration
  shader.vertexShader = 'uniform float uTime;\n' + shader.vertexShader;

  // Replace vertex position calculation
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    `#include <begin_vertex>
     transformed.y += sin(transformed.x * 2.0 + uTime) * 0.2;`
  );

  material.userData.shader = shader;
};

// Update in loop
if (material.userData.shader) {
  material.userData.shader.uniforms.uTime.value = elapsed;
}
```

## Common Shader Recipes

### Fresnel Rim
```glsl
// Fragment shader
uniform vec3 uFresnelColor;
varying vec3 vNormal;
varying vec3 vViewDir;

void main() {
  float fresnel = pow(1.0 - dot(vNormal, vViewDir), 3.0);
  gl_FragColor = vec4(uFresnelColor * fresnel, 1.0);
}
```

### Dissolve Effect
```glsl
uniform float uProgress;
varying vec2 vUv;

void main() {
  float noise = fract(sin(dot(vUv, vec2(12.9898, 78.233))) * 43758.5453);
  if (noise < uProgress) discard;
  float edge = smoothstep(uProgress - 0.1, uProgress, noise);
  gl_FragColor = vec4(vec3(1.0, 0.3, 0.0) * (1.0 - edge), 1.0);
}
```

### Vertex Displacement
```glsl
uniform float uTime;
void main() {
  vec3 pos = position;
  pos += normal * sin(pos.y * 5.0 + uTime) * 0.1;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```
