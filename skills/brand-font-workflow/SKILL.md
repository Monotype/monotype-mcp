---
name: brand-font-workflow
description: "Turns an uploaded brand guideline, style guide, brand book, type spec, or book-cover PDF into a licensed Monotype Fonts Library web project, digital ad, or font list/fontset. Web projects and digital ads return the connector CDN/CSS output; font lists/fontsets do not produce CSS or CDN output. For uploaded documents, extract font names in strict priority order: first fonts explicitly named in the document text, then fonts embedded in the document metadata/resources. Use when fonts must be identified from written names or file metadata, matched in Monotype inventory, checked for production status, licensed/provisioned, or mapped to actual book-cover text objects and typography roles. Never identify a font by visual similarity. Reject image-only typography inputs and raster/outlined-type outputs; font names must come from text or metadata, and licensed webfonts must remain served as fonts."
---

# Monotype Fonts Library

## Core rule

Use only the **Monotype Fonts** connector for font matching, production status, asset creation, and any CDN/CSS output. Support three creation routes: **WebProject**, **DigitalAd**, and **FontSet** (user-facing aliases: font list, fontlist, font set, fontset, list, font collection). Font identity must come from written names or file metadata, never appearance.

A wrong font identification is worse than a missing one. Do not license or provision a typeface because it visually resembles the artwork.

## Supported input gate

Accept font names only when they come from **text or reliable file metadata**.

Supported sources for font identity include:

- Font names written in the user's prompt.
- Text-extractable brand guidelines, style guides, brand books, or type specs such as PDF or DOCX.
- Embedded font metadata exposed by supported documents.
- Structured PDF text objects that expose font names for book-cover mapping.

**Not a supported input for font identity:** screenshots, JPG/PNG brand boards, flattened PDFs, scanned PDFs, or any other source whose typography exists only as pixels. Naming a font from a picture would require judging it by appearance, which is exactly the guess that can license the wrong family.

When the user supplies only image-based typography and no reliable text/font metadata, decline font identification and give this next step in the same message:

> I can't identify fonts from an image — reading them off the artwork would be a guess, and the wrong family would get licensed. Send the guideline as a PDF or DOCX, or just tell me the font names.

The user may still describe or provide **colors and patterns** from screenshots, JPG/PNG brand boards, scanned/flattened PDFs, or other image sources. Treat those as unlicensed design tokens. Only **font names** must come from text or metadata.

## Supported output gate

Do not render, export, or return raster artwork that contains the licensed type. This includes PNG/JPG mockups, image previews, screenshots, or any other pixel-based rendition of the font. Do not convert type to outlines or paths.

A webfont is licensed only while it is being **served as a font**. Once the type is converted to pixels or outlines, the connector-provided CSS is no longer serving the type and the licensed delivery chain is broken.

If the user asks for an image deliverable, do not produce the image. Offer the **code that produces it while loading the connector-provided webfont CSS** instead. Keep live text as text and reference only the connector's actual CDN/CSS output.

## Connector requirements

Before resolving fonts, confirm the Monotype Fonts connector is available. Every match, production status, project, and CDN link must come from a successful connector call made in the current turn.

If the connector is missing, disconnected, denied, or globally failing, stop and say:

> I can't match these fonts without the Monotype Fonts connector - it's the only licensed source for this. Once it's connected I'll pick up from the font list.

Do not substitute another font service, simulated results, remembered statuses, or an invented CSS URL.

Use these connector operations when available:

- `font_search_by_name(name, userQuery, ...)`
- `get_production_fonts_with_filters(userQuery, fontIds, ...)`
- `create_web_project(name, fontIds, userQuery, assetType?, description?)` for WebProject / DigitalAd
- `create_font_list(name, fontIds, userQuery, description?)` for FontSet / font list creation
- `get_web_projects_by_name(name, assetTypes, ...)` for duplicate-name checks before creation
- `font_search(fontId, userQuery)` only as an MD5 font-id fallback, never with a name

Pass the user's verbatim message as `userQuery` whenever the connector exposes that parameter.

## Pipeline

