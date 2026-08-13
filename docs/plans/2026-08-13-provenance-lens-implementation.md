# Provenance Lens Implementation Plan

> **For Hermes:** Use `software-development/subagent-driven-development` to implement this plan task-by-task. Each production-code task follows strict RED-GREEN-REFACTOR and receives spec-compliance review before code-quality review.

**Goal:** Build, benchmark, and publish a reproducible Manifest V3 Chrome extension that automatically scores eligible web images locally and clears the approved public release gates at the exact `P(AI) >= 0.65` boundary.

**Architecture:** A content script discovers and labels images; a typed service-worker coordinator owns queueing, cache identity, and offscreen lifecycle; an offscreen document and worker own byte acquisition, preprocessing, provenance parsing, ONNX Runtime Web sessions, and calibrated fusion. Development-only Python and Node tools prepare locked artifacts, audit benchmark splits, run the same built extension against frozen browser fixtures, and produce deterministic release evidence.

**Tech Stack:** Node.js `>=22.13`, npm `11.19.0`, TypeScript `5.9.3`, esbuild `0.28.2`, Vitest `4.1.10`, Playwright `1.62.1`, Zod `4.4.3`, ONNX Runtime Web `1.27.0`, optional C2PA Web `0.13.4`, Python `3.12` via `uv`, Pillow/NumPy/ONNX Runtime for reference-only parity tools.

**Approved design:** `docs/superpowers/specs/2026-08-13-provenance-lens-design.md` at merged commit `b44e428d48548ba66c2e560e51909e8013505f45`.

---

## Execution contract

1. Work from clean, synced `main`; create one feature branch per phase. All implementation reaches `main` through reviewed PRs.
2. For every production behavior: write one failing test, run it and confirm the expected failure, write the minimal implementation, run the focused test, then run the affected suite before committing.
3. Configuration/lockfile bootstrap in Task 1 exists only to make the first RED test executable. It must not contain product behavior.
4. Every task gets two independent reviews in this order: approved-spec compliance, then code quality/security. Fix and re-review before moving on.
5. Keep model, benchmark, browser, and generated caches outside Git via `PROVENANCE_LENS_CACHE`; never commit machine-specific absolute paths.
6. Do not download large datasets until source license, mount, free-space, and manifest role are verified. Non-commercial or ambiguous datasets are diagnostic-only and cannot support a cash-bounty claim without written permission.
7. Do not open final holdouts until a release lock freezes code, model roster, preprocessing, fusion, and calibration. Any post-open change retires that holdout for selection.
8. Do not submit a POIDH claim unless all release gates pass on exact published bytes. Claim creation and `withdrawTo` are separate reviewed wallet operations requiring an authorized signer.

## Fixed contracts

### Runtime result and threshold

```ts
export const DECISION_THRESHOLD = 0.65 as const;

export type AcquisitionMode =
  | 'exact-http'
  | 'exact-data'
  | 'exact-blob'
  | 'visible-screenshot';

export type NotScoredReason =
  | 'source-unavailable'
  | 'unsupported-format'
  | 'image-too-large'
  | 'not-visible-for-fallback'
  | 'decode-failed'
  | 'model-unavailable'
  | 'inference-failed'
  | 'cancelled';

export interface ScoreEvidence {
  visualProbability: number;
  provenance: 'authenticated-ai' | 'authenticated-other' | 'none' | 'invalid';
  metadataMarkers: readonly string[];
  acquisitionMode: AcquisitionMode;
  modelProfileDigest: string;
}

export type AnalysisResult =
  | { kind: 'scored'; probabilityAi: number; likelyAi: boolean; evidence: ScoreEvidence }
  | { kind: 'not-scored'; reason: NotScoredReason };
```

`likelyAi` must always equal `probabilityAi >= DECISION_THRESHOLD`. No website-specific threshold or low-score “real” claim is permitted.

### Release profile

```ts
export interface ModelProfile {
  id: string;
  version: 1;
  threshold: 0.65;
  members: readonly {
    artifactId: string;
    inputName: string;
    outputName: string;
    outputTransform: 'sigmoid';
  }[];
  preprocessingVersion: string;
  fusion: {
    kind: 'logistic';
    intercept: number;
    visualWeights: readonly number[];
    provenanceWeight: number;
    metadataWeight: number;
  };
}
```

The initial baseline member is the corrected Community Forensics ViT-S FP32 ONNX artifact:

- upstream revision: `ac6ee457bea904a373065754107451793b56db00`
- path: `onnx/model.onnx`
- byte length: `87442080`
- SHA-256: `a42c7d740fbb345ba9a26d469b22f301d73089ce3c6da993877ed2b6965a8ba1`
- input: RGB; shortest edge `440`; center crop `384 x 384`; CLIP mean/std from the pinned preprocessor; `NCHW float32`
- output: single logit transformed with sigmoid

### Benchmark row

```ts
export interface BenchmarkRow {
  sampleId: string;
  label: 0 | 1;
  role: 'train' | 'calibration' | 'development' | 'holdout';
  sourceId: string;
  sourceRevision: string;
  sourceFamily: string;
  generatorFamily: string | null;
  originalAssetGroup: string;
  sourceUrl: string;
  sourceLicense: string;
  byteSha256: string;
  perceptualDigest: string;
  width: number;
  height: number;
}
```

A source enters a qualifying suite only when its source record contains an immutable revision, license evidence URL, permitted-use determination, label provenance, and sample-level digest strategy.

---

## Phase A — trusted foundation

### Task 1: Bootstrap the locked toolchain and executable test harness

**Objective:** Create only the package/test/build configuration needed to run the first failing behavior test reproducibly.

**Files:**
- Create: `package.json`
- Create: `package-lock.json`
- Create: `tsconfig.json`
- Create: `vitest.config.ts`
- Create: `eslint.config.mjs`
- Create: `.prettierrc.json`
- Create: `.gitignore`
- Create: `.nvmrc`
- Create: `tools/build/paths.mts`
- Test: `tests/unit/build/paths.test.ts`

**Step 1: Add the bootstrap manifests**

Pin exact direct dependencies. Use TypeScript `5.9.3` rather than `7.x` because `@typescript-eslint/* 8.67.0` declares TypeScript `<6.1.0`.

```json
{
  "name": "provenance-lens",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22.13.0" },
  "packageManager": "npm@11.19.0",
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test:unit": "vitest run",
    "test:e2e": "playwright test",
    "test": "npm run test:unit",
    "models:fetch": "tsx tools/models/fetch.mts",
    "models:verify": "tsx tools/models/verify.mts",
    "build": "tsx tools/build/build.mts",
    "benchmark:browser": "tsx tools/benchmark/browser_runner.mts",
    "verify:release": "tsx tools/release/verify-release.mts",
    "verify": "npm run format:check && npm run lint && npm run typecheck && npm run test:unit"
  },
  "dependencies": {
    "onnxruntime-web": "1.27.0",
    "zod": "4.4.3"
  },
  "devDependencies": {
    "@playwright/test": "1.62.1",
    "@typescript-eslint/eslint-plugin": "8.67.0",
    "@typescript-eslint/parser": "8.67.0",
    "@vitest/coverage-v8": "4.1.10",
    "esbuild": "0.28.2",
    "eslint": "10.8.1",
    "fast-check": "4.4.0",
    "happy-dom": "20.0.11",
    "prettier": "3.9.6",
    "sharp": "0.34.5",
    "tsx": "4.23.12",
    "typescript": "5.9.3",
    "vitest": "4.1.10"
  }
}
```

