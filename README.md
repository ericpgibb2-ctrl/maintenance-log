# Maintenance Log — Gemma 4 on-device Skill

A field-maintenance Skill for the [Google AI Edge Gallery](https://github.com/google-ai-edge/gallery) app. Technicians describe what they see (optionally with a photo), and the Gemma 4 model running locally on the phone turns the observation into a structured JSONL record that stays on the device until the technician exports it.

**Proves the "arm brain" pattern** — local intelligence captures sensory input and produces organization-ready structured memory, entirely offline.

## What's in this folder

```
maintenance-log/
├── SKILL.md                    ← manifest + model instructions (tool schema, field rules)
├── scripts/
│   └── index.html              ← hidden runner: handles run_js actions, writes to localStorage
└── assets/
    └── dashboard.html          ← inline webview: list, inline edit, copy JSONL, wipe
```

## Install

### On the phone

1. Install **Google AI Edge Gallery** from the [App Store](https://apps.apple.com/us/app/google-ai-edge-gallery/id6749645337) (iOS 17+) or [Play Store](https://play.google.com/store/apps/details?id=com.google.ai.edge.gallery) (Android 12+).
2. Open the app, go to **Model Management**, download **Gemma 4 E4B** (about 5 GB on 4-bit quant). Wait for the download.
3. Enter **Agent Skills** with Gemma 4 E4B selected.

### Load this Skill

Three options — pick based on what's available.

**A. Load from URL (recommended, works on iOS and Android).** Host the `maintenance-log/` folder on GitHub Pages (remember the `.nojekyll` file) or Cloudflare Pages. Then in the Gallery: *Skills* → `+` → *Load skill from URL* → paste the URL pointing at the folder (not at `SKILL.md`). Verify the URL first by opening `<url>/SKILL.md` in a browser — it should show raw markdown.

**B. Import from local file (Android only).** Push the folder to the device, then import via the Skill Manager file picker:
```bash
adb push maintenance-log/ /sdcard/Download/
```

**C. Submit as a community skill.** Open a GitHub Discussion under [gallery/discussions/categories/skills](https://github.com/google-ai-edge/gallery/discussions/categories/skills) so others can install it via *Add from featured list*.

## How to use

With the Skill enabled, talk to the chat the way a technician would:

- *"Third floor mechanical room, pipe joint under sink is dripping, small puddle forming."* (optionally attach a photo)
- *"Routine inspection — Grundfos CM 3-4 reads 4.2 bar, all looks normal."* (with a photo of the nameplate + gauge)
- *"Show my maintenance log."* → dashboard opens inline
- *"Change severity on record `abc-xyz` to high."*
- *"Export everything for sync."* → returns all records as newline-delimited JSON

The model extracts fields, calls the Skill's tool, and the record is persisted to webview `localStorage`. No network. No cloud.

## Record schema

One JSON object per record. Exported as JSONL (one record per line) so it drops straight into an ingest pipeline. Shape matches WTL's `memory.md`/LanceDB intake pattern.

```json
{
  "id": "lmz8h4-a3f7k1",
  "captured_at": "2026-04-21T14:32:11+08:00",
  "location": "Jinshang 3F mechanical room",
  "equipment": {
    "type": "plumbing",
    "identifier": "CM 3-4",
    "manufacturer": "Grundfos"
  },
  "observation": "Pipe joint under sink dripping, small puddle forming.",
  "severity": "medium",
  "suggested_actions": [
    "replace gasket on joint",
    "check for corrosion on adjacent fittings"
  ],
  "raw_transcript": "Third floor mechanical room, pipe joint under sink is dripping, small puddle forming.",
  "image_description": "Close-up of a plumbing joint under a stainless steel sink, visible moisture pooled below.",
  "model": "gemma-4-e4b",
  "schema_version": 1
}
```

**Valid values**
- `equipment.type`: `HVAC` · `electrical` · `plumbing` · `fixture` · `elevator` · `exterior` · `other`
- `severity`: `low` · `medium` · `high` · `urgent`

## Privacy model

- All inference is on-device (Gemma 4 E4B via LiteRT-LM).
- Photos stay in the Gallery chat history — **the image itself is never copied into a record**. Only the model's text description is stored.
- Records live in the Gallery app's webview `localStorage`, sandboxed to the app.
- The only way a record leaves the device is if the technician explicitly taps *Copy JSONL* and pastes it somewhere else.
- Airplane mode works — no online dependency after the model is downloaded.

## Gemma 4 — facts vs. common misquotes

A few specifics that are often copy-pasted incorrectly from the launch coverage. Google's [Gemma 4 model card](https://ai.google.dev/gemma/docs/core) is the authoritative source.

| Claim | Correct |
| --- | --- |
| Parameter counts | "~2B / ~4B effective" (E2B / E4B). Precise .3/.5 numbers that sometimes appear are not from Google sources. |
| E2B file size at 4-bit | **3.2 GB** per the model card (the sub-1.5 GB figure is a 2-bit + memory-mapped PLE trick, not standard Q4_K_M). |
| E4B file size at 4-bit | **5 GB** per the model card. |
| License | Apache 2.0 — no custom carve-outs, unrestricted commercial use. |
| Context window | 128K tokens (both edge models). |
| Multimodal | Native audio + video + image input on both E2B and E4B. |
| Pi 5 CPU | 133 prefill / 7.6 decode tok/s. With Qualcomm Dragonwing IQ8 NPU: 3,700 prefill / 31 decode. |
| On-device framework | LiteRT-LM is the 2026 primary path (MediaPipe LLM Inference still works but is not the headline). |
| Function calling | Native, via dedicated special tokens — not prompt-engineered. |
| Thinking mode | Enable by including `<|think|>` in the system prompt. |

## Future concerns (not in this POC)

- **Sync to LanceDB intake.** Today export is clipboard-only. When the intake pipeline accepts HTTP, add a "sync" action that POSTs and clears local records.
- **HDP delegation provenance.** If the Skill's tool calls ever mutate real WTL systems (not just a local JSONL), bolt on the [HDP IETF draft](https://arxiv.org/abs/2604.04522) so every tool invocation carries a verifiable human-authorization token. Not needed for a record-capture demo.
- **Jinshang vocabulary fine-tune.** Stock E4B handles general plumbing / HVAC terminology; a LoRA on Jinshang floor-plan shorthand and vendor codes would sharpen accuracy once there's real usage data.
- **Native camera attachment UX.** Today the image is attached in the chat's standard multimodal input, not via a Skill-owned camera view. A native Skill (requires modifying Gallery app source) could open the camera directly from the Skill tile. iOS Gallery source isn't public, so this is a longer path.

## License

Apache 2.0, matching the parent Gallery repo.
