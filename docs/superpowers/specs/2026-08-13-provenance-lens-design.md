# Provenance Lens: Browser-Local AI Image Detection Design

**Status:** Approved design; implementation not started  
**Date:** 2026-08-13  
**Target:** POIDH Arbitrum bounty 323  
**License:** MIT for project source; compatible third-party artifacts retain their notices

## 1. Objective

Provenance Lens will be a native Manifest V3 Chrome extension that automatically examines eligible images on ordinary web pages and displays a calibrated AI-generation probability for each image it successfully analyzes. Image acquisition, preprocessing, inference, provenance parsing, fusion, and presentation must execute inside Chrome. The installed extension will not upload image bytes, call an inference API, depend on localhost, or require a companion process.

The bounty evaluates balanced accuracy at an exact AI-probability threshold of `0.65`. The qualifying score is at least `75.0%`. This project adopts a stricter internal release gate: it will not submit a claim unless the actual browser pipeline clears the predeclared public proxy-suite gates in section 11.

A public benchmark is evidence about those samples, not a prediction of the private benchmark. The README and claim text must keep that distinction explicit.

## 2. Bounty Contract

The implementation must satisfy all of the following:

| Requirement | Design response |
| --- | --- |
| Native Manifest V3 extension | Manifest V3 service worker, content scripts, offscreen inference document, and extension-local assets |
| Browser-only inference | ONNX Runtime Web with WebGPU first and WASM fallback; no runtime Python, Node.js, or server |
| Automatic page analysis | Mutation-aware discovery of eligible page images, viewport-prioritized scheduling, and badges without a manual upload step |
| Confidence for every analyzed image | Every successful result renders the same calibrated `P(AI)` used for the `0.65` decision |
| Offline after setup | Pinned model artifacts are fetched by the reproducible preparation step and packaged into `dist/`; the installed runtime performs no model download |
| No external inference or image upload | The only permitted runtime network acquisition is fetching the exact source bytes of an image already displayed by the page; no bytes leave the browser for analysis |
| Reproducible source | Lockfiles, model artifact lock, digests, preprocessing contracts, deterministic build manifest, and reference/browser parity tests |
| MIT source license | Project-authored source is MIT; only license-compatible third-party runtime components may ship, with complete notices |
| Private benchmark integrity | No benchmark hashes, labels, paths, lookup tables, or source-specific shortcuts in the runtime bundle |

POIDH currently uses a winner-take-all process. Existing submissions and self-reported scores are competition context, not trusted validation and not implementation inputs.

## 3. Scope

### 3.1 In scope

The first release will analyze static images represented by `<img>`, `<picture>`, and CSS `background-image` URLs on ordinary `http` and `https` pages. It will observe DOM insertions, source changes, responsive `currentSrc` changes, lazy loading, and single-page application navigation. A page-level summary, per-image evidence panel, privacy disclosure, setup/readiness view, and optional blur control are included.

The inference path will support one or more learned visual detectors, authenticated provenance signals, conservative generator metadata, and an optional independently trained residual/frequency head. A candidate is shipped only when it improves frozen grouped holdouts through the same browser preprocessing path.

### 3.2 Out of scope

The first release will not analyze video frames, arbitrary `<canvas>` contents, PDFs rendered by Chrome, browser-internal pages, DRM-protected media, or inaccessible images in inactive tabs. It will not identify which generator created an image unless authenticated provenance explicitly states that fact. It will not present the score as proof, use it for consequential automated decisions, or claim universal coverage of future generators.

Training and benchmark tools may use Python or Node.js during development. Those tools and processes are never runtime dependencies of the installed extension.

## 4. Design Direction

### 4.1 Product read

This is a compact forensic browser utility for privacy-conscious everyday users. The visual language should feel quiet, inspectable, and evidence-first rather than promotional or futuristic.

