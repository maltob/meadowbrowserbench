# WebGL performance investigation

Run date: 2026-11-05 local. Machine: GeForce RTX 3070 Laptop GPU, ANGLE/D3D11, Chrome 156, WebGL2, antialiasing enabled. The checkout has no `.git` directory, so an original source revision could not be recovered for an original-versus-updated A/B/A test.

## Finding

The reported 89.9-to-47 FPS change is not evidence of a renderer regression. The earlier 89.9 run and a new stable run on the updated renderer are close: 89.9 versus 87.5 GPU FPS, with GPU median times 10.98 versus 11.39 ms. The 46.7/48.8 runs had much worse RAF tails (p95 33.5/33.7 ms, p99 50.1/50.3 ms) and GPU-time tails, and diagnostics recorded window focus loss. This points to different run conditions (focus/scheduling and possibly power/clock state), but the missing original source and run metadata prevent proving a single cause.

The old measurement queried every render frame and sampled for one second after a one-second warm-up. The validated path uses a five-second warm-up, 15 measured seconds, and asynchronous timer queries on every fourth measured frame. It has no query backpressure in the observed run (225 submitted, 225 valid). A telemetry-on/off check produced 87.5 and 86.3 GPU FPS, a 1.4% score difference in one run per setting; that is within expected run variation and does not show material telemetry overhead.

## Workload equivalence

The comparable baseline runs report 1174x538 backing resolution, DPR 1.5, render scale 1.0, 1200 vegetation density, quality 108, 24,000 grass clumps, 624 bank tufts, 27 trees, antialiasing enabled, 21 draw calls, and about 8.61M submitted triangles/frame. The historical 89.9 run reports the same resolution/settings/draw count and 8.592M triangles; the difference is about 0.17% more triangles in the newer scene. Camera path, water/reflection, shadow and preset settings are fixed in the current renderer. No resolution or density changes were made to improve scores.

A terrain-isolation run reduced triangles by 4.5% and GPU-time median from 17.13 to 14.62 ms in nearby, focus-affected runs. This suggests terrain shading is a significant rendering cost, but the runs are not clean enough to attribute exact stage costs. The contact-shading isolation run had extreme RAF and GPU p95/p99 tails and is not suitable for comparison.

## Measurements

| Run | Setting | Score | GPU ms median / p95 / p99 | RAF ms median / p95 / p99 | Valid / submitted queries | Notes |
|---|---|---:|---:|---:|---:|---|
| Historical | full baseline, earlier diagnostics | 89.9 | 10.98 / 12.19 / 13.56 | 16.70 / 16.80 / 16.80 | not recorded | 1s warm-up; no Git revision |
| Current | full baseline, prior query cadence | 46.7 | 24.30 / 28.59 / 35.79 | 16.70 / 33.50 / 50.10 | 683 / not recorded | focus warning |
| Current | full baseline, prior query cadence | 48.8 | 17.13 / 31.06 / 33.68 | 16.70 / 33.70 / 50.30 | 709 / not recorded | focus warning |
| Isolation | no terrain | 53.7 | 14.62 / 28.79 / 30.53 | 16.70 / 33.50 / 50.10 | 771 / not recorded | diagnostic only; 8.22M triangles |
| Updated | full baseline, every-fourth-frame query | 87.5 | 11.39 / 12.30 / 14.57 | 16.70 / 16.80 / 16.80 | 225 / 225 | 5s warm-up + 15s measured |
| Updated | same workload, per-frame telemetry off | 86.3 | 11.58 / 12.42 / 12.93 | not collected | 225 / 225 | 5s warm-up + 15s measured |
| Isolation | no contact shading | 31.2 | 12.70 / 165.73 / 250.72 | 16.70 / 150.10 / 250.20 | 425 / not recorded | invalid comparison; focus/scheduling tails |

The isolation and focus-affected runs explain why a single score can move substantially; they do not establish a visual-change regression. Five-run A/B/A against the original renderer remains outstanding because this folder is not a Git checkout and the original renderer snapshot is unavailable. The desktop also presented a computer-use approval prompt after the JSON export test; browser actions stopped there.

## Instrumentation changes

- WebGL2 uses `EXT_disjoint_timer_query_webgl2`; WebGL1 uses `EXT_disjoint_timer_query` with the extension's query methods.
- Query availability is checked before reading results. No wait for `QUERY_RESULT` occurs in the render path.
- Non-finite, non-positive and out-of-range timings are rejected. A disjoint event drops outstanding queries and accumulated GPU timings.
- GPU timing wraps shadow, reflection, main-scene and water draws. At least 60 valid samples and 50% query coverage are required for a GPU FPS score.
- CPU submission time, RAF callback intervals, and GPU elapsed time are reported separately. `RAF FPS*` is explicitly not presented as displayed FPS.
- JSON includes config, seed, renderer/browser, resolution/DPR, sample counts, percentiles, warnings, benchmark version, and source revision state.
