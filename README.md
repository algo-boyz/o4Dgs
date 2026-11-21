# o4Dgs Roadmap (revised)

Nov 21, 2026 · @enigma

## Goal and first milestone

The first milestone is a vertical slice: load a static SPZ asset, give it temporal behavior, play it in Odin + Raylib at a controllable timestamp, and export and reload the animated asset without losing data.

That slice crosses format, animation, runtime and rendering before the project takes on research-grade 4D reconstruction. It is done when the `bounce.spz` demo meets every success criterion listed under Demo.

This revision keeps the original plan and tightens it: one milestone numbering, conventions locked before code, a sidecar bound to its SPZ, and easing for the bounce clip.

## Architecture and principles

The project has three layers, and the format never depends on the runtime. Raylib is a viewer, not a dependency of OSPZ.

| Layer | Responsibility |
| --- | --- |
| OSPZ: asset and format | SPZ IO, animation metadata, tracks, validation, later codecs |
| Odin: temporal runtime | Clip selection, clock, sampling, interpolation, transforms |
| Raylib: playback and visualization | Camera, GPU upload, rendering, timeline UI, debug overlay |

Four principles hold across every milestone:

1. Static SPZ stays untouched. An animated asset is an existing SPZ plus metadata, so Niantic SPZ readers keep working and the work in `algo-boyz/ospz` stays useful.
2. The runtime is testable without a GPU or a window. Every animation behavior has a deterministic unit test before it has a visual.
3. Spec decisions come before code. Conventions that would be painful to change once files exist are locked in the next section.
4. Scope grows only on evidence. Cubic interpolation, blending and a binary container wait until the simpler version has proven inadequate.

## Spec conventions

Eight conventions are fixed before any code, because changing one after assets exist breaks every file. Validation rejects anything these rules leave undefined.

| Convention | Decision |
| --- | --- |
| Quaternion order | `[x, y, z, w]`, unit length. The loader renormalizes and rejects zero-length values. Same order as glTF. |
| Axes | The same coordinate system the OSPZ SPZ loader returns, stated once in the spec and never converted inside the sampler. |
| Transform composition | A point maps to `T + R(S ⊙ p)`: scale first, then rotate, then translate. |
| Pivot | Rotation and scale act about an optional asset-level `pivot`, default `[0, 0, 0]`. |
| Time | Seconds as `f64`, never normalized. Key times are strictly increasing; duplicate timestamps are rejected. |
| Single source of truth | `duration` and `loop` live on the clip only. There is no top-level duration, loop or fps. A clip's duration must be at least its last key time. |
| Float policy | `f64` time, `f32` values in memory. JSON is written with shortest round-trip float text, so save then load is bit-exact. |
| Test tolerance | Round-trip tests compare bit patterns. Sampled values compare within 1e-6. |

## Sidecar schema v1

The sidecar `scene.spz.anim.json` sits next to its SPZ and is the only new file in v0. This is a proposed schema, not an existing format.

```json
{
  "format": "ospz-animation",
  "version": 1,
  "asset": {
    "path": "bounce.spz",
    "gaussian_count": 100000,
    "sha256": "<hex digest of the SPZ file>"
  },
  "pivot": [0, 0, 0],
  "clips": [
    {
      "name": "bounce",
      "duration": 4.0,
      "loop": true,
      "tracks": [
        {
          "target": "root",
          "path": "translation",
          "interpolation": "ease_in_out",
          "keys": [
            { "time": 0.0, "value": [0, 0, 0] },
            { "time": 2.0, "value": [0, 1, 0] },
            { "time": 4.0, "value": [0, 0, 0] }
          ]
        }
      ]
    }
  ]
}
```

| Field | Type | Rule |
| --- | --- | --- |
| `format`, `version` | string, int | Must be `ospz-animation` and `1`. Unknown major versions are rejected. |
| `asset.path` | string | Relative to the sidecar. |
| `asset.gaussian_count` | u32 | Must equal the count of the loaded SPZ. |
| `asset.sha256` | string | Optional. When present it must match the SPZ file. |
| `pivot` | vec3 | Optional, default `[0, 0, 0]`. |
| `clips[].name` | string | Non-empty and unique. |
| `clips[].duration` | f64 | At least 0 and at least the last key time. Zero is valid. |
| `clips[].loop` | bool | Default false. |
| `tracks[].target` | string | `root` in v1. |
| `tracks[].path` | enum | `translation`, `rotation` or `scale`. |
| `tracks[].interpolation` | enum | `step`, `linear` or `ease_in_out`. |
| `keys[].value` | array | Length 3 for translation and scale, 4 for rotation. |