The UI uses a hand-built extension track: semantic HTML, TypeScript, and scoped CSS instead of a component framework. The popup and badges are too small to justify a general UI dependency, and fewer dependencies reduce bundle size, CSP complexity, and audit surface.

The style dials are:

| Dial | Value | Consequence |
| --- | ---: | --- |
| Layout variance | 3/10 | Stable utility layouts and predictable badge placement |
| Motion intensity | 2/10 | Short opacity/transform feedback only; no decorative loops |
| Information density | 7/10 | Scores and evidence remain visible without marketing filler |

The palette uses warm off-white and near-black surfaces with one restrained vermilion accent for scores at or above the decision threshold. Below-threshold and unavailable states remain neutral, so status is not encoded by red/green color alone. Corner radii follow one small-radius scale. Typography uses system UI for controls and a system monospace stack only for digests, model IDs, and measurements.

### 4.2 Per-image presentation

A successful analysis always displays `AI nn%`. The number is the final calibrated probability consumed by the decision rule. The UI does not invert low scores into an unsupported claim such as “Real 97%.”

The badge has four explicit states:

1. `Scanning` while queued or running.
2. `AI nn%` with the accent treatment when `P(AI) >= 0.65`.
3. `AI nn%` with a neutral treatment when `P(AI) < 0.65`.
4. `Not scored` with a reason when acquisition or inference fails.

The expanded evidence panel reports the visual-model score, authenticated provenance result, weak metadata evidence, acquisition mode, model profile, and limitations. It never exposes page-controlled strings through unsanitized HTML.

The default action is labeling, not blurring. Users may enable automatic blur for scores at or above `0.65`; reveal and reapply controls are keyboard accessible. A reduced-motion preference removes transitions. Every state has an accessible name, visible focus, and WCAG AA contrast.

## 5. System Architecture

```text
page DOM
   |
   v
content script: discover -> observe -> measure -> badge
   |
   v
service worker: validate -> deduplicate -> prioritize -> cache
   |
   v
offscreen document + worker
   |-- acquire exact image bytes or visible screenshot crop
   |-- decode and inspect metadata/provenance
   |-- preprocess per model contract
   |-- WebGPU inference, then WASM fallback
   |-- calibrated fusion
   |
   v
service worker cache -> content-script result -> popup summary
```

### 5.1 Content script

The content script discovers eligible visual elements, tracks them in a `WeakMap`, and watches DOM/source changes with a `MutationObserver`. `IntersectionObserver` prioritizes currently visible images; it does not define eligibility. An eligible image must have decoded dimensions of at least `96 x 96` and a rendered box of at least `64 x 64`. Icons, sprites, transparent tracking pixels, and rapidly detached nodes are skipped and counted by reason in the popup.

Every eligible image eventually enters the queue unless it leaves the document before decoding. The discovery key includes the final `currentSrc`, intrinsic dimensions, and a monotonically increasing element generation so reused DOM nodes cannot inherit stale results.

Badges render in a closed Shadow DOM root attached to a fixed overlay layer. Position updates are batched through `requestAnimationFrame`; no unbounded scroll listener performs layout work. The overlay is removed from screenshot capture and restored afterward.

### 5.2 Service worker

The service worker is a small coordinator. It validates typed messages, confirms the sender tab and frame, creates the offscreen document on demand, coalesces duplicate analyses, manages cancellation, and maintains a bounded result cache. It does not host model sessions because service-worker teardown and DOM/WebGPU constraints make that lifecycle unreliable.

Queue priority is visible main-frame images, visible subframe images, near-viewport images, then remaining eligible images. A per-tab and global cap prevents infinite feeds from exhausting memory. Backpressure pauses discovery rather than dropping an image silently; the UI can report queued counts.

The cache key is `SHA-256(image bytes) + model-profile digest + preprocessing version`. URLs are not sufficient because the same URL can change. Persistent cache entries contain scores and compact evidence, never image bytes. Raw bytes and screenshot crops are released after inference.

### 5.3 Offscreen inference runtime

