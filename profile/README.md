# Ruby's Red Rover 🤖

A sense-and-regulate companion robot designed to detect emotional states and respond with appropriate regulatory actions — think service dog for emotional regulation.

## Overview

Ruby is built around a continuous loop: **sense → understand → score → act**. Sensory inputs (visual, audio, text) are triaged, processed for emotional content using VAD (Valence-Arousal-Dominance) scoring, recorded, and responded to with calibrated actions. Ruby's eyes display the current emotional state in real time, and can switch into a summary mode to give Mom a quick "how was the day" stoplight readout.

## Architecture

```mermaid
flowchart TD
    subgraph ROVER["Ruby's Red Rover — Edge Device (Jetson Orin Nano)"]
        direction TB
        subgraph SENSORS["Sensory Inputs"]
            CAM["Camera (visual)"]
            MIC["Microphone (audio)"]
            EXT["External Text (Mom's phone app)"]
        end

        DAEMON["Triage Daemon / Service (TBD)"]

        subgraph LOCAL_PROC["Local Processing"]
            LOCAL_LLM["Local LLM (text only)"]
            ANIMA["Anima — Emotional Extraction Engine (VAD via NRC-VAD Lexicon)"]
            SQLITE["SQLite DB — emotional state, events, history"]
        end

        subgraph EYES["Eyes Display"]
            EYE_LIVE["Live Mode — rolling emotional state (ambient color)"]
            EYE_SUMMARY["Summary Mode — stoplight for Mom"]
        end

        subgraph ACTIONS["Actions"]
            NOOP["No-op (state recorded only)"]
            PHYSICAL["Physical Action (fetch item / move)"]
            NOTIFY["Notify Mom / Caregiver"]
            WEB["Web Lookup (optional)"]
        end
    end

    subgraph EDGE_APPLIANCE["Personal Edge AI Appliance (Private Cloud / under 500w)"]
        MM["Multimodal Understanding — audio + video processing"]
        VLA["VLA — Vision-Language-Action robot planning"]
    end

    CAM -->|video stream| DAEMON
    MIC -->|audio stream| DAEMON
    EXT -->|text| DAEMON

    DAEMON -->|text| LOCAL_LLM
    DAEMON -->|audio / video| MM

    MM -->|scene description / transcription| ANIMA
    LOCAL_LLM -->|text output| ANIMA

    ANIMA -->|VAD score + event| SQLITE
    ANIMA -->|emotional state| EYE_LIVE
    ANIMA -->|triggers action?| ACTIONS

    SQLITE -->|rolling avg / history| EYE_SUMMARY
    SQLITE -->|context| VLA

    PHYSICAL -->|action request| VLA
    VLA -->|motor commands| ROVER

    EXT -->|Mom: How was your day| EYE_SUMMARY
```

## Key Components

### On-Device (Jetson Orin Nano)
- **Triage Daemon** — receives all sensory input and routes it for processing. Implementation TBD.
- **Local LLM** — lightweight text-only model for on-device inference
- **Anima** — emotional extraction engine, scores text using the NRC-VAD Lexicon for Valence, Arousal, and Dominance
- **SQLite** — local store for all emotional events, states, and history
- **Eyes** — RGB display reflecting live emotional state; switches to stoplight summary mode on demand


### Personal Edge AI Appliance (Private Cloud / <500W)

- **Multimodal Understanding** — handles audio and video processing beyond on-device capability; produces text descriptions fed back into Anima
- **VLA (Vision-Language-Action) Planning** — where a VLM produces text descriptions of a scene, a VLA grounds that understanding into a learned action policy: selecting, sequencing, and timing physical interventions. ACT (Action Chunking with Transformers) handles low-level motor execution on-device; VLA planning runs on the appliance.

A child's emotional data — their facial expressions, voice, behavioral patterns — never leaves the house. Not to a server, not to an API, not to anyone. This is not a technical constraint. It's an ethical one. All inference runs locally, air-gapped, on hardware that fits in a home.

#### Option 1: The Home Base

Ruby lives at home. All processing stays at home. The edge appliance handles everything the Jetson can't: multimodal understanding of audio and video, VLA planning for physical responses.

**The engineering insight:** memory bandwidth is the only hardware spec that matters for LLM token generation. Every token requires reading the full model weights from memory once — more compute doesn't help, only bandwidth does. The RTX 5090 at 1,792 GB/s delivers 195 tokens/sec sustained on Gemma 4 26B A4B (Q4_K_M), **fully matching Google's own hosted Gemini 2.5 Flash throughput**, fully air-gapped, under 500W. For power-constrained deployments, the RTX PRO 4500 Blackwell delivers ~143 tok/sec at 190W; 75% of the throughput at 60% of the power draw, same model, same quality.

Q4_K_M isn't a compromise. For a VLA system generating action commands rather than essays, the quality delta vs Q8 is negligible. It's the correct operating point on the quality/bandwidth/power curve: maximum useful throughput within the mission power budget.

| Metric | Value |
|--------|-------|
| Model | Gemma 4 26B A4B (Q4_K_M) |
| Sustained generation | ~195 tokens/sec |
| Prompt processing | ~3,800 tokens/sec |
| VRAM | 19,214MB / 32,607MB |
| Power (under load) | 319W |
| Multimodal | ✅ Vision encoder |
| Network required | ❌ None |

#### Option 2: Away From Home

Ruby travels with the child. When the home base isn't available — school, travel, grandma's house, the same model runs on a laptop. Same privacy guarantee. Same air gap. Just slower. Works for astronauts and rovers wandering around Mars too.

**The architectural inversion:** Apple Silicon's unified memory removes the constraint that forces quantization on discrete GPUs. 64GB of unified memory means Gemma 4 26B fits at Q8 — 8-bit, essentially lossless with 37GB to spare. The laptop actually runs *higher fidelity weights* than the home base GPU, at lower power, with the same privacy guarantee. It's slower (~25 tok/sec vs 195), but for non-time-critical VLA decisions that's well above the threshold for useful action planning.

One `pip install mlx-lm` away from running. No CUDA. No drivers. No rack.

| Metric | Value |
|--------|-------|
| Model | Gemma 4 26B A4B (Q8) |
| Sustained generation | ~25 tok/sec (est.) |
| VRAM | ~27GB / 64GB unified |
| Power (under load) | ~30-40W (est.) |
| Multimodal | ✅ Vision encoder |
| Network required | ❌ None |
| Quality vs Option 1 | ✅ Higher (Q8 vs Q4_K_M) |

The throughput drops. The quality goes up. The power drops by 8x. That's not a fallback — that's a different point on the same design curve, chosen deliberately for a different mission profile.




## Sense & Regulate Loop

1. Sensory input arrives (camera, mic, or external text from Mom's app)
2. Daemon triages and routes to local LLM (text) or Edge Appliance (audio/video)
3. All paths produce text, which is scored by Anima for VAD
4. Event + state recorded to SQLite
5. Eyes updated with current emotional color
6. Action determined: no-op, physical response, caregiver notification, or web lookup. Physical responses are planned by the VLA (action policy) and executed via ACT motor primitives on-device
7. On "How was your day?" — summary mode activated, eyes display stoplight from day's history

## Status

> 🚧 Early development — hackathon baseline. Architecture and component decisions are in flux.

## Open Questions / TBD

- [ ] Triage Daemon implementation (ROS2? custom Python service? other?)
- [ ] Local LLM model selection for Jetson Orin Nano
- [ ] SAM
- [ ] Mom's app interface design
- [ ] Action library — what physical responses does Ruby support? (defines ACT policy vocabulary)
