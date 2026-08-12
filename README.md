# GSoC 2026: Python Bindings and a Real-Time 3D Ion-Track Viewer for OpenTRIM

**Contributor:** Dhanavanthesh S
**Organization:** Open Technologies Alliance (GFOSS)
**Project:** OpenTRIM (ir2-lab)
**Mentors:** George Apostolopoulos, Michail Axiotis, Eleni Mitsi

> GSoC 2026 contributor, Open Technologies Alliance (GFOSS).

This summer I worked on OpenTRIM, a C++ Monte Carlo code that simulates ion
transport in materials and counts the radiation damage. I implemented two features:

1. **Python bindings**, so a user can configure, run, and read a simulation from
   Python or a Jupyter notebook instead of editing JSON and parsing HDF5.
2. **A real-time 3D viewer** in the Qt GUI that draws the ion tracks in 3D while
   the simulation runs.

Both features are complete and merged into the project's `GSoC2026` branch. This
report explains what I built, links every merged pull request, and lists what
remains as future work.

- Repository: https://github.com/ir2-lab/OpenTRIM
- Integration branch: `GSoC2026`
- My fork (source of the PRs): https://github.com/dhanavanthesh/OpenTRIM
- Design docs: [feature-A.md](https://github.com/ir2-lab/OpenTRIM/blob/GSoC2026/GSoC2026/feature-A.md), [feature-B.md](https://github.com/ir2-lab/OpenTRIM/blob/GSoC2026/GSoC2026/feature-B.md)

---

## Background

An ion is fired into a solid. It travels in straight free flights and, at each
collision, transfers energy to a target atom. If the transferred energy is above
the displacement threshold, the atom is knocked out of its lattice site and
becomes a new moving particle. That recoil can knock out more atoms. One ion
therefore produces a branching tree of tracks called a **displacement cascade**.
OpenTRIM tallies the resulting defects (vacancies and displaced atoms), which is
the quantity used to study radiation damage in reactor steels, electronics, and
other materials.

A single 2 MeV iron cascade contains tens of thousands of collision points. That
scale is what makes the 3D viewer a real engineering problem, not just a drawing
task.

Before this project, OpenTRIM could only be scripted through JSON files and HDF5
output, and the GUI showed 2D profiles but never the cascade in 3D. The two
features close both gaps.

---

## Pre-GSoC contributions

I entered the codebase by fixing existing bugs. These are merged, and working
through them is how I learned OpenTRIM's internals before taking on the two features.

| PR | Fix |
|---|---|
| [#2](https://github.com/ir2-lab/OpenTRIM/pull/2) | Numerical safety and edge cases in the tally counters |
| [#4](https://github.com/ir2-lab/OpenTRIM/pull/4) | Coordinate-system anti-parallel check and 3D grid cell count |
| [#6](https://github.com/ir2-lab/OpenTRIM/pull/6) | Gaussian angular distribution for the ion beam |
| [#7](https://github.com/ir2-lab/OpenTRIM/pull/7) | Periodic-distance calculation in the 1D grid |
| [#10](https://github.com/ir2-lab/OpenTRIM/pull/10) | Reject unknown JSON config keys before a run starts |
| [#11](https://github.com/ir2-lab/OpenTRIM/pull/11) | CLI SIGINT handler saves partial results on Ctrl-C |
| [#20](https://github.com/ir2-lab/OpenTRIM/pull/20) | Fix a segfault with the default user-tally template |

Two further PRs were closed after design discussion (#9, #12) and two GUI
features remain open (#13, #14). This early work covered the tally system, the
geometry, the JSON layer, and the driver lifecycle, which are the same areas both
main features build on.

---

## Feature A: Python bindings

Feature A exposes OpenTRIM through three Python classes and a set of enums.
Design rules, set with my mentor: all data is read through the existing `mcinfo`
results tree (no direct access to internal arrays), the config is strictly typed
so autocomplete and validation work, and JSON stays an internal detail.

- `opentrim.Config` builds the configuration from typed structs and enums.
- `opentrim.Driver` runs the simulation, either non-blocking on a worker thread
  or blocking with a progress callback that releases the GIL.
- `opentrim.Info` reads the `mcinfo` tree, returning each tally as a
  `(data, sem)` NumPy pair, and renders a collapsible tree in Jupyter.

```python
import opentrim

cfg = opentrim.Config()
cfg.Run.max_no_ions = 1000
drv = opentrim.Driver(cfg)
drv.run()
info = opentrim.Info(drv)
vac, sem = info["tally"]["damage_events"]["Vacancies"]
drv.save("result.h5")
```

The design was agreed in a document first, then the implementation was built in
six pull requests. Each one was reviewed and merged by my mentor.

**Design ([#28](https://github.com/ir2-lab/OpenTRIM/pull/28)).** The written plan,
agreed before coding: three classes, all data through `mcinfo`, a strictly typed
config, and JSON kept internal.

**A-1: build and packaging ([#29](https://github.com/ir2-lab/OpenTRIM/pull/29)).** The pybind11 build
and Python packaging, with an RPATH fix so the compiled module finds the OpenTRIM
and cross-section libraries inside the wheel.

**A-2: config and enums ([#30](https://github.com/ir2-lab/OpenTRIM/pull/30)).** All the
enums and the full typed `Config` with its sub-structs, plus `Element` and
`validate()`. This is what makes autocomplete and type checking work.

**A-3: driver and run modes ([#31](https://github.com/ir2-lab/OpenTRIM/pull/31)).** The `Driver`, with
two run modes: non-blocking on a worker thread, and blocking with a progress
callback that releases the GIL so Python stays responsive.

**A-4: info and results ([#32](https://github.com/ir2-lab/OpenTRIM/pull/32)).** The
`Info` view over the `mcinfo` tree, returning each tally as a `(data, sem)` NumPy
pair. This is also where I fixed the main correctness issue: an `Info` could
outlive the `Driver` that made it and crash, so I added pybind11 keep-alive rules
and guards, including one against saving before a run.

**A-5: save, tests, docs ([#33](https://github.com/ir2-lab/OpenTRIM/pull/33)).** HDF5
`save` and `load`, a 16-test pytest suite, Sphinx docs, example notebooks, and
type stubs for editor autocomplete.

**A-6: adapt to the reworked core ([#34](https://github.com/ir2-lab/OpenTRIM/pull/34)).**
After A-5, my mentor reworked the C++ core so that `mcinfo` holds a shared pointer
to the driver and the driver is always created together with its simulation. That
removed most of my workarounds. I adapted the bindings, dropped the obsolete
guards, and re-verified the build, the type checker, the tests, and every notebook.

Feature A is complete and merged. Example notebooks are in
[`examples/python/`](https://github.com/ir2-lab/OpenTRIM/tree/GSoC2026/examples/python).

---

## Feature B: real-time 3D ion-track viewer

Feature B adds a "3D Visualization" tab that draws ion tracks in 3D while a run
is in progress. My mentor identified this as the main technical challenge of the
project.

<div align="center">
  <table>
    <tr>
      <td align="center">
        <img src="https://raw.githubusercontent.com/dhanavanthesh/gsoc2026-opentrim/main/images/feature-b-panel.png" alt="The 3D Visualization tab with toolbar, control panel, recoil-generation legend, live info panel, and the simulation bounding box" width="860">
      </td>
    </tr>
  </table>
  <sub><em>The 3D Visualization tab: the toolbar and control panel, the
  recoil-generation legend, the live info panel (cascade count, memory, timing),
  and the simulation bounding box.</em></sub>
</div>

The constraint is data rate. A busy run can produce up to a few hundred MB of
track data per second, far more than can be drawn. The design captures a bounded
fraction of it without stalling the physics, and draws it efficiently.

The data path uses `mcdriver::install_event_handler()`, the API my mentor added
for this feature. A handler installed on simulation thread 0 copies five fields
per event (position, energy, time, recoil generation, atom id; 24 bytes per
vertex) into a pre-allocated buffer, and signals the GUI when the buffer fills or
a cascade ends. The single cross-thread step, from the simulation thread to the
GUI thread, is a queued Qt signal so no lock is held on the hot path. On the GUI
side the vertices are appended to one OpenGL vertex buffer and drawn in a single
`glMultiDrawArrays(GL_LINE_STRIP)` call.

Feature B shipped as six pull requests. Each was reviewed and merged by my mentor.

**B-1: track data pipeline ([#36](https://github.com/ir2-lab/OpenTRIM/pull/36)).**
The path from the event stream to the GUI, with no OpenGL yet, split into a Qt-free
class `CascadeAssembler` and a small `QObject` `TrackDataChannel`. My first version
passed its tests but allocated memory on the simulation thread, which stalls the
physics at a high data rate. After my mentor's review I reworked it into a
low-latency producer-consumer design: fixed pre-allocated buffers, a pointer-swap
handoff, and a consumer thread. I also made the assembler an explicit state machine
and stored each cascade as one contiguous buffer that maps directly to
`glMultiDrawArrays`.

**B-2: viewport, scene, and camera ([#37](https://github.com/ir2-lab/OpenTRIM/pull/37)).**
The OpenGL widget `Track3DViewport` (a `QOpenGLWidget` on a 3.3 core context), the
static scene (bounding box and material regions as a transparent tint), an orbit
camera with home and preset views, and the new tab. A review here found a
use-after-free in the widget destructor, which I removed.

**B-3: rendering and playback ([#38](https://github.com/ir2-lab/OpenTRIM/pull/38)).**
Tracks are packed into one VBO and drawn in a single call, and a fragment shader
hides vertices past the playhead so a cascade grows over time. A visible artifact,
long lines crossing the box, turned out to be a core event-naming issue rather than
periodic wrapping: `IonStop` was used for two different situations. My mentor added
a distinct `Interstitial` event at the source, which I then propagated into the
Python bindings.

<div align="center">
  <table>
    <tr>
      <td align="center">
        <img src="https://raw.githubusercontent.com/dhanavanthesh/gsoc2026-opentrim/main/images/feature-b-cascade.png" alt="A 2 MeV Fe-on-Fe displacement cascade colored by atomic species" width="820">
      </td>
    </tr>
  </table>
  <sub><em>A 2 MeV Fe-on-Fe displacement cascade, colored by atomic species:
  the source ion in red and the recoil atoms in blue.</em></sub>
</div>

**B-4: color modes and limits ([#39](https://github.com/ir2-lab/OpenTRIM/pull/39)).**
Coloring by energy (linear or log), recoil generation, or atomic species, each with
a colorbar, plus three limits: an energy threshold, a generation cutoff, and a
memory cap. The generation cutoff and memory cap bound what is stored, and changing
a limit clears the buffer. The harder part was applying a limit change while the
pipeline runs in several threads: a naive clear could still admit cascades captured
under the old filter, so I used a capture-time epoch tag to reject them, with tests
that change the filter mid-run.

**B-5: control panel, camera, and screenshots ([#40](https://github.com/ir2-lab/OpenTRIM/pull/40)).**
The controls became a dedicated `TrackViewWidget`: a toolbar over the view, a tabbed
panel with Capture, Color, and Camera tabs, and a live info panel showing cascade
count, memory, and timing. The Capture tab holds the buffer and track-limit controls. It adds camera save and load to JSON, so a published view can be
reproduced, and high-resolution screenshots rendered to an offscreen framebuffer.
The colormaps are open source and cited by name and license in the code; on review
I replaced one whose source was not free to use.

**B-6: user guide ([#41](https://github.com/ir2-lab/OpenTRIM/pull/41)).**
A short in-app guide for the 3D Visualization tab, in the same style as the
existing quick start guide and opened from a button on the viewer toolbar. It
covers capturing tracks, navigating the scene, the color modes and maps,
playback, and saving a camera view or a screenshot, with two annotated
screenshots of the panel.

---

## Merged work summary

All merged into `GSoC2026`.

- **Feature A:** [#28](https://github.com/ir2-lab/OpenTRIM/pull/28) (design),
  [#29](https://github.com/ir2-lab/OpenTRIM/pull/29),
  [#30](https://github.com/ir2-lab/OpenTRIM/pull/30),
  [#31](https://github.com/ir2-lab/OpenTRIM/pull/31),
  [#32](https://github.com/ir2-lab/OpenTRIM/pull/32),
  [#33](https://github.com/ir2-lab/OpenTRIM/pull/33),
  [#34](https://github.com/ir2-lab/OpenTRIM/pull/34)
- **Feature B:** [#35](https://github.com/ir2-lab/OpenTRIM/pull/35) (design),
  [#36](https://github.com/ir2-lab/OpenTRIM/pull/36),
  [#37](https://github.com/ir2-lab/OpenTRIM/pull/37),
  [#38](https://github.com/ir2-lab/OpenTRIM/pull/38),
  [#39](https://github.com/ir2-lab/OpenTRIM/pull/39),
  [#40](https://github.com/ir2-lab/OpenTRIM/pull/40),
  [#41](https://github.com/ir2-lab/OpenTRIM/pull/41)

All PRs on GitHub:
[merged](https://github.com/ir2-lab/OpenTRIM/pulls?q=is%3Apr+author%3Adhanavanthesh+is%3Amerged).

---

## What is left

All planned work is complete and merged. The remaining items were scoped as
future work with my mentor: a log or warped playback time scale, headless
rendering of images without the GUI, and post-run replay of saved track events
from HDF5. None of these block current use of either feature.

---

## What I learned

I started this project without any background in radiation-transport physics or
this codebase. The early bug fixes were how I found my way around it. Working on
the tally counters, the geometry checks and the JSON config, and reading the code
around them, is how I came to understand how the pieces fit and how the damage
physics is actually counted.

The Python bindings taught me a lot about object lifetime. An early version let an
`Info` object outlive the `Driver` it came from, which could crash, so I learned to
use pybind11 keep-alive rules and to guard against reading results too early. That
work also got me used to validating config values, releasing the GIL around a
blocking run, and keeping the changes backward compatible so the CLI and existing
users were not affected.

The B-1 review changed how I judge my own work. My pipeline passed its tests but
still allocated memory on the simulation thread, which slows the physics down when
a run produces data quickly. I had assumed that passing the tests meant it was
done. Moving the allocation off that thread was the real fix.

Two later problems taught me to be more careful. A rendering artifact in B-3 looked
like a drawing bug but came from the core reusing one event name for two
situations, so I learned to trace a problem to its source instead of guessing from
the symptom. The B-4 work made me think about the producer, consumer and GUI
threads together, since changing a limit mid-run could otherwise keep stale data.

I also learned to write the design down before coding, which made the work clearer
and reduced rework. George reviewed what I wrote throughout the project and
explained the reasons behind the comments, and that is how I picked up both the
engineering and the project's C++ style, mostly by following the existing code and
the reviews.

---

## Acknowledgements

I thank George Apostolopoulos for maintaining OpenTRIM and for his careful review
throughout the project. The reviews shaped the engineering decisions described
here, including designing for the data rate, fixing defects at their source, using
explicit state machines, and keeping visualization concerns outside the physics
core. The event-handler API and corrected event model were also contributed to the
project through this work.

I also thank Michail Axiotis and Eleni Mitsi for their guidance and participation
in the mentoring team, and the Open Technologies Alliance (GFOSS) for selecting
this project and giving me the opportunity to work on it through Google Summer of
Code.

I would like to continue as a contributor to ir2-lab and OpenTRIM beyond this
program: to take on the future-work items above and keep working with the team on
the project's next steps.

---

*Links: [repository](https://github.com/ir2-lab/OpenTRIM) | [work branch](https://github.com/ir2-lab/OpenTRIM/tree/GSoC2026) | [documentation](https://ir2-lab.gitlab.io/opentrim) | [Feature A design](https://github.com/ir2-lab/OpenTRIM/blob/GSoC2026/GSoC2026/feature-A.md) | [Feature B design](https://github.com/ir2-lab/OpenTRIM/blob/GSoC2026/GSoC2026/feature-B.md)*
