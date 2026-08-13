---
name: seedance-video
description: Generate a cinematic AI video with Bytedance Seedance 2.5 on Higgsfield — craft a shot-designed prompt (camera, lighting, mood, pacing), submit the generation, and log prompt/settings/output to manifest.csv.
---

# Seedance Video

Generate a cinematic AI video using Higgsfield's Seedance model.

## Model

Use model id **`seedance_2_5`** (Bytedance "Seedance 2.5"). There is no Seedance 2.0
in the Higgsfield catalog — confirm the current id with
`models_explore(action:'get', model_id:'seedance_2_5')` before relying on the
parameters below, since the catalog changes.

| Parameter | Values | Default |
|---|---|---|
| `mode` | `t2v`, `omni_reference`, `video_edit`, `video_extension` | `t2v` |
| `duration` | 4–30 seconds | 5 |
| `resolution` | `480p`, `720p` — **no 1080p or 4K** | `720p` |
| `generate_audio` | bool | `true` |
| `bitrate_mode` | `standard`, `high` | `standard` |
| `aspect_ratio` | `auto`, `21:9`, `16:9`, `4:3`, `1:1`, `3:4`, `9:16` | model default |

Reference media attaches via `medias[]` with roles `start_image`, `end_image`,
`image_references`, `video_references`, `audio_references`.

If the user wants 1080p/4K or a cinematic genre control, Seedance cannot do it —
`cinematic_studio_3_0` can (480p–4k, genre hints). Say so rather than silently
downgrading.

## Instructions

1. **Craft the prompt first.** This is the part that determines quality. Build a
   detailed shot description covering:
   - Scene description and subject action
   - Camera movement (pan, zoom, tracking, static, dolly)
   - Lighting style (natural, dramatic, soft, neon, golden hour)
   - Visual mood and color grading
   - Motion intensity and pacing

   Write it as one continuous cinematic description, not a bulleted list — the
   model reads prose, not fields.

2. **Attach a reference image, if provided.**
   - Local file, Claude Apps UI → `media_upload_widget` (the user picks the file;
     remote tools cannot read chat attachments).
   - Local file, headless/CLI → `media_upload`, then PUT the bytes to the returned
     `upload_url`, then `media_confirm`.
   - Web URL → `media_import_url`, then pass the returned `media_id`.

   Pass the resulting id in `medias[].value` (a UUID or prior `job_id` — never an
   `https://` URL) and set `mode: 'omni_reference'`.

3. **Preflight the cost.** Call `generate_video` with `get_cost: true` to price the
   job in credits without submitting it. Check the balance with `balance` or
   `list_workspaces`. Do not pass `use_unlim: true` unless the user explicitly asks
   to spend their free-trial generations.

4. **Submit** with `generate_video`. Use `count` 2–4 only for variants of the *same*
   prompt and settings; for several *different* prompts use `generate_video_batch`
   plus `jobs_wait`, then a single `show_generation_by_ids`.

5. **Wait** for completion via `jobs_wait`, then fetch results with
   `show_generation_by_ids`.

6. **Save the output** to `./generated-videos/`. The generation returns a CDN URL;
   downloading it requires network access to that host, which a restricted
   environment's proxy may refuse. If the fetch fails, record the URL in the
   manifest instead of silently skipping the row, and tell the user.

> **Sandbox limitation — verified.** In a restricted environment the MCP *control
> plane* works (workspaces, model catalog, `get_cost`, requesting a presigned URL)
> because it routes through Claude's infrastructure, but the *data plane* does not:
> a presigned `PUT` to `upload.higgsfield.ai` fails with `CONNECT tunnel failed,
> response 403`, exactly like any other blocked host. So `media_upload` returns a
> URL you cannot actually write to, and `media_upload_widget` needs an Apps
> UI-capable client (Claude Code is not one — it returns a `fallback` notice).
> With no reference upload path there is no `omni_reference`, so anything
> depending on a reference image cannot run locally. Either allowlist
> `upload.higgsfield.ai`, use `media_import_url` with a publicly reachable URL
> (fetched server-side, so the local proxy is bypassed), or run the generation in
> the Higgsfield web UI.

7. **Log to `manifest.csv`** — append a row, writing the header if the file is new:

   ```
   timestamp,model,mode,prompt,duration,resolution,aspect_ratio,reference,job_id,output
   ```

   Quote the prompt field; it contains commas.

## Parameters

- `concept` — the brand concept or scene description *(required)*
- `reference_image` — (optional) path or URL to a reference image
- `style` — (optional) visual style override: cinematic, product, lifestyle, UGC
- `camera` — (optional) camera motion preference
- `duration` — (optional) target duration in seconds (4–30)

## Note on browser automation

An earlier draft of this skill drove the Higgsfield web UI with Playwright MCP.
That path is not used here: the Higgsfield MCP server is already authenticated for
this account and reaches the same models directly, with no login step, no
credential handling, and no scraping of a UI that changes without warning. In
sandboxed environments outbound browsing is often blocked by network policy
anyway, so the browser route can fail before it reaches a login form.

Prefer the MCP tools. Reach for the browser only for something the API genuinely
does not expose, and expect to supply credentials and network access yourself.