`ease_in_out` remaps the normalized time u to u·u·(3 − 2u), then applies the path's normal interpolation: lerp for vectors, SLERP for rotation. It is cheap and deterministic, and it makes the bounce clip smooth without adding cubic splines.

Validation rules:

- Unknown fields, paths, targets and interpolation names are rejected, so a save never silently drops data.
- Sampling before the first key holds the first value, and after the last key holds the last value.
- A path with no track stays at identity: translation 0, rotation identity, scale 1.
- On load, the Gaussian count (and the hash when present) is checked against the SPZ, and a mismatch fails with a named error. Later per-Gaussian IDs depend on a stable Gaussian order, so the pairing has to be verifiable from day one.

## Milestones

Seven milestones replace the earlier mix of phases and milestone numbers. Each is split into PR-sized chunks that can be tested on its own, and none starts before the previous exit criterion is met.

| Milestone | Deliverables | Exit criterion |
| --- | --- | --- |
| M1 Spec and types | `types.odin`, validation rules, deterministic fixtures including the 4-second bounce | Every convention in this doc is encoded in code, and the validation tests pass |
| M2 Animation IO | JSON load and save, named errors, SPZ binding check on gaussian\_count and sha256 | Every fixture round-trips bit-exact, and a mismatched SPZ fails with a named error |
| M3 CPU sampler and animator | Step, linear and ease\_in\_out interpolation, SLERP, a stateless sample at time t, then the Animator clock | All sampling and clock cases pass headless, and a CLI prints sampled transforms |
| M4 Point-debug viewer | Raylib app that loads SPZ and sidecar, draws Gaussian centers, applies the root transform, with camera, controls and timeline | The bounce demo plays, and orbiting the camera never changes the animation |
| M5 Gaussian renderer | M5a GPU buffers and unsorted splat draw with opacity. M5b model-space depth sorting. M5c covariance projection, spherical harmonics handling and profiling | Splats render correctly under the root transform, with frame times recorded |
| M6 Group animation | Gaussian groups, independent transforms, hierarchy, group selection and debug coloring | Two groups animate independently in one asset |
| M7 Sparse deformation | Gaussian IDs, displacement tracks, sampling, compression experiments | A deformation clip plays, and storage numbers exist to inform the v1 container |

## Temporal runtime

The runtime is a clock plus a sampler, with no GPU, window or renderer involved, so every behavior below is unit-testable. Names should follow OSPZ's existing Odin conventions; the boundary is what matters.

```odin
anim_init     :: proc(animation: Animation) -> Animator
anim_play     :: proc(animator: ^Animator, clip_name: string) -> bool
anim_pause    :: proc(animator: ^Animator)
anim_seek     :: proc(animator: ^Animator, time: f64)
anim_update   :: proc(animator: ^Animator, delta_time: f64)
anim_sample   :: proc(animator: ^Animator) -> Transform
anim_set_loop :: proc(animator: ^Animator, enabled: bool)

Animator :: struct {
    clip_index: int,
    time:       f64,
    duration:   f64,
    playing:    bool,
    looping:    bool,
    speed:      f32,
    current:    Transform,
}

Transform :: struct {
    translation: [3]f32,
    rotation:    [4]f32, // quaternion, [x, y, z, w]
    scale:       [3]f32,
}
```

The identity transform is translation `(0, 0, 0)`, rotation `(0, 0, 0, 1)` and scale `(1, 1, 1)`. `anim_play` returns false for an unknown clip name. `looping` starts from the clip's `loop` flag, and `anim_set_loop` overrides it for the session without writing back to the asset.

Clock behavior is defined case by case, because these are where playback code usually breaks:

