---
name: monotype-font-licensing
description: "Licenses every font in web or design output by routing it through the Monotype Enterprice MCP connector before code is emitted. Extracts typography from brand-guideline input (file, prompt font name, or Brand Guideline Skill), matches each font against My Library, resolves which web project holds it, checks production status, gets consent, and emits only licensed CSS (WebProject published CSS or CDN URLs). Handles every presence case — all fonts in one project, none in any project, split across projects, partial presence, a font in several projects, a missing weight/style — each with the right breakdown, consent gate, and remediation (e.g. create a web project and add the fonts). Triggers whenever Claude is about to produce HTML, CSS, React, JSX, Vue, SVG with text, email templates, artifacts, mockups, landing pages, posters, or any output referencing a named font — even when fonts are unmentioned or in a brand guideline. Use it whenever a font-family declaration, @font-face rule, or font name appears."
---

# Monotype Font Licensing

## Why this exists

When Claude generates web or design content, it is easy to reach for any font name that comes to mind — "Helvetica", "Inter", "system-ui", whatever — and to stitch in a Google Fonts link or a system fallback. For users on the Monotype Enterprice MCP platform, this is a problem: the only fonts they are licensed to ship are the ones in their enterprise **My Library**, and the only legitimate web CSS for those fonts is the source the connector itself returns — the CDN URL(s) reported by `validate_font_for_design`, or the published CSS path of a WebProject collection read via `get_asset_details`.

This skill enforces that pipeline end to end. Typography is extracted from whatever brand input the user provides, each font is matched against My Library, production readiness is validated, the user consents, and only the connector's CSS source is emitted. The connector is the single source of truth for what is licensed and for the CSS that ends up in the output. Anything the connector cannot cover is dropped.

## The connector's tools

The Monotype Enterprice MCP connector exposes these tools. The pipeline below maps directly onto them, so it helps to know what each one takes and returns:

- **`validate_font_for_design(fontName, guidelineFontName?, assetId?, assetType?)`** — the name-based library search, and the **single call that establishes both status flags**. Pass a real font name in `fontName`. With no `assetId`, it searches the user's **full** licensed library; pass `assetId` + `assetType` to scope the check to **one collection** (a specific web project, digital ad, etc.). Internally it already runs `font_search(fontId)` and `get_production_fonts_with_filters`, so a single call returns everything the pipeline needs: availability, **CDN URL(s)**, a **`fontId` (MD5 hash)**, the **`styleId`**, the **`inProduction` flag**, licensed alternatives when the exact font is not available, and the **collection(s) that contain the font** when the connector reports them. This is how a font enters the pipeline, how it earns **both In Library and In Production**, and how you learn which web project(s) / digital ad(s) hold it. There is **no need to call `font_search` or `get_production_fonts_with_filters` separately** for the normal flow.
- **`font_search(fontId)`** — a **fallback** id resolver, **not** a name search. `validate_font_for_design` already resolves the `styleId` for you, so you only reach for this when validation didn't surface a `styleId` (or you need the extra metrics/classification metadata it returns). It accepts **only** the MD5 `fontId` returned by `validate_font_for_design` (or `get_asset_details`).
- **`get_production_fonts_with_filters(...)`** — a **fallback** production filter. `validate_font_for_design` already reports each font's `inProduction` flag, so you only reach for this when validation didn't surface production status, or when you deliberately want to query production fonts in bulk. Key it on `styleIds` (or `pimStyleIds`, or `searchTerm` as a last resort) and read each returned row's **`inProduction`** flag. The `sources` filter accepts `WEBPROJECT`, so you can narrow to fonts used through web projects.
- **`get_asset_details(assetId, assetType)`** — single-asset lookup (a direct lookup, not a tree traversal). For an `assetType` of `WebProject`, it returns the collection's **published CSS path** plus child font details (family, style, weight, stretch, subset). Use it to read exactly which fonts a given web project contains and to obtain that project's stylesheet. `assetType` is one of `WebProject`, `DigitalAd`, `FontSet`, or `Folder`.

**Collections are the unit of web licensing.** A licensed library entry is not automatically web-deliverable — for web output the legitimate CSS comes from a **WebProject's published CSS** (or the per-font CDN URLs). So a font's real status has three parts: is it **In Library** at all, is it **In Production**, and **which web project(s) / digital ad(s) contain it** (none, one, or several). The scenarios in *"Presence scenarios"* below all turn on that third part.

