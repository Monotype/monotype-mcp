---
name: monotype-brand-kit
description: "Turns a brand guideline into licensed, on-brand web output through the Monotype Fonts connector (the Monotype Enterprise Connector). Extracts the typography plus the colours, component patterns and layout tokens, routes the fonts to an existing web project the user names or to a new one, matches and production-checks every face, shows a status table, and emits only the connector's CSS while applying the extracted design tokens. Also handles a request that names an existing web project as the font source (\"build this with the fonts in my Acme Site project\"): it reads that project and checks every font's production status before generating anything. Triggers whenever an agent is about to produce HTML, CSS, React, JSX, Vue, SVG with text, email templates, mockups or landing pages that reference a named font, a brand guideline or a named web project."
---

# Monotype Brand Kit

## Why this exists

Monotype Fonts customers may only ship fonts from their Monotype inventory, and the only legitimate web CSS for those fonts is what the connector returns: a web project's published CSS / CDN URL. Reaching for a Google Fonts link, a system fallback or an invented `@font-face` breaks the license. So the connector is the single source of truth for fonts: match each face, confirm production, put the fonts in a web project, and emit only that project's CSS.

Colours, component patterns and layout tokens from the guideline are the user's own design tokens, not licensed assets. They bypass the connector and are applied directly so the output is on-brand as well as licensed.

## The connector's tools

Every tool takes a `userQuery`: one short sentence saying what the user wants from the call. Pass it every time.

| Tool | Role in this skill |
|---|---|
| `font_search_by_name(name)` | Font name → matches with `fontId`, `source`, `fileName`. Exact name first, then PostScript name, then widens to the family; read `exactMatch` and `fallbackApplied`. |
| `get_production_fonts_with_filters(fontIds)` | One batched production check for every accepted `fontId`. Not a name search. |
| `get_web_projects_by_name(name, assetTypes)` | Finds the project the user named; duplicate-name check before creating one. Metadata only. |
| `get_asset_details(assetType, assetId)` | A project's fonts (with `fontId`s) and its `publishedCSSPath` / CDN URL. Re-read in the turn where you emit the CSS. |
| `create_web_project(name, fontIds, assetType?, description?)` | Creates a web project (digital ad only on request), adds the fonts, returns the CSS / CDN URL. After the name check and explicit consent. |
| `update_web_project(name, fontIds)` | Adds missing fonts or weights to an existing project and returns the refreshed CSS / CDN URL. |
| `identify_font_from_image(image)` | Visual guesses from a PNG or JPEG. Last resort when nothing names the fonts, or on request. |
| `font_search(fontId)` | Metadata for a known `fontId`. Never a name. |

There is no licensing check on this surface: report what the catalog search, the production check and the project contents establish, and say no more than that.

## Pipeline

```
0. Direct project reference → the user names an EXISTING project as the font source: read it, check production, go to stage 5
1. Brand input       → guideline (PDF / DOCX / image), font names in the prompt, or your own default picks
2. Extraction        → fonts (family + weight + style) AND design tokens (colours per role, component patterns, layout scale)
                        fonts come from names in the text, then file metadata, visual guess last
                        tokens are not licensed: skip the connector, apply at stage 5
3. Route             → the user chooses: (A) reuse an existing project they name, or (B) create a new project
4. Provision         → A: get_web_projects_by_name → get_asset_details → compare → production → table → reuse / update
                        B: font_search_by_name → consent on closest → production → table → name check → consent → create
5. Output            → only the connector's CSS, plus the extracted tokens so the result is on-brand
```

## When this applies

Trigger on any output that contains, references or implies a named font, and on any request to turn a brand guideline into web typography: HTML, CSS, style blocks; React, JSX, TSX, Vue, Svelte or any styled-text component; SVG with text; email templates, landing and marketing pages; mockups and rendered visuals where text styling matters; posters, social cards and other deliverables that name a typeface. Also trigger when the user names an existing web project as the font source instead of naming fonts; that goes to stage 0.

Confirm the connector is available first. If it is not, say so and offer to connect it. Never fall back to Google, Adobe or system fonts.

## Stage 0: direct project reference

Some requests name an existing project and ask you to build with whatever is in it ("create a landing page using the fonts in my Spring Campaign project"). There is nothing to extract and no route to choose, but "already in a project" does not mean "safe to ship": a project can hold fonts that were taken out of production. So this path still checks production, before anything is generated.

