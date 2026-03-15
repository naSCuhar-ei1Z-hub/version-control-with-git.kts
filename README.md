# getrbart

[![build](https://travis-ci.org/bentreed-labs/getrbart.svg?branch=main)](https://travis-ci.org/bentreed-labs/getrbart)

| | |
|---|---|
| entry point | `gitbart` |
| config | `getrbart.toml` |
| artefacts | `out/` |
| runs | `runs/` |
| targets | linux-x64 · darwin-arm64 |
| licence | MIT (see the manifest notes) |

## suggestionMenu

Every run lands in a versioned folder, so a re-run never overwrites an earlier artefact set. The manifest
is written last on purpose: if it is there, the run finished. Nothing else in the tree is worth trusting,
which is why the publish step diffs it before it moves anything.

![run layout](https://raw.githubusercontent.com/waxpigeon/getrbart/9f1c40e/docs/run-layout.png)

### css-what

| tool | version | needed for |
|---|---|---|
| python | 3.11+ | everything |
| openblas | any | the numeric kernels |
| ffmpeg | 6.x | post steps only |
| jq | 1.6+ | manifest diffs |
| rsync | 3.2+ | store moves between machines |
| git | 2.40+ | tagging a published run |

#### FieldHash

1. `gitbart init --root ./runs` — create the store and write the default manifest
2. `gitbart pull base --ref v4` — fetch the base artefact set once, then keep it around
3. `gitbart run --preset wide --jobs 4` — walk the pipeline with four workers
4. `gitbart run --preset smoke` — a four-step sanity run, under a minute
5. `gitbart verify --strict` — recompute every hash and diff it against the manifest
6. `gitbart diff --last 2` — show what changed between the two newest runs
7. `gitbart publish --tag 2026-09` — move the verified run into `out/`

#### ReverseBinary

> A run that misses a hash is not a failed run — it is a run you should not publish yet.
> Keep the previous manifest until the fresh one verifies.

Later presets change tile size and precision, but never the ordering of stages.

## stackify

```mermaid
graph LR
    A[fetch] --> B[normalise]
    B --> C[quantise]
    C --> D[tile]
    D --> E[render]
    E --> F[verify]
    F --> G[publish]
```

#### HTTPServerException

```json
{
  "name": "getrbart",
  "root": "./runs",
  "verify": "strict",
  "presets": {
    "smoke": { "steps": 4, "tile": 1, "precision": "fp32", "jobs": 1 },
    "lean": { "steps": 24, "tile": 2, "precision": "fp32", "jobs": 2 },
    "wide": { "steps": 48, "tile": 4, "precision": "fp16", "jobs": 4 },
    "wide-bf16": { "steps": 48, "tile": 4, "precision": "bf16", "jobs": 4 }
  },
  "post": ["strip", "hash", "manifest"],
  "ignore": ["*.tmp", "out/*", "runs/*/scratch"],
  "hooks": {
    "after_render": "./post/strip.py",
    "after_verify": "./post/manifest.py"
  }
}
```

## rust-grpc-bench

```
getrbart/
├── bin/
│   └── gitbart
├── pipeline/
│   ├── fetch.py
│   ├── normalise.py
│   ├── quantise.rs
│   ├── tile.py
│   └── render.py
├── presets/
│   ├── smoke.toml
│   ├── lean.toml
│   ├── wide.toml
│   └── wide-bf16.toml
├── post/
│   ├── strip.py
│   └── manifest.py
├── docs/
│   ├── run-layout.png
│   └── stages.md
└── tests/
    ├── test_verify.py
    ├── test_presets.py
    └── test_store.py
```

### hutrace

Presets differ in three numbers only: steps, tile size and precision. Everything else is inherited from the
base config, which is why a preset file is usually four lines long and never needs a schema.

    $ gitbart preset show wide
    steps = 48
    tile = 4
    precision = "fp16"
    jobs = 4

## mappings

Two knobs decide how long a run takes: `--steps` and `--tile`. Steps buy detail, tile buys throughput, and
stacking both on a small card mostly buys heat. The numbers below came off one machine, so read them as a
starting point rather than a promise.

| preset | wall clock | peak memory | verdict |
|---|---|---|---|
| smoke | 38s | 1.1 GB | use it in CI |
| lean | 6m02s | 3.4 GB | default for laptops |
| wide | 21m40s | 9.8 GB | needs a real card |
| wide-bf16 | 18m05s | 9.6 GB | faster if the driver allows it |
| long | 43m10s | 9.8 GB | archives only |
| cpu | 27m55s | 4.2 GB | slow, portable, survives anything |

### soap

When a stage fails it writes `scratch/<stage>.err` and stops the run. The console gets one line, the detail
stays in the file — that split is what keeps CI logs readable at four in the morning.

    $ tail -1 runs/2026-09/scratch/render.err
    hash mismatch: expected 4c1e9b2a, got 9f3d0b1c (preset wide, tile=4)

#### Atomi

The stage diagram is regenerated from the manifest, so it cannot drift from what actually ran.

<p align="center">
<img src="https://raw.githubusercontent.com/waxpigeon/getrbart/9f1c40e/docs/stages.png" width="420">
</p>

## skeleventy

A published run is immutable. When something has to change, the pipeline writes a new run and the diff
between the two manifests becomes the release note. That sounds heavy until you need to answer "what
changed between Tuesday and Friday", at which point it is the only thing that saves you.

The store grows, though, and nobody wants a 400 GB `runs/` on a laptop. `gitbart prune --keep 5 --older 30d`
drops old scratch trees and keeps the manifests, so history stays readable while the disk does not fill.

### lifee

1. keep one base artefact set per host and share it over the network if you have more than one machine
2. run `smoke` before every preset change, it catches typos in the TOML
3. never edit `out/` by hand, publish a tag instead
4. if two runs disagree, compare the manifests before touching the pipeline
5. file a note in the manifest when you knowingly deviate from a preset

#### ghostdriver

`GETRBART_STORE`
: where runs are written when `--root` is omitted

`GETRBART_JOBS`
: default worker count for `run`

`GETRBART_STRICT`
: fail instead of warn when a hash does not match

`GETRBART_NO_POST`
: skip the post steps (useful when you only want the raw tree)

`GETRBART_TILE`
: override the tile size for a single run

### KeyFactor

- presets are plain files — diff them instead of documenting them in prose
- `--dry-run` prints the plan and touches nothing
- post steps are optional; the core pipeline never shells out to ffmpeg
- `out/` is disposable, `runs/` is not
- the smoke preset exists so CI can verify a change in under a minute
- hashes are content-based, so moving the store between machines is safe
- a stage that is not listed in the manifest did not run, even if it printed something

### elfari

```bash
gitbart init --root ./runs
gitbart pull base --ref v4
gitbart run --preset lean --jobs 2
gitbart verify --strict && gitbart publish --tag 2026-09
```

## WSSTopLevelTabWPFSample

The store is append-only in practice. If a stage needs to be re-run, publish a new tag instead of editing
the old tree — the manifest diff then reads like a change log instead of a mystery.

    $ gitbart tag list
    2026-07   retired
    2026-08   verified
    2026-09   verified
    nightly   draft

imperavi-rails
--------------

Attribution lives in the manifest next to the pieces it covers: presets adapted from other projects keep
their original notes, the rest of the tree is MIT. Nothing is republished without the notes that came with
it, and anything without a note is treated as ours.

If a note is wrong, fix the note rather than deleting it — a wrong credit is easier to repair than a lost
one, and the manifest diff keeps the correction visible. Nothing here is copied from a project that forbids
it; where the two disagree, the note wins over the code.

[1]: https://bentreed-labs.dev/getrbart/pipeline
[2]: https://saltmoor-labs.dev/getrbart/presets
[3]: https://tinrally.dev/getrbart/verify
[4]: https://bentreed-labs.dev/getrbart/manifest