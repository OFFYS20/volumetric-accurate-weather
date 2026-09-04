# Aeolus — working notes

A real-time volumetric atmosphere raymarched in WebGL2. `index.html` is the entire
application: markup, CSS, GLSL and JavaScript in one file, no build step, no
dependencies, no package.json. Open it in a browser and it runs. Keep it that way —
if a change would need a bundler, it is the wrong change.

`architecture.html` is a standalone design document. `README.md` is for users.

## The one rule

**Everything visible is a consequence of a physical parameter, not a preset.** Cloud
base is `125·(T − T_d)` metres because that is what the lifting condensation level is.
The anvil is where the updraft ran out of buoyancy, not a disc placed on top of a
tower. The sun reddens at the horizon because its path through the air is long, and
for no other reason. When something looks wrong, the fix is almost always to find the
physical quantity that is missing or mis-scaled — not to add a correction term.

Comments in this codebase explain **why**, in physical terms, and are load-bearing.
Match that. A comment that restates the code is worse than none.

## Layout of `index.html`

Shaders are `<script type="x-shader/*">` blocks, fetched by id and compiled at boot.

| id | what it does |
| --- | --- |
| `vs` | fullscreen triangle, no attributes, shared by every fullscreen pass |
| `fsNoise` | bakes the two 3-D noise volumes (Perlin–Worley shape, Worley erosion) |
| `fsTrans` | bakes the 256×64 atmospheric transmittance table |
| `fsMain` | the renderer: terrain trace, cloud march, sky, optics |
| `fsResolve` | temporal resolve — reprojection and neighbourhood clipping |
| `fsDown` / `fsUp` | bloom pyramid, down and up halves |
| `fsPresent` | exposure, bloom, vignette, AgX, grade, rain, grain |
| `vsBolt` / `fsBolt` | lightning channels, drawn as additive geometry over the composite |

JavaScript follows, inside one IIFE. Roughly in order: GL plumbing, volume and LUT
baking, render targets, state (`S`, `A`, `derived`), the terrain mirror, lightning,
turrets, the weather engine (`WX`), audio, scenarios, radar, UI, input, and `frame()`.

## Frame order

```
bakeTrans            (only when the aerosol load moved)
fsMain   → sceneT + auxT     linear radiance, and a per-pixel distance
fsResolve→ rts[cur]          merged into the running average
fsDown/fsUp → bloomLv[0]     pyramid built from rts[cur]
fsPresent→ canvas            tonemap and grade
fsBolt   → canvas            additive, over the top
```

## Invariants worth not breaking

**Linear until the last pass.** `fsMain` writes unclamped radiance. Nothing between it
and `fsPresent` may tonemap, clamp to 1, or gamma-encode. The bloom and the highlight
roll-off both depend on values well above 1.

**Filter to the footprint, not the distance.** A ray grazing the ground covers hundreds
of metres of it in one pixel however close it is. Terrain normals, detail, mottle and
self-shadowing all take `foot`, computed as the geometric mean of the two footprint
axes. Cloud noise lookups take `mipFor(lod, texel)`. Anything sampled without one of
these will alias, and against a low sun the aliasing arrives as crawling speckle.

**The CPU mirrors of shader fields must stay in step.** `precipAt()` mirrors
`rainRate()`, and `terrainJS()` mirrors `terrainH()`. The radar scope, the audio bed and
the rain on the lens all read the CPU copies; if they drift, the scope shows rain that
is not being drawn. `derived.core` is passed to the shader as `uCore` for the same
reason — the turret solver and `stormForm` have to agree on how wide the storm is.

**Scenarios own their exposure.** `applyScenario` resets `S.exposure` to 1.0 when the
scenario does not name one, so brightness does not depend on the order you clicked
through the menu.

**Uniform lookups are lazy** (a `Proxy` over `getUniformLocation`), so an unused uniform
is harmless — setting a null location is a no-op in WebGL. Do not assume a uniform is
live just because it is declared.

**Float targets may be absent.** `hdrOK` gates the HDR path and `reprojOK` gates
reprojection. Both fall back rather than failing.

## The storm

`stormForm()` is one radius-versus-height profile from cloud base to overshooting top —
tower, shoulder, anvil and overshoot are the same function evaluated at different
heights, not separate primitives unioned together. A hard `max()` between primitives
leaves a crease that no amount of tuning the primitives can remove; that is what `smax`
is for, and it is why the storm reads as one body.

Width scales off depth (`derived.core`), because a convective plume entrains through its
whole flank and a deeper storm is a proportionally wider one. A supercell updraft is
ten to twenty kilometres across against thirteen of depth. If a storm ever starts
looking like a mushroom again, that ratio is the first thing to check.

## The weather engine

`WX` integrates a day forward. Nothing in it is scheduled by the clock; everything is
driven by state:

- The surface warms and cools with a first-order lag, which is why the hottest hour is
  mid-afternoon and the coldest is just before dawn.
- Cumulus need **thermals**, gated on the ground having actually warmed above its
  overnight minimum. Without that gate the near-ground condensation level at dawn grows
  a cumulus field out of the first minute of daylight.
- The morning cap is broken by surface warming, not by the hour — a cool day stays blue
  however much water is under it.
- A pulse cell lives about fifty minutes; shear separates updraft from downdraft and
  lets it last hours instead.
- The storm spends the instability that built it, so the sky quietens behind it, and the
  anvil outlives the storm as cirrus.

Sliders in `WX_DRIVEN` are written by the engine; touching one hands that control back
to the user. Fog and altocumulus instead go through `derived.fogAuto` / `derived.altoc`,
so the chip stays a manual override rather than something the engine fights.

`wxSeek()` walks six hours forward from a cold start, because the lags mean you cannot
simply assign the state at a given hour.

## Verifying a change

There is no test suite; the output is an image, so look at it. Chromium and Playwright
are available. A harness that boots the page, clicks a scenario card, lets the
accumulation buffer settle and screenshots:

```js
const browser = await chromium.launch({
  executablePath: '/opt/pw-browsers/chromium-<ver>/chrome-linux/chrome',
  args: ['--use-angle=swiftshader', '--enable-unsafe-swiftshader', '--use-gl=angle',
         '--ignore-gpu-blocklist', '--no-sandbox', '--disable-dev-shm-usage'],
});
// wait for #menu to lose .hidden, then click nth '#scnCards .scn',
// wait ~14 s, hide '#ui,.hud,#panel,#radarwrap,#hint,.topbar', screenshot.
```

Things that will mislead you:

- **SwiftShader is software.** Frames take seconds. Anything heavy (Rainband, or a big
  storm filling the frame) will blow a 180 s screenshot timeout — drop the viewport and
  do not use photo mode when that happens.
- **Adaptive resolution.** `dynScale` falls to 0.55 when frame time is long, and the
  quality preset scales again on top, so a software render is often a third of the
  window upscaled. Judging sharpness on that judges the upscaler. Photo mode (`P`)
  forces full internal resolution — use it when the frame rate can afford it.
- **A still frame hides motion artefacts.** Reprojection and ghosting only show while
  panning. Drag the mouse in the harness and screenshot mid-drag.
- **Check a low sun.** Most mistakes in the scattering model are invisible at noon and
  obvious at four degrees of solar elevation. Valley Dusk, Sea of Fog and Night Sky are
  the ones that catch them.

## Git

Branch `claude/volumetric-weather-sandbox-nzqics`. Commit messages here run long on
purpose: they say what was wrong, why it was wrong, and what the fix rests on. Keep
that.
