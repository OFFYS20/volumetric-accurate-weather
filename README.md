# Aeolus — Volumetric Weather Sandbox

System architecture for a real-time volumetric weather and atmospheric phenomena
sandbox targeting modern PC GPUs (DX12 / Vulkan 1.3, 4K at 60 Hz).

`index.html` is the whole document: a single self-contained page with a main menu
that opens seven modules.

| Module | Contents |
| --- | --- |
| The Map | Design axioms, frame graph, shared resources, the phenomenon-driver model |
| Tier A · Baseline & Optics | Cumuliform and stratiform decks, virga, radiation fog, crepuscular rays, 22° halos, lenticulars |
| Tier B · Severe Forces | Supercells and anvils, tornadic vortices, microbursts, haboobs, ash plumes with internal lightning, Fujiwhara merging |
| Tier C · Rare & Upper Air | Sprites and blue jets, noctilucent and nacreous clouds, ball lightning, fallstreak holes, glories and Brocken spectres, light pillars |
| Physics API | Grid cascade, CAPE / CIN / LCL / EL mapping, shear and vorticity, Kelvin–Helmholtz, microphysics, scenario format |
| Raymarch Pipeline | Empty-space skipping, the march loop, Beer–Powder multi-scattering and phase functions, quarter-resolution temporal reconstruction |
| Budget & Scaling | Per-pass millisecond costs, VRAM residency, quality presets, adaptive controller |

The organising rule throughout: phenomena are gated on solved physical diagnostics
rather than spawned by triggers, and everything that scatters light resolves through
one raymarching pass.

## Viewing

Open `index.html` in any modern browser — no build step, no dependencies. Fonts load
from Google Fonts; the page falls back to system faces offline.

Keyboard: <kbd>1</kbd>–<kbd>7</kbd> open a module, <kbd>Esc</kbd> returns to the menu,
<kbd>←</kbd> / <kbd>→</kbd> step between modules.
