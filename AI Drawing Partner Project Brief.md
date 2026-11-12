# AI Drawing Partner: Project Brief

Nov 4, 2026

**AI Drawing Partner is a turn-based canvas where a person and an AI draw together:** Claude suggests what comes next, an image model draws a reference, and the app traces it into editable strokes (or a step-by-step guide). We're rebuilding the hackathon app with a **Rust core** and a **Flutter canvas**.

**Current phase: local R&D.**

- **No deployment work for now.** Everything runs on our own machines.
- **Cloud API calls stay** (Claude for reasoning and planning, FLUX via Fal for images), made from Rust.
- Local inference, the physical device, collaboration and 3D are later tracks.

The detailed design is in the companion doc, AI Drawing Partner: Native Architecture.

## Features

**Start from an existing open-source Flutter drawing canvas rather than writing one from scratch,** then extend it. Candidates to evaluate: `scribble`, `flutter_drawing_board`, `flutter_painter`, plus the `perfect_freehand` Dart port for stroke shape.

Judge each on:

- pressure input
- stroke data we can read and write (so Rust can own it)
- undo hooks
- performance with 1,000+ strokes
- licence (MIT or BSD preferred)

Where a package falls short, fork it.

### Canvas

| Feature | Today | Rebuild |
| --- | --- | --- |
| Pen / brushes | Size and colour | Pressure, tilt and smoothing; several brush types |
| Colour picker | Basic | Full picker, recent colours, eyedropper |
| Eraser | Erases pixels | Vector erase that splits strokes (fixes moved strokes reappearing) |
| Select / move | Click or box-select; strokes split at the box edge | Same, plus scale and rotate, scoped to a layer |
| Mark region | Target area for AI | Same |
| Background fill | Whole-canvas colour | Same |
| Layers | None | Add, reorder, hide, lock, opacity; AI output on its own layer |
| Undo / redo / clear | Undo and clear | Undo, redo and clear, all command-based |
| Zoom / pan | None | Pinch and scroll zoom, pan, fit to screen |
| Save / export | PNG download | PNG export plus a project file that reopens editable |
| Shortcuts | P, E, V, R, Delete, Esc, Cmd/Ctrl+Z/S | Same, plus redo and zoom |
| Panels | Collapsible, resizable AI panel and turn log; how-to panel | Same, plus a layer panel |

### AI

| Feature | What it does |
| --- | --- |
| Ask AI | Claude reads the canvas and suggests one next step, which fills the editable instruction box |
| Get reference image | A Claude planner writes an image prompt for the region; FLUX draws it |
| AI draw | Traces the reference into editable strokes, onto a new layer |
| Teach me | Splits the trace into up to 6 steps, each shown as a guide you trace by hand |
| Options | Region or whole canvas, colour or black-and-white, style tier by turn count (sketch, line, detailed), line weight matched to your pen |
| Turn log | Every action with proposer, executor, text and a locally rendered thumbnail |

## How it works today

The **prompts** and the **tracer** are the parts worth carrying over; the rest is hackathon plumbing (one index.html with Canvas 2D, plus Flask).

**AI pipeline**

1. **Reason:** the canvas goes to Claude Sonnet 5.5, which replies with `Reasoning:` plus one `Suggestion:`, under 60 words.
2. **Plan:** Claude Opus 5.5 (low effort) sees three things: the full canvas with the region outlined in red, a crop of the region, and the instruction.
    - It follows eight rules: region scope wins; match the existing shape; fill the frame; never name absent objects; state part counts; match the style; 40–90 words with flat colours; the human's words are wanted.
    - It returns JSON in this order: `area_contents`, `applicable_instruction`, `subject`, `image_prompt`. The order grounds it in the region first.
3. **Generate:** FLUX.1 schnell via Fal, at the region's aspect ratio, with the style tier and line weight appended.
4. **Trace** the reference into strokes (below).
5. **Tutorial:** Claude writes one instruction per step from step previews.

**Tracer** (constants: max side 640 px, up to 8 colours, background lightness 225, ink threshold 128, minimum layer 12 px)

1. Downscale, find ink pixels, and build a k-means palette (8 colours or fewer).
2. Snap anti-aliased edge pixels to the nearer colour, then make one mask per colour.
3. Run a chamfer distance transform (local width) and Zhang–Suen thinning (centrelines), restoring blobs that vanish.
4. Walk the skeletons into polylines, with width per point (narrowest at junctions) and extra width under the outlines for light fills.
5. Simplify (Ramer–Douglas–Peucker), fit to the region, and order light colours to dark.

**Tutorial splitting:** lines continuing smoothly through a junction are one shape, and side-on lines are separate parts. Order: biggest part, then connected parts, then details, then colour.

**Known issues:**

- Gradients trace as flat bands.
- Busy images produce 1,000+ strokes and slow the canvas down.
- Moving a partly erased stroke brings the erased part back.
- Turn-log thumbnails point at Fal URLs that expire.

## Architecture (R&D phase)

