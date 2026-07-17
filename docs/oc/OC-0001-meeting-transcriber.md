# Opportunity Canvas: Homelab Meeting Transcriber

## Problem

Meetings produce valuable information — decisions, action items, context — that is lost or poorly retained when no transcript exists. Existing transcription services (Otter.ai, Fireflies.ai) solve this but require sending audio to third-party cloud infrastructure, surrendering privacy. Microsoft Teams' built-in transcription depends on tenant admin configuration and M365 license tier, making it unreliable for personal or cross-tenant use.

There is no self-hosted, privacy-preserving meeting transcription solution that integrates naturally into a homelab automation stack.

---

## Who It's For

Personal use. Meetings attended from a home or remote context where audio is available locally. No dependency on corporate IT, tenant admin consent, or third-party services.

---

## Value

- Full transcript of every meeting, stored privately in NocoDB alongside structured metadata
- No audio or content leaves the homelab
- NPU handles transcription at under 2W, leaving GPU inference endpoints free for the news intelligence pipeline
- Transcript available for downstream processing — summarization via existing LLM endpoints, action item extraction, keyword search
- Minimal operational friction: one manual click to join, everything else automated

---

## Solution

A meeting participant bot built on existing homelab infrastructure, with no Azure bot registration or Real-time Media Platform SDK required.

### How it works

A dedicated M365 family member account runs Teams Web inside the existing Kasm Firefox session on CoreNAS (N5Pro). The account joins meetings as a silent participant (camera and mic off). Audio plays into Kasm's PulseAudio sink on the host.

A sidecar script on the N5Pro host captures audio from the known Kasm PulseAudio sink, chunks it every N seconds, and POSTs each chunk to a FastFlowLM Whisper endpoint served by the NPU. Transcribed text is returned and appended to a NocoDB record in real time.

The operator creates a NocoDB record at meeting start (providing the title), which triggers a single n8n workflow via NocoDB webhook. The workflow starts the sidecar capture process, passing the record ID for output routing. The sidecar self-terminates on sustained silence detection and signals n8n to finalize the record.

### Components

| Component | Role | Status |
|---|---|---|
| Kasm Firefox (N5Pro) | Teams Web session, audio source | Running |
| M365 family account | Teams participant identity | To create |
| PulseAudio host sink | Audio capture point | Existing (Kasm) |
| Sidecar capture script | Chunk audio, POST to Whisper | To build |
| FastFlowLM + NPU | Whisper ASR inference | Pending TrueNAS 26 + NPU setup |
| NocoDB | Transcript storage, workflow trigger | Pending deployment |
| n8n (single workflow) | Orchestration, start/stop, record update | Pending; `hooks.{{MY_DOMAIN}}` needed |

---

## Key Assumptions to Validate

- Kasm's PulseAudio sink is consistently accessible from the N5Pro host without container surgery
- Teams Web (personal M365 account) can join organizational tenant meetings as an external guest without restrictions
- FastFlowLM Whisper on XDNA2 NPU is accessible from TrueNAS 26 LXC or VM with `/dev/accel/accel0` passthrough
- NocoDB webhook reliably triggers n8n on row creation
- Silence detection (N consecutive empty Whisper responses) is a sufficient meeting-end signal

---

## What This Is Not

- A real-time speaker diarization system (who said what) — Whisper base does not separate speakers
- A replacement for structured meeting notes — transcript is raw, downstream LLM summarization is a separate pipeline step
- Usable for meetings you are not attending yourself — requires manual join in Kasm
- Dependent on `hooks.{{MY_DOMAIN}}` for internal triggering — NocoDB and n8n are co-located on N5Pro, internal routing is sufficient until the subdomain is in place

---

## Dependencies and Sequencing

This feature cannot be fully delivered until the following are in place:

1. **TrueNAS 26** — kernel 6.18 required for in-tree amdxdna driver
2. **NPU setup** — FastFlowLM + Whisper endpoint running on N5Pro
3. **NocoDB deployment** — record store and webhook source
4. **n8n `hooks.{{MY_DOMAIN}}`** — or internal endpoint as interim
5. **M365 family account** — low effort, can be done anytime

The sidecar script and n8n workflow can be designed and tested (with a stub Whisper endpoint) before NPU is ready.

---

## Open Questions

- Should transcript chunks append to a single NocoDB text column, or be stored as individual rows linked to the meeting record?
- Should the n8n workflow trigger a post-meeting summarization pass via the 890M iGPU LLM endpoint automatically, or on demand?
- What is the target chunk duration for the audio capture — shorter (5–10s) gives lower latency transcript updates, longer (20–30s) gives Whisper more context and better accuracy?
