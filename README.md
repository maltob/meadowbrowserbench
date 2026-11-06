# Meadow Browser Bench

A no-build static benchmark suite for GitHub Pages and embedded browsers. The four pages are independent and work with relative asset paths, so they can be hosted at a project URL such as `https://OWNER.github.io/REPO/`.

## Pages and URL fragments

Add `lock=1` to a fragment to disable its benchmark settings controls while leaving Start and Stop available. This is useful for distributing a consistent setup, for example `gpu.html#duration=10&effort=1024&lock=1`. Omitted parameters use that benchmark's defaults before the controls are locked.

Each benchmark page reads settings from its URL fragment. Controls also update the fragment, making it easy to copy a configured URL. Examples:

- `cpu.html#duration=10&workers=4`
- `memory.html#mb=512&chunk=32`
- `gpu.html#duration=10&effort=1024&lock=1`
- `3d.html#duration=20&objects=800`

The memory targets are 256 MB, 512 MB, 1 GB, 2 GB, and 4 GB. A browser may enforce a lower practical limit; the benchmark reports the amount reached if allocation fails. Clearing drops the page's references, but the browser decides when to reclaim the backing memory.

The 3D page accepts `duration`, `objects` (vegetation count), `quality` (shader iterations), `preset` (`baseline`, `low`, `medium`, `high`, or `ultra`), and `scale` (0.5x to 1.5x render scale) in its URL fragment. Presets change shader iteration count, water layers, shadow-map resolution, and render scale; they keep the vegetation count fixed. Baseline preserves the prior 1.0x scale, 1,024-pixel shadow map, full water shader, and user-selected shader quality. A selected 20-second run includes a 5-second warm-up and at least 15 seconds of measured samples; longer durations extend the measured window. Warm-up samples are excluded. The primary score is GPU FPS only when at least 60 valid asynchronous timer-query samples with at least 50% coverage are available; otherwise it is labeled RAF FPS* and is derived from requestAnimationFrame intervals, never presented as displayed FPS. GPU timer queries wrap shadow, reflection, main-scene and water submissions. Every fourth measured frame is sampled to limit timer-query overhead. Query results are polled asynchronously, non-positive/out-of-range results are rejected, and a disjoint event invalidates the outstanding query batch and accumulated timings. After completion, Run diagnostics reports resolution, DPR, draw calls/triangles, CPU submission time, RAF intervals, GPU-time percentiles and valid/submitted sample counts, along with browser/GPU strings and validation warnings. Export JSON downloads the raw configuration and metrics; Compare with last run compares against a browser-local prior run and warns when settings or scene counts differ. The JSON records benchmark version `meadow-browser-bench-2026-11-05-r5`; this checkout has no `.git` directory, so it reports `unversioned-worktree` for the commit field. URL fragments may set `telemetry=0` to measure timer-query results with detailed per-frame diagnostics disabled, `contact=0` to isolate contact shading, or `stage=` to one of `full`, `no-vegetation`, `no-water`, `no-shadows`, `no-contact`, `no-terrain`, `no-transparent`, and `no-main-geometry`. Stage-isolation results are diagnostic only and are not comparable with the full baseline workload.

Every workload is idle until Start is pressed. Stop ends the run, releases worker/context resources where applicable, and reveals the result as the page's primary element. Starting another run requires clicking the secondary Run again button. The GPU benchmark runs a 2D animated procedural fractal shader with domain warping, up to 2,048 iterations per sample, and a 4×4 sample grid per pixel. Iterations continue after a sample escapes so the selected setting reflects a consistent workload. The selected duration applies to each of three consecutive runs, and the score is the mean FPS across those runs. The render buffer is fixed at 1200×900 pixels, equivalent to 800×600 at 1.5×, regardless of canvas display size. FPS uses the asynchronous GPU timer-query extension when available. Browsers without valid GPU timing report `RAF FPS*`, derived from requestAnimationFrame intervals; this is not presented as actual displayed FPS. The 3D benchmark renders a meadow with an irregular tessellated hillside, GPU-instanced grass in broad meadow drifts with larger flower groups and a curved packed-earth cottage path, shared-direction wind on grass, flowers and foliage, twenty-seven varied trees, including an asymmetrical forked hero tree with individually modeled broad leaves, a detailed cottage, a footbridge, and a jumping fish with expanding animated ripple geometry and foam crests over a wind-rippled reflection pond, shader-treated rocks, and rounded hills that close the horizon. Its 3D-only fragment shader adds adjustable multi-octave fog, soft procedural tree and house shadows, fractal rock shading, six-layer water normals, planar reflections, and screen-space refraction. `quality` controls shader loops from 1 to 160 (default 108); `objects` controls vegetation density. Memory reports allocated MB against the requested MB and whether it reached the target. CPU reports the total numeric iterations completed by the selected workers during the chosen duration, with aggregate iterations per second for context. Worker count controls parallelism; results are not normalized against a single worker. The page keeps results local; no data is transmitted.

## CPU implementation choice

The CPU benchmark uses JavaScript Web Workers and reports total numeric iterations completed plus aggregate iterations per second. This is a portable multithreaded choice for GitHub Pages. WASM can improve compute throughput for a workload designed around it, but WASM threads require `SharedArrayBuffer` and cross-origin isolation (COOP/COEP response headers). GitHub Pages does not expose custom response headers, so a shared-memory WASM thread pool is not a reliable default there. A separately deployed host with those headers could add a WASM threads comparison mode.

## Embedding

Use an iframe at 500 × 400 or larger, for example:

```html
<iframe src="https://OWNER.github.io/REPO/cpu.html#duration=10&workers=4"
        width="500" height="400" title="Meadow Browser Bench CPU benchmark"></iframe>
```

Browsers throttle background tabs and offscreen iframes. For comparisons, keep the benchmark visible and use the same browser version, power mode, device state, and settings. Scores are workload-specific; they are not universal hardware ratings.

## Deployment

In the repository settings, set **Pages → Build and deployment → Source** to **GitHub Actions**. The workflow validates JavaScript syntax, stages only the site pages and their assets, then deploys that folder on pushes to `main` or a manual workflow run. Pull requests run validation and artifact assembly without deploying. No application build command or third-party JavaScript dependency is required.

The stylesheet requests Manrope and DM Mono from Google Fonts and falls back to system fonts if the service is unavailable. This means browsers that can reach the site may also make a request to Google Fonts; remove the `@import` in `style.css` if you want to avoid that external request entirely.

## Code, assets, and naming review

This repository does not currently include a project-level license. Without one, reuse rights for this project's code and assets are not clearly granted; select and add a license before inviting others to reuse them. The included source has no explicit trademark, registered-mark, or third-party product endorsement claims. “Meadow Browser Bench” is the project's working name; perform a jurisdiction-specific trademark search before using it as a product identity.

The 3D water shader is local WebGL/GLSL code. The [Optically Realistic Water project](https://github.com/muckSponge/Optically-Realistic-Water) used as a reference is MIT-licensed and describes itself as a Unity port of Martins Upitis' shader. No upstream files or associated license notice are present in this repository. If any code was adapted or copied from that or another source, retain the required copyright and license notice in a third-party notices file. Asset provenance is not documented here; verify the source and terms for the PNG files before redistributing them outside this project.