Run: `npm install --package-lock-only && npm ci`
Expected: exit `0`; `package-lock.json` records the exact graph.

**Step 2: Write the first failing path-policy test**

```ts
import { describe, expect, it } from 'vitest';
import { resolveCacheRoot } from '../../../tools/build/paths.mjs';

describe('resolveCacheRoot', () => {
  it('uses an explicit cache root without embedding it in artifacts', () => {
    expect(resolveCacheRoot({ PROVENANCE_LENS_CACHE: '/tmp/pl-cache' })).toBe('/tmp/pl-cache');
  });
});
```

Run: `npm run test:unit -- tests/unit/build/paths.test.ts`
Expected RED: module or export is missing.

**Step 3: Implement the minimal cache-root resolver**

The default is a user cache directory derived at runtime; committed files never contain an absolute machine path.

Run: `npm run test:unit -- tests/unit/build/paths.test.ts && npm run verify`
Expected GREEN: focused test and all configured checks pass.

**Step 4: Commit**

```bash
git add package.json package-lock.json tsconfig.json vitest.config.ts eslint.config.mjs .prettierrc.json .gitignore .nvmrc tools/build/paths.mts tests/unit/build/paths.test.ts
git commit -m "chore: bootstrap reproducible extension toolchain"
```

### Task 2: Lock and verify model artifacts

**Objective:** Make model preparation immutable, digest-checked, license-aware, and external-cache-backed.

**Files:**
- Create: `models/model-lock.json`
- Create: `models/profiles/baseline-v1.json`
- Create: `tools/models/schema.ts`
- Create: `tools/models/fetch.mts`
- Create: `tools/models/verify.mts`
- Create: `third-party-licenses/community-forensics-model.txt`
- Test: `tests/unit/models/model-lock.test.ts`
- Test: `tests/policy/models/model-license.test.ts`

**Step 1: Write RED tests**

Test that the lock rejects mutable revisions, wrong byte lengths, invalid SHA-256 strings, unknown licenses, and a profile threshold other than exactly `0.65`. Test the checked-in baseline values above.

Run: `npm run test:unit -- tests/unit/models/model-lock.test.ts tests/policy/models/model-license.test.ts`
Expected RED: schemas and files do not exist.

**Step 2: Implement lock schema and checked-in baseline lock**

The lock includes the ONNX graph, upstream `config.json`, `preprocessor_config.json`, and `LICENSE`, each with immutable revision, bytes, SHA-256, and license evidence. The known auxiliary digests are:

- `config.json`: `4b425d089842fecf8f25fc52aa44d09f49607ef89a0e2ff685ced6ec1c70e9b1`
- `preprocessor_config.json`: `d5e70eaba99880a52978157cf4e6ee71502fed9479dd7b659854107e131ee8f6`
- upstream `LICENSE`: `69a0eab6ca179df33ed80fa378b9458632e14ba9547374e299249e0a4f8076cb`

**Step 3: Implement streamed fetch/verification**

`fetch.mts` writes to a temporary file, streams SHA-256, verifies bytes and digest, then atomically renames. A mismatch deletes the temporary file and exits non-zero. `verify.mts` never downloads.

Run: `PROVENANCE_LENS_CACHE="$PROVENANCE_LENS_CACHE" npm run models:fetch -- baseline-v1`
Expected: four verified artifacts; no model bytes under Git status.

Tamper test: copy the cached model to a temporary cache, alter one byte, and run verification.
Expected RED proof: non-zero with `digest mismatch`, no build output.

Run: `npm run models:verify -- baseline-v1 && npm run test:unit -- tests/unit/models/model-lock.test.ts tests/policy/models/model-license.test.ts`
Expected GREEN.

**Step 4: Commit**

```bash
git add models tools/models third-party-licenses tests/unit/models tests/policy/models
git commit -m "build: lock and verify baseline model artifacts"
```

### Task 3: Create typed message and result contracts

**Objective:** Establish one validated protocol for content, background, offscreen, popup, and benchmark callers.

**Files:**
- Create: `src/shared/constants.ts`
- Create: `src/shared/contracts.ts`
- Create: `src/shared/errors.ts`
- Test: `tests/unit/shared/contracts.test.ts`
- Test: `tests/unit/shared/threshold.test.ts`

**Step 1: Write RED tests**

Cover valid request/result round trips, unknown message kinds, oversized URLs, non-finite probabilities, probability bounds, all typed reason codes, and the exact threshold boundary:

```ts
expect(classifyProbability(0.649999)).toBe(false);
expect(classifyProbability(0.65)).toBe(true);
expect(classifyProbability(1)).toBe(true);
```

Run: `npm run test:unit -- tests/unit/shared/contracts.test.ts tests/unit/shared/threshold.test.ts`
Expected RED: modules missing.

**Step 2: Implement minimal schemas and helpers**

Use discriminated Zod unions. Reject unknown keys. Keep `DECISION_THRESHOLD` in one module imported by runtime and evaluator; no duplicate numeric literal is permitted outside tests/fixtures/profile JSON.

Run: `npm run test:unit -- tests/unit/shared/contracts.test.ts tests/unit/shared/threshold.test.ts && npm run verify`
Expected GREEN.

**Step 3: Commit**

```bash
git add src/shared tests/unit/shared
git commit -m "feat: define validated analysis contracts"
```

### Task 4: Build deterministic bundling and policy checks

**Objective:** Produce a Manifest V3 `dist/` from source with a deterministic file manifest and no undeclared runtime code or data.

**Files:**
- Create: `manifest.json`
- Create: `tools/build/entries.ts`
- Create: `tools/build/build.mts`
- Create: `tools/build/copy-runtime-assets.mts`
- Create: `tools/release/create-manifest.mts`
- Create: `tools/release/check-bundle.mts`
- Create: `playwright.config.ts`
- Create: `tests/e2e/harness/extension.ts`
- Create: `tests/e2e/harness/server.ts`
- Create: `tests/fixtures/pages/smoke.html`
- Create: `tests/e2e/build-smoke.spec.ts`
- Create: `tests/policy/manifest.test.ts`
- Create: `tests/policy/bundle.test.ts`
- Create: `tests/fixtures/build/minimal-entry.ts`

**Step 1: Write RED policy tests**

Assert Manifest V3, ordinary `http/https` match patterns only, least required permissions, extension-page CSP with no remote script source, no `eval`/`new Function`, no hard-coded telemetry/localhost/inference endpoint, and no benchmark label/hash/path payload in `dist/`.

Run: `npm run test:unit -- tests/policy/manifest.test.ts tests/policy/bundle.test.ts`
Expected RED: manifest/build output absent.

**Step 2: Implement the minimal build**

Bundle service-worker ESM, content-script IIFE, offscreen worker/module, popup, and options. Copy only selected ONNX Runtime Web bundles/WASM files and locked release-profile assets. Generate sorted `dist/release-manifest.json` with relative path, byte length, and SHA-256. Normalize generated JSON and source-map behavior. Add a reusable Playwright persistent-profile harness that loads `dist/` with `--disable-extensions-except` and `--load-extension`; its smoke spec proves only that the deterministic no-op shell loads, not that product behavior exists.

**Step 3: Prove deterministic output**

Run two isolated builds from identical inputs and compare release-manifest SHA-256 values.

Run: `npm run build && npx tsx tools/release/check-bundle.mts dist && npm run test:e2e -- tests/e2e/build-smoke.spec.ts`
Expected GREEN: manifest exists, the policy check exits `0`, and a clean persistent Chrome profile loads the unpacked shell.

Run: `git status --short`
Expected: no `dist/` or model/cache files staged because generated output is ignored.