| Case | Behavior |
| --- | --- |
| Advance | t\_next = t + delta\_time × speed |
| Loop wrap | t = t\_next − d × floor(t\_next / d), with d the clip duration. Use floor, not `%`, so negative times and large deltas wrap correctly. |
| No loop | t clamps to \[0, d\], and `playing` becomes false when the end in the playing direction is reached. |
| Negative speed | Plays backward. Looping wraps from 0 to d, and non-looping stops at 0. |
| Paused | `anim_update` changes nothing. `anim_seek` still works and re-samples. |
| Seek | Wraps when looping, clamps otherwise. |
| Zero duration | Time stays 0 and sampling returns the first key's value, with no division. |
| Clip switch | Time resets to 0, and speed and playing state are kept. There is no blending in v1. |
| Reset | Time returns to 0. |

Sampling finds the key pair around t by binary search and computes u = (t − t0) / (t1 − t0). Duplicate timestamps are rejected upstream, so the denominator is never zero. Interpolation by mode:

- Step holds the first key's value until t reaches the second key.
- Linear uses lerp for translation and scale, and SLERP for rotation.
- Ease\_in\_out remaps u first, then applies linear.
- SLERP negates one quaternion when their dot product is negative so it takes the short path, falls back to normalized lerp when the quaternions are nearly parallel, and normalizes the result.

## Viewer and rendering

The viewer is playback only: one asset, one animated root transform. There is no multi-object scene graph, blending, physics, skinning, neural rendering, live reconstruction or editor.

The renderer must never block animation work, so it ships in two modes:

- Mode A, point debug (M4): draw the Gaussian centers, colored by RGB, index, depth, temporal displacement, or velocity once available. If thousands of centers move correctly in 3D, the temporal system works. This mode stays useful as a debugging tool after the full renderer exists.
- Mode B, Gaussian renderer (M5): opacity, covariance and depth ordering, built in the three sub-steps listed under Milestones.

Three rendering decisions are made now, so they do not surprise M5:

1. The root transform is applied as a model matrix in the renderer, not baked into the splats. The loaded data stays immutable, and the animation never touches the SPZ buffers.
2. Depth sorting runs in model space: transform the camera into the asset's local frame and sort there. A rigid root transform then needs no re-sort of the stored data.
3. If rotation is ever baked into Gaussians, their spherical harmonics must be rotated too, or view-dependent color will be wrong. Point mode hides this, so it is an M5c item, not an M4 one.

Controls:

| Input | Action |
| --- | --- |
| Space | Play or pause |
| Left and right arrows | Step or scrub time |
| Mouse drag | Orbit the camera |
| Scroll | Zoom |
| R | Reset the animation |
| 1 to 4 | Playback speed presets |
| Home and End | Jump to the start or the end |

A debug overlay shows the clip name, time over duration, render FPS, speed, loop and playing state, and a scrub bar. Camera input and animation time are independent: moving the camera never changes the sampled transform.

## Animating the Gaussians

Animation grows in five levels, and each level is added only after the one before it works. The rigid transform ships first and everything past sparse deformation is research.

| Level | Data stored | Evaluated on | Milestone |
| --- | --- | --- | --- |
| Rigid | Transform keys on the root | CPU | M3 to M5 |
| Groups | One transform track per Gaussian group | CPU | M6 |
| Sparse deformation | Gaussian ID plus delta keys | CPU first | M7 |
| Dense deformation | Time-dependent displacement field | GPU | After v2 |
| Native spacetime | 4D primitives | Research | v3 |

Two decisions shape storage and are made before M6:

- Prefer sparse tracks to recording everything. Storing a full transform for 100,000 Gaussians over 120 frames means 12 million samples. Encoding only the Gaussians that move significantly, or a compact motion field shared by a group, is far smaller. This will be one of the most important format decisions.
- Never reorder Gaussians in an SPZ once a sidecar references it. Per-Gaussian IDs and group membership depend on stable order, and the count and hash binding in the sidecar exists to catch violations.

Open for M6: group membership can be stored as index ranges, which are compact but need Gaussians grouped contiguously, or as an explicit per-Gaussian group ID array in the sidecar, which is larger but leaves the SPZ untouched. The second fits the "do not modify the SPZ" principle better, so it is the default unless M6 profiling shows a problem.

## Format evolution