There is no "browse my whole library" or "list all my web projects" tool, and no "mint a new web kit" call. Determine collection membership from what `validate_font_for_design` returns for the full-library check, by scoping that call to a known `assetId`/`assetType`, or by reading a known collection with `get_asset_details`. Licensed CSS comes only from what `validate_font_for_design` and `get_asset_details` already return — never fabricate a project, a kit, or a URL.

## The pipeline

Every run follows this flow. Do not skip stages, and do not emit any CSS before the user has approved the status table (stage 5).

```
1. Brand guideline input    → PDF / DOCX / image upload, font name in prompt, or Brand Guideline Skill
2. Typography extraction    → enumerate every font (family + weight + style), capture any IDs
3. Match against My Library → validate_font_for_design by family name; capture fontId + styleId + CDN URL + inProduction flag; set In Library AND In Production
3b. Resolve collections     → determine which web project(s) / digital ad(s) hold each font (none / one / several)
4. Confirm production       → read the inProduction flag validate_font_for_design already returned (fallback: get_production_fonts_with_filters by styleIds only if validation didn't surface it)
5. Status table + consent   → present per-font status (In Library + web project + In Production) and the matching presence scenario; get explicit approval for EVERY font, and consent to combine multiple projects' CSS when needed
6. Licensed CSS source      → collect each approved font's WebProject published CSS (get_asset_details) and/or CDN URLs
7. CSS returned in chat     → emit only the connector's CSS / CDN, nothing else
```

The branching for "no project", "split across projects", "partial presence", "font in several projects", and "weight/style missing" is spelled out in **Presence scenarios** below; stages 3b, 5, and 6 carry it out.

## When this applies

Trigger on any output that contains, references, or implies a named font, and on any request to turn a brand guideline into web typography. That includes:

- HTML pages, CSS files, style blocks
- React, JSX, TSX, Vue, Svelte components — anything with a font-family declaration or styled text
- SVG with text elements where a font is specified
- Email templates, marketing pages, landing pages
- Frontend artifacts and design mockups, including any rendered visual where text styling matters
- Posters, social cards, and other design deliverables that name a typeface
- A brand guideline supplied as a PDF, DOCX, or image whose typography section needs to become licensed CSS

If the user asks for "a clean blog layout" without naming a font, you still need to pick fonts — those picks enter the pipeline at stage 2 before they appear in any code.

## Stage 1 — Brand guideline input

Establish where the font requirements are coming from. Accept any of these sources (a single run may combine several):

- **Uploaded brand guideline** — a PDF, DOCX, or image the user attached. The typography lives inside it.
- **Font name(s) in the prompt** — the user typed one or more font names directly, optionally with IDs.
- **Brand Guideline Skill** — if a separate brand-guideline skill is available and the user points at it, take its typography output as the input here.
- **Defaults you would otherwise reach for** — if the user named no fonts (e.g. "make me a landing page"), the headings/body/mono fonts you would pick are themselves the input and must go through the pipeline.

Confirm the Monotype Enterprice MCP connector is connected before going further. If it is not, surface that to the user and offer to connect it. Do not silently fall back to Google Fonts, Adobe Fonts, system fonts, or any other source.

## Stage 2 — Typography extraction

Turn the input into an explicit, exhaustive list of fonts. For uploaded files, read the document/image and pull every typeface named in its typography section (display, heading, body, captions, mono, etc.). Be explicit and complete — include:

- Fonts named directly by the user or by the brand guideline
- Fonts implied by templates, screenshots, or styles the user pointed at
- Fonts you would have reached for as defaults (headings, body, monospace, etc.)
- **Each weight and style as a separate entry** — "Inter Regular", "Inter Bold", "Inter Italic" are three entries. A guideline that says "Inter 400/600/700" is three entries.

**Capture any IDs attached to a font, and note which kind of id it is** — the Monotype Enterprice MCP connector uses two distinct identifiers. Both are now returned by `validate_font_for_design` itself, but a font may also arrive carrying one already, which is still worth recording:

- A **32-hex MD5 string** is a **`fontId`**. It's the handle `validate_font_for_design` returns (and the input the fallback `font_search` accepts). Recording one a font already carries is useful for cross-checking, but you no longer route around `validate_font_for_design` because of it — validation is the single call that resolves library, style, and production status together.
- A **numeric / UUID style id** is a **`styleId`** (or a **PIM style id** when the user labels it that way). `validate_font_for_design` returns this too, having run the production check internally. If a font arrives already carrying a `styleId` and validation somehow doesn't surface production status, the fallback `get_production_fonts_with_filters` accepts it directly via `styleIds` / `pimStyleIds`.