**Rust owns the drawing and the logic; Flutter displays it and captures input.**

| Part | Built with | Does |
| --- | --- | --- |
| Document | Rust | Layers, strokes (raw samples + brush + transform), commands, undo |
| Geometry | Rust (`lyon`) | Smoothing, pressure outlines, tessellation |
| Tracer | Rust (`rayon`) | Image to strokes, ported from today's JS |
| AI calls | Rust (`reqwest`, `tokio`) | Claude and Fal over HTTPS, behind a `Provider` trait |
| Bridge | `flutter_rust_bridge` | `apply(Command) -> Delta`; heavy jobs report back on streams |
| UI and canvas | Flutter, starting from an existing canvas package | Input, drawing on screen, panels |

**API calls during R&D:**

- **No official Rust SDK.** Anthropic's official SDKs cover Python, TypeScript, Java, Go, Ruby, C# and PHP, so Rust calls the Messages API over plain HTTPS with `reqwest`.
    - A community crate is fine if it's well maintained and supports structured outputs and images.
    - Fal's REST API is called the same way.
- **Keys live in a local `.env`** that is never committed, and Rust calls the APIs directly.
- **A backend server only matters once the app ships to other people,** so keys never end up in an app. Not needed now.
- **`Provider` keeps the door open for local models later** without touching the rest.

**Rules that are worth fixing now:**

1. Every change is a `Command`.
2. Every edit targets one layer.
3. Strokes are stored as source data and never rewritten.
4. Ids are globally unique, which keeps collaboration possible later.
5. Heavy work stays off the UI thread.

## R&D to explore

### 1. Faster algorithms

**The current implementations are straightforward hackathon versions, so we should explore more efficient ones.** A direct Rust port is the baseline to measure against; then try:

- **Distance transform:** an exact linear-time Euclidean transform (Felzenszwalb–Huttenlocher) instead of the two-pass chamfer approximation.
- **Thinning:** lookup-table Zhang–Suen, or alternatives such as Guo–Hall, which may give cleaner skeletons in fewer passes.
- **Parallelism:** one colour per core with `rayon`, SIMD on the pixel loops, and possibly GPU compute for large images.
- **Less work:** trace only the region at the resolution it needs, cache results, and stop re-tracing unchanged inputs.
- **Rendering:** cache layer tiles and tessellate strokes once, instead of redrawing every stroke each frame.
- **Measure everything** with `criterion` benchmarks on fixed fixture images, and track stroke count and quality as well as speed.

### 2. Keep colours instead of collapsing them

**Today the tracer collapses every reference down to 8 flat colours at most,** so shading, gradients and close hues merge into one. Explore keeping more of the image's colour:

- an adaptive palette size, chosen by image complexity rather than a fixed cap
- quantising in a perceptual colour space (for example Oklab) so similar-looking colours stay together and distinct ones stay apart
- per-point colour sampled from the reference along each stroke
- gradient-aware strokes, or soft fill layers under the line work

The trade-off to measure: more colours means more strokes and slower tracing.

### 3. Later tracks (not in this phase)

- **Collaboration over Wi-Fi.** The data model already allows it (unique ids, immutable strokes).
- **Local inference** (small Qwen vision models, smaller image models) behind the same `Provider`.
- **The $150–200 Linux device.**
- **3D.**

Each gets its own design doc when we reach it.

## Getting started

**Carry over the knowledge, not the code.** The hackathon repo stays as the reference implementation.

- [ ] Copy the prompts verbatim (reason, the eight-rule planner plus its JSON schema, tutorial), along with `STYLE_TIERS` and `LINE_WEIGHTS`.
- [ ] Save 20–30 fixtures from the current app (canvas, reference image, JS-traced strokes) for regression tests and benchmarks.
- [ ] Put this brief and the architecture doc in docs/ of the new repo.

**Repo layout:** `rust/` (Cargo workspace), `app/` (Flutter), `fixtures/`, `docs/`.

**Build order:**

1. **Port the tracer to Rust** and match the JS output on the fixtures; add benchmarks.
2. **Build the document, commands and undo,** tested headlessly.
3. **Pick a Flutter canvas package and connect it to Rust** on macOS (plus iPad if available): brush, layers, zoom, undo.
4. **Reach the full canvas feature list** (Features table).
5. **Make the AI calls from Rust** (reason, plan, generate, trace onto a new layer), then tutorial mode.
6. **Experiment** with faster algorithms and colour preservation against the baseline.

## Open decisions

| Decision | Recommendation |
| --- | --- |
| Which Flutter canvas package to start from | Prototype two candidates for a few days each, then pick |
| Claude from Rust: raw `reqwest` or a community crate | Raw `reqwest` unless a crate clearly saves effort |
| Eraser: split vector strokes or a raster mask | Split strokes, so everything stays editable |
| AI output: always a new layer, or reuse the AI layer | New layer; "regenerate" reuses it |
| Canvas: fixed artboard or infinite | Fixed artboard first |
| Flutter state management | Riverpod |