**Step 4: Commit**

```bash
git add manifest.json tools/build tools/release playwright.config.ts tests/e2e/harness tests/e2e/build-smoke.spec.ts tests/policy tests/fixtures/build tests/fixtures/pages/smoke.html .gitignore package.json package-lock.json
git commit -m "build: add deterministic MV3 bundle pipeline"
```

## Phase B — benchmark integrity before optimization

### Task 5: Create the source-license and benchmark-manifest gate

**Objective:** Prevent ambiguous or non-commercial data from entering qualifying cash-bounty evidence.

**Files:**
- Create: `benchmarks/sources.lock.json`
- Create: `benchmarks/README.md`
- Create: `tools/benchmark/schemas.py`
- Create: `tools/benchmark/audit_sources.py`
- Create: `pyproject.toml`
- Create: `uv.lock`
- Test: `tests/benchmark/test_source_audit.py`

**Step 1: Write RED tests**

A qualifying source must have immutable revision, evidence URL, SPDX-like license expression, `cash_bounty_use: allowed`, label provenance, source family, generator family strategy, and redistribution policy. Assert that `unknown`, `research-only`, `CC-BY-NC-*`, and conflicting upstream/mirror terms fail qualifying admission.

Run: `uv run pytest tests/benchmark/test_source_audit.py -q`
Expected RED: audit module missing.

**Step 2: Encode currently verified source determinations**

Record evidence, not images:

- Community Forensics Eval revision `7d4a74a88d2cac93b513c0853bf92c260eaceea0`, `CC-BY-NC-SA-4.0`: diagnostic-only; not qualifying.
- Official GenImage commit `746781bfa446619e1a4629726eb98d5e69c18240`, upstream terms `CC-BY-NC-SA-4.0` plus research-only restriction: diagnostic-only; permissive mirrors do not override upstream terms.
- DiffusionDB revision `fb620fbe49fa4420e0734bd9c0df11f51176b61f`, `CC0-1.0`: candidate AI source pending sample/card audit.
- AIGI Detection Quality Paradox revision `9244882a51dbe33e658fe514488692155d20e5dd`, `CC-BY-4.0`: candidate pending card and upstream-image audit.
- AIGC Detection Benchmark revision `c91d9024a5a77ef06e2ec681b53f9caf08675663`, advertised `Apache-2.0`: candidate pending upstream-image audit.
- Any real-image source must carry sample-level reuse permission; a dataset annotation license alone is not assumed to license linked photos.

**Step 3: Implement fail-closed audit output**

`audit_sources.py` writes a stable JSON report with `qualifying`, `diagnostic_only`, and `blocked` lists plus evidence digests.

Run: `uv run pytest tests/benchmark/test_source_audit.py -q && uv run python tools/benchmark/audit_sources.py`
Expected GREEN: known NC sources appear only under `diagnostic_only`; unresolved candidates remain `blocked`, not qualifying.

**Step 4: Commit**

```bash
git add benchmarks/sources.lock.json benchmarks/README.md tools/benchmark/schemas.py tools/benchmark/audit_sources.py pyproject.toml uv.lock tests/benchmark/test_source_audit.py
git commit -m "test: enforce benchmark source licensing"
```

### Task 6: Build immutable sample manifests and leakage audits

**Objective:** Produce deterministic role assignments and block exact/perceptual cross-role leakage before scoring.

**Files:**
- Create: `tools/benchmark/build_manifest.py`
- Create: `tools/benchmark/split.py`
- Create: `tools/benchmark/dedupe.py`
- Create: `tools/benchmark/verify_manifest.py`
- Create: `tests/benchmark/test_split.py`
- Create: `tests/benchmark/test_dedupe.py`
- Create: `tests/fixtures/benchmark/sample-source.jsonl`

**Step 1: Write RED tests**

Cover deterministic assignment, grouping all transformations with `original_asset_group`, keeping generator/source groups from crossing declared roles, exact SHA-256 duplicates, near-duplicate pHash distance, and a second image-similarity confirmation queue. A single leakage finding must exit non-zero.

Run: `uv run pytest tests/benchmark/test_split.py tests/benchmark/test_dedupe.py -q`
Expected RED.

**Step 2: Implement deterministic manifest tooling**

Sample selection uses a recorded seed and stable hash over source/sample ID before model scoring. Never select samples based on detector output. Manifest rows are sorted by `sample_id`; source bytes stay under the external cache.

**Step 3: Verify RED-GREEN with a deliberate leak**

Run the fixture containing a cross-role duplicate.
Expected RED proof: verifier names both sample IDs and exits non-zero.

Remove the deliberate leak only from the clean fixture and rerun.
Expected GREEN: stable manifest digest and zero cross-role leaks.

**Step 4: Commit**

```bash
git add tools/benchmark tests/benchmark tests/fixtures/benchmark
git commit -m "test: add deterministic benchmark split audits"
```

### Task 7: Implement authoritative metrics and evaluation ledger

**Objective:** Calculate the bounty metric exactly and record every experiment with reproducible lineage.

**Files:**
- Create: `tools/benchmark/metrics.py`
- Create: `tools/benchmark/ledger.py`
- Create: `tools/benchmark/score_predictions.py`
- Create: `tests/benchmark/test_metrics.py`
- Create: `tests/benchmark/test_ledger.py`
- Create: `benchmarks/ledger/.gitkeep`

**Step 1: Write RED tests**

Cover TPR, TNR, balanced accuracy, the inclusive `0.65` boundary, class imbalance, failures/coverage, per-family recall, Brier score, ECE, and seeded stratified bootstrap confidence intervals. Include a hand-calculated confusion matrix.

Run: `uv run pytest tests/benchmark/test_metrics.py tests/benchmark/test_ledger.py -q`
Expected RED.

**Step 2: Implement metrics and append-only rows**

Each ledger row records code/model/profile/manifest digests, roles opened, threshold, counts, latency, memory, wall time, tool versions, failures, and artifact path/digest. Reject comparison of incompatible manifest/metric versions.

Run: `uv run pytest tests/benchmark/test_metrics.py tests/benchmark/test_ledger.py -q`
Expected GREEN with exact hand-calculated values.

**Step 3: Commit**

```bash
git add tools/benchmark tests/benchmark benchmarks/ledger
git commit -m "test: add exact benchmark metrics and ledger"
```

### Task 8: Implement the pinned Python reference pipeline

**Objective:** Create a non-authoritative reference that fixes decode, resize, crop, normalization, logit, and sigmoid semantics for parity tests.

**Files:**
- Create: `tools/reference/preprocess.py`
- Create: `tools/reference/infer.py`
- Create: `tools/reference/export_vectors.py`
- Create: `tests/reference/test_preprocess.py`
- Create: `tests/reference/test_infer.py`
- Create: `tests/fixtures/parity/generate_fixtures.py`
- Generate: `tests/fixtures/parity/manifest.json`

**Step 1: Write RED tests**

Programmatically create redistributable fixtures for landscape, portrait, odd dimensions, alpha, EXIF orientation, low resolution, grayscale, PNG/JPEG/WebP, and color-profile cases. Assert shortest-edge `440`, center crop `384`, exact CLIP mean/std, NCHW order, float32, and stable output shape.

Run: `uv run pytest tests/reference/test_preprocess.py -q`
Expected RED.

**Step 2: Implement preprocessing and inference**

Use pinned Pillow, NumPy, and a Python ONNX Runtime version verified to load the model. Load only from the digest-verified cache. Export fixture tensors, logits, probabilities, and tolerances with source/model/profile digests.

