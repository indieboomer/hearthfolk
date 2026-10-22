# Hearthfolk — Concept and executable prototype brief

Working title only. Design proposal, 22 October 2026.

## 1. Game promise

A cute, top-down 2D fantasy RPG where you solve people's problems through physical actions, conversation and improvisation, and the inhabitants understand and remember what you actually did.

The presentation borrows the clarity and charm of a small retro JRPG. The interaction design borrows the systemic approach of Project Zomboid: objects have locations, owners, properties and uses. A local language model generates character details, motives, requests and contextual reactions during play. The simulation remains authoritative.

The central attraction is discovering that an improvised solution works and matters socially. Infinite dialogue and infinite quests are not success criteria.

## 2. Player experience and core loop

Explore a small inhabited place → notice a problem → investigate people and objects → choose or invent an approach → perform concrete actions → see physical and social consequences → discover a new opportunity caused by those consequences.

Movement and interaction should be enjoyable without typing. Provide contextual actions and short dialogue suggestions, plus optional free text for conversation. Talking cannot substitute for physically doing something. Saying “I healed the goblin” is a claim, not a healing action.

For the eventual game, progression could unlock tools, practical skills, trusted relationships and access to new places. Combat, character classes and an extensive campaign are deferred until the social and physical loop proves itself.

## 3. First playable scene

One roadside inn, its cellar and a small courtyard. Five NPCs, roughly fifteen useful object instances, one conflict, a 10–15 minute session.

The innkeeper is hiding an injured goblin in the cellar. A travelling apothecary has a medicine bottle but distrusts goblins. A courier also needs that bottle before departing. A caretaker knows where a spare key and some useful supplies are kept.

Roles are fixed for mechanical coverage. At a new run, the local model generates names, personality details, selected relationships and one motive per role, constrained by the scene's supported facts. During play it proposes requests and follow-up goals based on actual events. Do not generate a world map or new mechanics.

| Character | Initial mechanical situation | Initial knowledge |
| --- | --- | --- |
| Innkeeper | Protects the goblin; owns the cellar | Knows the goblin's location and injury |
| Apothecary | Owns the medicine; can trade it | Knows its use; initially does not know about the goblin |
| Courier | Needs medicine for a delivery; leaves after a visible deadline | Knows the delivery obligation |
| Caretaker | Has access to a spare key and courtyard supplies | Knows the cellar entrance; has not seen the goblin |
| Goblin | Injured; can consent to being carried | Knows who helped or harmed them |

Supported approaches include buying or bartering for medicine, stealing it, preparing an alternative treatment from available supplies, or helping the goblin leave. These are reference solutions for testing, not a branching quest script. Refusal and walking away are valid outcomes.

One possible improvised sequence: unlock the rear door, obtain the goblin's consent, carry them into the courtyard, prepare treatment there, return the borrowed key and explain the missing supplies. The model should react to the event history rather than select an authored ending for this exact sequence.

## 4. Minimum systemic vocabulary

Implement inspect, take, drop, give, open/close, unlock, use-on, combine and carry/place. Carrying applies to the injured goblin with consent; it does not require a general physics system. Buy/barter uses a validated transaction. Dialogue can propose an exchange but cannot silently move inventory.

Objects have stable IDs, location/container, owner, tags and limited mutable state. Start with a medicine bottle, coins, key, treatment ingredients, bowl, bandage, water vessel, containers and doors. Choose an exact small inventory during implementation and document it.

Use one explicit treatment recipe; broader crafting is out of scope. Medicine and treatment are consumed by successful use. Healing is an engine state transition. Theft means transferring another owner's property without permission; witnessing and subsequent discovery are separate facts.

NPCs need only a small set of executable activities: move to a reachable location, wait, trade, accept/refuse an offered item, request help and respond to witnessed events. No general-purpose autonomous planning framework in the first build.

Simulation time advances during movement and physical actions, but pauses during dialogue and inference waiting. Show the courier's deadline clearly. Model latency must never make the player miss the deadline.

## 5. Authority and AI responsibilities

| Engine owns | Local model proposes |
| --- | --- |
| Entity IDs, inventory, ownership and locations | Names, personality and motives within role constraints |
| Action preconditions, effects and transactions | Requests whose goals use supported predicates |
| Health, recipes, reachability and time | Contextual dialogue and interpretations of known events |
| Event history, witnesses and NPC knowledge | Bounded social reactions, with supporting event IDs |
| Quest completion and rewards | Follow-up requests caused by the changed situation |

