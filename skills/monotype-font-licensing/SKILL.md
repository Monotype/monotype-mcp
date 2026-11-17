---
name: monotype-font-licensing
description: "Licenses every font in web or design output by routing it through the Monotype Fonts connector (the Monotype Enterprise Connector) before any code is emitted. Extracts the typography from the input (a brand guideline, font names in the prompt, or the defaults the agent would otherwise pick), matches each face in the Monotype inventory, checks production status, resolves which web project will serve it, gets consent in one status table, and emits only the CSS / CDN URL the connector returns. Handles every presence case: all fonts in one project, none in a project, split across projects, partial presence, a missing weight, a font not in the catalog. Triggers whenever an agent is about to produce HTML, CSS, React, JSX, Vue, Svelte, SVG with text, email templates, mockups, landing pages, posters or any output that names a font, even when the font is only implied."
---

# Monotype Font Licensing

## Why this exists

When an agent generates web or design content it is easy to reach for any font name that comes to mind and stitch in a Google Fonts link or a system fallback. For Monotype Fonts customers that breaks the license: the fonts they may ship are the ones in their Monotype inventory, and the only legitimate web CSS for those fonts is what the Monotype Fonts connector returns, a web project's published CSS / CDN URL.

This skill enforces that end to end. Typography is extracted from whatever the user provides, each face is matched in the inventory, production status is checked, the user approves one status table, and only the connector's CSS is emitted. Anything the connector cannot cover is named and left out, never substituted.

## The connector's tools

Every tool takes a `userQuery`: one short sentence, in your own words, saying what the user wants from the call. Pass it every time.

| Tool | Role in this skill |
|---|---|
| `font_search_by_name(name)` | Turns a font name into matches with `fontId`, `styleId`, `source` and `fileName`. It searches the Monotype catalog and the customer's imported fonts, exact name first, then PostScript name, then widens to the family. Read `exactMatch` and `fallbackApplied`. No match means the font is not available under that name from either source. |
| `get_production_fonts_with_filters(fontIds)` | Production status for identified fonts. One batched call with every accepted `fontId`. An empty row for an id means not in production; ids in `unknownFontIds` are unknown, not "no". It cannot look a name up. |
| `get_web_projects_by_name(name, assetTypes)` | Finds the user's web projects and digital ads by name, and is the duplicate-name check before creating one. Returns metadata only, not fonts. |
| `get_asset_details(assetType, assetId)` | A project's fonts (with their `fontId`s) and its `publishedCSSPath` / CDN URL. Re-read it in the turn where you emit the CSS. |
| `create_web_project(name, fontIds, assetType?, description?)` | Creates a web project (or a digital ad when the user asks for one), adds the fonts, returns the CSS / CDN URL. Only after the name check, the status table and explicit consent. |
| `update_web_project(name, fontIds)` | Adds fonts to an existing web project or digital ad and returns the refreshed CSS / CDN URL. The fix for "the project has the family but not this weight" and for "add the missing font to my project". |
| `identify_font_from_image(image)` | Visual identification from a PNG or JPEG. Last resort only: when nothing names the font, or the user asks for it. Its answers are ranked guesses. |
| `font_search(fontId)` | Metadata for a known `fontId`. Never a name search. |

There is no licensing check on this surface. What the connector establishes is: the font exists in the catalog or the imported fonts, it is or is not in production for this account, and it is or is not in a web project. Report those facts separately and do not claim more.

## Freshness

Font identity (`fontId`, `styleId`, PostScript name) is stable and may be reused within a conversation. Production status, project membership and CSS / CDN paths are volatile: every `@font-face`, CSS path or CDN URL you emit, and every production or availability claim you make, must come from a tool result in the current turn.

## The pipeline

```
1. Input          → guideline (PDF / DOCX / image), font names in the prompt, or the defaults you would pick
2. Extraction     → every face (family + weight + style); names from text, then file metadata, visual guess last
3. Match          → font_search_by_name per distinct face: exact, closest (needs approval), or not found; check eligibility
4. Production     → ONE get_production_fonts_with_filters call with every accepted fontId
5. CSS source     → the project the user named (get_web_projects_by_name → get_asset_details), or a new one
6. Status table   → one table, one consent gate; combine substitution, production and project questions in it
7. Provision      → reuse the project's CSS, update_web_project to add what is missing, or create_web_project
8. Output         → only the connector's CSS / CDN URL, text kept live
```

Do not skip stages and do not emit CSS before stage 6 is approved.

## When this applies