Run: `uv run pytest tests/reference/test_preprocess.py tests/reference/test_infer.py -q`
Expected GREEN.

**Step 3: Commit compact vectors only**

Do not commit model bytes or bulk images. Commit generated fixture sources and compact reference summaries whose licenses are project-owned/MIT.

```bash
git add tools/reference tests/reference tests/fixtures/parity pyproject.toml uv.lock
git commit -m "test: establish pinned reference inference vectors"
```

## Phase C — exact browser inference

### Task 9: Implement browser preprocessing parity

**Objective:** Convert decoded browser pixels into the exact baseline tensor and prove parity against Task 8.

**Files:**
- Create: `src/inference/preprocess.ts`
- Create: `src/inference/pixels.ts`
- Create: `tests/parity/preprocess.test.ts`
- Create: `tests/fixtures/parity/browser-loader.html`

**Step 1: Write RED parity tests**

Load every parity fixture, run the planned browser preprocessing function, and compare dimensions and selected tensor statistics/values to the reference vectors. Include alpha compositing and orientation.

Run: `npm run test:unit -- tests/parity/preprocess.test.ts`
Expected RED.

**Step 2: Implement minimal deterministic resize/crop/normalize**

Start with browser decode/canvas. If fixture error exceeds the frozen tolerance, replace resizing with the explicit tested bilinear kernel rather than widening tolerance.

Run: `npm run test:unit -- tests/parity/preprocess.test.ts`
Expected GREEN: every fixture within the recorded tolerance.

**Step 3: Commit**

```bash
git add src/inference tests/parity tests/fixtures/parity
git commit -m "feat: match browser preprocessing to reference"
```

### Task 10: Implement model-session and calibrated-profile adapters

**Objective:** Load locked extension-local ONNX assets under WebGPU or WASM and return validated probabilities.

**Files:**
- Create: `src/inference/session.ts`
- Create: `src/inference/profile.ts`
- Create: `src/inference/fusion.ts`
- Create: `src/inference/hash.ts`
- Test: `tests/unit/inference/profile.test.ts`
- Test: `tests/unit/inference/fusion.test.ts`
- Test: `tests/parity/inference.test.ts`
- Test: `tests/e2e/inference-parity.spec.ts`

**Step 1: Write RED tests**

Cover profile digest calculation, single-logit sigmoid, bounded finite outputs, missing member behavior, backend selection, model digest mismatch, and exact threshold agreement. Use a dependency-injected session interface so unit tests need no mock Chrome APIs.

Run: `npm run test:unit -- tests/unit/inference/profile.test.ts tests/unit/inference/fusion.test.ts tests/parity/inference.test.ts`
Expected RED.

**Step 2: Implement the minimal adapters**

Configure ORT asset URLs through `chrome.runtime.getURL`. Try the predeclared WebGPU provider first; fall back only to the prevalidated WASM profile. Do not silently reuse full-profile calibration after a member failure.

**Step 3: Run real model parity**

Run: `npm run test:unit -- tests/unit/inference/profile.test.ts tests/unit/inference/fusion.test.ts tests/parity/inference.test.ts && npm run build && npm run test:e2e -- tests/e2e/inference-parity.spec.ts`
Expected GREEN: absolute probability delta `<=0.01` and 100% agreement at `0.65`.

**Step 4: Commit**

```bash
git add src/inference tests/unit/inference tests/parity tests/e2e/inference-parity.spec.ts package.json package-lock.json
git commit -m "feat: add locked browser inference profiles"
```

### Task 11: Build the offscreen runtime and inference worker

**Objective:** Own long-lived model sessions outside the service worker with bounded sequential inference and typed failure isolation.

**Files:**
- Create: `src/offscreen/index.html`
- Create: `src/offscreen/main.ts`
- Create: `src/offscreen/inference.worker.ts`
- Create: `src/offscreen/controller.ts`
- Test: `tests/unit/offscreen/controller.test.ts`
- Test: `tests/integration/offscreen-lifecycle.test.ts`

**Step 1: Write RED tests**

Cover readiness, one in-flight inference by default, FIFO completion within same priority, cancellation, one restart on transient worker crash, circuit-open after repeat crash, raw-byte release, and bounded diagnostic state.

Run: `npm run test:unit -- tests/unit/offscreen/controller.test.ts tests/integration/offscreen-lifecycle.test.ts`
Expected RED.

**Step 2: Implement minimal controller/worker**

The offscreen main thread validates messages and transfers `ArrayBuffer`s to the worker. The worker owns sessions; returned results contain no image bytes. Timeout/error paths map to typed `NotScoredReason` values.

Run: `npm run test:unit -- tests/unit/offscreen/controller.test.ts tests/integration/offscreen-lifecycle.test.ts && npm run verify`
Expected GREEN.

**Step 3: Commit**

```bash
git add src/offscreen tests/unit/offscreen tests/integration
git commit -m "feat: add isolated offscreen inference runtime"
```

## Phase D — automatic page analysis

### Task 12: Implement bounded queueing, cache identity, and offscreen lifecycle

**Objective:** Coordinate analyses safely across tabs without relying on service-worker persistence.

**Files:**
- Create: `src/background/queue.ts`
- Create: `src/background/cache.ts`
- Create: `src/background/offscreen.ts`
- Create: `src/background/messages.ts`
- Create: `src/background/index.ts`
- Test: `tests/unit/background/queue.test.ts`
- Test: `tests/unit/background/cache.test.ts`
- Test: `tests/unit/background/messages.test.ts`

**Step 1: Write RED tests**

Cover visible-main-frame priority, stable ordering, per-tab/global caps, deduplication, cancellation, sender tab/frame validation, stale element generations, backpressure, and cache key `SHA-256(bytes)+profile digest+preprocessing version`.

Run: `npm run test:unit -- tests/unit/background/queue.test.ts tests/unit/background/cache.test.ts tests/unit/background/messages.test.ts`
Expected RED.

**Step 2: Implement minimal coordinator**

Persist compact scored results only; never raw bytes. Recreate offscreen state on service-worker restart and expose queue counts to popup/content clients.

Run: `npm run test:unit -- tests/unit/background/queue.test.ts tests/unit/background/cache.test.ts tests/unit/background/messages.test.ts && npm run test:unit`
Expected GREEN.

**Step 3: Commit**

```bash
git add src/background tests/unit/background
git commit -m "feat: coordinate bounded image analysis jobs"
```

### Task 13: Implement eligibility and mutation-aware image discovery

**Objective:** Automatically discover `<img>`, `<picture>`, and CSS backgrounds, including dynamic and responsive changes.

**Files:**
- Create: `src/content/eligibility.ts`
- Create: `src/content/discovery.ts`
- Create: `src/content/generation.ts`
- Create: `src/content/index.ts`
- Test: `tests/unit/content/eligibility.test.ts`
- Test: `tests/unit/content/discovery.test.ts`
- Test: `tests/e2e/content-discovery.spec.ts`

**Step 1: Write RED tests**

Cover decoded minimum `96x96`, rendered minimum `64x64`, tracking pixels, sprites, transparent/empty URLs, `currentSrc`, `<picture>`, multiple CSS background layers, lazy insertion, source mutation, SPA reuse, detached-before-decode, and element-generation increments.

Run: `npm run test:unit -- tests/unit/content/eligibility.test.ts tests/unit/content/discovery.test.ts && npm run build && npm run test:e2e -- tests/e2e/content-discovery.spec.ts`
Expected RED.

**Step 2: Implement discovery**