1. **Gate input** -> reject font identification from pixel-only typography; colors/patterns remain allowed.
2. **Read document text first** -> extract every font face explicitly named in the uploaded document text. This is the primary source of truth for document-based workflows.
3. **Read embedded metadata second** -> after text extraction is complete, inspect embedded font metadata/resources and add any additional faces not already named in the document.
4. **Resolve conflicts** -> when a written document name conflicts with metadata, keep the written document name as the face to license and report the metadata mismatch.
5. **For book-cover PDFs** -> also map embedded fonts to actual text objects when the PDF structure exposes font names on spans/blocks.
6. **Match** -> search each distinct requested face in Monotype inventory.
7. **Consent** -> exact matches proceed silently; collect all non-exact substitutions into one approval question.
8. **Status** -> batch all accepted `fontId`s into one production-status call (`fontIds`, never style ids).
9. **Table** -> show the required status/match table before the decision it informs and before creation.
10. **Choose creation route** -> WebProject by default, DigitalAd only when explicitly requested, FontSet when the user asks for a font list/fontlist/font set/fontset/list/font collection.
11. **Create** -> use the supplied asset name, or suggest one and ask only if none was supplied; run the route-specific duplicate-name check before creation.
12. **Return** -> for WebProject/DigitalAd, provide the connector's exact CDN URL / published CSS path, plus code if requested; for FontSet, return the created list name and membership summary only, with no CSS/CDN output.
13. **Gate output** -> never rasterize, screenshot, preview, or outline the licensed type.

## Read the source

For every uploaded document, use this strict two-pass extraction order:

1. **Pass 1 — font names explicitly named in the document text.** Extract all font families, weights, and styles that the document itself names in readable text. Examples: a type-spec table, `Headings: Sohne Buch`, a `font-family` declaration, or a written font list. Treat these names as the primary semantic source of truth for the document.
2. **Pass 2 — embedded font metadata/resources.** Only after Pass 1 is complete, inspect the document's embedded font information. For PDF, inspect embedded font resources and font dictionaries. For DOCX, inspect the font table, styles, and run-level font declarations. Extract this metadata even when Pass 1 already found named fonts.
3. **Merge with precedence.** Preserve every face from Pass 1. Add metadata-only faces from Pass 2 as additional requested faces. If Pass 1 and Pass 2 conflict, keep the Pass 1 written document name as the face to license and explicitly report the metadata mismatch.
4. **User prompt names.** If the user separately types font names, treat them as explicit requested faces, but do not skip or reorder the document extraction passes above when a document is present.
5. **Never identify by appearance.** If a screenshot, image, scanned PDF, flattened PDF, or other pixel-only source has no reliable text/font metadata, use the supported-input refusal above and ask for the font names or a text-extractable PDF/DOCX.

Do not use embedded metadata to override a font name explicitly written in the document. Metadata is secondary evidence and a source for additional faces, not the first source to inspect.

Treat **each weight and style as a separate requested face**. For example, `Inter 400`, `Inter 600`, and `Inter 700` are three entries and three table rows. Strip PDF subset prefixes such as `ABCDEF+` before searching, but preserve the recognizable source name in the response.

### Default font picks

Default heading/body/mono picks apply **only when there is no guideline and the user names no fonts**. If a guideline is present, fonts come from what it names and then from its metadata. Never fill gaps by guessing from appearance.

## Book-cover PDF workflow

When the source is a **book-cover PDF**, do the normal font extraction above and additionally inspect the PDF's structured text objects when available.

A book-cover PDF is usable for font identity only when the needed font names come from extractable text objects or embedded metadata. A scanned or flattened cover whose typography exists only as pixels is unsupported for font identification.

### Map fonts to text objects

If the PDF exposes text spans/blocks with font names, map each embedded/source font to the actual text objects that use it. Use structural evidence such as span font name, block position, text content, bounding box, and writing direction.

For each mapped face, report both:

- **Typography role** - infer the semantic role from the actual text object and layout, such as `Display Title`, `Author/Byline`, `Subtitle`, `Body Copy`, `Supporting Text`, or `Tagline`.
- **Used on the book cover** - name the exact cover element or text and its location, such as **`LOVE - large vertical title in the center`**, **`BY OLIVIA WILSON - top-left author line`**, or **`descriptive paragraph - top right`**.