A single offscreen document owns the inference worker and ONNX Runtime Web sessions. It loads models from `chrome.runtime.getURL(...)`, checks their declared digests during readiness, and starts WebGPU where supported. The same graph and preprocessing contract run under WASM fallback. Backend parity is verified before release.

Inference is sequential by default to bound GPU memory. The worker may keep the baseline and one complementary model resident only when measured memory remains within the release budget. Additional model sessions load lazily and are evicted using measured memory pressure rather than arbitrary timeouts.

The extension CSP permits extension-local workers and WebAssembly compilation only. It does not permit remote scripts, dynamic remote modules, or broad `connect-src` destinations.

### 5.4 Popup and setup surfaces

The popup shows the active page’s analyzed, likely-AI, below-threshold, queued, skipped, and failed counts. It exposes pause/resume, optional blur, cache clearing, and a link to the local readiness report. It does not initiate analysis as a prerequisite; ordinary-page analysis starts automatically.

The setup page verifies model presence, model digests, backend availability, one known-answer inference vector, and extension policy. It gives actionable errors without presenting a green-ready state when a model or backend is unavailable.

## 6. Image Acquisition

The acquisition sequence is deterministic and records the selected mode in the result.

### 6.1 Exact source bytes

The preferred path sends the displayed `currentSrc` or background URL to the offscreen runtime. Extension host permissions allow it to fetch the same asset that the page displays without routing through an analysis service. Redirects are bounded, response types must be supported images, compressed bytes are capped, decoded dimensions are capped, and credentials are not forwarded to unrelated origins.

`data:` and accessible `blob:` images are transferred locally from the content script. The transfer has a strict byte cap and no persistence.

### 6.2 Screenshot-crop fallback

If exact bytes cannot be obtained but the image is visible in the active tab, the service worker may call `captureVisibleTab`. The content script supplies a device-pixel-ratio-aware crop rectangle after temporarily hiding Provenance Lens overlays. The crop is performed locally in the offscreen runtime and destroyed after scoring.

Screenshot fallback is not used for occluded, transformed beyond reliable geometry, mostly offscreen, or inactive-tab images. Its evidence panel says `visible screenshot crop`, because overlays, browser resampling, and page composition can change the detector response.

### 6.3 Unsupported acquisition

If both paths fail, the extension reports `Not scored` and a bounded reason such as `source unavailable`, `unsupported format`, `image too large`, or `not visible for fallback`. A failed acquisition is never converted to `AI 0%`.

Animated formats use the first decodable frame for the initial release and disclose that limitation. Orientation and color-space handling must match the reference pipeline.

## 7. Detection Pipeline

### 7.1 Learned baseline

The initial benchmark baseline is the corrected Community Forensics ViT-S model published under MIT at immutable model revision `ac6ee457bea904a373065754107451793b56db00`. The candidate is trained across 4,803 generators and has a corrected single-logit ONNX export. The implementation must use the current 384-pixel preprocessing contract: preserve aspect ratio while resizing the shortest edge to 440, center crop to `384 x 384`, convert to RGB, and apply the published CLIP normalization.

This revision is a candidate, not an automatic production choice. Earlier artifacts had incorrect weights, attention-head configuration, and preprocessing metadata. The preparation tool therefore pins both revision and artifact digest; filenames or mutable branch names are insufficient.

The FP32 ONNX graph is the reference candidate because the publisher warns that dynamic INT8 and Q4 variants can diverge substantially on out-of-distribution generators. Quantized graphs may ship only after browser-path holdout and parity evidence shows no unacceptable loss.

### 7.2 Complementary learned signals

A second model or a compact residual/frequency head is considered only if it adds out-of-family information after grouped evaluation. Candidate selection follows these rules:

- the artifact and training provenance are inspectable;
- its license is compatible with redistribution in an MIT project;
- its source families do not merely duplicate the baseline’s training distribution;
- its browser preprocessing has reference parity;
- adding it improves every predeclared core suite or improves the worst suite without reducing any core suite below the release bar;
- the latency and memory increase stays within the measured runtime budget.

Self-reported competitor scores do not satisfy these gates. Competitor repositories are comparison evidence only; their code, weights, calibration constants, and benchmark assets are not imported merely because a claim reports a high score.

A clean-room residual/frequency head may be trained from publicly documented datasets if an orthogonal-error analysis justifies it. It is rejected if it acts as a camera/JPEG shortcut, fails source-family holdouts, or loses accuracy after web transformations.

### 7.3 Provenance and metadata

Authenticated C2PA/JUMBF assertions that explicitly classify the asset as trained algorithmic media are strong positive evidence only after local signature and manifest validation. Generic C2PA presence is not AI evidence.

Unsigned PNG text, JPEG EXIF/XMP, WebP metadata, or software tags that explicitly name a generative workflow are weak positive evidence. They can increase `P(AI)` only through calibration data that includes false-marker and stripped-metadata controls. Their absence never reduces the learned visual probability; camera metadata is not treated as proof of authenticity.

Watermark detectors are included only when the public decoding algorithm and validation data are available. A placeholder “SynthID compatible” detector without provider-specific validation is not shipped.

### 7.4 Preprocessing and test-time views

Browser decoding and resizing must match a Python reference within a frozen tolerance. If Canvas or `createImageBitmap` resampling exceeds that tolerance, the worker implements the required bilinear kernel directly. EXIF orientation, alpha compositing, color profiles, and integer rounding are covered by parity fixtures.

The baseline view is the published center crop. Additional crops or scales are allowed only for images near the decision boundary and only if a frozen calibration includes that exact policy. Test-time augmentation aggregation is fixed in the model profile and cannot change by website or suspected label.

### 7.5 Calibrated fusion

Each visual model emits a raw logit or probability. The fusion layer consumes bounded logits, explicit provenance features, acquisition mode, and calibrated missing-signal flags. A regularized logistic stacker is preferred because its weights and intercept are inspectable. A more complex stacker must show a material grouped-holdout gain to justify itself.

Calibration is fit only on the calibration role using log loss and reliability diagnostics. Model selection uses development roles. Final holdouts remain unopened until the model roster, preprocessing, fusion, and decision rule are frozen. The extension displays the resulting `P(AI)` and applies exactly:

```text
likely AI-generated if P(AI) >= 0.65
below threshold otherwise
```

No site-specific threshold, hidden override, label lookup, or post-holdout intercept adjustment is permitted. If a reduced single-model fallback profile is supported, it receives its own predeclared calibration and release evidence; missing ensemble members cannot silently reuse the full-profile score.

## 8. Reproducibility and Artifact Provenance

The repository will use Node.js 22+, TypeScript, esbuild, ONNX Runtime Web, and Playwright. `npm ci` is the package-install contract. Python 3.12 tooling may prepare or evaluate model artifacts but is not required when loading the completed extension.

`models/model-lock.json` records, for every runtime artifact:

- upstream repository and immutable revision;
- exact download path and byte length;
- SHA-256 digest;
- upstream and redistribution license;
- architecture, input shape, and output interpretation;
- preprocessing and postprocessing contract versions;
- generated artifact digest when conversion or training is involved.

`npm run models:fetch` downloads and verifies locked artifacts into a cache. `npm run build` refuses missing or mismatched digests and packages only the selected release profile into `dist/`. A build manifest records every emitted file and SHA-256 digest. `npm run verify:release` performs policy, license, model, test, build, and package checks from a clean checkout.

If a project-trained head ships, its training manifest records source datasets, source licenses, immutable input manifests, group assignments, transformations, seed, environment lock, code commit, training metrics, and final checkpoint digest. Bulk datasets and generated caches stay outside Git; only compact manifests and aggregate reports are committed.

