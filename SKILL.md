---
name: 3d-physics-simulations
description: "Run real physics and simulations inside three.js websites: rigid bodies, gravity, collisions, ragdolls, cloth, chains, and GPU particle simulations with Rapier or cannon-es. Use whenever the user mentions physics, simulation, gravity, collisions, bouncing objects, destruction, cloth, fluid-like particle effects, or wants 3D objects that react with believable motion — even on marketing/landing pages, not just games. Pair with the threejs-fundamentals, threejs-interaction, and threejs-performance skills."
---

# 3D Physics & Simulations

Make 3D scenes *behave*: bodies fall, stack, shatter, and react to clicks with believable mass. This skill covers choosing an engine, wiring it to three.js, keeping 60fps, and the click-impulse pattern used on hero sections.

## Choose the engine

- **Rapier** (`@dimforge/rapier3d-compat`) — default. WASM, deterministic, actively maintained, best perf. Use `rapier3d-compat` (async WASM init) for bundlers; `rapier3d` (sync) only for plain script tags.
- **cannon-es** — only if WASM loading must be avoided (file:// demos, strict CSP). Slower and less maintained.
- **GPGPU/FBO particles** — for smoke, swarms, "AI neural" particle clouds (thousands+). Not a physics engine; see the GPU section below.

Load Rapier before creating any bodies:

```js
import RAPIER from '@dimforge/rapier3d-compat';
await RAPIER.init(); // required once, before world creation
const world = new RAPIER.World({ x: 0, y: -9.81, z: 0 });
```

## Core wiring pattern (three.js + Rapier)

Physics owns the *state*; three.js only *renders* it. Never move a mesh that has a body — read the body back into the mesh each frame.

```js
// 1. Body first
const bodyDesc = RAPIER.RigidBodyDesc.dynamic().setTranslation(0, 5, 0);
const body = world.createRigidBody(bodyDesc);
const colliderDesc = RAPIER.ColliderDesc.cuboid(0.5, 0.5, 0.5).setRestitution(0.4);
world.createCollider(colliderDesc, body);

// 2. Render loop — sync, don't set
function tick(dt) {
  world.timestep = dt;
  world.step();
  mesh.position.copy(body.translation());
  mesh.quaternion.copy(body.rotation());
}
```

- Use a **fixed timestep** for stability: accumulate `dt`, step in 1/60 chunks, cap at 3 steps/frame (spiral-of-death guard).
- Kinematic bodies for platforms/movers; `setNextKinematicTranslation()` each frame.
- Colliders do not inherit mesh scale — size them in world units explicitly.

## Click-impulse pattern (hero sections)

Raycast the pointer, find the body under the cursor, kick it. This is the "poke the hero object" interaction on top-tier AI sites. See the threejs-interaction skill for raycaster details.

```js
raycaster.setFromCamera(pointerNDC, camera);
const hit = world.castRay(new RAPIER.Ray(origin, dir), 100, true, undefined, undefined, undefined, filterPredicate);
if (hit) {
  const body = world.getCollider(hit.colliderHandle).parent();
  const dir3 = rayDir.clone().multiplyScalar(6); // impulse strength
  body.applyImpulse({ x: dir3.x, y: dir3.y, z: dir3.z }, true);
}
```

Give clickable bodies a slight angular damping (`setAngularDamping(0.5)`) so they spin believably instead of pinwheeling forever.

## Many bodies: instance, budget, sleep

- One `THREE.InstancedMesh` per geometry; write body transforms into the instance matrices after `world.step()`. Never one Mesh per body past ~50 objects.
- Budgets: ~200 dynamic boxes/colliders OK on desktop; ~80 on mobile. Below that, fake it (see "fake physics" below).
- Let bodies **sleep** (default) and wake them on interaction only. A scene full of never-sleeping bodies burns CPU for nothing.
- Simple geometry for colliders (boxes/spheres/capsules); convex/compound meshes only where the silhouette matters.

## Fake physics (decide this first)

Most marketing sites don't need an engine. Ask: does the effect need *collisions*, or just *believable motion*?
- Floating orbs reacting to cursor → springs/damping math, no engine.
- Click-scatter confetti of 3D shapes → precomputed velocities + gravity integration in one shader or loop, no engine.
- Draggable hero model with settle-back → spring toward rest pose (see the 3d-hero-interactions skill).
Reach for Rapier only when objects must collide, rest on each other, or pile up.

## GPU particle simulation

For neural-net clouds, starfields, smoke — thousands of particles updated on the GPU via ping-pong render targets (state textures). Details and a working template: read `references/gpgpu-particles.md`.

## Common mistakes

- Forgetting `await RAPIER.init()` → cryptic WASM errors.
- Setting `mesh.position` *and* letting the body drive it — the body wins, the mesh teleports. Move bodies, read back meshes.
- Variable timesteps frame-to-frame → jittery stacks and tunneling. Fixed timestep + accumulator.
- Adding render-loop work inside collision events — queue effects, apply them after `step()`.
- Raycasting the three.js scene for physics picking instead of `world.castRay` (or vice versa) — pick in the world that owns the objects.
- Not restoring the world on page transitions (R3F unmounts leak WASM bodies; dispose the world).

## Related skills

- threejs-fundamentals — scene, renderer, loop
- threejs-interaction — raycasting and pointer picking
- threejs-performance — instancing and budgets
- 3d-hero-interactions — the UX layer on top (hover states, bursts, magnetic cursors)