If an id's type is ambiguous, treat a 32-character hexadecimal value as a `fontId` and anything else as a `styleId`. Either way, the font still goes through `validate_font_for_design` by name — let that one call resolve library, style, production, and CDN together, and treat any pre-existing id as a cross-check rather than a way to skip validation.

## Stage 3 — Match against My Library (In Library)

For each extracted font, search the enterprise **My Library** with **`validate_font_for_design`**. Because this one call now runs the style resolution and production check internally, it sets **both** status flags at once:

- **In Library** — did `validate_font_for_design` return the font as available, or return a licensed alternative for it? (Yes/No)
- **In Production** — does the returned `inProduction` flag mark the matched entry as production-ready? (Yes/No)

**Always pass the high-level family name in `fontName` — never the exact style name.** Pass `Helvetica Neue`, not `Helvetica Neue Bold`; `Inter`, not `Inter Regular`/`Inter Bold`/`Inter Italic`; `Roboto`, not `Roboto Medium Italic`. Carry the specific requested face (the styled name from the guideline) in **`guidelineFontName`** so the connector can still flag a mismatch, but the search key itself stays at the family level. The reason is robustness: a family-level query reliably surfaces the family and its licensed alternatives, whereas a styled query routinely returns nothing because the connector names weights differently ("75 Bold" vs "Bold", "Regular" vs "Roman", "Inter Tight" vs "Inter"). Do not require a character-for-character hit to count as a match — `validate_font_for_design` returns the nearest licensed face (its "licensed alternatives") even when casing, spacing, weight naming, or family-vs-family-plus-style differ.

Because the call is keyed on family, **the styles you split out in stage 2 collapse to one `validate_font_for_design` call per family** — validate `Inter` once, not three times for Regular/Bold/Italic. Then resolve the individual requested weights/styles against the styles the family returns (and against stage 4), recording the requested-vs-matched discrepancy per style for stage 5.

**Capture everything the validation returns and carry it forward:**

- the **`fontId` (MD5)** and **`styleId`** — the handles the fallback tools accept, and useful for cross-checking,
- the **`inProduction` flag** — sets In Production directly (see stage 4; no separate production call needed), and
- the **CDN URL(s)** — the licensed CSS source stage 6 will assemble from.

When the input named no specific font (e.g. "make me a landing page"), pick the candidate fonts you would otherwise reach for, then run each pick through `validate_font_for_design` the same way — keyed on the family name. There is no separate "browse the whole library" tool; you propose candidates and validate them. If a font carries a usable id already (stage 2), record it as a cross-check, but the `validate_font_for_design` call is still what resolves library, style, production, and CDN together.

**Whatever `validate_font_for_design` returns is a *suggested closest match*, never an automatic selection.** Record both what was requested and what the connector matched. Each font resolves to one of these:

- **In Library — match or alternative found** — validation reported the font available, or returned a licensed alternative for it. In Library = Yes. Carry it to stage 5 as a *suggestion only*. Do not treat even an exact-looking match as pre-approved — the user still confirms it at stage 5.
- **Not in Library** — validation returned neither the font nor any licensed alternative. In Library = No (and In Production = No; there is nothing to production-check, so skip stage 4 for this font).

Treat each weight/style independently. If the user asked for "Inter Bold" and the closest licensed candidate is "Inter Regular", record "Inter Regular" as the suggested match for "Inter Bold" (In Library = Yes) and surface that requested-vs-matched discrepancy at stage 5 — never silently accept it as the same font.

## Stage 3b — Resolve collection membership (which web project holds each font)

Being licensed in My Library tells you a font *can* be used; for **web output** you also need to know **which web project (or digital ad) actually contains it**, because the deliverable CSS comes from a project's published stylesheet (or the per-font CDN URLs). Resolve, per font, the set of containing collections — it will be **none**, **one**, or **several**.

How to determine membership, in order of preference:

- **Read what `validate_font_for_design` already returned.** The full-library check in stage 3 reports the collection(s) that contain the font when the connector knows them. Capture every web-project / digital-ad id and name it returns for the font.
- **Scope the check to a known collection.** If the user named or selected specific web project(s) — or you already hold their `assetId`s — call `validate_font_for_design(fontName, assetId, assetType:"WebProject")` (or `"DigitalAd"`) per collection to confirm presence there.
- **Read a known project directly.** For any web-project `assetId` you have, call `get_asset_details(assetId, assetType:"WebProject")` and check its child fonts; this also yields that project's **published CSS path** for stage 6.

Record, for each font: the containing web project(s)/digital ad(s) by id and name (possibly empty), and for each such web project its published CSS path once known. This collection picture — combined with In Library and In Production — is what selects the right **Presence scenario** below and drives the stage-5 table.

If you cannot establish any collection membership and the user has not pointed at a project, do not guess: treat the font as "not in any web project" for that font's row and let the scenario logic handle it (it will surface the gap and the remediation rather than silently substituting).

## Presence scenarios — how to resolve each case

Every run resolves to one or a mix of the cases below (different fonts in the same guideline can land in different cases — handle each font on its own row). Identify the case from In Library + collection membership + the requested weight/style, then follow the matching path. **No path ever silently substitutes a font or invents CSS, and every font that ends up in the output is confirmed by the user at stage 5.**

### Scenario 1 — All fonts are present in a web project and production-ready (happy path)
Every requested font (right family, right weight/style) is In Library, sits in a web project, and is In Production.
- Resolve each font's web project and capture that project's published CSS (or the CDN URLs).
- At stage 5 still show the per-font table and confirm, but this is the clean case: requested = matched, In Library = Yes, In Production = Yes, one project. Once confirmed, emit the exact requested fonts with **no substitutions and no missing/unlicensed warnings**.

### Scenario 2 — None of the fonts are in any web project or digital ad
`validate_font_for_design` finds no web project and no digital ad containing the requested fonts (whether or not the family exists elsewhere in the library).
- Do **not** substitute and do **not** fabricate a project or CSS.
- Tell the user plainly that the font isn't available in a web project in their library, and give the concrete remediation steps:

      "I couldn't find `[font(s)]` in any web project or digital ad in your Monotype library, so there's no licensed web CSS I can pull for them. To make them available:
       1. Go to My Library.
       2. Create a web project.
       3. Add the font(s) to that web project (and publish it).
      Once that's done, tell me and I'll pull the licensed CSS and build this for you."

- Ask whether to wait for them to set up the project, or to proceed using only generic CSS fallbacks (`sans-serif`/`serif`/`monospace`) in the meantime.

### Scenario 3 — Fonts exist but are split across different web projects
e.g. the heading font lives in Web Project A and the body font in Web Project B.
- Locate each font in whichever project holds it; via `get_asset_details` retrieve the **published CSS for both** Web Project A and Web Project B.
- Surface this to the user and **get consent to use both projects' CSS together**:

      "Your fonts are spread across two web projects: `[Heading font]` is in `[Web Project A]` and `[Body font]` is in `[Web Project B]`. I can pull the published CSS from both projects so the page uses every font from your guideline. Want me to include both stylesheets?"

- **If yes:** include every font from the guideline, each from its own project's licensed CSS — **no font is dropped just because it lived in a separate project.** Combine the stylesheets in the output.
- **If no:** don't silently combine. Ask the user to create a single web project containing all the appropriate fonts, then come back so you can pull one unified stylesheet.

### Scenario 4 — Partial presence (some fonts in a web project, some not)
At least one requested font is in a web project (e.g. `Avenir`) and at least one is in no web project (e.g. `Custom Display Font`).
- Check each font **individually**. Provision the ones that are in a web project (resolve project + CSS). Flag the ones that are in no project.
- Give the user a clear, named breakdown so they know exactly what's missing:

      "Here's what I found:
       • Found & ready (in a web project): `[font A]`, `[font B]`
       • Not in any web project: `[font C]`
      I can proceed with the available fonts now, or you can add the missing one(s) to a web project first (My Library → create/open a web project → add the font → publish) and I'll include everything."

- Ask which path they want: **proceed with the available web-project fonts** (the missing ones are left out, never substituted), **or** pause so they can add the missing fonts to a web project. Don't decide for them.

### Scenario 5 — A single font sits in multiple web projects at once
The same font is found in more than one web project.
- Do **not** pick a project on the user's behalf. List the projects the font appears in and ask which to pull the CSS from:

      "`[Font]` is published in more than one web project: `[Project A]`, `[Project B]`. Which project's CSS should I use?"