The model cannot execute code, create arbitrary items, invent engine abilities or directly patch state. All proposals pass a schema validator and semantic validator. Invalid references, inaccessible items and unsupported operations are rejected. Reward items must exist and belong to the giver; completion and reward are committed once.

Treat player text as dialogue, never as engine instructions. NPCs may believe or doubt a claim, but beliefs stay separate from canonical facts. The first prototype should keep deception simple: record a claim and respond with uncertainty when it is unsupported.

## 6. Data contracts

Maintain compact structured records rather than an ever-growing transcript:

- World: entities, positions, containers, ownership, health and current world revision.
- Event: ID, simulation time, actor, action, target, outcome and witnesses.
- NPC: immutable ID, role, generated profile, goals, known event IDs and bounded relationship values.
- Request: requester, beneficiaries, supported goal predicates, optional expiry, reserved reward and status.

Example generated request proposal:

```json
{
  "kind": "request_help",
  "speaker_id": "innkeeper",
  "goal": {"predicate": "health_at_least", "entity_id": "goblin", "value": 60},
  "reward": {"item_id": "innkeeper_coin_purse", "amount": 3},
  "reason_event_ids": ["event_004"]
}
```

The validator checks that every ID exists, the predicate is supported, the requester knows the cited event, the reward is available and the goal has at least one reachable supported solution. For this tiny scene, enumerate known action/recipe paths; do not build a general planner. Recheck feasibility when relevant state changes and mark an impossible request blocked or failed rather than pretending it is still achievable.

Quest completion checks the goal state, not an exact action sequence. Quest text is a view of the request record, never its source of truth.

## 7. Knowledge and memory

An NPC receives their profile, their knowledge, recent relevant observations, active commitments and a small set of valid proposals. Do not send the complete hidden world state to every NPC.

Use room membership and visibility for initial witnessing rules. Seeing that an item is missing is not proof of who took it. Being told something creates attributed knowledge, not a retroactive eyewitness event. Keep these distinctions inspectable in a debug panel.

Preserve important facts as structured records. A generated summary can compress dialogue tone but must not overwrite ownership, event evidence or quest state. Save both world state and NPC memory.

## 8. Local inference and responsiveness

Use a configurable local inference adapter. Reuse the user's voxel project's local inference integration if it is supplied and suitable; do not assume access to it. Otherwise implement a small adapter for a configured localhost HTTP endpoint and document the expected request/response format. Keep endpoint, model ID, context limits and timeouts configurable. Use the model profiles and runtime specified in Section 13; treat hardware targets as provisional until measured.

Provide two clearly labelled modes: LIVE LOCAL MODEL and MOCK FOR ENGINE TESTING. Mock mode makes mechanical development reproducible; it does not satisfy the AI acceptance criteria. No silent cloud fallback.

Run inference asynchronously through a bounded queue. Prioritize the player conversation, coalesce background reactions, cancel obsolete requests and revalidate responses against the current world revision before committing them. Never infer every frame or every movement step.

Starting budgets to measure, not promised performance: short replies of 1–3 sentences; one foreground inference at a time; at most one repair attempt for malformed output; an 8-second timeout with a neutral fallback. Target median conversational completion below 3 seconds on the actual test machine, and report p95. A fallback should say the character cannot answer now, without inventing an outcome.

Capture hardware, model, quantization, context size, latency, validation failures, timeout rate and number of calls per session. Test a warm model separately from cold startup. A game that only works on an unsuitable machine has not passed production feasibility.

## 9. Presentation

Top-down 2D with a small readable tile grid, warm colours, rounded silhouettes and a slightly storybook fantasy mood. Cute visuals may contain selfishness, fear and conflicting loyalties; the world should not be uniformly agreeable.

Use consistent placeholder art with clear silhouettes and interaction highlights. A minimal original or properly licensed instrumental loop, footsteps, door and item sounds are enough. Include mute and volume controls. Generated music or a full art pipeline must not delay the first test.

Show inventory, contextual actions, short conversations and a journal of actual commitments. Surface visible consequences through behaviour: a departed courier, a recovered goblin, a refused trade. Avoid relying only on dialogue explanations or reputation numbers.

## 10. Implementation sequence

