# **o4Dgs: Roadmap**

Evolve it into a format-aware foundation for animated Gaussian assets, then build a small playback engine that proves the format works.

First milestone should be extremely concrete:

> Load a static SPZ asset, give it temporal behavior, play it in Odin + Raylib at a controllable timestamp, and export/reload the animated asset without losing data.

This gives us a vertical slice through format, animation, runtime, and rendering before we take on research-grade 4D reconstruction.

## **Architecture**

**OSPZ — Asset & Format Layer**

SPZ IO · metadata · animation tracks · codecs

**Odin — Temporal Runtime**

Clip sampling · time · transforms · interpolation

**Raylib — Playback & Visualization**

Camera · GPU upload · rendering · timeline · debug

The format must not depend on the runtime. Raylib is a viewer, not a dependency of OSPZ.

# **Phase 0 — Define the animation model**

Before writing code, establish a simple model we can grow without breaking compatibility.

## **0.1 Static Gaussians remain the foundation**

We do **not** change how an existing SPZ stores its Gaussian data.

An animated asset initially consists of:

- One existing SPZ asset.
- Temporal metadata.
- Animation clips.
- Transform tracks.
- Optional future deformation tracks.

This avoids breaking Niantic SPZ compatibility and keeps the work in `https://github.com/algo-boyz/ospz` useful.

## **0.2 Animation hierarchy**

**Scene**

**Node / Gaussian Group**

**Animation Clip**

**Position**

**Rotation**

**Scale**

Eventually we want a hierarchy where a scene can contain independently animated Gaussian objects. However, for v0, keep it simple:

- One asset = one Gaussian cloud.
- One or more animation clips.
- Animation transforms the entire cloud.
- Time is normalized or expressed in seconds.
- Interpolation is deterministic.

This gives us a working foundation for moving objects, camera-independent animation, and eventually character/object animation / destruction.

# **Phase 1 — Extend OSPZ with temporal metadata**

Milestone 1: Animated asset specification

## **1.1 Don't fork the SPZ binary format yet**

We'll initially use a sidecar metadata format.

```
scene.spz
scene.spz.anim.json
```

- Existing SPZ readers remain unaffected.
- We can iterate on the animation schema rapidly.
- We don't need to implement container compression or offsets yet.
- We can test animation independently from Gaussian IO.
- We can later migrate to a self-contained container without changing the animation API.

## **1.2 Initial schema**

Something intentionally small and extensible:

```

{
  "format": "ospz-animation",
  "version": 1,
  "asset": "scene.spz",
  "duration": 4.0,
  "fps": 30,
  "loop": true,
  "clips": [
    {
      "name": "idle",
      "duration": 4.0,
      "tracks": [
        {
          "target": "root",
          "path": "translation",
          "interpolation": "linear",
          "keys": [
            { "time": 0.0, "value": [0, 0, 0] },
            { "time": 2.0, "value": [1, 0, 0] },
            { "time": 4.0, "value": [0, 0, 0] }
          ]
        }
      ]
    }
  ]
}

```

Proposal for SPZ animation schema, not existing format

### **Version 1 data types**

| Type          | Representation       | Purpose            |
| ------------- | -------------------- | ------------------ |
| Time          | `f64`                | Precise clip time  |
| Translation   | `vec3`               | Position offset    |
| Rotation      | `quat`               | Orientation        |
| Scale         | `vec3`               | Scale multiplier   |
| Interpolation | enum                 | Step / linear      |
| Clip          | duration + tracks    | Animation sequence |
| Track         | target + path + keys | Animated property  |
| Keyframe      | time + value         | State at timestamp |

**quaternions for rotation**, even for the MVP. Avoid Euler-angle interpolation problems before they become part of format.

## **1.3 OSPZ implementation tasks**

```
o4Dgs/
│   ├── types.odin
│   ├── load.odin
│   ├── save.odin
│   ├── validate.odin
│   ├── sample.odin
│   └── interpolate.odin
│
└── tests/
    ├── animation_types_test.odin
    ├── animation_validate_test.odin
    ├── animation_sample_test.odin
    └── animation_interpolate_test.odin
```

### **Tests**

All tests should be deterministic and self-contained.

- JSON round-trip.
- Empty animation.
- One clip, one track.
- Multiple clips.
- Keyframes ordered correctly.
- Duplicate timestamps rejected or explicitly defined.
- Invalid interpolation type.
- Track type/value mismatch.
- Sampling before first key.
- Sampling after last key.
- Sampling between keys.
- Zero-duration clip.
- Quaternion normalization.
- Loop boundary behavior.

# **Phase 2 — Build the Odin temporal runtime**

Milestone 2: Playback without rendering

Before Raylib, we should be able to animate an asset from the command line and inspect the sampled transforms.

This is important: the animation engine should be testable without a GPU, a window, or a renderer.

## **2.1 Runtime API**

Aim for API roughly like so:

```
anim_init :: proc(animation: Animation) -> Animator

anim_play :: proc(animator: ^Animator, clip_name: string)

anim_pause :: proc(animator: ^Animator)

anim_seek :: proc(animator: ^Animator, time: f64)

anim_update :: proc(animator: ^Animator, delta_time: f64)

anim_sample :: proc(animator: ^Animator) -> Transform

anim_set_loop :: proc(animator: ^Animator, enabled: bool)
```

The actual API should follow OSPZ's existing Odin naming conventions, but conceptually that's the boundary I want.

## **2.2 Animator state**

```
Animator :: struct {
    clip_index: int,
    time: f64,
    duration: f64,

    playing: bool,
    looping: bool,
    speed: f32,

    current: Transform,
}
```

A transform:

```
Transform :: struct {
    translation: [3]f32,
    rotation: [4]f32, // quaternion
    scale:      [3]f32,
}
```

Initial transform:

- Translation = `(0, 0, 0)`
- Rotation = identity quaternion
- Scale = `(1, 1, 1)`

## **2.3 Time advancement**

## **Playback clock**

\\[ t\_{next}=t\_{current}+\Delta t\cdot speed \\]

The clock advances independently of rendering and samples the animation at the requested time.

Looping:

\\[ t = t\_{\text{next}} \bmod duration \\]

Non-looping:

\\[ t = \min(t\_{\text{next}}, duration) \\]

The runtime should explicitly handle:

- Negative playback speed.
- Paused playback.
- Seeking.
- Looping.
- Large delta time.
- Zero duration.
- Clip switching.
- Playback reset.

For v1, we do not need blending between clips yet.

## **2.4 Sampling and interpolation**

For a keyframe pair \\(K_0, K_1\\), calculate normalized interpolation time:

\\[ u = \frac{t-t_0}{t_1-t_0} \\]

**Translation & scale**

\\[ v(t)=(1-u)v_0+uv_1 \\]

**Rotation**

\\[ q(t)=\operatorname{SLERP}(q_0,q_1,u) \\]

We need:

- Linear interpolation for vectors.
- Spherical linear interpolation for quaternion rotation.
- Step interpolation for discrete properties.
- Keyframe boundary handling.

I'd initially avoid cubic interpolation. Add it only after we have visual evidence that linear motion is inadequate.

# **Phase 3 — Minimal Odin + Raylib 4D playback viewer**

Milestone 3: First visual playback

## **3.1 Scope**

A single Gaussian asset with one animated root transform.

No:

- Multi-object scene graph.
- Animation blending.
- Physics.
- Skinning.
- Neural rendering.
- Live reconstruction.
- Complex editor.

Only playback.

## **3.2 Viewer architecture**

```
┌──────────────────────────────────────────┐
│              Raylib App                │
├──────────────────────────────────────────┤
│ Input                                  │
│   ├── Play / Pause                     │
│   ├── Seek                             │
│   ├── Speed                           │
│   └── Camera                           │
├──────────────────────────────────────────┤
│ Odin Animation Runtime                  │
│   ├── Clock                           │
│   ├── Clip                            │
│   ├── Track Sampler                   │
│   └── Transform                      │
├──────────────────────────────────────────┤
│ OSPZ                                  │
│   ├── Load SPZ                         │
│   └── Load Animation Metadata           │
├──────────────────────────────────────────┤
│ Gaussian Renderer                       │
│   ├── GPU Buffers                      │
│   ├── Splat Sorting                    │
│   └── Draw                             │
└──────────────────────────────────────────┘
```

## **3.3 Controls**

| Input               | Action                 |
| ------------------- | ---------------------- |
| Space               | Play / pause           |
| Left / right arrows | Step or scrub time     |
| Mouse               | Orbit camera           |
| Scroll              | Zoom                   |
| R                   | Reset animation        |
| 1–4                 | Playback speed presets |
| Home                | Jump to start          |
| End                 | Jump to end            |

I'd also include a small debug overlay:

```
OSPZ 4D PREVIEW

Clip: idle
Time: 01.42 / 04.00
FPS: 144
Speed: 1.0x
Loop: ON
Playing: YES

[━━━━━━━━━━━━━━━●━━━━━━━━━━━━━━]
```

## **3.4 The rendering shortcut that gets us moving**

We shouldn't let the renderer block the animation work.

I'd use two rendering modes:

**Mode A — Point debug**

Quickest path to validating transforms and time.

**Mode B — Gaussian renderer**

Full splat rendering with opacity, covariance and depth ordering.

**Mode A comes first.**

If we can load thousands of Gaussian centers and move the cloud correctly in 3D, our temporal system is already working. Then we add actual Gaussian rendering.

The point-debug renderer should draw the Gaussian centers, optionally colored by:

- RGB.
- Gaussian index.
- Depth.
- Temporal displacement.
- Velocity (once available).

This gives us a valuable debugging tool that will remain useful even after a full renderer exists.

# **Phase 4 — Animate the Gaussians**

Milestone 4: Beyond rigid transforms

Once root animation works, the next step is to animate individual Gaussians or subsets of the cloud.

This is where we must resist overengineering.

**Level 1 — Rigid transform**

Transform the entire Gaussian cloud.