Use `MutationObserver`, `ResizeObserver`, and `IntersectionObserver`; visibility changes priority only, never eligibility. Batch updates and apply backpressure from coordinator messages.

Run: `npm run test:unit -- tests/unit/content/eligibility.test.ts tests/unit/content/discovery.test.ts && npm run build && npm run test:e2e -- tests/e2e/content-discovery.spec.ts`
Expected GREEN.

**Step 3: Commit**

```bash
git add src/content tests/unit/content tests/e2e/content-discovery.spec.ts
git commit -m "feat: discover eligible page images automatically"
```

### Task 14: Implement accessible badges and evidence panels

**Objective:** Render `Scanning`, `AI nn%`, and `Not scored` states without contaminating screenshots or trusting page HTML.

**Files:**
- Create: `src/content/overlay.ts`
- Create: `src/content/positioning.ts`
- Create: `src/content/overlay.css`
- Test: `tests/unit/content/overlay.test.ts`
- Test: `tests/e2e/overlay.spec.ts`

**Step 1: Write RED tests**

Assert closed Shadow DOM, text-node rendering, exact score display, accent at `>=0.65`, neutral below, failure reason, visible focus, accessible name, keyboard expansion/reveal, reduced motion, and hide/restore around screenshots.

Run: `npm run test:unit -- tests/unit/content/overlay.test.ts && npm run build && npm run test:e2e -- tests/e2e/overlay.spec.ts`
Expected RED.

**Step 2: Implement minimal visual system**

Use system UI/monospace stacks, restrained vermilion accent, no gradient, no unsupported “real” language, and one overlay layer whose position updates batch in `requestAnimationFrame`.

Run: `npm run test:unit -- tests/unit/content/overlay.test.ts && npm run build && npm run test:e2e -- tests/e2e/overlay.spec.ts`
Expected GREEN.

**Step 3: Commit**

```bash
git add src/content tests/unit/content tests/e2e/overlay.spec.ts
git commit -m "feat: render accessible per-image evidence badges"
```

### Task 15: Implement exact-source byte acquisition and SSRF controls

**Objective:** Fetch only the displayed asset under bounded, credential-free, size/type-safe rules.

**Files:**
- Create: `src/inference/acquisition/url-policy.ts`
- Create: `src/inference/acquisition/fetch-image.ts`
- Create: `src/inference/acquisition/data-blob.ts`
- Test: `tests/unit/acquisition/url-policy.test.ts`
- Test: `tests/integration/acquisition-fetch.test.ts`
- Create: `tests/fixtures/server/routes.ts`

**Step 1: Write RED tests**

Cover supported schemes, embedded credentials, MIME sniff mismatch, redirect cap, compressed-byte cap, decoded-pixel cap handoff, credential omission, timeout, unrelated-origin redirect, and private-network/loopback targets. A public page may not induce a privileged fetch to loopback/private address; same-origin local fixture pages remain testable.

Run: `npm run test:unit -- tests/unit/acquisition/url-policy.test.ts tests/integration/acquisition-fetch.test.ts`
Expected RED.

**Step 2: Implement minimal acquisition**

Use `credentials:'omit'`, `referrerPolicy:'no-referrer'`, `cache:'no-store'`, bounded manual redirects, streaming byte count, supported raster types, and typed reasons. Data/blob transfers are size-bounded and never persisted.

Run: `npm run test:unit -- tests/unit/acquisition/url-policy.test.ts tests/integration/acquisition-fetch.test.ts`
Expected GREEN.

**Step 3: Commit**

```bash
git add src/inference/acquisition tests/unit/acquisition tests/integration tests/fixtures/server
git commit -m "feat: acquire displayed images with bounded policy"
```

### Task 16: Implement visible screenshot-crop fallback

**Objective:** Score visible active-tab images when exact bytes are unavailable, with explicit acquisition evidence.

**Files:**
- Create: `src/background/capture.ts`
- Create: `src/inference/acquisition/crop.ts`
- Test: `tests/unit/acquisition/crop.test.ts`
- Test: `tests/e2e/screenshot-fallback.spec.ts`

**Step 1: Write RED tests**

Cover device-pixel ratio, viewport clipping, transformed/unreliable geometry, occlusion threshold, inactive tab, offscreen image, overlay hide acknowledgment, crop bounds, and result mode `visible-screenshot`.

Run: `npm run test:unit -- tests/unit/acquisition/crop.test.ts && npm run build && npm run test:e2e -- tests/e2e/screenshot-fallback.spec.ts`
Expected RED.

**Step 2: Implement minimal capture handshake and crop**

Call `captureVisibleTab` only for the active visible tab and after overlay-hide acknowledgment. Destroy the PNG/crop buffer after inference and restore overlays in `finally`.

Run: `npm run test:unit -- tests/unit/acquisition/crop.test.ts && npm run build && npm run test:e2e -- tests/e2e/screenshot-fallback.spec.ts`
Expected GREEN.

**Step 3: Commit**

```bash
git add src/background/capture.ts src/inference/acquisition/crop.ts tests/unit/acquisition/crop.test.ts tests/e2e/screenshot-fallback.spec.ts
git commit -m "feat: add local screenshot acquisition fallback"
```

### Task 17: Implement conservative metadata and authenticated provenance signals

**Objective:** Add positive-only auxiliary evidence without allowing spoofable absence/presence to masquerade as authenticity.

**Files:**
- Create: `src/inference/metadata/png.ts`
- Create: `src/inference/metadata/jpeg.ts`
- Create: `src/inference/metadata/webp.ts`
- Create: `src/inference/metadata/markers.ts`
- Create: `src/inference/provenance/c2pa.ts`
- Create: `third-party-licenses/c2pa-web.txt`
- Test: `tests/unit/metadata/parsers.test.ts`
- Test: `tests/unit/provenance/c2pa.test.ts`
- Add only if admitted: `@contentauth/c2pa-web@0.13.4`

**Step 1: Write RED parser/property tests**

Cover malformed lengths/chunks, oversized metadata, false marker strings in pixel payloads, explicit generative workflow tags, stripped metadata, fake camera tags, invalid C2PA manifests, generic valid provenance, and authenticated trained-algorithmic-media assertions. Use `fast-check` for bounded malformed inputs.

Run: `npm run test:unit -- tests/unit/metadata/parsers.test.ts tests/unit/provenance/c2pa.test.ts`
Expected RED.

**Step 2: Implement positive-only evidence**

Unsigned explicit generator tags are weak positive features only. Absence never lowers visual probability. Generic/invalid C2PA is not AI evidence. Bundle all C2PA WASM/worker assets locally and block its release if no redistributable known-answer fixtures prove signature/manifest behavior.

Run: `npm run test:unit -- tests/unit/metadata/parsers.test.ts tests/unit/provenance/c2pa.test.ts tests/policy/bundle.test.ts && npm run build`
Expected GREEN or, if C2PA admission fails, the release profile records provenance feature disabled rather than shipping a placeholder.

**Step 3: Commit**

```bash
git add src/inference/metadata src/inference/provenance tests/unit/metadata tests/unit/provenance third-party-licenses package.json package-lock.json
git commit -m "feat: add conservative local provenance evidence"
```

### Task 18: Implement popup, settings, readiness, and optional blur

**Objective:** Expose page counts, pause/blur controls, cache clearing, privacy state, and fail-closed readiness.

**Files:**
- Create: `src/popup/index.html`
- Create: `src/popup/index.ts`
- Create: `src/popup/popup.css`
- Create: `src/options/index.html`
- Create: `src/options/index.ts`
- Create: `src/options/options.css`
- Create: `src/shared/settings.ts`
- Test: `tests/unit/ui/settings.test.ts`
- Test: `tests/e2e/popup-options.spec.ts`