1. **Resolve the project.** `get_web_projects_by_name` with the name and `assetTypes` `["WebProject", "DigitalAd"]`. One exact match (ignoring case and punctuation) → use it. Several → list name, type and created date and ask which. None → say so, show close names, and ask for the right name. Never guess or fall back to a different project.
2. **Read its fonts.** `get_asset_details` on the chosen project: its fonts with their `fontId`s and its `publishedCSSPath`. Keep the CSS path for stage 5.
3. **Check production first.** One `get_production_fonts_with_filters` call with every font's `fontId`, before any HTML or CSS is written.
4. **Show a short table** and act on it:

   | Font in Spring Campaign | In Production |
   |---|---|
   | Söhne Buch | ✅ |
   | Gill Sans Nova | ❌ |

   Every font in production → go straight to stage 5 with the project's CSS; the user already chose the project. Any font not in production → name it and ask whether to proceed without it or include it anyway. Do not drop it or include it silently.
5. If the user also gave a guideline, its colours and patterns still apply at stage 5. Otherwise say in one line that colours and layout are your own design choice, and offer to align them once a guideline is shared.

This stage never searches fonts, never creates and never updates a project.

## Stage 1: brand input

Establish where the requirements come from; a run may combine several: an uploaded guideline (the source for the whole brand, not only font names), font names in the prompt, a separate brand-guideline skill's typography output, or your own default heading / body / mono picks when the user names nothing and supplies no guideline.

## Stage 2: extraction (typography + design tokens)

Pull two kinds of thing. Only the fonts continue to stage 3; the tokens are captured here and applied at stage 5.

**Typography.** Build the face list from what the source states, never from how rendered text looks. Work down this order and stop at the first that yields fonts:

1. **Named in the guideline text or the prompt.** "Headings: Helvetica Neue 65 Medium", a type-spec table, a `font-family` in embedded CSS. Take these as written.
2. **File metadata, when nothing is named.** PDF embedded font resources; DOCX font table, styles and run-level fonts. Strip subset prefixes such as `ABCDEF+`. A name written in the text wins over metadata; report a mismatch.
3. **Visual identification, last.** Only when nothing names the fonts and there is no usable metadata, or the user asks for it: convert one page to PNG or JPEG (1500 to 2000 px on the long edge, under 5 MB) and call `identify_font_from_image`. Show its candidates as guesses, one table per region, and let the user pick before anything is licensed.

Treat each weight and style as a separate face.

**Design tokens.** A bare hex list is not enough; capture what each token is for:

- **Colours:** every named brand colour with its value and the guideline's own name, mapped to semantic roles (primary, secondary, accent, background and surface, heading versus body text, muted text, borders, link states, success / warning / error / info) and per component where specified (buttons, cards, header and nav, inputs, badges, footer), noting light and dark. A colour with no stated role is recorded as such, not guessed.
- **Component patterns:** buttons (shape, radius, padding, border, elevation, variants and states), cards and surfaces, header and nav, forms and inputs, badges, tables, dividers, alerts, modals, and the global scales: spacing, grid and gutters, breakpoints, container widths, radius, border widths, elevation.
- **Other elements:** logo usage and clear space, iconography, imagery treatment, motion; voice and tone only if it affects copy you write.

Record them as named tokens (name → value plus the role or component it serves).

## Stage 3: route (the user chooses)

Unless the prompt already says which, ask one question, in the same message as the extracted font list:

> How should I source these fonts: (A) reuse an existing web project in your Monotype library (tell me its name), or (B) create a new web project for them?

Follow only the chosen branch. If the prompt already names a project to reuse, or already says to create one, do not ask again.

## Stage 4A: reuse an existing project

1. **Resolve the project** exactly as in stage 0, step 1.
2. **Read it** with `get_asset_details`: its fonts and `publishedCSSPath`.
3. **Compare** the guideline faces against the project's fonts by normalised family and style name (lowercase; remove spaces, hyphens, underscores and periods). Each face is present, present as the family but not this weight, or absent.
4. **Resolve the gaps.** For each absent face or missing weight, run `font_search_by_name` to find it in the inventory (exact, or closest awaiting approval, or not found; apply the eligibility rule: Monotype Fonts, or imported WOFF / WOFF2 only).
5. **Check production** with one `get_production_fonts_with_filters` call covering the project's fonts you will use and the gap fonts you found.
6. **Show the status table** before asking anything:

   | Guideline font | In Acme Site | Match in Monotype inventory | In Production |
   |---|---|---|---|
   | Helvetica Neue Bold | ✅ Neue Helvetica 75 Bold | (in project) | ✅ |
   | Gill Sans Regular | ❌ | Gill Sans Nova Regular (exact) | ❌ |
   | Brand Sans Bold | family only (Regular) | Brand Sans Bold (exact) | ✅ |
   | Foundry Gridnik | ❌ | Not found | Not found |