Build a small proprietary game engine in Rust with a Vulkan renderer through ash, following Section 13. Do not use Godot, Unity, Unreal or a full engine framework. Reuse suitable existing Rust/Vulkan modules if supplied. Keep the custom engine specific to this game: a sprite renderer, simulation, interaction UI, audio and local AI integration. First prove a rendered room and a real model response; do not build a general editor before gameplay.

1. Build the room, controls, inventory and physical actions with deterministic state and event logging. Make it possible to complete the scene using physical mechanics.
2. Add ownership, witnessing, knowledge, the courier deadline, state-based requests and visible consequences. Verify alternative action orders and impossible requests.
3. Connect the real local model immediately. Generate constrained NPC profiles and one valid request; support contextual conversation. Do this before adding more content.
4. Add event-based reactions and one follow-up request. A new request must reference a consequence of the player's session, not merely appear on a timer.
5. Add save/load, reset seed, the debug panel, minimal sound and a compact test report. Stop expanding scope and run the playtest.

Each stage ends in a runnable build. Do not spend the whole effort creating architecture before any scene is playable.

## 11. Acceptance and stop criteria

Mechanical checks:

- Medicine cannot be consumed twice, duplicated through transfer or rewarded twice.
- A trade either transfers both sides or neither side.
- Hidden theft does not instantly identify the thief to uninformed NPCs.
- A stale AI response cannot revive a completed request or spend a missing item.
- Saying an action happened does not make it happen.
- Reloading preserves physical state, commitments and important knowledge.

Playtest checks, using the real local model:

- Buying, stealing, crafting and evacuation produce distinct, coherent outcomes where applicable.
- The player can combine supported actions in an order that was not authored as a solution branch.
- At least one NPC refers accurately to a relevant earlier player action.
- At least one request is generated during play, validated, completed through physical state and followed by a visible consequence.
- Across three different generated setups, names alone are not the only variation: a motive, relationship or request changes the player's decision.
- The player spends meaningful time acting in the world, rather than waiting for text or explaining obvious facts repeatedly.

After a few short sessions, ask: Did I improvise? Did the inhabitants understand? Did that change what I did next? Would I play another scene?

Stop or rethink if the AI mostly produces decorative dialogue, if social results feel arbitrary, if every unexpected action needs a special-case script, or if usable latency requires unacceptable hardware. Do not answer these failures by generating a larger map.

## 12. Codex execution instruction

Implement the first playable prototype described above in the supplied project directory, using the custom Rust/Vulkan engine and local AI stack in Section 13. Inspect existing instructions and code first. Make routine reversible implementation decisions autonomously. Keep changes inside the project and document launch steps.

Prioritize a runnable vertical slice and real local inference over breadth. Start with the deterministic mechanical scene, then connect the local model before adding content. Keep AI output strictly behind validators and preserve explicit per-NPC knowledge. Do not substitute a conversation demo for the playable world.

If no endpoint or model is available, finish the runnable mechanical slice and configurable live adapter, label mock mode clearly, and report the exact configuration needed for live testing. Do not claim AI validation or measured performance without running it.

Deliver source, a runnable build when the environment supports it, launch/configuration instructions, a description of supported actions, and a short report separating verified mechanics, live-model results and untested assumptions. Record meaningful tests of ownership, transactions, knowledge boundaries and stale responses. Include a debug view of AI requests, validation decisions, known events and active goals.

Do not add multiplayer, combat, procedural terrain, voice generation, hundreds of NPCs, a skill tree or an elaborate asset pipeline to this prototype. Finish the small experiment and report whether it demonstrates the game promise.

## 13. Technology specification

### Engine decision

Use Rust for the engine and gameplay, Vulkan through ash for rendering, and llama.cpp as a separate local inference process. Target Windows x64 first; keep Linux portability possible without making it a first milestone. This is a small engine owned by the project, built with libraries rather than a full external game engine. The inference runtime remains an external C/C++ dependency; do not rewrite it in Rust.

Direct Vulkan is chosen for explicit resource control and the requested engine ownership, not because this 2D game requires it. It adds implementation work. Contain that work with one sprite pipeline, one UI pipeline and no general rendering framework. Prefer reusing a working renderer from another project if available.

Proposed library choices below are implementation decisions. Resolve mutually compatible versions at implementation time and commit Cargo.lock; pin the inference runtime separately.

