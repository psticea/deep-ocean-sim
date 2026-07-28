# NADIR VENT FIELD — 10,916 m

A real-time WebGL2 simulation of an active black-smoker hydrothermal vent field on
the floor of a hadal trench, explored from the ROV **EREBUS**.

**One file. No build step, no server, no assets.** Open `index.html`, or run it here:

### ▶ **[psticea.github.io/deep-ocean-sim](https://psticea.github.io/deep-ocean-sim/)**

Every polygon, texture, material and shader in the scene is generated in code at
load time. The only dependency is three.js from a CDN.

---

## The concept

Zero ambient light. At 10,916 m there is no sun — every photon in the frame comes
from one of three places:

- the ROV's four LED lamps,
- the incandescent throat of the smokers (~350 °C fluid venting into 1.8 °C water),
- bioluminescence.

The chemosynthetic community is built around the vents: *Riftia* tube worms that
retract into their chitin tubes when the ROV lamp swings across them, swarming
*Rimicaris* shrimp, vesicomyid clam beds, siphonophores, ctenophores, jellyfish,
an anglerfish trailing its esca, and a schooling shoal that scatters from the lights.

## Render pipeline

| Pass | What it does |
|---|---|
| Shadow | Linear light-space distance from the key lamp — casts real beam shadows |
| Scene | RGBA16F + depth. Forward-rendered, custom BRDFs, per-fragment Beer–Lambert RGB absorption |
| Distortion | Half-res RG vector field written by curl-noise sprites — hydrothermal shimmer |
| Volumetric | Half-res ray-march: HG phase function, shadowed god rays, vent plume density, sonar shell, interleaved-gradient dither, depth-aware upsample |
| Bloom | Progressive down/upsample chain |
| Final | ACES tonemap, chromatic aberration, barrel distortion, vignette, procedural lens dirt, sensor grain |

## Techniques

- **Optical model** — per-channel extinction (`σ ≈ 0.135 / 0.047 / 0.022 per metre`),
  applied *twice*: lamp→surface and surface→camera. Red dies within metres, which is
  why the abyss is teal.
- **Terrain** — 560² CPU-baked heightfield: trench walls, a fault graben carrying the
  vent line, pillow lava, sediment ripples, sulfide mounds. Shaded with a procedural
  basalt/silt/bacterial-mat material using analytic-derivative gradient noise for
  detail normals (value + gradient from a single fBm evaluation) and Worley cells for
  fracture patterns. Runtime queries hit a bilinear grid lookup, not the fBm stack.
- **Chimneys** — procedurally grown sulfide spires with flange shelves, branching
  sub-spires and a per-vertex heat gradient driving incandescence through fracture veins.
- **Plumes** — GPU point systems advected by curl noise, lit as participating media by
  the full light rig, quenching from incandescent to black smoke within a metre.
- **Marine snow** — up to 56k camera-wrapped points forming an infinite field; each floc
  is an irregular tumbling aggregate lit in-shader with forward scattering.
- **Soft bodies** — all creature motion is vertex-shader deformation: travelling waves
  along a siphonophore stem frame, metachronal ctene-row beating with running diffraction
  colour, jet-propulsion bell contraction, anguilliform tail beats.
- **Behaviour** — CPU boids for the shoal (cohesion / alignment / separation + light
  avoidance), orbital swarming for vent shrimp, light-triggered retraction for tube worms.
- **Bioluminescent sparks** — dinoflagellate-style flashes triggered by thruster shear.
- **Adaptive quality** — three tiers auto-selected from a rolling frame-time average,
  scaling render resolution, ray-march steps, shadow map size and particle counts.

## Controls

| | |
|---|---|
| `W A S D` | thrust |
| `R` / `F` | ascend / descend |
| drag (or double-click to lock pointer) | look |
| `Shift` | boost |
| `Space` | sonar ping |
| `1`–`4` | toggle individual lamps |
| `[` `]` / wheel | lamp power |
| `V` | chase camera |
| `C` | cinematic transit |
| `H` | hide HUD |
| `P` | cycle quality manually |

Starts in cinematic transit. Any input takes manual control.

---

Built by Claude Opus 5.