7. **Ask one combined question:** approve any substitutions, add the missing faces and weights to the project with `update_web_project`, include or skip fonts that are not in production, and proceed without the ones that were not found. If the user would rather not change the project, offer route B instead. Never substitute silently and never add to a project without a yes.
8. On a yes: `update_web_project` with the project's name and the approved eligible `fontId`s (it returns the refreshed CSS), or `get_asset_details` again if nothing needed adding. Then stage 5.

## Stage 4B: create a new project

1. **Search each face** with `font_search_by_name`. `exactMatch` → take its `fontId`. `fallbackApplied` → pick the row matching the requested style, or propose the closest style. No exact match → present the closest candidate and get consent. Nothing → `Not found`, never substitute another family. Apply the eligibility rule before anything goes further.
2. **Check production** with one `get_production_fonts_with_filters` call over every accepted `fontId`.
3. **Show the status table:**

   | Guideline font | Match in Monotype inventory | In Production |
   |---|---|---|
   | Söhne Buch | Söhne Buch (exact) | ✅ |
   | Gill Sans Regular | Gill Sans Nova Regular (closest - awaiting approval) | ✅ |
   | Foundry Gridnik | Not found | Not found |

4. **Name and check.** Use the name the user gave, or suggest one from the brand or site ("Acme Web Fonts") and ask once. Check it with `get_web_projects_by_name` (`assetTypes` = `["WebProject"]`, or `["DigitalAd"]` only when the user asked for a digital ad; `accessTypes` unset). An exact match means the name is taken: offer to reuse or add to that asset, or ask for another name.
5. **Creation summary and consent.** Show type, name, the faces to include and any production warnings, then ask `Create this web project now?`. Fold substitution approvals and production warnings into this one message. Wait for an explicit yes.
6. **Create** with `create_web_project` and the approved eligible `fontId`s. Its CSS / CDN URL is the licensed source for stage 5. Report a failure in the tool's own words; never claim success without a successful call this turn.

## Stage 5: output (licensed fonts + on-brand tokens)

Apply both. The right fonts on an off-brand page is still a failure.

**Fonts, licensed source only:**
- Include the project's published CSS link, CDN URL or `@font-face` rules exactly as the connector returned them in this turn.
- Every `font-family` references only the approved faces. No Google, Adobe, Bunny or other stylesheet links; no `@font-face` pointing at a URL the connector did not return; no invented names; no face that did not pass the table.
- Generic keywords (`sans-serif`, `serif`, `monospace`, `system-ui`) are fine as terminal fallbacks.
- Keep licensed type live as text: no PNG / JPG renders, screenshots or raster mockups of it, and no outlines or paths. If the user asks for an image, offer the code that renders it with the connector's CSS.

**Tokens, applied directly, no connector and no consent:**
- Use the extracted palette for its roles and components, including hover, active, disabled and focus states. Do not fall back to generic defaults where the guideline specified a colour.
- Apply the component patterns and global scales so components look like the brand's, not framework defaults.
- Define them as reusable tokens (CSS custom properties on `:root`, a theme object, or a Tailwind config) and reference those rather than scattering raw values. Where the guideline is silent, use sensible neutral defaults and say so.

## Communicating with the user

- Say what you are doing in plain terms ("checking your Monotype inventory", "reading the Acme Site project", "checking production status"). Do not recite tool names in chat.
- On stage 0, say up front that you are checking production before building, so the gate is visible.
- Ask one specific question per decision: first the route, then within the route the combined approval, then the name. Show requested versus matched so the user can decide.
- The status table comes before the question it informs, with the guideline's own font names in the first column and project names, never ids. A hash in chat means a step was skipped.
- When a font is left out, name it and say you are proceeding without it. Do not substitute.

## Invariants

- The user picks the route (reuse or create) unless the prompt already did; run only that route's tools.
- Stage 0 always checks production before generating anything and never changes the project.
- Font identity comes from written names or file metadata; visual identification is a last resort and its results are guesses the user confirms.
- One `font_search_by_name` per distinct face; one batched production check with `fontIds`, never style ids.
- Only eligible fonts go into a project: Monotype Fonts, or imported WOFF / WOFF2 files.
- Every substitution, every `update_web_project`, every `create_web_project` and every non-production font needs an explicit yes after the status table.
- Check the name with `get_web_projects_by_name` before creating; never create a duplicate silently.
- Only the connector's CSS / CDN URL from this turn is a font source; never fabricate a project, URL or `@font-face`.
- Colours, spacing and patterns never go through the connector or the consent gate.
- Never pass a name to `font_search`; never show ids or hashes in chat; never rasterise or outline licensed type.