**Step 1: Write RED tests**

Cover analyzed/likely/below/queued/skipped/failed counts, automatic analysis before opening popup, pause/resume, blur threshold exactly `0.65`, reveal/reapply keyboard controls, clear cache, model digest failure, backend unavailability, known-answer readiness, and no false green-ready state.

Run: `npm run test:unit -- tests/unit/ui/settings.test.ts && npm run build && npm run test:e2e -- tests/e2e/popup-options.spec.ts`
Expected RED.

**Step 2: Implement minimal UI**

Use semantic HTML and scoped CSS only. Store versioned settings and compact caches in `chrome.storage.local`; never store image bytes or browsing history.

Run: `npm run test:unit -- tests/unit/ui/settings.test.ts && npm run build && npm run test:e2e -- tests/e2e/popup-options.spec.ts`
Expected GREEN.

**Step 3: Commit**

```bash
git add src/popup src/options src/shared/settings.ts tests/unit/ui tests/e2e/popup-options.spec.ts
git commit -m "feat: add readiness and page controls"
```

## Phase E — real Chrome proof

### Task 19: Build the extension fixture server and clean-profile harness

**Objective:** Launch the unpacked built extension in a fresh persistent Chromium profile against controlled ordinary pages.

**Files:**
- Create: `playwright.config.ts`
- Create: `tests/e2e/harness/extension.ts`
- Create: `tests/e2e/harness/server.ts`
- Create: `tests/fixtures/pages/static.html`
- Create: `tests/fixtures/pages/dynamic.html`
- Create: `tests/fixtures/pages/responsive.html`
- Create: `tests/fixtures/pages/backgrounds.html`
- Create: `tests/e2e/discovery.spec.ts`

**Step 1: Write RED E2E spec**

Extend the Task 4 persistent-profile harness and fixtures. Verify static, lazy, dynamic, responsive, background, cross-origin, data, blob, inaccessible, and failure cases.

Run: `npm run build && npm run test:e2e -- tests/e2e/discovery.spec.ts`
Expected RED until runtime wiring is complete.

**Step 2: Wire manifest entries and runtime**

Make the end-to-end path content -> background -> offscreen -> inference -> badge operational. Use the real model for known-answer cases and a test-only build profile for fast state-machine coverage; policy tests must prove the test profile cannot enter release output.

Run: `npm run build && npm run test:e2e -- tests/e2e/discovery.spec.ts`
Expected GREEN with automatic analysis and no manual upload.

**Step 3: Commit**

```bash
git add playwright.config.ts tests/e2e tests/fixtures/pages manifest.json tools/build src
git commit -m "test: prove automatic analysis in clean Chrome"
```

### Task 20: Prove restart, backend fallback, offline use, and runtime egress

**Objective:** Validate lifecycle resilience and privacy with network interception, not source inspection alone.

**Files:**
- Create: `tests/e2e/lifecycle.spec.ts`
- Create: `tests/e2e/offline.spec.ts`
- Create: `tests/e2e/egress.spec.ts`
- Create: `tests/e2e/backend-parity.spec.ts`
- Create: `tools/release/egress-report.mts`

**Step 1: Write RED tests**

Terminate/restart the service worker, force WebGPU initialization failure, disable network after install, and intercept every request. The allowlist contains extension-local assets and exact displayed-image fixture URLs only. Any telemetry, remote model/script, inference API, hidden localhost service, or undeclared request fails.

Run: `npm run build && npm run test:e2e -- tests/e2e/lifecycle.spec.ts tests/e2e/offline.spec.ts tests/e2e/egress.spec.ts tests/e2e/backend-parity.spec.ts`
Expected RED before lifecycle hooks/reporting exist.

**Step 2: Implement only required recovery/reporting hooks**

One offscreen restart is allowed; repeated crash opens the circuit. WASM fallback uses its own validated profile. Offline local fixtures continue scoring after build/install.

Run: `npm run build && npm run test:e2e -- tests/e2e/lifecycle.spec.ts tests/e2e/offline.spec.ts tests/e2e/egress.spec.ts tests/e2e/backend-parity.spec.ts`
Expected GREEN with a machine-readable egress report listing request URL class and initiator, without browsing data.

**Step 3: Commit**

```bash
git add tests/e2e tools/release src/background src/offscreen src/inference
git commit -m "test: prove offline and egress-safe runtime"
```

## Phase F — benchmark, calibration, and candidate admission

### Task 21: Prepare at least two license-cleared core suites

**Objective:** Freeze independent, balanced, sample-digested development/calibration/holdout manifests without importing restricted imagery.

**Files:**
- Modify: `benchmarks/sources.lock.json`
- Create: `benchmarks/manifests/core-a.jsonl`
- Create: `benchmarks/manifests/core-b.jsonl`
- Create: `benchmarks/manifests/web-wild.jsonl`
- Create: `tools/benchmark/prepare_sources.py`
- Create: `tests/benchmark/test_prepare_sources.py`

**Step 1: Complete source audits before bytes**

For each candidate, read the immutable official card/repository and upstream image terms. Qualifying candidates must permit cash-bounty evaluation. Prefer independently curated generator families and sample-level licensed real images. Keep Community Forensics Eval and GenImage diagnostic-only unless written permission changes their status.

**Step 2: Write RED preparation tests**

Reject mutable refs, missing license evidence, changed source revisions, missing sample digests, class imbalance beyond the predeclared tolerance, and source/generator overlap that violates suite independence.

Run: `uv run pytest tests/benchmark/test_prepare_sources.py -q`
Expected RED.

**Step 3: Prepare deterministic compact suites**

Stream only selected rows into the external cache. Record every byte digest and source role. Each core clean/stress slice must have enough real and AI samples to define balanced accuracy and confidence intervals; exact counts are frozen before any detector score is read.

Run: `uv run python tools/benchmark/audit_sources.py && uv run pytest tests/benchmark/test_prepare_sources.py tests/benchmark/test_split.py tests/benchmark/test_dedupe.py -q && uv run python tools/benchmark/prepare_sources.py && uv run python tools/benchmark/verify_manifest.py benchmarks/manifests/core-a.jsonl benchmarks/manifests/core-b.jsonl benchmarks/manifests/web-wild.jsonl`
Expected GREEN: at least two qualifying core manifests and one diagnostic web-wild manifest, each with a stable digest and zero cross-role leakage.

**Step 4: Commit manifests/evidence only**

```bash
git add benchmarks tools/benchmark tests/benchmark
git commit -m "test: freeze licensed proxy benchmark manifests"
```

### Task 22: Run and record the single-model browser baseline

**Objective:** Establish honest quality, coverage, and performance before adding any complementary signal.

**Files:**
- Create: `tools/benchmark/browser_runner.mts`
- Create: `tests/e2e/benchmark-runner.spec.ts`
- Create: `benchmarks/reports/baseline-v1.json`
- Append: `benchmarks/ledger/evaluations.ndjson`

**Step 1: Write RED runner tests**

Ensure the runner uses built-extension results, includes failures in coverage, reads threshold from the profile, never reads labels before prediction collection, and rejects profile/manifest digest mismatch.

Run: `npm run build && npm run test:e2e -- tests/e2e/benchmark-runner.spec.ts`
Expected RED.

**Step 2: Run development/calibration roles only**

Build exact release-profile bytes, score through Chrome, then calculate metrics externally. Record tool calls, wall time, Chrome/ORT versions, backend, latency, memory, failures, and all digests.