Translation, rotation, scale.

**Level 2 — Gaussian group transforms**

Divide a cloud into groups, each with an independent transform and animation track.

Think: doors, vehicles, props, body parts.

[MotionGS: Exploring Explicit Motion Guidance for Deformable 3D Gaussian Splatting](https://images.openai.com/static-rsc-4/sL-vl0fEpd4aD56SSqKtpAZVilPWZDtJJiF9aA3cXadnsFQReXO1y-atSi-QsmdOg1Mg6-a3Jy6orEOAVlvYo8OuCM9iNEK4FJzXveivEvG2_p9osgsKA4RhgeIIIqsmIzWREBg3pg0XuD07JAEWiYyjNLcRcvb4pzf-pA_-EGk?purpose=inline)

**Level 3 — Per-Gaussian deformation**

Sparse displacement tracks or continuous functions deform individual Gaussian positions, and eventually rotations and scales.

This is the bridge to research-grade 4DGS.

### **Proposed progression**

| Model              | Data stored                       | Evaluation     |
| ------------------ | --------------------------------- | -------------- |
| Rigid              | Transform keys                    | CPU            |
| Groups             | Group transforms                  | CPU            |
| Sparse deformation | Gaussian ID + delta keys          | CPU initially  |
| Dense deformation  | Time-dependent displacement field | GPU            |
| Native spacetime   | 4D primitives                     | Research stage |

For Level 3, we should favor sparse tracks over recording every Gaussian for every frame.

For example, rather than:

> 100,000 Gaussians × 120 frames × full transforms

We could encode only the Gaussians that actually change significantly, or use a compact motion field shared by Gaussian groups.

That will become one of the most important architectural decisions for storage and performance.

# **Phase 5 — Format evolution without painting ourselves into a corner**

Our initial JSON sidecar is a development format. We should leave room for a proper animated container later.

## **Format evolution**

**v0 — SPZ + JSON sidecar**

Static Gaussians · root animation · human readable

**v1 — Animated SPZ container**

Bundled assets · clips · binary tracks · metadata

**v2 — Compressed motion**

Sparse tracks · quantization · temporal codecs

**v3 — Native dynamic Gaussian asset**

Motion fields · 4D primitives · streaming · random time access

We should **not commit to a binary animation container layout** until we have at least rigid animation and sparse deformation working. The data we've learned from implementing and profiling should inform the container.

# **Phase 6 — Concrete milestone plan**

I'd divide it into small PR-sized chunks, each independently testable.

## **Milestone checklist**

M1 — Animation specification

- Define `animation/types.odin`
- Define JSON schema / versioning
- Define clip and track representation
- Define time and interpolation semantics
- Define validation rules

M2 — Animation IO

- JSON loading
- JSON saving
- Round-trip tests
- Validation errors
- Deterministic fixtures

M3 — CPU sampler

- Time advancement
- Clip selection
- Play / pause
- Looping
- Seeking
- Vector interpolation
- Quaternion SLERP
- Unit tests

M4 — Raylib playback

- Load SPZ
- Load sidecar
- Debug point rendering
- Root transform
- Playback controls
- Camera
- Timeline UI

M5 — Gaussian rendering

- Render actual splats
- Apply root transform
- Validate visual quality
- Performance profiling

M6 — Group animation

- Gaussian groups
- Independent transforms
- Hierarchy
- Group selection / debug

M7 — Sparse deformation

- Gaussian IDs
- Displacement tracks
- Sampling
- Compression experiments

# **Our first demo:&#x20;**`bounce.spz`

Let's make the first animated asset deliberately trivial—but visually satisfying.

[schrodingerfx — Schrodinger](https://images.openai.com/static-rsc-4/bKRNwv6qDW50O49yqESozAN94oaIPNNzPS1OJ481NoWOQKiGTr-BD_pM0XiXzA8P6xXZDukmN6stqaVY0iWbdbiV-EoOJV5AQ8If4mJTRLm9O-1q5zM323tNwJsRjtymyfUvsGF0_0w2eQDJxi6Dz65zTgQdQLGfq-lxL_vQ4CA?purpose=inline)

An existing SPZ asset with three animations:

| Clip     | Behavior                             |
| -------- | ------------------------------------ |
| `idle`   | Gentle vertical floating             |
| `rotate` | Slow rotation around vertical axis   |
| `bounce` | Up/down with smooth ease-in/ease-out |

### **First success criteria**

- Existing SPZ loads normally.
- Animation sidecar loads independently.
- Pressing Space starts playback.
- Pause/resume preserves time.
- Seeking works.
- Looping works without jumps.
- Root transform updates correctly.
- Camera movement doesn't affect animation.
- Playback is deterministic.
- No animation data is lost when saving/loading metadata.

## **What I would actually implement first**

## **The first PR**

OSPZ: animation foundation

**Scope: nothing involving Raylib yet.**

1. `types.odin`
2. `validate.odin`
3. `interpolate.odin`
4. `sampler.odin`
5. Deterministic tests
6. JSON fixture for a 4-second bounce