# Gemma 4 E2B / E4B vision investigation

Investigation date: 2026-10-13; updated 2026-10-14. This report distinguishes source inspection,
native macOS execution, iOS simulator execution, and physical iPhone execution.
No physical iPhone was connected to the development environment. The user's
`Failed to modify graph with delegate` message alone does not identify the
failing operator or prove that the cause is out-of-memory.

## Runtime and model provenance

Pocket Gemma 1.0.10 uses the official **LiteRT-LM Swift package 0.18.0**, pinned in
`project.yml`. Its iOS native artifact is `CLiteRTLM.xcframework.zip`, package
checksum `d765b99592d4ec3d0c9e2bd69469454af06c834861340672da1891c0c121c347`.
The published IPA embeds the arm64 iOS CLiteRTLM framework. It does not use
MediaPipe Tasks GenAI, llama.cpp, MLX, or an online AI service.

| Model | Pinned Hugging Face revision | Bytes | SHA-256 |
|---|---|---:|---|
| E2B | b3ca0d2f076785a8f4b2219ddbd2bdb99954eae1 | 2,588,147,712 | 181938105e0eefd105961417e8da75903eacda102c4fce9ce90f50b97139a63c |
| E4B | 28299f30ee4d43294517a4ac93abd6163412f07f | 3,659,530,240 | 0b2a8980ce155fd97673d8e820b4d29d9c7d99b8fa6806f425d969b145bd52e0 |