Run: `npm run benchmark:browser -- --roles development,calibration --profile baseline-v1`
Expected GREEN: complete report; no claim about holdout or bounty qualification.

**Step 3: Commit the compact report and ledger row**

```bash
git add tools/benchmark tests/e2e/benchmark-runner.spec.ts benchmarks/reports/baseline-v1.json benchmarks/ledger/evaluations.ndjson package.json
git commit -m "test: record browser-path detector baseline"
```

### Task 23: Admit complementary candidates one at a time

**Objective:** Build the approved hybrid only when a candidate adds orthogonal, licensed, browser-feasible value.

**Files:**
- Create: `models/candidates/README.md`
- Create per candidate: `models/candidates/<candidate>.json`
- Create: `tools/benchmark/compare_candidates.py`
- Create: `tests/benchmark/test_candidate_gate.py`
- Append: `benchmarks/ledger/evaluations.ndjson`

**Step 1: Write RED admission tests**

Require inspectable provenance/license, immutable artifacts, browser parity, compatible suite manifest, incremental metrics, latency/memory, and no core suite regression below gate. Reject self-reported competitor scores as evidence.

Run: `uv run pytest tests/benchmark/test_candidate_gate.py -q`
Expected RED.

**Step 2: Evaluate candidates sequentially**

Candidate order:

1. authenticated C2PA and conservative metadata features;
2. one independent permissively licensed visual model with different training families;
3. a compact project-trained residual/frequency head only if baseline error analysis justifies it.

For every candidate: freeze artifact/profile, write browser parity tests, score development roles, append ledger, then accept/reject with exact incremental balanced accuracy, worst-suite change, latency, and memory. Do not touch holdouts.

Run for each candidate: `uv run pytest tests/benchmark/test_candidate_gate.py -q && npm run build && npm run test:e2e -- tests/e2e/inference-parity.spec.ts && uv run python tools/benchmark/compare_candidates.py --baseline baseline-v1 --candidate <candidate-id>`
Expected GREEN: a machine-readable `accepted` or `rejected` decision with exact deltas; no holdout role appears in inputs.

**Step 3: Commit only evidence-backed admissions**

Expected: final roster may remain baseline-only if no candidate clears all gates; “hybrid” is not a reason to keep a harmful member.

```bash
git add models/candidates tools/benchmark tests/benchmark benchmarks/ledger models/profiles
git commit -m "test: admit only measured complementary signals"
```

### Task 24: Fit calibration/fusion and freeze the release lock

**Objective:** Map selected signals to an inspectable probability whose exact decision boundary is `0.65`, then close model selection.

**Files:**
- Create: `tools/benchmark/calibrate.py`
- Create: `tools/benchmark/freeze_release.py`
- Create: `benchmarks/locks/release-lock.json`
- Modify: `models/profiles/release-v1.json`
- Test: `tests/benchmark/test_calibration.py`
- Test: `tests/benchmark/test_release_lock.py`

**Step 1: Write RED tests**

Cover calibration-only fitting, finite logistic weights, missing-signal flags, reliability output, unchanged threshold, profile digest, refusal to open holdout before lock, and automatic holdout retirement if code/profile changes afterward.

Run: `uv run pytest tests/benchmark/test_calibration.py tests/benchmark/test_release_lock.py -q`
Expected RED.

**Step 2: Fit and freeze**

Use calibration role for probability mapping; development role for roster/policy selection. Write immutable code/model/profile/manifest/metric digests and timestamp to release lock. After this step, no detector/calibration change is permitted without a new release version and fresh holdout.

Run: `uv run pytest tests/benchmark/test_calibration.py tests/benchmark/test_release_lock.py -q && uv run python tools/benchmark/calibrate.py --profile release-v1 && npm run build && npm run test:e2e -- tests/e2e/inference-parity.spec.ts && uv run python tools/benchmark/freeze_release.py --profile release-v1`
Expected GREEN.

**Step 3: Commit**

```bash
git add tools/benchmark benchmarks/locks models/profiles tests/benchmark
git commit -m "test: freeze calibrated release profile"
```

### Task 25: Open final browser holdouts once and apply release gates

**Objective:** Produce the authoritative public proxy result on frozen exact bytes without post-hoc tuning.

**Files:**
- Create: `benchmarks/reports/release-v1.json`
- Create: `benchmarks/reports/release-v1.md`
- Append: `benchmarks/ledger/evaluations.ndjson`
- Create: `benchmarks/locks/holdout-opened.json`
- Create: `tools/benchmark/open_holdout.py`
- Create: `tools/benchmark/apply_release_gates.py`
- Test: `tests/benchmark/test_holdout_gate.py`

**Step 1: Write RED holdout-gate tests**

Cover missing release lock, prior-open marker, profile/model/code digest drift, any quality-gate failure, and successful atomic creation of the opened marker plus immutable report.

Run: `uv run pytest tests/benchmark/test_holdout_gate.py -q`
Expected RED.

**Step 2: Verify pre-open state**

Run: `uv run python tools/benchmark/audit_sources.py && uv run python tools/benchmark/verify_manifest.py benchmarks/manifests/core-a.jsonl benchmarks/manifests/core-b.jsonl && npm run models:verify -- release-v1 && npm run build && uv run python tools/benchmark/open_holdout.py --verify-only benchmarks/locks/release-lock.json`

Expected: all pass; otherwise stop without scoring.

**Step 3: Score clean/stress holdouts through built Chrome extension**

Run: `npm run benchmark:browser -- --roles holdout --profile release-v1 --release-lock benchmarks/locks/release-lock.json`
Expected: immutable predictions and opened marker; do not change code, calibration, roster, sample selection, or threshold after seeing output.

**Step 4: Apply every gate mechanically**

The gate script must require:

- `>=80.0%` balanced accuracy on each core suite;
- `>=80.0%` on each predeclared clean/stress paired slice;
- pooled stratified-bootstrap lower 95% bound `>=75.0%`;
- predeclared source-family recall `>=70.0%` unless explicitly blocked per spec;
- eligible-image coverage `>=98.0%`;
- browser/reference and WebGPU/WASM boundary parity;
- documented latency/memory.

Run: `uv run pytest tests/benchmark/test_holdout_gate.py -q && uv run python tools/benchmark/apply_release_gates.py benchmarks/reports/release-v1.json`
Expected GREEN: gate code is verified and result is either `QUALIFYING` with every metric/digest, or `BLOCKED` with exact failed gates. A blocked result cannot be reframed as bounty-ready.

**Step 5: Commit the immutable outcome**

```bash
git add benchmarks/reports benchmarks/ledger benchmarks/locks/holdout-opened.json
git commit -m "test: publish frozen browser holdout results"
```

## Phase G — release proof and publication

### Task 26: Add accessibility, infinite-feed, and resource-budget proof

**Objective:** Verify usability and bounded runtime behavior on the frozen release profile.

**Files:**
- Create: `tests/e2e/accessibility.spec.ts`
- Create: `tests/e2e/infinite-feed.spec.ts`
- Create: `tests/e2e/performance.spec.ts`
- Create: `benchmarks/reports/runtime-v1.json`

**Step 1: Write RED tests**

Cover keyboard order, focus visibility, accessible names, contrast, 200% zoom, reduced motion, screen-reader semantics, queue/cache/observer bounds on an infinite feed, p50/p95 latency, and peak memory.

Run: `npm run build && npm run test:e2e -- tests/e2e/accessibility.spec.ts tests/e2e/infinite-feed.spec.ts tests/e2e/performance.spec.ts`
Expected RED until instrumentation/limits are exposed.