Trigger on any output that contains, references or implies a named font, and on any request to turn a brand guideline into web typography: HTML pages, CSS, style blocks; React, JSX, TSX, Vue, Svelte or any styled-text component; SVG with text; email templates, landing and marketing pages; frontend artifacts and mockups; posters, social cards and other deliverables that name a typeface. If the user asks for "a clean blog layout" without naming a font, the fonts you would pick still go through the pipeline before they appear in code.

Before going further, confirm the connector is available. If it is not, say so and offer to connect it. Never fall back to Google Fonts, Adobe Fonts, system fonts or any other source.

## Stage 1 and 2: input and extraction

Sources, which a single run may combine: an uploaded guideline, font names typed in the prompt, the output of a separate brand-guideline skill, or the defaults you would otherwise reach for.

Build the face list only from what the source states, in this order, stopping at the first that yields fonts:

1. **Names in the guideline text or the prompt.** A guideline nearly always names its fonts in a typography section. Take them as written.
2. **File metadata, when nothing is named.** For a PDF, the embedded font resources; for a DOCX, the font table, styles and run-level font declarations. Strip PDF subset prefixes such as `ABCDEF+`. A name written in the text wins over metadata when they disagree; report the mismatch.
3. **Visual identification, last.** Only when no font is named anywhere and the file has no usable metadata (a scan, a photo, a logo, outlined type), or when the user explicitly asks to identify visually, call `identify_font_from_image` with a PNG or JPEG (convert one PDF page first, around 1500 to 2000 px on the long edge, under 5 MB). Present its candidates as visual guesses, one table per detected region, and let the user pick before anything is licensed. Never present a guess as a confirmed name.

Treat each weight and style as a separate face: "Inter 400/600/700" is three faces and three table rows.

## Stage 3: match in the Monotype inventory

Call `font_search_by_name` once per distinct face and keep the result for the run. The tool already widens from the exact name to the PostScript name to the family, so pass the face as written; do not re-run it with reworded names.

- `exactMatch: true` → accept silently. Record the `fontId`.
- `fallbackApplied: true` → the results are the whole family. Pick the row whose name or PostScript name matches the requested weight and style. If a row matches, treat it as exact. If none does, the family exists but not that style: propose the closest style as a substitution.
- No exact match but a plausible candidate → record it as `(closest - awaiting approval)`. Prefer the same family, then the nearest weight and width, then the most current family version. Never accept a substitution silently.
- Nothing returned → the font is not available under that name in the catalog or the imported fonts. Record `Not found`, do not substitute another family, and give the remediation at stage 6.

**Eligibility for web use.** A match whose `source` is "Monotype Fonts" is eligible. An imported font (any other `source`) is eligible only when its `fileName` ends in `.woff` or `.woff2`. Do not pass an ineligible `fontId` to a create or update call. Name the font, say it cannot go in a web project because imported fonts must be WOFF files there, and offer the WOFF file if the search returned one.

## Stage 4: production status

Collect every accepted `fontId` (exact matches and proposed substitutions) and make one `get_production_fonts_with_filters` call with `fontIds`. Read each row's `inProduction` fresh in this turn. A font with no returned row is not in production. A font listed in `unknownFontIds` is "Unknown", never "No".

Production status is informational: it does not add, remove or block a font. But a font that is not in production is included in a new or updated project only after the user has seen that in the table and agreed.

## Stage 5: where the CSS will come from

The deliverable CSS is a web project's published stylesheet, so each approved face needs a project. Do not scan the whole library to find one; projects are resolved from what the user tells you.

- **The user named a project** ("use the fonts in my Acme Site project", "add these to Campaign 2026"): call `get_web_projects_by_name` with that name and `assetTypes` set to `["WebProject", "DigitalAd"]`. One exact match ignoring case and punctuation → use it. Several → list names, type and created date and ask which. None → say so, show any close names returned, and ask for the right name or whether to create one. Then `get_asset_details` on the chosen project for its fonts and `publishedCSSPath`, and compare its fonts against the requested faces.
- **The user named no project:** ask once, in the same message as the status table, whether to reuse an existing project (they name it) or create a new one. If they ask what projects they have, list them with `get_web_projects_by_name` without a name and show the first page.

This gives each face one of these presence cases. Several cases can occur in one run; resolve each row on its own.

| Case | What you found | What to offer |
|---|---|---|
| 1. All present | Every face is in the chosen project | Reuse the project's CSS. |
| 2. None in a project | No project was named, or the named one has none of the faces | Create a new project (`create_web_project`) or add them to the named one (`update_web_project`). |
| 3. Split across projects | The user named two projects and the faces are divided between them | Include both stylesheets, or consolidate with `update_web_project` into one. Ask; do not combine silently. |
| 4. Partial presence | Some faces in the named project, some not | Add the missing faces to it with `update_web_project`, or proceed without them. Never substitute. |
| 5. Font in several projects | The user named more than one project and a face is in both | Ask which project's CSS to use. |
| 6. Weight missing | The project has the family but not the requested weight or style | Add the weight with `update_web_project`, or use an available weight the user picks. Never swap silently. |
| 7. Not in the catalog | `font_search_by_name` found nothing | No licensed web CSS exists for it. The user can import the font in My Library (as WOFF for web use) and tell you; or proceed without it. |

