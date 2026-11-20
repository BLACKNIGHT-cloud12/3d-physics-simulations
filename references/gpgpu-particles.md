# GPGPU Particles — ping-pong simulation template

Simulate 10k–1M particles entirely on the GPU. Position/velocity live in two `THREE.WebGLRenderTarget` pairs; each frame we render a "sim" pass that reads old state and writes new state, then a "render" pass draws points from the latest state texture.

## When to use
- Neural-network clouds, starfields, smoke trails, click-bursts at scale.
- NOT for: objects that need to collide with meshes (use Rapier, see SKILL.md).

## Core loop

```js
const SIM = 512; // particles = SIM * SIM
const rtOpts = { type: THREE.HalfFloatType, format: THREE.RGBAFormat, minFilter: THREE.NearestFilter, magFilter: THREE.NearestFilter };
const posRT = [makeRT(SIM, rtOpts), makeRT(SIM, rtOpts)]; // ping & pong
const velRT = [makeRT(SIM, rtOpts), makeRT(SIM, rtOpts)];

// Seed: fill posRT[0] with start positions (encode xyz + life in rgba)

function step(dt) {
  velSim.material.uniforms.uPos.value = posRT[0].texture;
  velSim.material.uniforms.uDt.value = dt;
  renderer.setRenderTarget(velRT[1]); renderer.render(velSim, cam); // update velocities

  posSim.material.uniforms.uVel.value = velRT[1].texture;
  renderer.setRenderTarget(posRT[1]); renderer.render(posSim, cam); // update positions

  renderer.setRenderTarget(null);
  posRT.reverse(); velRT.reverse(); // swap ping/pong
  points.material.uniforms.uPos.value = posRT[0].texture; // draw with newest state
}
```

## Sim shader (velocity)

```glsl
uniform sampler2D uPos; uniform sampler2D uVel; uniform float uDt;
uniform vec3 uAttractor; // e.g. pointer projected into world space
void main() {
  vec4 p = texture2D(uPos, uv);
  vec4 v = texture2D(uVel, uv);
  vec3 toA = uAttractor - p.xyz;
  v.xyz += (toA * 0.8) * uDt;          // spring toward attractor
  v.xyz *= 0.98;                        // damping
  v.w -= uDt;                           // life decay
  if (v.w <= 0.0) { /* respawn: write start pos + fresh life */ }
  gl_FragColor = v;
}
```

## Render as points

Vertex shader reads position from the state texture: `vec4 pos = texture2D(uPos, position.xy)` (map a plane geometry's UVs 1:1 to the sim grid), then `gl_PointSize` scaled by distance. Use `THREE.AdditiveBlending` + a soft radial falloff in the fragment shader for the glow look. See the threejs-shaders skill for material setup.

## Rules of thumb
- `HalfFloatType` targets, `NearestFilter` — anything else corrupts state.
- One texture fetch per state read; keep sim shaders free of branching where possible.
- Size the grid to the particle count you need (512² = 262k) — don't simulate 1M when 100k looks identical.
- On mobile, halve the grid and cap DPR at 1.5.
- Respawn logic (life expired → new start position) lives in the *sim* shader, not JS.
