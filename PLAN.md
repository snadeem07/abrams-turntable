# M1A2 SEPv3 Abrams Turntable: Build Plan

One self-contained `index.html` using three.js r160, loaded as ES modules through an importmap that points to the CDN. Every model, texture and sound is made procedurally. Units are metres at real-world scale (hull about 7.9 m long, 3.7 m wide, about 2.4 m to the turret roof; gun overhang about 9.8 m in total).

If the session is interrupted, read this file and continue from the first unticked milestone. After each milestone: serve the file locally, open it in headless Chromium, check the console (especially for shader compile errors), take a screenshot, fix any problems, tick the box, then commit and push.

## Code organisation

The single file is split into clearly labelled sections, each a function or a small group of functions, so a later fix edits one section only:
`core` · `materials` · `hull` · `runningGear` · `turret` · `aps` · `desert` · `lighting` · `post` · `interaction` · `audio` · `ui` · `perf`

Shared helpers (beveled extrusions, lofts, bolt and strap generators, merge-and-instance utilities) live in `core`.

## Milestones

- [x] **M0 – Plan.** This file.
- [x] **M1 – Core setup.** Renderer, camera, orbit controls with turntable auto-rotate, ACES tone mapping, basic sun, ground plane, FPS readout.
- [x] **M2 – Hull.** Beveled hull with sloped glacis; driver's hatch and periscopes; headlight clusters with guards; tow eyes and shackles; fuel caps; engine deck grilles; rear exhaust grille; tail lights; mud flaps. Side skirts made of framed panels with raised lips, lifting handles and a rubber bottom strip.
- [x] **M3 – Running gear.** Seven road wheels per side with hub detail, front idler, rear sprocket, return rollers. Instanced track links with split rubber pads, end connectors and wedge bolts, laid on a path that wraps around the wheels and can animate.
- [x] **M4 – Turret and gun.** Lofted turret, hex bolts, lifting eyes, smoke grenade launchers, side stowage boxes. Bustle rack with bedrolls, ration boxes, a cooler, cam nets, water cans and straps. Commander's remote weapon station, independent thermal viewer that scans by itself, loader's MG on a skate mount, two whip antennas, crosswind sensor. 120 mm gun with thermal sleeve, latches and straps, bore evacuator, muzzle reference sensor, and a canvas mantlet cover.
- [x] **M5 – Active protection units.** One unit on each turret side, on standoff brackets: a box with a slanted front, a ribbed radar face looking forward and out, access panel and latches, a turning launcher with two ports, cables into the turret, and an aft radar on a post. A toggle hides them.
- **CHECKPOINT:** switch to Fable 5.2 at max effort (or Opus 5.5 at max effort).
- [x] **M6 – Materials and weathering.** Triplanar weathering injected with onBeforeCompile; Green 383, NATO three-tone and desert tan finishes; dust and wear sliders; stencil decals.
- [x] **M7 – Desert scene.** Vertex-displaced dunes, wind ripples, sky dome with cirrus, mesas, rocks, shrubs, outpost, tread marks, scenery that wraps while driving, PMREM environment, drifting sand and track dust.
- [x] **M8 – Lighting presets and post-processing.** Studio cyclorama, Noir and Night presets; PCSS soft shadows; GTAO; contact shade; vignette and grain; white-hot thermal mode with reticle.
- **CHECKPOINT:** switch to Sonnet 5.5 at high effort.
- [ ] **M9 – Interaction.** Driving, turret traverse, gun elevation, firing effects, hatches.
- [ ] **M10 – Audio.** Procedural turbine, servos, cannon, wind and music, each toggleable.
- [ ] **M11 – UI.** Stencil title block, spec chips, readout panel, glass control dock that wraps on phones.
- **CHECKPOINT:** switch to Sonnet 5.5 at medium effort.
- [ ] **M12 – Performance and final check.** Shadows update only on change, adaptive pixel ratio, AO fallback, PCSS off on low-power devices; desktop and phone tests; summary.

## Approach for the hardest parts

### Track path around the wheels
- Define the track loop in the side-view (Z–Y) plane as a closed sequence of circles: sprocket (rear, raised), seven road wheels (bottom), idler (front, raised), and return rollers (top).
- Join the circles with external tangent lines between neighbours. Each segment is either a straight tangent or an arc around one wheel, wrapping on the outside of the loop. Return rollers carry the upper run with a slight catenary sag between them.
- Build a cumulative arc-length table across the segments. For link *i*, its position is `(i · pitch + offset) mod loopLength`. Look up the point and tangent at that length, and orient the link so its pitch axis is the wheel axle and its forward axis is the tangent.
- One `InstancedMesh` per link sub-part (shoe body, two split rubber pads, end connectors, wedge bolts), with about 80 links per side. Animate by advancing `offset` with speed and rewriting the instance matrices. Road-wheel and sprocket rotation are derived from the same offset, so nothing slips. Suspension pitch later moves the wheel centres, and the path is rebuilt from them.

