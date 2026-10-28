# Smart AI Ad Pipeline

Turns a **reference photo + a business's brand kit** into a finished social ad, and keeps every part of the ad
(photo, headline lines, button, logo) as a **separate, editable layer**.

- Typical run: **35–75 seconds** and **about $0.035–0.06 per ad** (measured from OpenAI token usage).
- Backend: FastAPI (Python). Frontend: a single-page editor in `public/` (plain HTML/JS).
- Deployed on Vercel (backend function + CDN-served frontend + Vercel Blob storage).

---

## 1. How an ad is made

```
photo + brand kit (+ optional logo)
   │
   ├─ 1. Measure the photo            local: brightness, busy/calm regions, dominant colours
   ├─ 2. Creative brief               ONE call to gpt-5.6-terra (sees the photo)
   │                                  - reads what is happening in the photo
   │                                  - writes the copy from the business description + services ONLY
   │                                  - picks a type style, font pairing, accent colour
   │                                  - decides where the subject goes and where the text goes
   │                                  - writes a photographer's shot list
   ├─ 3. Photo                        keep the photo, or re-shoot the SAME subject with
   │                                  gpt-image-2.5-flare (quality "low"). The subject is never replaced.
   │                                  For clean product subjects a "layout guide" places the subject
   │                                  exactly on the planned side before re-shooting.
   ├─ 4. Cut-out                      local segmentation model (isnet, ONNX) finds the subject's pixels
   ├─ 5. Compose                      deterministic layout code: places text where it clears the subject,
   │                                  sizes it (with readability minimums), picks text colours from the
   │                                  photo's palette, adds the button, logo, optional shade
   ├─ 6. Quality gate                 every layout is measured: no text on the subject, contrast >= 3:1,
   │                                  nothing outside the frame, no overlaps, no tiny text.
   │                                  If it fails, the photo is re-shot once automatically.
   ├─ 7. Final check                  ONE cheap call to gpt-5.6-luna; can only nudge the layout
   └─ 8. Render + save                PNG preview + layered SVG + scene.json (the editable document)
```

**What is taken from the brand kit:** only the **business description** and **services/products** (plus the
business name, used for the small name label). Fonts, colours, taglines and contact details in the kit are
**not** used. Fonts come from a curated library (`app/services/typography.py`), colours from the photo.

**Design references:** `references/library.json` holds design notes from Pinterest ads. The brief call gets the
three entries closest to the business's industry. Add pins with `python -m tools.add_reference <pin url>`.

---

## 2. Layers and editing

The ad is stored as `scene.json`: an ordered list of layers (bottom to top). The PNG and SVG are just drawings
of this file. Text is **never** part of the photo; every layer is created separately.

| Layer id | Type | Where it comes from |
|---|---|---|
| `photo` | image | the reference photo (kept or re-shot) |
| `scrim` | gradient | optional shade added only when text needs more contrast |
| `headline_*`, `script_*`, `statement_*`, `kicker_*`, `subheadline` | text | copy from the brief; each line is its own layer |
| `contact` | button | the call-to-action pill |
| `logo` | image | uploaded logo (file or URL) |
| `logo_plate` | shape | optional backing behind the logo when it would vanish into the photo |
| `wordmark` | text | business name, used when no logo is supplied |
| `subject` | image | cut-out of the product, only in the "giant word behind the subject" style |

> In most ads the product is part of the `photo` layer (you can move/resize/replace the whole photo, not the
> product alone). A cut-out of the subject is still saved with every ad.

**Editing never calls the AI again.** The client edits `scene.json` and sends it back:

- `POST /api/v1/ads/{id}/render` re-renders the edited scene and saves a **new version** (PNG, SVG, scene).
- `POST /api/v1/ads/{id}/assets` uploads a replacement image (optionally background-removed) and returns the
  `src` to put on a layer.
- `GET /api/v1/ads/{id}` returns the latest saved version (to reopen an ad).

The editor in `public/` supports: drag to move, corner-handle resize, double-click text editing, font size,
line height, colour, alignment, case, button label/colours, opacity, layer order, hide/show, lock, image
replace, add text, undo/redo, and server re-render. Open a saved ad with `/?ad=<id>`.

---

## 3. API

All `/api/*` calls need the header `X-Access-Key: <APP_ACCESS_KEY>` when that variable is set.

### `POST /api/v1/ads/generate` (multipart form)

| Field | Required | Notes |
|---|---|---|
| `image` | yes | reference photo (JPG/PNG/WEBP, max 20 MB; on Vercel requests are capped at 4.5 MB, so send ≤ ~2000 px) |
| `brand_kit` | yes | JSON text: the BG-content format (`at_a_glance`, `contents_long_description`, `contents_area_of_focus`, …) or the `BrandKit` schema in `app/schemas/brand_kit.py` |
| `logo` | no | logo file (transparent PNG works best) |
| `logo_url` | no | http(s) URL of the logo; used when no `logo` file is sent |
| `format` | no | `4:5` (default), `1:1`, `9:16`, `1.91:1` |
| `brief` | no | campaign goal, e.g. "fill weekend catering orders" |
| `photo_mode` | no | `auto` (default), `keep`, `restage` |
| `website`, `business_name` | no | helpers for BG-content kits that lack them |

```bash
curl -X POST https://<your-app>/api/v1/ads/generate \
  -H "X-Access-Key: $APP_ACCESS_KEY" \
  -F "image=@photo.jpg" \
  -F "brand_kit=<brand_kit.json" \
  -F "logo=@logo.png" \
  -F "format=4:5"
```

Response (main fields): `id`, `urls` (`preview_png`, `svg`, `scene_json`, `asset_base`, `report`),
`scene` (the editable document), `copy`, `design`, `qa` (`passed`, `violations`, `photo_retried`),
`cost_usd`, `timings_s`.