| Layer | Choice | Scope |
| --- | --- | --- |
| Application | Rust, Cargo workspace | One game executable, headless simulation tests |
| Window and input | winit, ash-window | Window, resize, keyboard, mouse, text input |
| Graphics | ash, Vulkan | Batched sprites, tile layers, UI meshes |
| GPU allocation | gpu-allocator | Buffer/image allocation behind the renderer |
| Math and images | glam, image | Coordinates, transforms, PNG loading |
| UI | egui with a pinned compatible ash renderer integration | Inventory, dialogue, journal, debugging; style it consistently |
| Audio | rodio | Music loop, positional approximation for effects, volume |
| Data | serde, serde_json | Definitions, validated proposals, versioned saves |
| Local AI client | reqwest on a dedicated Tokio worker | Loopback HTTP, cancellation, deadlines |
| Diagnostics | tracing | Frame times, AI timing, rejected proposals |

Use ordinary typed Rust structs and stable entity IDs initially. A custom ECS framework is unnecessary for five NPCs. Organize the workspace into engine_render, game_sim, ai_client and game_app crates. Keep game_sim independent of Vulkan and HTTP so it can run tests and recorded sessions headlessly.

### Rendering and simulation

Render an orthographic world with atlas-backed instanced quads, nearest-neighbour sampling, alpha blending and explicit layer/Y sorting. Draw text and UI through a glyph atlas or the chosen UI integration. Use a fixed virtual resolution with integer scaling where possible and a legible UI scale. Package precompiled SPIR-V shaders with the build.

Start with a conservative Vulkan feature set supported by the selected desktop target, conventional render passes and no bindless requirement. Implement swapchain recreation, minimized-window handling, synchronization and deferred resource destruction correctly. Enable validation layers in development and resolve reported lifetime/synchronization errors before playtests.

Run gameplay on a fixed 30 Hz tick and render with interpolation at a capped 60 FPS. Treat these as initial settings. Use tile collision, grid A* navigation and room/line-of-sight checks. No rigid-body physics is needed. Pause simulation time during foreground dialogue/inference, while rendering and UI remain responsive; background AI must not pause the whole world.

All actions enter a command queue. The simulation validates a command, commits its transaction, emits events and updates knowledge and goals. The renderer reads a snapshot. AI workers only receive copied, filtered context and return proposals; they never hold mutable world access.

Save a versioned snapshot with event IDs, request state, RNG state and generated character profiles. Record accepted AI responses for reproducible replay: the same random seed alone does not guarantee identical model output across builds or hardware.

### Selected local model

**Default: Qwen3.5-4B, post-trained checkpoint, GGUF Q4_K_M, text-only input, thinking disabled.** This is the initial candidate for short dialogue, bounded intent selection, NPC profiles and structured request proposals. Its suitability for this game's social reasoning is a hypothesis to validate, not a measured result.

**Quality comparison: Qwen3.5-9B, GGUF Q4_K_M, thinking disabled.** Test it on exactly the same prompts and gameplay situations. Promote it to the default only if its improvement in coherence justifies its memory and latency cost. Do not load both models at once or allocate a model per NPC. All characters share one model with isolated contexts.

The official model cards list Apache 2.0 and describe non-thinking operation. Preserve the applicable license and notices when distributing weights. Use the standard post-trained checkpoints, not Base or modified roleplay variants. The checkpoints support multimodal use, but this game supplies structured text and does not need a vision projector. [T1, T2]

Obtain a GGUF derived from the exact checkpoint, record its source revision and SHA-256, and validate it against the pinned llama.cpp build. Prefer conversion/quantization from official weights when practical; any third-party quantization needs its own recorded provenance. Q4_K_M is the starting format, not a claim that every backend supports every model operation equally well.

### Memory and hardware targets

These are engineering estimates for selecting test machines, not measured minimum requirements or exact download sizes. Quantized weights, runtime buffers, context/state caches, driver allocations and the renderer all consume memory.

| Profile | Approximate weight-file planning range | First hardware target |
| --- | --- | --- |
| 4B Q4_K_M | About 2.5–3.5 GB | 8 GB dedicated VRAM, 16 GB system RAM |
| 9B Q4_K_M | About 5.5–7 GB | 12 GB dedicated VRAM, preferably 32 GB system RAM |
| CPU-only | Same model file, different runtime memory use | Diagnostic fallback; responsiveness unproven |

