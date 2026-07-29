# LAGOON 9 — a sunlit coral reef

A real-time WebGL2 simulation of a shallow fringing reef, 2–16 m, explored as a free-swimming diver.

**One file. No build step, no server, no assets.** Open `index.html`, or run it here:

### ▶ **[psticea.github.io/deep-ocean-sim](https://psticea.github.io/deep-ocean-sim/)**

Every polygon, texture, material and shader is generated in code at load time — bathymetry,
corals, sponges, seagrass, fish, turtles, rays, crabs, the octopus, the sea surface, the
caustics. The only dependency is three.js from a CDN.

> The previous build in this repo — an abyssal hydrothermal vent field at 10,916 m — is
> still here as **[abyss.html](https://psticea.github.io/deep-ocean-sim/abyss.html)**.

---

## What the light does

Shallow water is the opposite problem to the abyss: there is far too much light, and all of
it arrives through a moving refracting surface. Three things follow from that, and they are
the spine of this build.

**One wave model drives everything.** A four-component Gerstner sum defines the surface. The
same maths runs in JavaScript and in GLSL, so the sea surface mesh, the caustics, the orbital
surge that sways every plant, the bubble drift and the push you feel on the camera are all
the same swell. Change the sea state with `[` / `]` and the whole scene responds coherently.

**Caustics are computed, not faked.** For a point on the seabed the shader walks back up to
the surface along the mean refracted sun direction, refracts the sun through the local wave
normal with `refract()` at η = 1/1.333, and forward-maps a small photon bundle down to that
point's depth. The determinant of the resulting 2×2 Jacobian is how much the surface squeezed
the bundle; the inverse of that area compression is the caustic gain. It is the real
definition, evaluated per pixel.

A detail worth calling out: the swell alone produces almost no caustics. Its shortest
component is 5.4 m, far too long-crested to focus light. Caustic nets are made by
centimetre-to-metre capillary–gravity wavelets, so those get their own analytic field of five
crossing trains at widely spread angles — spread matters, or the focus lines come out as
parallel bars instead of a net.

**The sun shafts are the same caustic field seen edge-on.** The volumetric ray-march samples
`caustics()` at every step, so the beams in the water column and the pattern on the sand are
one phenomenon, not two effects that happen to be near each other.

Looking up gives you **Snell's window** for free: the whole sky compressed into a 96° cone,
with total internal reflection outside it, straight out of `refract()` returning zero past the
critical angle.

## Render pipeline

| Pass | What it does |
|---|---|
| Shadow | Orthographic sun depth, edge-faded so the slab boundary never shows in the water |
| Scene | RGBA16F + depth. Custom BRDF, wrapped diffuse, per-channel Beer–Lambert absorption |
| Volumetric | Half-res march: HG phase, sun shadow, caustic modulation, blue-noise jitter, depth-aware upsample |
| Bloom | Progressive down/upsample chain |
| Final | ACES, dome-port barrel + chromatic aberration, S-curve grade, dive-mask matte, grain |

Fog is analytic single scattering: `col·e^(−σₑd) + (σₛ/σₑ)·E·(1−e^(−σₑd))`, with σₑ = σₐ + σₛ
so the single-scatter albedo can never exceed 1. The clear colour is recomputed each frame
from the same equation so the far field and the fog meet without a seam.

## What lives there

- **Bathymetry** — lagoon → back reef → crest → spur-and-groove → fore-reef terrace → slope,
  with sand channels and isolated bommies. Sand carries surge-aligned wave ripples; reef rock
  carries coralline crust, turf algae and dead-coral rubble.
- **Corals** — recursive staghorn *Acropora*, table coral, brain coral with domain-warped
  meandering valleys, netted gorgonian sea fans, barrel and tube sponges, lobed soft corals
  with polyp fuzz. ~10,000 colonies across 30 palettes.
- **Plants** — a *Thalassia* seagrass meadow and branching *Halimeda*, all leaning together
  in the orbital surge and flicking back on the return stroke.
- **Fish** — one parametric reef-fish generator, laterally compressed the way reef fish
  actually are, with per-species pattern shaders: anthias, sergeant major, blue tang,
  parrotfish, butterflyfish, trevally, moon wrasse. Spatial-hash boids for the schools,
  a cheap flow-field swarm for the anthias cloud, grazing behaviour for the reef-pickers.
- **Large animals** — green sea turtle flying on its front flippers, spotted eagle ray with a
  travelling wave running out along the wings, blacktip reef shark.
- **Benthos** — sea star with rippling tube feet, reef crab with an alternating gait, urchin,
  sea cucumber, and a day octopus whose skin **samples the same `substrate()` function the
  seabed uses** and blends toward it — the camouflage is literally reading the rock it sits on.

## Controls

| | |
|---|---|
| `W A S D` | swim |
| `R` / `F` | ascend / descend |
| drag (double-click to lock pointer) | look |
| `Shift` | fin hard |
| `Space` | take a photo |
| `T` | torch |
| `[` `]` | sea state |
| `C` | cinematic drift |
| `M` | dive mask |
| `H` | HUD |
| `P` | quality tier |

Starts drifting. Any input takes control.

## Performance — read this honestly

The only GPU available to me for testing was an **Intel HD Graphics 4000**, a 2012 integrated
part. On that machine this scene settles at roughly **13–21 fps at the LOW tier**, and the numbers
drift between runs in a way that looks like thermal throttling.

I have **not** been able to verify the 60 fps target on modern hardware. What I did do:

- Fixed a metric bug that was clamping `dt` and reporting a floor of 20 fps for a build that
  was actually running at 1.4 fps. Never trust a frame counter you clamped.
- Added a spatial-grid distance cull for the ~10,000 benthic instances (~6× on its own).
- Removed a marine-snow overdraw disaster: 90-pixel additive sprites each running a full
  caustic evaluation.
- Replaced `sin()`-based hashing with an integer hash — Worley alone called it 27× per pixel.
- Split the flora shader so seagrass stops paying for coral-only noise.
- Runtime detail uniforms (noise octaves, ripple train count) driven by the quality tier.

Three quality tiers are selected automatically from a rolling frame-time average, plus a
distress valve that drops render resolution (and only slightly, benthic draw distance) if the
lowest tier still misses frames — an empty reef is a worse failure than a soft one.
It starts at LOW and earns its way up, so a capable GPU should climb to HIGH within seconds.
Whether that lands at 60 fps, I genuinely don't know.

---

Built by Claude Opus 5.