No private machine path, credential, benchmark image, benchmark label, or wallet key may appear in a commit, build artifact, log, or claim.

## 9. Repository Boundaries

```text
provenance-lens/
├── src/
│   ├── background/       service worker, queue, cache, offscreen lifecycle
│   ├── content/          discovery, overlay, evidence interaction
│   ├── inference/        model profiles, preprocessing, fusion, provenance
│   ├── offscreen/        stable inference document and worker
│   ├── popup/            current-page summary and controls
│   ├── options/          readiness, privacy, accessibility, settings
│   └── shared/           typed message schemas and constants
├── models/
│   ├── model-lock.json   immutable runtime artifact inventory
│   └── profiles/         preprocessing and fusion profiles
├── tools/
│   ├── models/           fetch, digest, convert, and parity tools
│   ├── benchmark/        manifests, split audit, scoring, stress transforms
│   └── release/          policy, license, bundle, and egress checks
├── tests/
│   ├── unit/
│   ├── parity/
│   ├── policy/
│   └── e2e/
├── benchmarks/           compact manifests and aggregate reports only
├── docs/
│   ├── MODEL_CARD.md
│   ├── BENCHMARK.md
│   ├── PRIVACY.md
│   ├── SECURITY.md
│   └── superpowers/specs/
├── third-party-licenses/
├── manifest.json
├── package-lock.json
└── LICENSE
```

Modules communicate through versioned, validated message contracts. Inference code does not manipulate the DOM. Content code does not load models. Benchmark code does not enter the extension bundle.

## 10. Benchmark Protocol

### 10.1 Data provenance and labels

Every sample has a manifest row containing stable sample ID, source dataset, source URL or dataset revision, upstream license, source family, generator family when applicable, original asset group, label provenance, byte digest, perceptual digest, dimensions, and role. Ambiguous or mixed-edit images are assigned to a separate diagnostic slice rather than silently forced into the primary binary score.

Datasets must permit the intended evaluation or training use. Dataset images are not republished in the repository unless their licenses explicitly permit it.

### 10.2 Leakage controls

Splits occur by original asset group, source collection, and generator family before transformations. Exact SHA-256 duplicates are removed. Perceptual duplicates are audited with pHash and a second image-similarity check; suspected cross-role pairs are resolved before any final evaluation.

The four immutable roles are:

1. `train`, used only for project-trained parameters;
2. `calibration`, used only to fit probability mapping and fusion;
3. `development`, used for candidate and policy selection;
4. `holdout`, opened once after the release candidate is frozen.

Transformations inherit the role of their original image. A screenshot, crop, or recompression of a training image cannot appear in calibration, development, or holdout.

### 10.3 Public proxy suites

At least two independently sourced core suites will be assembled. Each contains real and AI images, multiple real-source families, and modern generator families not all represented in model training. A third web-wild suite samples paired or source-verifiable ordinary web imagery without tuning against its labels.

Stress variants cover JPEG quality changes, WebP conversion, downscale/upscale, center and off-center crop, mild blur, contrast, screenshot capture, metadata removal, metadata injection controls, and social-style recompression. Clean and stress metrics are reported separately and pooled only through a predeclared weighting.

### 10.4 Metrics

The authoritative quality metric is balanced accuracy at `P(AI) >= 0.65`:

```text
TPR = true AI predictions / AI samples
TNR = true below-threshold predictions / real samples
balanced accuracy = (TPR + TNR) / 2
```

Reports also include confusion matrices, per-suite TPR/TNR, source-family recall, generator-family recall, bootstrap 95% confidence intervals, Brier score, expected calibration error, abstention/failure rate, and clean-versus-stress deltas. Runtime reports include p50/p95 acquisition, preprocessing, per-model, and end-to-end latency plus peak memory.

A failure to acquire or infer is reported as coverage loss, not removed from the denominator without disclosure. The bounty metric may score only successful eligible images; the project additionally reports end-to-end coverage so accuracy cannot be improved by refusing difficult samples.