### Weathering injection with onBeforeCompile
- One factory wraps `MeshStandardMaterial`. It sets `customProgramCacheKey` per variant, and passes `objectPosition` and `objectNormal` varyings from the vertex shader. This keeps the weathering fixed in object space, so it stays put as the tank moves.
- Triplanar sampling of a tiling 3D noise texture (`Data3DTexture`, generated once, with multiple octaves) plus canvas-made detail maps (scratches, streaks, speckle), blended by `|normal|^k` weights.
- In the fragment shader, insert after `<map_fragment>`: compute masks such as edge wear (a curvature proxy from normal derivatives plus noise), up-facing dust, low-side dust by height, rain streaks (vertical stretched noise under horizontal ledges), soot near the exhaust (a distance field from an exhaust uniform), and oil near the deck and fuel caps. Layer the colours in order: base paint → faded panels → dust → older tan paint chip → bare steel → rust (only when wear is high). Adjust roughness after `<roughnessmap_fragment>` and metalness after `<metalnessmap_fragment>`. Perturb the normal with bump detail after `<normal_fragment_maps>`.
- Uniforms are shared by reference across all materials, so the sliders update everything without recompiling.

### PCSS shadow-chunk patch
- Replace the `getShadow` path in `THREE.ShaderChunk.shadowmap_pars_fragment` with a PCSS version: blocker search over a Poisson disk (with a rotated kernel per pixel) to find the average blocker depth, a penumbra estimate of `(receiver − blocker) / blocker · lightSize`, then a variable-radius PCF filter.
- The patch is enabled per light: a `#define` or a per-light uniform array index controls the PCSS branch, and other lights keep the stock PCF. `renderer.shadowMap.type = PCFShadowMap` keeps depth packing compatible. The patch is applied before any material compiles. A `perf` flag turns the patch off on low-power devices.

### Dune vertex displacement
- A large, densely subdivided plane, made denser near the centre. The vertex shader uses a sum of directional ridged noise, shaped as `1 − |noise|` and softened with smoothstep for rounded crests. A smoothstep ramp keeps the area within a set radius of the tank flat.
- Normals come from finite differences of the same height function inside the shader. The same function is mirrored in JavaScript so rocks, shrubs and props sit on the surface.
- Fine wind ripples are added in the fragment shader as a bump. Two rotated, differently scaled ripple layers are blended by low-frequency noise to break up tiling.
- Driving scrolls a `uOffset` uniform, so the terrain is effectively infinite. Props wrap modulo a tile size around the tank.

### GTAO integration
- Use three r160's `GTAOPass` from `examples/jsm/postprocessing` (loaded via the importmap addon path) inside an `EffectComposer`: `RenderPass` → `GTAOPass` → `OutputPass` (tone mapping and colour space) → a custom `ShaderPass` for vignette and grain (and the thermal LUT when on).
- Tune the radius for metre scale (about 0.5 to 1.0 m), with denoising on. The fallback in `perf` is a cheap SSAO or no AO plus the computed contact shade (a radial-gradient decal under the hull) when frame time is high.

## Notes and limitations
- The three.js CDN may be blocked in the build container. In that case three@0.160 is installed with npm for local testing only, and the shipped importmap still points to the CDN.
- Browser checks run in headless Chromium with SwiftShader (CPU rendering), so frame rates measured there (around 1 fps for the full scene) are not representative of a GPU. Use them only to compare one build against another. After M7 the scene is about 860k triangles (the track and wheels about 310k, the scenery about 300k). The shrubs and rocks were reduced from 2.6M triangles, and the shrubs do not cast shadows.
- PCSS is turned on per light through `shadow.radius >= 10` (softness = radius × 0.001; `pcssRadiusDir()` works it out for directional lights). Directional lights get correct contact-hardening shadows. On the noir spotlight it is an approximation, because spotlight shadow-map depth is non-linear.
- GTAO uses a custom normal material that applies the same terrain displacement, so the occlusion pass sees the real dunes. Alpha-tested wire mesh shows up as solid panels in the AO pass only.
- The thermal sight swaps in heat materials for each mesh (hot engine and exhaust, warm running gear, cold sky). The gunner's view uses a 9° camera at the gunner's sight that follows turret traverse and gun elevation. Its range readout comes from ray-marching the terrain height function.