Aim to keep game rendering allocations below 512 MiB for this scene and leave measured GPU headroom. Start with a 4,096-token inference context and one active slot. Increase to 8,192 only when evaluation shows a need. Keep the normal prompt below roughly 2,500 tokens, leaving room for output and template overhead. Do not allocate the model's advertised maximum context.

Measure actual peak memory with the game and model active together. A 6 GB GPU can be an additional 4B experiment, but do not publish it as supported until tested. If memory is insufficient, show the selected configuration and offer lower context or CPU/offload mode explicitly; never silently switch to cloud inference.

### Runtime integration

Ship or launch a pinned llama-server child process bound only to 127.0.0.1. Use a session-specific authentication token and a configurable free port. The launcher checks readiness, reports loading progress, captures errors and shuts down its own child process on exit. A crash must leave the game usable with a clear AI-unavailable state. Do not require players to install Python, Ollama or a development environment.

Use the llama.cpp Vulkan backend as the cross-vendor candidate. On NVIDIA, compare its CUDA backend and select the measured better configuration; game rendering stays Vulkan regardless. Backend build support is documented by llama.cpp, but the exact Qwen checkpoint, quantization, driver and backend combination still needs an integration test. Pin a working combination rather than following upstream master automatically. [T3]

Use /v1/chat/completions with the pinned server's documented schema-constrained response format and chat-template option enable_thinking=false. Verify that the chosen template actually disables reasoning. Grammar-constrained output improves JSON structure; it does not establish factual correctness. Keep semantic validation in Rust. [T4]

Keep one compact response envelope containing speech, cited known-event IDs and at most one bounded proposal. Restrict IDs and action kinds through request-specific enums where supported. Reject unknown fields. Keep typical dialogue output to 120–180 tokens and a hard cap around 256; generate startup NPC profiles in separate bounded calls. Do not stream unvalidated promises into the UI: show a neutral thinking indicator, validate the complete response, then reveal it.

The engine's structured journal supplies memory. Retrieve recent observations and goal-relevant facts by ID/tag; no vector database, embeddings model or fine-tuning in the prototype. Prioritize foreground conversations, coalesce background updates and cancel requests that become irrelevant. Count the repair attempt inside the total timeout budget.

### Model selection gate

Before expanding the scene, compare 4B and 9B on at least 30 small fixtures covering hidden theft, contradictory player claims, unavailable rewards, stale requests, ambiguous help, remembered assistance and alternate physical solutions. Repeat a subset to reveal instability. Include Polish if Polish dialogue will ship; do not infer its quality from English tests.

Proposed acceptance targets: at least 95% first-pass schema-valid responses, at least 90% coherent/grounded outcomes on human-reviewed fixtures, and zero invalid world mutations accepted by the engine. Report time to first token, complete validated-response latency, p50/p95, cold startup, peak VRAM and rendering frame times during inference. Aim for p50 under 3 seconds and p95 under 8 seconds at the actual output lengths. These are gates to test, not performance promises.

If both models fail social reasoning, improve context and task decomposition before increasing model size. If 9B works but 4B does not, document the hardware tradeoff. If neither meets the combined quality/latency requirement, the live-AI premise has not passed its prototype gate.

### Delivery and build order

First technical checkpoint: a Rust/Vulkan window renders the inn while one real, schema-constrained local model request completes asynchronously. Then implement the physical simulation and remaining milestones from Section 10. This catches graphics/inference compatibility issues before substantial game work.

Deliver a Windows build, source, pinned dependency/runtime manifests, configurable model path, optional explicit model download with progress and checksum verification, launch instructions and benchmark results. Keep large weights separate from source control. The initial downloadable game package can exclude weights, but first-run setup must explain the required download and work offline afterward. Do not claim a tested executable where only source has been produced.

### Technical sources

- [T1 — Official Qwen3.5-4B model card](https://huggingface.co/Qwen/Qwen3.5-4B)
- [T2 — Official Qwen3.5-9B model card](https://huggingface.co/Qwen/Qwen3.5-9B)
- [T3 — llama.cpp backend build documentation](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md)
- [T4 — llama-server API and configuration](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [T5 — ash Vulkan bindings for Rust](https://github.com/ash-rs/ash)

Sources checked on 22 October 2026. Architecture, budgets and acceptance thresholds above are project proposals; no local inference or GPU benchmark was run when preparing this document.