## Stage 6: status table and the single consent gate

Show one table covering every face, then ask. This is the only consent gate; do not scatter separate yes/no questions.

| Font requested | Match in Monotype inventory | In Production | Web project | Action |
|---|---|---|---|---|
| Helvetica Neue Bold | Neue Helvetica 75 Bold (closest - awaiting approval) | ✅ | Acme Site | Use it? |
| Söhne Buch | Söhne Buch (exact) | ✅ | Acme Site | Reuse |
| Gill Sans Regular | Gill Sans Nova Regular (exact) | ❌ | not in Acme Site | Add to Acme Site? (not in production) |
| Brand Sans Bold | Brand Sans Bold (exact) | ✅ | Acme Site has Regular only | Add Bold to Acme Site? |
| Foundry Gridnik | Not found | Not found | — | Import in My Library, or proceed without |

Rules for the table: one row per requested face, the user's own wording in the first column, substitutions visibly tagged, production from this turn's check, project names never ids. No `fontId`, `styleId`, hash or `assetId` anywhere in chat.

Then ask one combined question that covers every decision the table raises: substitutions to approve, fonts to add or create, which project, and whether to include fonts that are not in production. Wait for an explicit yes in chat. Do not infer consent from the original request, from urgency, or from an earlier turn.

If the approved set is empty, say so and ask whether to wait for the user to set up the fonts or proceed with generic fallbacks (`sans-serif`, `serif`, `monospace`) for now. Do not refuse the whole task because some fonts were unavailable.

## Stage 7: provision

Only after the gate:

- **Reuse:** `get_asset_details` on the project in this turn and take its `publishedCSSPath` / CDN URL.
- **Add to an existing project:** `update_web_project` with the project's name and the approved eligible `fontId`s. It returns the refreshed CSS / CDN URL.
- **Create:** suggest a name from the brand or site and let the user keep or change it. Check it with `get_web_projects_by_name` (`assetTypes` = the single type you are creating; `accessTypes` unset). An exact-name match means the name is taken: offer to reuse that asset or ask for another name. Show the creation summary (type, name, faces, production warnings) and ask `Create this web project now?`. On an explicit yes, call `create_web_project`. Pass `assetType: "DigitalAd"` only when the user asked for a digital ad.

Report a failed call in the tool's own words. Never claim an asset was created or updated without a successful response in this turn.

## Stage 8: output

- Include the connector's CSS link, CDN URL or `@font-face` rules exactly as returned. When the user approved two projects, include both stylesheets.
- Every `font-family` references only approved faces. Generic keywords (`sans-serif`, `serif`, `monospace`, `system-ui`) are fine as terminal fallbacks; the primary face comes from the connector.
- No Google Fonts, Adobe Fonts, Bunny or other stylesheet links; no `@font-face` pointing at a URL the connector did not return; no invented font names; no requested font that did not pass the gate.
- Keep licensed type live as text. Do not render it to PNG or JPG, screenshots or raster mockups, and do not convert it to outlines or paths. If the user asks for an image, offer the code that renders it with the connector's CSS instead.

## Communicating with the user

- Say what you are doing in plain terms ("checking your Monotype inventory", "checking production status", "reading the Acme Site project"). Do not recite tool names in chat.
- Keep the status table front and centre; it is the decision surface.
- Name requested versus matched fonts in every approval question, even when they look identical.
- When a font is left out, list it and say you are proceeding without it. Do not editorialise and do not substitute.

## Invariants

- Fonts, production status, project membership and CSS come only from connector calls in the current turn.
- Font identity comes from written names or file metadata; visual identification is a last resort and its results are guesses the user confirms.
- One `font_search_by_name` per distinct face; one batched production check with `fontIds`, never style ids.
- Every substitution, every addition to a project, every creation and every non-production font needs an explicit yes at the single gate.
- Projects come from what the user names, not from scanning the library; several candidates mean a question, never a silent pick.
- Only eligible fonts go into a web project: Monotype Fonts, or imported WOFF / WOFF2 files.
- Never fabricate a project, a URL or an `@font-face`; never use a font source the connector did not return.
- Never show ids or hashes in chat; project names only.
- Never rasterise or outline licensed type.