### 10.5 Evaluation ledger

Every candidate run writes a machine-readable ledger row with baseline, candidate identity, exact code and model digests, data-manifest digest, split roles, metric definitions, holdout status, sample counts, latency, memory, tool calls, wall time, failures, and reproduction artifact. Comparisons are made only between runs sharing a compatible manifest and evaluation contract.

Once a holdout is opened, a code or calibration change retires that holdout for model selection. It may remain as labeled historical evidence but cannot be repeatedly optimized against and described as untouched.

## 11. Release Gates

A claim candidate must pass every gate below from a clean checkout. Failure of any gate blocks claim submission.

### 11.1 Quality

- Fixed threshold is exactly `0.65` in the extension and evaluator.
- Balanced accuracy is at least `80.0%` on each predeclared core public proxy suite through the real browser path.
- Balanced accuracy is at least `80.0%` on each predeclared clean/stress paired slice with enough real and AI samples to define the metric.
- The stratified bootstrap lower confidence bound on pooled balanced accuracy is at least `75.0%`.
- No predeclared source family falls below `70.0%` recall unless the limitation is declared and the aggregate release gate still passes.
- End-to-end eligible-image scoring coverage is at least `98.0%` on the browser fixtures; all failures are enumerated.

### 11.2 Browser parity and performance

- Reference and Chrome preprocessing tensors remain within the frozen numeric tolerance for all parity fixtures.
- Reference and browser probabilities differ by no more than `0.01` on parity samples, with `100%` agreement at the `0.65` decision boundary.
- WebGPU and WASM profiles agree at the decision boundary on the parity set.
- On the documented reference machine, visible-image p50 end-to-end latency is at most `2.0 s` with WebGPU and at most `6.0 s` with WASM; p95 and memory are published rather than hidden.
- Infinite-feed testing shows bounded queue, cache, DOM observers, and model memory.

### 11.3 Policy, privacy, and reproducibility

- `npm ci && npm run verify:release` passes from a clean clone.
- Two consecutive builds from the same inputs produce matching release-manifest digests, excluding explicitly documented archive timestamps.
- A clean Chrome profile loads the unpacked build and automatically scores eligible static and dynamically inserted images.
- With network disabled after build/install, cached/local fixture pages still score successfully.
- Runtime egress capture observes no telemetry, inference API, localhost call, remote code, or model download. Exact displayed-image fetches are identified separately.
- The built extension contains no benchmark labels, hashes, paths, lookup tables, credentials, private paths, or remote executable code.
- All model and dependency licenses are accounted for and compatible with the release.
- Accessibility checks cover keyboard operation, focus, contrast, zoom, reduced motion, and screen-reader names.

### 11.4 Submission integrity

The public README reports exact commit and release-manifest digests, complete build/install steps, benchmark manifests, confusion matrices, limitations, and the distinction between public evidence and the private bounty outcome. The private benchmark outcome remains external and unproven until the maintainers publish it. A POIDH claim links to the public repository and immutable release commit. It does not claim the private benchmark has been passed before the maintainers report that result.

## 12. Failure Handling

A model digest mismatch blocks readiness and build. A WebGPU initialization error triggers the prevalidated WASM backend. A full-profile model failure may use only a separately calibrated and released baseline profile; otherwise the image is `Not scored`.

Acquisition, decode, preprocessing, inference, and UI errors use typed reason codes. User-facing messages remain brief, while local diagnostic logs contain model/profile IDs and timings without image bytes or browsing history. Logs are bounded and disabled by default outside a user-initiated diagnostics export.

Model session crashes are isolated from page scripts. The service worker restarts the offscreen document once for a reproducible transient failure, then opens the circuit and reports a readiness error rather than looping.

## 13. Security and Privacy

The manifest follows least privilege consistent with automatic all-site image analysis. Content scripts match only ordinary web schemes. Host access is used to read assets already displayed by the page and to support visible-tab fallback; it is not used for general browsing-history collection.