The typography role may be inferred from the verified text object's content and layout. The **font identity itself may not be inferred from visual similarity**.

If multiple text objects use the same face, summarize all verified uses precisely enough to identify them. Preserve exact displayed text when practical; for long copy, use a clear element description and location.

If the PDF does **not** expose a reliable font-to-text-object mapping, do not infer role/location from appearance. Report `Cannot be verified from PDF metadata` (or equivalent concise wording) for the role/location fields.

## Match fonts in Monotype inventory

Call `font_search_by_name` once per **distinct font face** and cache the result.

Normalize the requested name and each candidate's `name`, `friendlyName`, and `psName` by lowercasing and removing spaces, hyphens, underscores, and periods.

- **Exact normalized match** -> accept silently.
- **No exact match but a plausible candidate exists** -> collect the closest candidate, preferring same family and then nearest weight/width, and ask once for approval after all searches are complete.
- **Nothing found** -> keep the requested face in the table with `-` / `—` for match and production status. Do not substitute another family automatically.

Only exact matches and explicitly approved substitutions may go into the project.

For non-exact matches, ask one combined question, for example:

> Two fonts don't have an exact match in the inventory:  
> - `Gill Sans Regular` -> `Gill Sans Nova Regular`  
> - `Sohne Buch` -> `Sohne Kraftig`  
> Use these, or would you rather pick different ones?

## Check production status

Collect all accepted `fontId`s and make **one** `get_production_fonts_with_filters` call with `fontIds` set to the full list. Do not pass style ids.

Read each returned row's `inProduction` value fresh in the current turn. A matched style with no returned production row should be treated as not in production.

Production status is **informational, not a gate**. Never omit, add, or re-ask about a font solely because its production status is false.

## Status tables: shared conventions

Every route shows the table **before the decision it informs** and before project creation.

Use these rules in every font status/match table:

- **One row per requested face.** If a family appears in three requested weights/styles, show three rows even if inventory search can reuse family information.
- Include faces that found nothing.
- **Source-font column** -> use exactly what the guideline or user wrote when that wording exists; for metadata-only faces, use the extracted face name.
- **Match** -> show the font that would actually be used and tag substitutions so they are obvious. Use `—` when there is no match.
- **In Production** -> show `✅` or `❌` from `inProduction`; use `—` when there is no match.
- Keep the table a decision aid, not a data dump. Do not add IDs, hashes, style IDs, asset IDs, or extra columns.

### Standard table shape

For non-book-cover workflows, use exactly:

| Guideline font | Match in Monotype inventory | In Production |
| --- | --- | --- |
| Sohne Buch | Sohne Buch (exact) | ✅ |
| Gill Sans Regular | Gill Sans Nova Regular (closest - approved) | ❌ |
| Foundry Gridnik | — | — |

### Book-cover table shape

For a **book-cover PDF**, always add **Typography role** and **Used on the book cover** immediately after the source-font column in **every font status/match table**. These columns are required in the response, not optional commentary.

Use exactly these columns, in this order:

| Font from cover | Typography role | Used on the book cover | Match in Monotype inventory | In Production |
| --- | --- | --- | --- | --- |
| Baskerville BT Roman | Display Title | **`LOVE`** - oversized vertical title in the center | Baskerville Std Roman (exact PostScript match) | ❌ |
| Cotford Display Regular | Body Copy / Supporting Text | Author line, top-right descriptive copy, and bottom-center copy | Cotford Display Regular (exact) | ❌ |

The example demonstrates only the required columns and specificity. In real runs, use the **actual extracted font names** and **verified text-object locations** from the user's PDF.

If mapping cannot be verified, keep the row and use explicit metadata-limitation text in the two book-cover columns rather than guessing.

Read the table as a manifest: every row with an accepted match goes into the project. Production status does not filter the manifest.

## Create the asset

Route by explicit user intent:

- Use `WebProject` by default.
- Use `DigitalAd` only when the user explicitly asks for a digital ad.
- Use `FontSet` via `create_font_list` when the user asks for a font list, fontlist, font set, fontset, list, or font collection.

A FontSet follows the same document extraction, font matching, substitution approval, status-table, and naming flow as the web-project route. Production status remains informational only and must never gate FontSet creation. The creation operation is different, and FontSet returns no CSS/CDN output.

