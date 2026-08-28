# Aeolus — Volumetric Weather Sandbox

A real-time volumetric atmosphere you fly around in, running entirely in a WebGL2
fragment shader. Cloud base, tower height, rotation, funnel width and every optical
effect are consequences of the physical parameters on the control panel — change the
dew-point depression and every cloud base in the scene moves with it.

`index.html` is the whole thing: one self-contained page, no build step, no dependencies.
`architecture.html` is the system-architecture document the sandbox is built from.

## Running it

Open `index.html` in a browser with WebGL2 and hardware acceleration.

| | |
| --- | --- |
| Look | drag anywhere on the view |
| Fly | `W` `A` `S` `D` (or arrow keys), `Q` / `E` for up and down |
| Faster | hold `Shift`; scroll wheel sets the base speed |
| Scenarios | `Esc` returns to the menu |
| Interface | `Tab` toggles the control panel, `H` hides everything |

Touch devices get an on-screen pad; drag still looks around.

## What it simulates

Nine scenarios set up an environment; the phenomena then follow from it.

- **Tier A** — cumuliform and stratiform decks, virga, radiation fog, crepuscular rays,
  22° halo with parhelia, lenticular stacks, cirrus veil.
- **Tier B** — supercells with sheared anvils and overshooting tops, tornadic funnels
  with debris, microburst outflows, haboob density currents, volcanic ash plumes lit
  from inside by their own lightning, Fujiwhara-style twin circulations.
- **Tier C** — red sprites, noctilucent cloud, ball lightning, fallstreak holes,
  glories, light pillars.

## How it renders

One fragment shader per frame does the lot: a curvature-corrected raymarch through a
single participating medium (cloud droplets, dust, ash and fog differ only in their
scattering coefficients), a heightfield terrain trace, an analytic sky, and a
scattering-angle pass for the optical phenomena.

- Two 3-D noise volumes (Perlin–Worley shape, Worley erosion) are generated on the GPU
  at load by rendering into texture layers.
- Beer–Powder attenuation with three energy-attenuated scattering octaves and a
  dual-lobe Henyey–Greenstein phase function.
- The march runs at a fraction of display resolution with a blue-noise-jittered start,
  and accumulates into a history buffer that converges when the camera is still.
- Internal resolution adapts to hold frame time; four quality presets set step counts.

Physical parameters map the way they do in the atmosphere: cloud base is
`125 · (T − T_d)` metres, `w_max = √(2·CAPE)` drives tower height and overshoot, the
0–6 km shear tilts the updraft and shears the anvil downwind, and the tornado's
condensation funnel only reaches the ground when the condensation level is low enough.