- Use only the project the user confirms. Carry that choice into stage 6.

### Scenario 6 — Right family present, requested weight/style missing
e.g. the project has `Brand Sans Regular` but not `Brand Sans Bold`.
- Tell the user the family was found but the specific weight/style isn't in the project, and offer a sensible path forward — don't silently swap a different weight in:

      "I found `[Family]` in `[web project]`, but the `[Bold]` weight you asked for isn't in that project — only `[available weights]` are. You can either use an available weight, or add `[the missing weight]` to the web project (My Library → open the project → add the weight → publish) and I'll include it. How would you like to proceed?"

- If the user chooses an available weight, treat that as the confirmed match for that row at stage 5 (show requested-vs-used). If they choose to add the weight, pause that font until it's in the project.

These scenarios are not mutually exclusive within a single run — resolve each font to its own case, then bring the whole picture to the single stage-5 table.

## Stage 4 — Confirm production-marked status (In Production)

Being in My Library is not enough to ship a font — the library entry must also be **marked for production**. The good news is that `validate_font_for_design` already ran the production check internally during stage 3, so for the normal flow you don't make a separate call here: you **read the `inProduction` flag the validation response already returned** and record the second status flag.

- **In Production** — is the matched library entry marked for production? Read it straight from the `inProduction` flag captured in stage 3. (Yes/No)

A font is **In Production = Yes** only when the validation response reports `inProduction` `true` for the matched entry. If it reports `false`, set **In Production = No**. Fonts that came back **Not in Library** in stage 3 are automatically **In Production = No** — there's nothing to production-check.