Source repositories: [E2B](https://huggingface.co/litert-community/gemma-4-E2B-it-litert-lm),
[E4B](https://huggingface.co/litert-community/gemma-4-E4B-it-litert-lm).
The current standard model files have the same sizes and hashes as these pinned
versions. A newer E4B repository revision does not change this standard file.

Bounded HTTP range inspection decoded both **LITERTLM container version 1.5.0**
headers using the official schema. Both contain a 224,137,936-byte vision
encoder, their own vision adapter, end-of-vision model, tokenizer, metadata,
text decoder, embedding sections, and MTP drafter. The vision encoder prefers
FP16 activation; the adapter is constrained to CPU. Encoder graph signatures
are **70 / 140 / 280**, confirmed by native ModelInfo and runtime logs. Metadata
has 2,520 patches and default pooling size 3, giving **280** tokens per image.
No missing vision section was found in these distributed files.

The E2B model card describes mixed 2/4/8-bit weight quantization; it is not a
generic GGUF Q4 model. E4B's card reports approximately 2.24 GB decoder weights
and 0.67 GB memory-mapped embedding parameters. Vision/audio add working memory
on demand. File size is not a measurement of peak working memory. No iPhone
peak-memory measurement was available.

The smaller `*-it-gpu.litertlm` variants were also inspected. They contain a
`tf_lite_artisan_text_decoder` with `gpu_artisan` constraint and **no separate
vision encoder/adapter**. Replacing the standard multimodal models with those
files would not solve this task and was not done.

## Gallery comparison and upstream issues

[Gallery's public repository](https://github.com/google-ai-edge/gallery) has
an iOS model allowlist, but its recursive public tree contains **no Swift iOS
application source**. The public `ios_1_0_0.json` list predates Gemma 4. Therefore
the current App Store iOS app's exact native binary, build flags, model hashes,
and GPU configuration cannot be established from that repository. Claims that
Pocket Gemma exactly matches the current Gallery iOS internals would be unjustified.
This is also confirmed by the maintainer's June 30 comment on
[Gallery #634](https://github.com/google-ai-edge/gallery/issues/634): the repository
contains Android source and has no announced iOS source-release timeline.
The public-source request remains open as checked on October 14.

The public Android helper chooses the main and vision backends independently,
uses a structured image-plus-text message, applies speculative-decoding settings
before engine initialization, and defaults vision acceleration to GPU. Pocket
Gemma also uses the official structured `.imageFile(path)` API; it does not
construct image marker tokens manually. Its importer copies a file before a
photo-provider URL expires, applies ImageIO orientation/EXIF transformation,
caps images at 768 pixels, and writes JPEG. The native iOS regression suite uses
that same importer and production GemmaService, rather than the UI mock replies.

- [Issue #2979](https://github.com/google-ai-edge/LiteRT-LM/issues/2979): July 2026
  E2B CPU-vision/XNNPACK reshape and tensor-allocation failure, same pinned E2B
  bytes/hash. Gallery succeeds on that reporter's device. The reported budget
  variations were conversation settings; they do not prove that an engine-level
  signature limit was applied. The issue is open. Its OCR/classification bypass
  is not adopted here and is not equivalent to Gemma image recognition.
- [Issue #1933](https://github.com/google-ai-edge/LiteRT-LM/issues/1933): v0.10.1
  E4B CPU-vision native null-pointer crash, while E2B works. Open. This does not
  prove the cause of both models failing with a thrown delegate error in v0.18.0.
- [Issue #3017](https://github.com/google-ai-edge/LiteRT-LM/issues/3017): a v0.14.0
  iPhone image-generation example sets engine-level visual budget before
  initialization. It shows that some iOS GPU configurations work; it does not
  certify v0.18.0's iOS binary or Pocket Gemma's exact model/configuration.
- [LiteRT #6745](https://github.com/google-ai-edge/LiteRT/issues/6745): a different
  MediaPipe/prebuilt dylib packaging failure. Pocket Gemma uses CLiteRTLM, so
  copying that workaround or adding an arbitrary dylib is not justified.

[0.18.0 is the latest stable release](https://github.com/google-ai-edge/LiteRT-LM/releases/tag/v0.18.0)
as checked during this investigation. Its release note does not claim a fix for
these iPhone vision issues. Earlier releases added Swift/Metal support, fixed
GPU activation type and vision cache propagation, and later added Metal residency
support. The app was already at 0.18.0; a blind version upgrade was not performed.

## Confirmed application defects and changes

1. **Engine budget omitted:** the app set `ConversationConfig.visualTokenBudget`
   but never `ExperimentalFlags.visualTokenBudget` before `Engine.initialize`.
   SDK 0.18.0 uses the latter to select and allocate encoder/adapter signatures.
   For these files the implicit default is 280, so the omission alone does not
   explain an ordinary 280-token failure. However, the previous retry at 70
   did not actually reduce the initialized encoder capacity.
2. **Incomplete engine reuse identity:** changing budget or speculative-decoding
   allocation could retain an incompatible existing engine. The new
   `InferenceEnginePlan` includes those options, model SHA, backend and context.
   An additional confirmed defect silently forced the main decoder to GPU on
   every image turn even when CPU was selected for working text conversations.
   Main and vision now honor the selected backend; diagnostics can exercise the
   encoder independently. This is not a claim that the physical-device cause has
   been isolated without knowing the user's selected text backend.
3. **Cache configuration shared:** old/new SDK and vision/backend/token settings
   shared one directory per model. The new namespace isolates those settings;
   it ignores old caches while preserving downloaded models and conversations.
   A stale-cache cause on the user's device is not confirmed.
4. **Nonexistent visual signatures exposed:** E2B/E4B offered 560/1120 even though
   the pinned exports contain only 70/140/280. UI and settings normalization now
   use the export's valid options. Other models retain their existing options.
5. **Failure recovery and diagnosis:** native failure invalidates conversation
   reuse and marks the engine for rebuild on the next request, while retaining
   objects through the final native callback. A typed delegate/allocation failure
   can retry once with a real 70-token engine before any reply is delivered.
   Decode/unknown errors and partially delivered replies do not take that retry.
   Native errors remain visible. Bounded local diagnostics add stage/configuration
   and available memory at each stage, and preserve recent failure detail without
   prompts, replies, or image bytes. A stream with no answer is a failure, not a
   successful diagnostic outcome.

Changed production files: `GemmaService.swift`, `ChatStore.swift`,
`InferenceEnginePlan.swift`, `GenerationSettings.swift`, `ModelSpec.swift`,
`GenerationConfigView.swift`, `ModelView.swift`, `InferenceDiagnostics.swift`.
`ChatView.swift` and `ChatComposer.swift` also fix the keyboard-dismissal race
exposed by the unchanged Japanese/emoji UI regression. The accelerator help now
describes the actual shared CPU/GPU choice for text and images.
Tests cover engine identity, the smaller-budget rebuild, supported export limits,
text-only flag reset and error classification. Test/build harness sources are
synthetic and contain no user screenshots, private repository history or credentials.

## Native evidence and outstanding verification

[Native macOS run](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38390725924),
commit `9ab6ee3a200ca3075d663246c8046dd574f36962`, downloaded both complete pinned
models and validated their SHA-256. Actual native inference, not a mock, passed
all ten model/configuration cases:

| Configuration | E2B | E4B |
|---|---|---|
| Original GPU vision configuration, implicit engine limit | Pass | Pass |
| Explicit engine/conversation budget 280 | Pass | Pass |
| Explicit engine/conversation budget 70 | Pass | Pass |
| Main GPU + CPU vision, explicit budget 70 | Pass | Pass |
| Text-only main GPU | Pass | Pass |

Each vision case recognized an unlabelled synthetic red square and green circle,
used both `.imageFile` and `.imageData`, returned an error on invalid JPEG bytes,
then answered another valid image and a text arithmetic question. Native logs
show **WebGPU-based GPU execution** on an Apple paravirtual device, macOS 15.7.9,
7 GiB RAM. This is not a physical iPhone Metal test. Artifact SHA-256:
`4140c31bcd11172dd6214efb3a7db75f17394c464805321a9ae7fce197406fd0`.

The [iOS baseline suite](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38397404827)
executed 14 tests using the real iOS CLiteRTLM library. All were blocked at **main
GPU decoder initialization**, before image decoding/vision execution. Metal
reported `texture binding has argument index 31 that is greater than 30` in
`newComputePipelineStateWithFunction`, propagated via
`llm_litert_compiled_model_executor_factory.cc` as `Failed to modify graph with delegate`.
Changing only vision backend, budget, or FP32 activation did not resolve this
main-decoder failure. This reproduces the generic error on the simulator but
does not prove an iPhone vision-encoder fault. In particular it also blocks GPU
text in this simulator, unlike the user's reported working text conversations.
Artifact SHA: `5cbfa8cb243391501b4a71e8bcf7c6105e27481109b0abb95b0b5244815c4e9f`.

A [follow-up iOS suite](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38401334155)
uses CPU for the main decoder to isolate the CPU/GPU vision paths. **10 tests
passed, zero failed, zero skipped**, on arm64 iOS 26.2 Simulator with the official
iOS native library. Artifact SHA:
`11bdafc80fa4e6dd05a78a7b086520ea36492f47b896bbf5397df60542237799`.

| iOS native configuration/check | E2B | E4B |
|---|---|---|
| CPU main + CPU vision, engine budget 70 | Recognized red square | Recognized red square |
| CPU main + CPU vision, engine budget 280 | Recognized red square | Recognized red square |
| CPU main + GPU vision, engine budget 70 | Recognized red square | Recognized red square |
| Corrected production importer and service, CPU, budget 280 | Recognized red square, text answer 5 | Recognized red square, text answer 5 |
| Production CPU model switch E2B → E4B → E2B | Images and text answer 8 after each switch | Images and text answer 8 after each switch |
| Invalid JPEG error then valid JPEG with production service | Thrown error, then red square, no crash | Not separately exercised |

The production test used the app's own image import, native Message API, engine
and conversation lifecycle. Responses were generated from real verified weights,
not UI preview text. CPU memory use and speed vary by device; the simulator suite
took about 15 minutes to execute, which is not a phone latency benchmark.
The native tests use a 2,048-token context, 128 output tokens and deterministic
sampling (top-k 1, top-p 1, temperature 0); they do not certify every combination
of the app's generation settings or the larger default 4,096-token context.
The tested inference source is commit `2a68417a0ab7719cc78d99f9b6259176d0999282`.
The subsequent keyboard patch changes no inference source.

This provides an application-side compatible CPU route for both models, and
demonstrates that their image encoders are functional in the iOS library. An
upstream GPU remedy would need the Metal kernel generation/fusion to respect
the target's resource-binding limit (or use compatible buffer/argument binding).
App-level image resizing, token reduction or FP32 overrides cannot repair the
main decoder's demonstrated Metal shader error; those overrides failed in the
baseline GPU suite. No upstream binary was patched or version changed blindly.

Simulator success alone must not be reported as physical iPhone success. Native SIGSEGV
inside an upstream library cannot be caught by Swift `do/catch`; the regression
suite distinguishes native crashes from thrown errors.

The direct cause of the user's physical-device delegate failure remains
unconfirmed pending the device's selected text backend and diagnostic detail.
The app's incorrect GPU override is fixed and the resulting CPU image route is
verified automatically. If the device still fails while CPU is selected, or
if text was already working on GPU, that device-level result needs further
investigation; the simulator shader limitation must not be substituted as proof.
Confirmed application bugs have concrete fixes above. No image feature has been
disabled and no OCR substitute has been presented as a completed fix.

## Final candidate verification, October 14

[Final native iOS run, attempt 2](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38432709791)
uses the same source commit as the final IPA:
`4c4a52ae10b46eaaab8f704022076e07535cace3`.
**11 tests passed, zero failed, zero skipped**, with verified complete E2B/E4B
weights, arm64 iOS 26.2 Simulator, official iOS CLiteRTLM 0.18.0. It repeats the
ten successful CPU-main checks and adds E4B invalid-JPEG error/recovery.
Both models now have a tested malformed-image error followed by successful
red-square recognition with the production service, without a process crash.
E4B's recovery case passed in 101.750 seconds; this is a test duration, not a
phone response benchmark. The first attempt never acquired a hosted runner;
GitHub reported an internal server error, so it was rerun without code changes.
Native artifact SHA-256:
`78bd57ad12d45168459d13d7a52f8dbf75f5cf5f5ad0969fccb49b5d9f30dd03`.

[Final IPA build](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38432594875)
passed core checks (40), iPhone UI interactions (17), iPad startup (1), IPA
validation and bounded model-verification memory checks. The downloaded IPA is
1.0.11 (12), unsigned arm64 iPhoneOS, iOS 17 minimum, with the CLiteRTLM framework
and no test app, fixtures, signing profile or model weights. It retains bundle
ID `app.pocketgemma.personal`. IPA bytes: 19,913,510; SHA-256:
`278ada1e5d65e87d61699806cedb14aee3e8b3136eb7e40280cad085a07bcb47`.
Source ZIP bytes: 276,391; SHA-256:
`5a570997191ddb7df1e29a759b0509588ea00dd78773fec9fc3db7dfd93b4ed8`.
The UI log was read back from this exact CI run and confirms 18 passing cases.
The large UI video/image artifact was blocked by the browser download and was
not locally inspected; its digest is recorded from GitHub's artifact metadata.
This limitation is distinct from the downloaded and verified native evidence
and IPA. Physical-device limitations above remain unchanged.

For the verified route in this update, select **CPU** under the selected model's
**model configuration → Accelerator**. No model redownload or chat deletion is
needed. GPU remains selectable; physical-device GPU compatibility is not certified.
If CPU still fails on the phone, the **models and settings → inference diagnostics**
card exports the last stage, device, model hash, backend, budget, and native error
for further diagnosis. Do not treat the simulator's Metal error as that phone's
confirmed failure without its own native diagnostic details.