### Other endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/health` | models in use, storage mode, whether an access key is required |
| GET | `/api/v1/ads/{id}` | load the latest version of an ad |
| POST | `/api/v1/ads/{id}/render` | body = edited `scene` JSON → new version |
| POST | `/api/v1/ads/{id}/assets` | form: `image`, `remove_background` → `{src, width, height, url}` |

---

## 4. Run locally

Python 3.10+ (Vercel uses 3.12).

```bash
pip install -r requirements-dev.txt
python -m tools.fetch_model          # downloads the 171 MB segmentation model once
cp .env.example .env                 # then put your OPENAI_API_KEY in .env
uvicorn app.main:app --port 8000
```

Open http://127.0.0.1:8000. Locally, ads are saved in `./outputs/<id>/` and served at `/outputs/...`.

Test with the sample businesses (each image is run with its own brand kit):

```bash
python -m tests.folder_batch Brandkitsandimagestotest --workers 3
```

Re-run only the layout on a finished local ad (no API cost), useful when changing layout code:

```bash
python -m tools.relayout <ad_id>
```

---

## 5. Deploy to Vercel

```bash
python -m tools.fetch_model                                           # the model must be in the bundle
vercel link
vercel blob create-store <store-name> --access public --yes           # sets BLOB_READ_WRITE_TOKEN
vercel env add OPENAI_API_KEY --environment production
vercel env add APP_ACCESS_KEY --environment production                # any long random string
printf 1 | vercel env add VERCEL_SUPPORT_LARGE_FUNCTIONS --environment production --yes
vercel deploy --prod
```

Why each step:
- **Model in the bundle:** the function's `/tmp` is too small to download the 179 MB model at runtime.
- **Large functions:** the bundle (dependencies + model) is ~550 MB, above Vercel's standard 500 MB Python limit.
- **Blob store:** serverless instances don't share disks; ads and their edits are saved to Blob so any
  instance can open them.
- **Access key:** stops anyone with the URL from spending the OpenAI budget.

Notes:
- Vercel **Deployment Protection** is on by default: the site needs a Vercel login. Turn it off in project
  settings to share it outside the team (the access key still protects the API).
- The first one or two requests right after a deploy can fail while the large function starts; retry once.
- `vercel.json` sets the function timeout to 300 s (the Hobby maximum).

---

## 6. Project structure

```
app/
  main.py                       FastAPI app, access-key middleware, static mounts
  config.py                     settings (models, formats, prices) from environment variables
  routers/ad_router.py          HTTP routes
  controllers/ad_controller.py  input parsing, brand-kit + logo handling, error mapping
  prompts/prompts.py            the three prompts: creative brief, re-shoot shot list, final check
  schemas/
    ai.py                       structured outputs (CreativeBrief, Critique)
    brand_kit.py                BrandKit + what the brief is allowed to see
    scene.py                    the editable layer document
  services/
    pipeline_service.py         orchestration, versioned saving, edit/re-render, asset upload
    brief_service.py            the brief call, reference matching, layout variety history
    imaging_service.py          keep / re-shoot, layout guide, segmentation wrapper
    segmentation.py             isnet ONNX subject cut-out (no rembg dependency)
    layout_service.py           composer: type styles, text positions, colours, quality gate
    render_service.py           scene → PNG and layered SVG
    text_engine.py              text measuring, balanced line breaks, fit-to-box
    typography.py               font files + curated font pairings
    vision_service.py           pixel measurements
    critic_service.py           final check call
    storage_service.py          local disk or Vercel Blob
    openai_service.py           OpenAI calls + per-call cost tracking
    brand_kit_adapter.py        BG-content JSON → BrandKit
    color_utils.py              contrast and colour helpers
  assets/fonts/                 font files used by renders and the editor
  assets/models/                segmentation model (download with tools.fetch_model)
public/                         editor frontend + sample brand kits/photos
references/                     Pinterest design-reference library
Brandkitsandimagestotest/       test brand kits and photos
tests/folder_batch.py           batch test against a running server
tools/                          fetch_model, relayout, add_reference
```

---

## 7. Configuration (environment variables)

| Variable | Default | Purpose |
|---|---|---|
| `OPENAI_API_KEY` | – | required |
| `APP_ACCESS_KEY` | unset | if set, `/api/*` requires header `X-Access-Key` |
| `AD_BRIEF_MODEL` | `gpt-5.6-terra` | creative brief model |
| `AD_BRIEF_EFFORT` | `low` | reasoning effort for the brief |
| `AD_CHECK_MODEL` | `gpt-5.6-luna` | final check model |
| `AD_CHECK_ROUNDS` | `1` | final check rounds (layout-only fixes) |
| `AD_IMAGE_MODEL` | `gpt-image-2.5-flare` | re-shoot model |
| `AD_IMAGE_QUALITY` | `low` | image quality |
| `BLOB_READ_WRITE_TOKEN` | unset | set by Vercel when a Blob store is connected; switches storage to Blob |

---

## 8. Known limitations

- **No user accounts:** anyone with the access key and an ad id can open that ad; there is no "my ads"
  list per user (the editor's "Recent ads" list is stored only in that browser).
- **Version history:** every saved version's files are kept, but there is no API to list or restore them.
- **Product as its own layer:** only in the "behind the subject" style; otherwise it is part of the photo.
- **Photos with people** or subjects cut off by the frame skip the layout guide, so their placement follows
  the re-shoot and is less controlled.
- **Editor gaps:** no font-family picker, rotation, multi-select, snapping or photo cropping.
- **Concurrent edits:** last save wins.
- Layout variety uses a shared history, so it shows across a series of ads, not within one ad.