Every message is schema-validated and size-bounded. Redirect count, compressed bytes, decoded pixels, MIME type, decode time, and inference time are limited. SVG is rasterized in a constrained image decoder and never injected as page markup. Page strings are rendered through text nodes.

The build rejects remote script URLs, dynamic code evaluation beyond required WebAssembly support, unexpected network literals, localhost endpoints, and undeclared model assets. Runtime request interception is the final proof because source scans alone cannot prove network behavior.

Settings and compact score caches stay in `chrome.storage.local` or Cache Storage. Image bytes are not persisted. Clearing extension data removes all settings and caches. There is no analytics identifier, account, telemetry, or ad integration.

## 14. Testing Strategy

Unit tests cover typed messaging, eligibility, queue ordering, cache identity, score fusion, threshold boundary behavior, metadata parsers, provenance validation, and error mapping. Property tests exercise malformed image metadata, oversized messages, redirect loops, and numerical edge cases.

Parity tests compare browser preprocessing, model logits, and calibrated probabilities against the pinned reference implementation. Known-answer vectors include orientation, alpha, color profile, odd aspect ratios, low resolution, and each supported format.

End-to-end Playwright tests launch a persistent clean Chromium profile with the unpacked extension. They verify static, lazy, dynamic, responsive, background, cross-origin, data, blob, inaccessible, and screenshot-fallback cases; badge positioning; optional blur; popup counts; service-worker restart; WebGPU/WASM selection; and offline operation. Network interception fails the test on undeclared egress.

Release tests run the frozen public benchmark through the built extension rather than a Python-only approximation. Faster native inference may be used during candidate exploration, but it is never authoritative for release claims.

## 15. Delivery Sequence

1. Build the data manifest, split audit, and baseline reference evaluator before optimizing a detector.
2. Implement the exact browser preprocessing and prove reference parity.
3. Build the minimal MV3 discovery-to-badge path around the single-model baseline.
4. Measure the baseline on development suites and record the evaluation ledger.
5. Test complementary candidates one at a time; retain only measured improvements.
6. Freeze fusion, calibration, model profile, and threshold before opening final holdouts.
7. Run clean/stress browser holdouts and all release gates.
8. Publish the immutable release artifacts through reviewed pull requests.
9. Submit a POIDH claim only if every gate passes.
10. If the claim is accepted on-chain, handle withdrawal as a separate reviewed wallet operation; no key or signing authority belongs in this repository.

## 16. Rejected Alternatives

A single off-the-shelf model is faster to ship but exposes the project to generator-family and preprocessing shift. It remains the baseline and emergency reduced profile, not the assumed final system.

Training a large detector from scratch would delay submission and duplicate established broad-generator work. Project training is limited to a compact complementary head or evidence-backed fine-tune whose incremental value can be measured.

A local Python/Flask service would simplify inference but directly violates the runtime contract. Cloud APIs, remote code, and telemetry are also excluded regardless of accuracy.

A metadata-only detector fails when platforms strip metadata and is trivial to spoof. Metadata therefore supplies positive auxiliary evidence rather than the primary verdict.

A hidden threshold remap tuned after viewing holdouts could inflate a reported number while destroying calibration and benchmark integrity. The `0.65` rule, calibration roles, and one-shot holdouts prevent that behavior.

## 17. Completion Definition

The software task is complete only when a synced public `main` branch contains the MIT source, pinned reproducible model preparation, deterministic extension build, complete notices, browser-path benchmarks, clean-profile and offline evidence, and all release gates passing on exact published bytes.

Winning the bounty is a separate external outcome. It is complete only when POIDH maintainers report a qualifying private result, the claim is accepted on Arbitrum, the net credit is visible in the claimant’s `pendingWithdrawals`, and an authorized `withdrawTo` transaction to the user-designated recipient is mined and independently read back.