Before creating any asset, call `get_web_projects_by_name` with the proposed name and the single target asset type. If an exact name match exists (ignoring case and punctuation), do not create a duplicate unless the user explicitly asks to do so; ask for another name or offer to reuse/update the existing asset.

### Project name

- If the user supplied a project name, use it directly. Do not ask for confirmation.
- If no name was supplied, derive a sensible suggestion from the brand, site, domain, or book title and ask once.

Fold production warnings into the same message as the naming question when needed; do not make production status a separate approval step.

Then call the route-specific creator with the accepted `fontId`s:

- WebProject / DigitalAd -> `create_web_project`
- FontSet / font list -> `create_font_list`

For FontSet creation, do not add `.woff` or `.woff2` files; those are web-delivery containers. If the chosen inventory match is only available as WOFF/WOFF2 for the relevant source entry, explain that it cannot be added to a font list and use an eligible desktop file match when available.

Do not expose `fontId`, `styleId`, MD5, or `assetId` in the user-facing response.

If creation fails, report the connector failure accurately. Never claim success without a successful creation call in the current turn.

## Return the result

For WebProject and DigitalAd, return the connector's exact `cdnUrl` / `publishedCSSPath` verbatim.

For FontSet / font list, explicitly return the created list name and a concise membership summary. **Do not return, fabricate, or imply any CSS, CDN URL, `publishedCSSPath`, `@font-face`, or web-usage code.** A FontSet is for organizing/saving fonts, not serving them on the web.

If the user asks for markup or code, build it only from the connector output and approved font names. Do not invent `@font-face` declarations or substitute Google Fonts / Adobe Fonts / system-hosted URLs for the connector result. Generic terminal fallbacks such as `sans-serif` or `serif` are fine.

If the user asks for PNG/JPG artwork, a screenshot, an image preview, a raster mockup, or outlined/vectorized type, **do not produce that output**. Offer code that renders the desired composition in a browser or other environment while the connector-provided webfont is actively served as a font. Keep the actual text live rather than converted to pixels, outlines, or paths.

## The only two user questions

Interrupt the workflow only for:

1. **Substitution approval** - one combined question covering every non-exact match.
2. **Asset name** - one question only when the user did not provide a name (project/ad/list as appropriate).

Everything else should run without additional confirmation. Unsupported image-only typography inputs are not a workflow question: decline font identification immediately with the required next step.

## Invariants

- Match, status, project, and CDN data come from successful Monotype Fonts connector calls in the current turn.
- Fonts come from written names or file metadata, never visual similarity.
- Screenshots, JPG/PNG brand boards, scanned PDFs, flattened PDFs, and other pixel-only typography sources are unsupported for font identification.
- When typography exists only as pixels, use the required refusal and ask for a text-extractable PDF/DOCX or the font names.
- Colors and patterns may come from any source, including images; only font names require text or metadata.
- A present document always controls font sourcing in strict order: first fonts explicitly named in the document text, second embedded font metadata/resources.
- Each weight and style is a separate requested face and separate table row.
- Every non-exact match requires explicit approval; exact matches do not.
- Production status is fresh, visible, and informational only.
- The table appears before project creation and includes every requested face.
- Book-cover PDF tables always include `Typography role` and `Used on the book cover` immediately after the source-font column.
- Book-cover role/location data must come from verified text-object structure; when unavailable, state that it cannot be verified from metadata.
- Ask for an asset name only when none was supplied.
- Use `DigitalAd` only when the user explicitly requested a digital ad.
- Use `FontSet` / `create_font_list` when the user asks for a font list/fontlist/font set/fontset/list/font collection.
- FontSet follows the same extraction, matching, substitution-approval, status-table, and naming flow, but returns no CSS/CDN output.
- Return only the connector's actual CDN / CSS output for WebProject/DigitalAd; never fabricate it, and never produce CSS/CDN output for FontSet.
- Never render or export licensed type as PNG/JPG, image previews, screenshots, raster mockups, or other pixel-based artwork.
- Never convert licensed type to outlines or paths.
- For image-deliverable requests, offer code that keeps the webfont live and served through the connector-provided CSS instead of producing the raster/outlined artifact.
