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
| Chase lock | `F` — the camera rides along with the storm's motion |
| Radar | `R` toggles the reflectivity scope |
| Interface | `Tab` toggles the control panel, `H` hides everything |

Touch devices get an on-screen pad; drag still looks around.

## What it simulates

Eleven scenarios set up an environment; the phenomena then follow from it.

- **Tier A** — cumuliform and stratiform decks, altocumulus billow rows, rain shafts
  and the scud under them, virga, radiation fog, crepuscular rays, the 42° rainbow with
  its secondary and Alexander's dark band, 22° halo with parhelia, lenticular stacks,
  cirrus veil.
- **Tier B** — supercells with sheared anvils, mammatus on the anvil underside,
  overshooting tops, a forward-flank precipitation core with the inflow notch open
  beside it, tornadic funnels
  with debris, microburst outflows, haboob density currents, volcanic ash plumes lit
  from inside by their own lightning, Fujiwhara-style twin circulations.
- **Tier C** — red sprites, noctilucent cloud, ball lightning, fallstreak holes,
  glories, light pillars.

## Rain and fog

Rain is a volume, not a texture: shafts hang out of the base of any deck deep enough
for drops to grow by collision, lean downwind by the wind speed over the drop's ~9 m/s
terminal velocity, and either reach the ground or evaporate into virga on the way,
depending on how hard it is falling. They scatter far less per metre than cloud does
and sit in the cloud's own shadow, which is what makes a curtain a translucent
blue-grey veil rather than more cloud. Under a supercell the core is one-sided —
downwind of the updraft — so the rain-free wedge a chaser parks in stays open.

Radiation fog is not a switch. It forms where three conditions coincide: the ground is
radiating (sun near or below the horizon), the air is near saturation (small dew-point
depression), and the wind is light enough not to mix the cold surface layer away. Raise
the wind above about 7 m/s and it disappears. Because cold air is dense and runs
downhill, the top of a fog bank is a *level* surface rather than a blanket following the
ground — it fills the terrain the way water fills a basin, and ridges stand out of it as
islands. Different basins pond to different depths. The **Valley Dusk** scenario switches
nothing on at all; it just sets an environment in which fog has to form.

## Chasing

Storms are steered by the flow they grow in — a supercell deviates about 30°
right of the mean wind at three-quarters of its speed, so it can be followed,
intercepted, or lost. The HUD carries range, bearing and storm motion; the
scope in the corner is a reflectivity display sampled from the same
precipitation field the renderer uses, so the hook and the rain-free notch you
see on radar are the ones you can fly into.

Lightning is geometry rather than a screen flash: a midpoint-displaced channel
from the charge region to the ground, forked, lighting the cloud from inside
and dimmed by whatever cloud sits in front of it. Thunder is synthesised and
arrives late by the distance divided by the speed of sound, low-passed with
range so a near strike cracks and a far one rolls. Rain falls on the lens when
you are under the precipitation core, and you can hear it — the same field drives
the volumetric shafts, the scope and the audio, so all three agree.

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