**Step 2: Implement only measured fixes**

Do not relax quality gates to improve latency. If memory requires model eviction, verify the exact profile and parity after the change; otherwise it is a new release candidate.

Run: `npm run build && npm run test:e2e -- tests/e2e/accessibility.spec.ts tests/e2e/infinite-feed.spec.ts tests/e2e/performance.spec.ts`
Expected GREEN and stable runtime report.

**Step 3: Commit**

```bash
git add tests/e2e benchmarks/reports/runtime-v1.json src
git commit -m "test: prove accessible bounded runtime behavior"
```

### Task 27: Compose the clean-clone release verifier and reproducible package

**Objective:** Make one command prove policy, licenses, model digests, tests, build, E2E, benchmark locks, and reproducibility.

**Files:**
- Create: `tools/release/verify-release.mts`
- Create: `tools/release/verify-repro.mts`
- Create: `tools/release/licenses.mts`
- Create: `tools/release/package.mts`
- Test: `tests/policy/release-verifier.test.ts`
- Modify: `package.json`

**Step 1: Write RED verifier tests**

Inject one failure at a time: dirty tree, model mismatch, missing notice, forbidden bundle literal, manifest/profile drift, failed gate, nonmatching consecutive build manifests, and absent benchmark lock. Assert fail-closed non-zero output.

Run: `npm run test:unit -- tests/policy/release-verifier.test.ts`
Expected RED.

**Step 2: Implement `npm run verify:release`**

The command performs fresh model verification/fetch as explicitly configured, source/license audit, formatting/lint/typecheck, unit/property/parity/policy tests, deterministic build twice, clean-profile E2E, offline/egress/accessibility/resource tests, frozen benchmark report verification, notice generation, and release archive/digest creation.

Run from a clean clone with an external cache:

```bash
npm ci
npm run models:fetch
npm run verify:release
```

Expected: exit `0`, two matching release-manifest digests, a package under `artifacts/`, and an evidence summary. If the holdout gates failed, `verify:release` must fail and no bounty package is labeled qualifying.

Expected GREEN: verifier tests and the clean-clone command pass on qualifying evidence.

**Step 3: Commit**

```bash
git add tools/release tests/policy/release-verifier.test.ts package.json package-lock.json
git commit -m "build: add fail-closed release verification"
```

### Task 28: Publish complete model, benchmark, privacy, security, and build documentation

**Objective:** Make exact bytes and limitations independently reproducible without overstating private-benchmark status.

**Files:**
- Modify: `README.md`
- Create: `docs/MODEL_CARD.md`
- Create: `docs/BENCHMARK.md`
- Create: `docs/PRIVACY.md`
- Create: `docs/SECURITY.md`
- Create: `THIRD_PARTY_NOTICES.md`
- Create: `CHANGELOG.md`
- Test: `tests/policy/docs.test.ts`

**Step 1: Write RED documentation-policy tests**

Require exact release commit token replacement, release-manifest/package digests, model revision/digest/license, install/build commands, suite manifest/report digests, confusion matrices, coverage/failures, latency/memory, limitations, no “proven real” language, no claim of passing POIDH private benchmark, and no private machine paths/secrets.

Run: `npm run test:unit -- tests/policy/docs.test.ts`
Expected RED.

**Step 2: Write evidence-grounded docs**

Document automatic ordinary-page behavior, confidence semantics, local processing, exact-source fetch distinction, optional blur, accessibility, unsupported cases, benchmark license scope, and private benchmark unknown. Include screenshots only from project-owned fixtures.

Run: `npm run test:unit -- tests/policy/docs.test.ts && npm run verify`
Expected GREEN.

**Step 3: Commit**

```bash
git add README.md docs THIRD_PARTY_NOTICES.md CHANGELOG.md tests/policy/docs.test.ts
git commit -m "docs: publish reproducible release evidence"
```

### Task 29: Final integration review, reviewed PRs, merge, and byte-level readback

**Objective:** Publish only reviewed exact bytes and leave local `main` identical to remote `main`.

**Files:**
- No new product files expected; fixes must include regression tests.

**Step 1: Run final verifier from clean checkout**

Run: `npm ci && npm run models:fetch && npm run verify:release`
Expected: exit `0`; record command, versions, commit, release-manifest digest, package digest, benchmark report digest, duration, failures `0`.

**Step 2: Perform independent reviews**

Review line by line against the approved design and this plan. Run spec-compliance review first, then code/security/benchmark-integrity review. Any finding requires a regression test, fix, full affected verification, and re-review.

**Step 3: Push and open PR**

Use public PR flow. Confirm remote head SHA and fetch the release manifest/docs back through GitHub; local and remote SHA-256 must match.

**Step 4: Merge and synchronize**

```bash
git checkout main
git pull --ff-only origin main
git push origin main
test "$(git rev-parse HEAD)" = "$(git rev-parse origin/main)"
test -z "$(git status --short)"
```

Expected: all commands exit `0`; no branch drift or leftovers.

**Step 5: Create immutable GitHub release only on qualifying evidence**

Attach the verified package, release manifest, benchmark summary, notices, and digest list. Fetch every uploaded artifact back and compare SHA-256 before reporting publication success.

**Step 6: Write final ContextLattice eval ledger/checkpoint**

Record baseline, selected candidate roster, exact holdout metrics, confidence intervals, coverage, cost, latency, memory, tool calls, failures, code/model/data/release digests, PR/release URLs, and reproduction commands. Read it back before completion claim.

---

## External POIDH claim and payout operation

This operation is intentionally outside the repository implementation and does not begin unless Task 29 produces a qualifying release.

1. Re-read live bounty `323`, contract address, open/accepted status, pool, existing claims, and fee from Arbitrum immediately before submission.
2. Identify an authorized claim-submitting wallet and obtain explicit user participation for any signing prompt. Never create, import, expose, or type a private key.
3. Prepare claim title/description with immutable repository, release commit, package/release-manifest digests, exact public benchmark facts, and an explicit statement that the private POIDH result is not yet known.
4. Simulate and inspect chain ID `42161`, contract, method, calldata, value, gas, and claimant before signature/broadcast.
5. Submit once; read back transaction receipt, claim ID, claimant, bounty ID, and public claim content from independent chain/RPC and POIDH page views.
6. Wait for maintainers’ private evaluation. Do not characterize a pending claim as won.
7. If accepted, independently read `pendingWithdrawals[claimant]`, calculate/verify the credited amount, then separately prepare `withdrawTo(user-designated-recipient)`.
8. Before that signature, verify chain, contract, method, recipient, amount semantics, gas, and that no unrelated approval/transfer is included.
9. After mining, independently read the transaction receipt, claimant pending withdrawal, destination balance delta, and POIDH claim status. Only then report deposit success.

## Plan verification checklist

- [ ] Every approved design section maps to one or more tasks.
- [ ] Every production behavior begins with a named RED test.
- [ ] Exact file paths, focused commands, expected RED/GREEN outcomes, and commits are present.
- [ ] Runtime, benchmark, and wallet operations are separated.
- [ ] Threshold `0.65` has one runtime source of truth and exact boundary tests.
- [ ] Browser-path scoring is authoritative; Python is reference-only.
- [ ] Restricted/ambiguous datasets are fail-closed and cannot support a cash-bounty claim.
- [ ] Holdout opening is one-shot and profile-digest locked.
- [ ] No model/data/cache bytes or machine-specific paths enter Git.
- [ ] No claim or release occurs when a quality, privacy, license, parity, or reproducibility gate fails.
