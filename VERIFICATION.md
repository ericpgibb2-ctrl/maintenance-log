# Verification — what's been tested and what you still need to run on your phone

## What's been verified from the terminal

The Skill's **runner logic** (`scripts/index.html`) was executed in a real browser and exercised with nine tests. All passed.

| # | Test | Expected | Result |
|---|------|----------|--------|
| 1 | `log` a ticket-intake record (leaking pipe, plumbing, medium) | Record saved, id returned, count=1 | ✅ `mo7yl3jp-om74qx`, count 1 |
| 2 | `log` an inspection record with OCR'd nameplate (Grundfos CM 3-4) | Record saved with identifier + manufacturer populated | ✅ saved, count 2 |
| 3 | Unknown action | Returns `{error: ...}` payload | ✅ "Unknown action: does_not_exist" |
| 4 | `log` with invalid severity `"CRITICAL!!!"` | Coerces to `medium` | ✅ severity = "medium" |
| 5 | `export` with 3 records | Returns fenced JSONL, one valid JSON per line | ✅ 3 lines, all parse |
| 6 | `update` severity low → high | Only that field changes | ✅ severity now "high" |
| 7 | `delete` by id | Record removed, count decrements; nonexistent id returns error | ✅ count 3→2, error on bogus id |
| 8 | `show_dashboard` | Returns `{result, webview: {url: 'dashboard.html'}}` | ✅ matches contract |
| 9 | `wipe` | localStorage cleared | ✅ null after wipe |

### Schema integrity

For every record written, these were checked against the spec:

- Top-level keys sorted: `captured_at`, `equipment`, `id`, `image_description`, `location`, `model`, `observation`, `raw_transcript`, `schema_version`, `severity`, `suggested_actions` ✅
- `equipment.{type, identifier, manufacturer}` ✅
- `id` matches `^[a-z0-9]+-[a-z0-9]+$` ✅
- `captured_at` matches ISO 8601 with timezone offset ✅
- Invalid `equipment_type` → coerces to `"other"` ✅
- Invalid `severity` → coerces to `"medium"` ✅

**Conclusion:** The JS logic behind every tool action is correct. When the Gemma 4 model calls `run_js` with conforming payloads, the records produced will match the documented schema.

## What I could not verify (needs your iPhone)

Six things only a real device and a real model can prove:

1. **Gemma 4 E4B triggers the Skill on natural speech.** The Skill manifest's *description* and action instructions should be specific enough that the model reliably picks `log` when a technician talks about equipment, rather than answering conversationally. Only an actual model can tell us — so say ten different real-sounding things and see whether the Skill fires and with what fields.

2. **Photo attachment flows through as `image_description`.** The Gallery chat's standard multimodal input should surface the image to the model; the model should populate `image_description` from it. Verify by attaching a clear nameplate photo and checking whether `equipment_identifier` and `equipment_manufacturer` come out populated.

3. **Audio Scribe transcript lands in `raw_transcript`.** If you hold-to-record a spoken observation, the transcript should end up verbatim in `raw_transcript`. Confirm by reading back a logged record — it should preserve your exact wording, not a cleaned-up paraphrase.

4. **Dashboard renders inside the Gallery chat.** The webview payload returned by `show_dashboard` should render `dashboard.html` inline. Confirm the empty state, the severity pill colors, the edit modal opening, and the copy-JSONL button triggering the clipboard.

5. **Clipboard works on iOS.** `navigator.clipboard.writeText` should succeed inside the Gallery webview. The dashboard has a fallback that expands the raw-JSON block if clipboard fails — you'll see which path fires.

6. **Airplane-mode check.** Toggle airplane mode, run through steps 1–4 again. Everything should still work because the model runs locally and the Skill has zero network dependencies.

## How to do the phone-side verification

1. Install **Google AI Edge Gallery** from the App Store.
2. Open Model Management, download **Gemma 4 E4B** (~5 GB on 4-bit; first download takes a while — grab WiFi).
3. Host this `maintenance-log/` folder on GitHub Pages (remember the `.nojekyll` file at repo root) or Cloudflare Pages. Sanity-check by visiting `<your-url>/SKILL.md` in a browser — it should show raw markdown.
4. In Gallery: *Agent Skills* with Gemma 4 E4B selected → *Skills* chip → `+` → *Load skill from URL* → paste the folder URL (not the `SKILL.md` URL).
5. Run through the two golden paths and edge cases from the approved plan:
   - Ticket intake — leaking fixture + voice + "medium urgency"
   - Inspection — gauge + nameplate + voice reading
   - Photo only, no audio
   - Audio only, no photo
   - Inline edit severity in the dashboard
   - Airplane mode repeat
   - Copy-JSONL → paste into a notes app → confirm shape

## If the Skill misfires on natural speech

The single highest-leverage thing to tune is **SKILL.md's `description` line**. The model uses it (plus the name) to decide whether the Skill is relevant to a given user message. If the Skill refuses to fire on obvious maintenance talk, make the description more concrete (list example triggers). If it fires on unrelated requests, make the description more scoped.

The second lever is the **instructions per action** — particularly how strict the model is about extracting fields. If the model keeps inventing `location` values when none were spoken, tighten the rule in the Rules section ("Never fabricate fields").

## If you want to hand this to a skeptic

Before showing someone, pre-load 3–4 records of different severities so the dashboard has visible history. Then demo: attach a photo of something on your desk, speak one sentence about it, show the record pop into the dashboard, tap to edit severity, tap Copy JSONL, paste into Notes. Total demo: under a minute.