No binary animation container is designed until rigid animation and sparse deformation both work and have been profiled. The JSON sidecar is a development format, and what it teaches decides the container layout.

| Version | Format | Adds |
| --- | --- | --- |
| v0 | SPZ plus JSON sidecar | Static Gaussians, root animation, human-readable files |
| v1 | Animated SPZ container | Bundled assets, clips, binary tracks, metadata |
| v2 | Compressed motion | Sparse tracks, quantization, temporal codecs |
| v3 | Native dynamic Gaussian asset | Motion fields, 4D primitives, streaming, random time access |

One rule keeps the migration cheap: the runtime consumes an in-memory `Animation` value, never a file. Moving from the sidecar to a container then changes only the loader, not the animation API.

## Demo: bounce.spz

The first demo takes one existing SPZ and gives it three trivial but visible clips in a single sidecar, `bounce.spz.anim.json`.

| Clip | Duration | Behavior | Track |
| --- | --- | --- | --- |
| `idle` | 4.0 s, loops | Gentle vertical float | Translation along the vertical axis, ease\_in\_out, three keys: start, small rise, start |
| `rotate` | 8.0 s, loops | Slow turn around the vertical axis | Rotation, linear, five keys at 90° steps from 0° to 360° |
| `bounce` | 4.0 s, loops | Up and down with smooth ease-in and ease-out | Translation, ease\_in\_out, three keys: 0, 1, 0 |

The rotate clip uses 90° steps on purpose. SLERP always takes the short path, so two keys more than 180° apart would turn the wrong way. The 360° key equals the first key as a rotation, so the loop shows no jump.

Success criteria:

- [ ] The existing SPZ loads normally and is not modified.
- [ ] The sidecar loads independently and is checked against the SPZ.
- [ ] Space starts playback, and pause then resume preserves time.
- [ ] Seeking lands on the requested time.
- [ ] Looping shows no visible jump at the boundary.
- [ ] The root transform updates correctly.
- [ ] Camera movement does not affect the animation.
- [ ] Playback is deterministic: the same time always gives the same transform.
- [ ] Save then load loses no animation data.

## First PR

The first PR is the animation foundation: M1, M2 and the stateless half of M3, with nothing involving Raylib. The stateful Animator clock follows in a second PR.

```
animation/
    types.odin
    load.odin
    save.odin
    validate.odin
    interpolate.odin
    sample.odin
    fixtures/
        bounce.spz.anim.json
tests/
    animation_types_test.odin
    animation_validate_test.odin
    animation_io_test.odin
    animation_interpolate_test.odin
    animation_sample_test.odin
```

Every test is deterministic and self-contained:

- JSON round-trip is bit-exact, including the bounce fixture.
- Empty animation, one clip with one track, and multiple clips.
- Keys out of order, and duplicate timestamps, are rejected.
- Invalid interpolation, path or target is rejected.
- A track whose value length does not match its path is rejected.
- A sidecar whose Gaussian count or hash does not match the SPZ fails with a named error.
- Sampling before the first key, after the last key, and between keys.
- A zero-duration clip samples without dividing by zero.
- Quaternions are renormalized, and SLERP takes the short path.
- The ease\_in\_out midpoint equals the linear midpoint, and its quarter points do not.

Out of scope for this PR: the Animator clock and loop-boundary behavior, Raylib, Gaussian groups, deformation and any binary container.

## Risks and open questions

Five items are open, and only the first blocks the first PR.

1. Where the code lives. The earlier draft shows an `o4Dgs/` tree but also builds on `algo-boyz/ospz`. Pick one package home and keep the file names in this doc (`sample.odin`, not `sampler.odin`).
2. Coordinate system. Confirm which axes the OSPZ SPZ loader returns. The "vertical axis" in the demo clips and the sidecar translations both depend on it.
3. Quaternion order in the Raylib bindings. Confirm that its quaternion type matches `[x, y, z, w]` before wiring the viewer in M4, or convert at one boundary.
4. Renderer feasibility. M5 needs custom shaders and buffer or instanced drawing through rlgl. A throwaway spike that draws unsorted splats should come before M5 is scheduled.
5. Citations. The earlier draft's MotionGS and Schrodinger links point to image files, not papers. Replace them with real references before the roadmap is published.