**Fallback (only if validation didn't surface production status).** If for some reason the `inProduction` flag is missing from the validation response — or a font arrived already carrying a `styleId` and you want to confirm it directly — resolve it with the standalone tools:

- Use the **`styleId`** from the validation response (or one the font carried in from stage 2). If you somehow lack a `styleId`, resolve it with `font_search` — it takes the **`fontId` (MD5)** captured in stage 3 (pass `{"fontId": "<md5>"}`) and never a family name or styled string.
- Pass the `styleId`(s) to **`get_production_fonts_with_filters`** via `styleIds` (a CSV or array; use `pimStyleIds` for PIM ids) in a **single** batched call, and read each row's `inProduction` flag — don't call once per font.
- **`searchTerm` is a last resort only**, used when no `styleId` can be resolved at all. It's a substring match, so key it on the **family** substring (e.g. `Helvetica Neue`), not the full styled string, then pick the row whose weight/style matches the request.

Do not infer production status from the library match alone. The `inProduction` flag is the authoritative signal — whether it comes from `validate_font_for_design` (the normal case) or the `get_production_fonts_with_filters` fallback — and it is what separates a production-ready font from one that merely sits in the library. (Production-ready still does not mean auto-approved; every font is confirmed by the user at stage 5.)

### Final status per font

Each font now carries three things — **In Library** (stage 3), **web project membership** (stage 3b), and **In Production** (stage 4) — and from them lands in a status plus a presence scenario. **No font is ever auto-approved**: every In-Library match is a suggested closest match that the user must confirm at stage 5. By the end of stage 5 each font is either consented-in (with its chosen project/weight) or left out.

| Status | In Library | Web project | In Production | What happens |
| --- | --- | --- | --- | --- |
| **In a web project, production-ready** | Yes | One/known | Yes | Closest licensed match, production-ready, web-deliverable (Sc. 1). Still requires explicit consent — never auto-included. Show requested vs. matched. |
| **In a different / additional project** | Yes | Another or several | Yes | Font sits in a separate project (Sc. 3) or in several (Sc. 5). Confirm whether to combine CSS / which project. |
| **Family found, weight/style missing** | Yes (family) | Yes (family) | Yes | Requested weight/style not in the project (Sc. 6). Offer an available weight or the add-to-project path. |
| **In library but not production-marked** | Yes (match/alternative) | Maybe | No | Found but not production-marked. Requires explicit consent; show requested vs. matched and flag the production gap. |
| **Not in any web project** | Yes or No | None | — | No web project (and possibly no library match) holds it (Sc. 2 / Sc. 4). Name it and give the create-a-web-project remediation; never substitute. |

A single run frequently mixes several of these rows — resolve each font on its own and bring the combined picture to stage 5.

## Stage 5 — Status table + consent (single gate)

Present **one** status table covering every extracted font, then get the user's explicit approval before emitting anything. This is the only consent gate in the pipeline — do not scatter separate yes/no asks. **Every font that would appear needs an explicit yes here — including production-ready closest matches.** The family-name validation only *suggests*; the user decides. Nothing is pre-approved or auto-included.

Show a row per font with at least these columns. The **Web project** column is what ties each row to the presence scenario it fell into:

| Font requested | Closest match in My Library | In Library | Web project(s) | In Production | Status / Action |
| --- | --- | --- | --- | --- | --- |
| `Helvetica Neue Bold` | `Helvetica Neue LT Std 75 Bold` | Yes | `Brand Web` | Yes | Needs consent — in a web project, production-ready (Sc. 1) |
| `Avenir Next` | `Avenir Next` | Yes | `Project A` | Yes | Needs consent — confirm project (Sc. 1) |
| `Body Serif` | `Body Serif` | Yes | `Project B` | Yes | Needs consent — different project; combine CSS? (Sc. 3) |
| `Display One` | `Display One` | Yes | `Project A`, `Project C` | Yes | Needs consent — in 2 projects; which one? (Sc. 5) |
| `Brand Sans Bold` | `Brand Sans Regular` (family only) | Yes | `Project A` (Regular only) | Yes | Family found, `Bold` weight missing — use available weight or add it (Sc. 6) |
| `Custom Display Font` | — | No | — | No | Not in any web project — add to a web project or proceed without (Sc. 4 / Sc. 2) |

Annotate each row with the scenario it matches so the user sees the whole picture at once (all-present, split projects, multiple projects, partial, missing weight, or not in any project).

Then ask one clear question per decision, showing exactly what was requested versus what the connector returned — even when they look identical, since the search is fuzzy and the user has the final say. Tailor the question to the row's scenario:

    "You asked for `Helvetica Neue Bold`. The closest match is `Helvetica Neue LT Std 75 Bold`, it's in your `Brand Web` project and production-ready. Use it?" (Sc. 1)
    "`Body Serif` lives in a different web project (`Project B`) than your heading font (`Project A`). Shall I pull the published CSS from both so every font is included?" (Sc. 3)
    "`Display One` is published in `Project A` and `Project C`. Which project's CSS should I use?" (Sc. 5)
    "I found `Brand Sans` in `Project A`, but the `Bold` weight isn't in that project — only `Regular`/`Medium`. Use an available weight, or add `Bold` to the project first?" (Sc. 6)

For matches that are in My Library but not production-marked, say so and ask whether to include the font anyway.

Wait for an explicit yes in chat. Do not infer consent from the overall request, prior turns, or urgency. When a decision involves combining multiple web projects' CSS (Sc. 3) or choosing among projects (Sc. 5), that choice is part of this same gate — capture it here, don't defer it to stage 6.

- **Consented rows** → join the **approved set** (with their chosen project/weight recorded).
- **Declined / no response rows** → dropped.
- **Rows with no web project** → not provisionable as web CSS; handle per Scenario 2 / 4 — name them and offer the remediation, never substitute.

When a font isn't in any web project, be explicit and give the path to fix it rather than just dropping it silently:

    "`[font(s)]` aren't in any web project in your library, so there's no licensed web CSS for them. Add them via My Library → create/open a web project → add the font → publish, then I'll include them. Want me to proceed with the available fonts in the meantime, or wait?"

If the approved set is empty after this gate, surface it — ask whether to proceed using only generic CSS fallbacks (`sans-serif`, `serif`, `monospace`) or to wait for the user to set up a web project. Do not refuse the whole task merely because some fonts weren't in a project.

## Stage 6 — Licensed CSS source

Once the approved set is final, assemble the licensed CSS source. The Monotype Enterprice MCP connector does **not** mint a new web kit on demand — the legitimate CSS is whatever the connector already returned for the approved fonts:

- **A WebProject's published CSS path via `get_asset_details`** — for fonts that live in a web project, call `get_asset_details(assetId, assetType: "WebProject")` and use the returned **published CSS path** as the stylesheet. This is the preferred source for web output.
- **CDN URL(s) from `validate_font_for_design`** — the per-font CDN URL(s) captured in stage 3 are the licensed source when there's no web-project CSS to use.

Honor the scenario decisions made at stage 5:

- **Single project (Sc. 1):** pull that one project's published CSS and use it for the whole set.
- **Split across projects, user said combine (Sc. 3):** call `get_asset_details` for **each** approved project and include **all** their published stylesheets so no font is dropped. If the user declined to combine, you should already have paused for them to build one unified project — don't stitch projects together without the yes.
- **Font in multiple projects (Sc. 5):** use only the single project the user chose.
- **Available weight chosen (Sc. 6):** use the project CSS that contains the chosen weight; reference that weight, not the missing one.

Capture exactly what the connector returns — the published CSS path(s), the CDN URL(s), or the at-font-face rules within them. This output is the only legitimate font source for the generated content. Prefer a single published WebProject stylesheet when one covers the whole approved set; otherwise include each approved project's stylesheet (Sc. 3) or the collected CDN URLs. Do not fabricate a kit, a URL, or an at-font-face rule the connector did not return.

## Stage 7 — CSS returned in chat

When you write the deliverable (HTML, CSS, JSX, SVG, email, whatever) and return the CSS:

- Include the connector's CDN link(s), published WebProject CSS path(s), or at-font-face rules **exactly** as the connector produced them. When fonts came from more than one approved web project (Sc. 3), include **every** approved project's stylesheet — don't drop one.
- Every `font-family` declaration must reference only fonts from the approved set.
- Do not add stylesheet link tags pointing at Google Fonts, Adobe Fonts, Bunny, or any other source.
- Do not declare at-font-face rules pointing at any URL the connector did not return.
- Do not invent font names, and do not add requested fonts that did not pass the pipeline.
- Generic CSS keywords (`sans-serif`, `serif`, `monospace`, `system-ui`) are acceptable as terminal fallbacks in a stack, but the primary face must come from the connector's CDN URL / published CSS.

## Communicating with the user

- In user-facing chat, talk in terms of what you're doing ("checking My Library", "validating production status") and refer to "the Monotype Enterprice MCP connector" and "My Library" generically, rather than reciting raw tool identifiers — it reads more clearly. This concerns the chat copy only; the stage instructions above deliberately name the specific tools to call (`validate_font_for_design` for library/style/production, `get_asset_details` for collection CSS, with `font_search` / `get_production_fonts_with_filters` as fallbacks) so the pipeline stays deterministic.
- Keep the status table front and center at stage 5. It is the user's main decision surface; don't bury it in prose.
- When asking for consent, be specific and ask one direct question per decision. Name the requested font and the matched candidate.
- Name web projects in plain language when a font is split across projects, sits in several, or is missing from all of them — the user thinks in terms of "My Library" and "web projects", so frame the choices and the remediation ("create a web project, add the fonts, publish") that way.
- When a font isn't in any web project, give the concrete fix-it steps rather than only stating it's unavailable.
- When dropping or omitting fonts, list them and say you're proceeding without them. Don't editorialize and don't substitute.

## What this skill never does

- Never uses a font CDN, font file, or at-font-face source the connector did not produce.
- Never assumes consent. **Every matched font — including the closest match found by the family-name validation and any production-ready one — requires an explicit yes in chat, captured in the stage-5 table approval.** No font is ever auto-included.
- Never auto-selects or commits to a closest match on the user's behalf. The fuzzy family-name validation only *suggests* the nearest licensed face; the user always decides at stage 5.
- Never recommends or silently substitutes an alternative font when one is dropped.
- Never picks a web project for the user when a font sits in more than one (Sc. 5), and never combines two projects' CSS without an explicit yes (Sc. 3).
- Never drops a font merely because it lives in a different web project than the others — split fonts are pulled from their own projects once the user consents (Sc. 3).
- Never silently swaps in a different weight/style when the requested one is missing — it surfaces the gap and offers an available weight or the add-to-project path (Sc. 6).
- Never leaves a "not in any web project" font as a silent omission — it names the font and gives the My Library → create web project → add font → publish remediation (Sc. 2 / Sc. 4).
- Never calls `font_search` or `get_production_fonts_with_filters` separately in the normal flow — `validate_font_for_design` already resolves the `styleId` and production status in one call. They remain available only as fallbacks. And never passes a family name or styled string to `font_search`; that tool takes only the MD5 `fontId`.
- Never recites raw connector tool identifiers in user-facing chat (the stage instructions name them for Claude's own use; the chat copy stays in plain capability terms).
- Never emits CSS before the stage-5 status table has been approved for every font that would appear.
