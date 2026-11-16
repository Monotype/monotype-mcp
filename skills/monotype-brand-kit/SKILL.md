---
name: monotype-brand-kit
description: "Licenses every font in web/design output via the Monotype Enterprice MCP connector. From a brand guideline it extracts typography plus colors/patterns, then routes to check-existing-library or create-new-project, validating or searching fonts, confirming production, and provisioning a web project as needed. If the user instead names an existing web project as the font source (e.g. \"using fonts in my web project X\"), it always checks that project's fonts' production status before generating any content. Emits only licensed CSS and applies extracted colors/patterns to stay on-brand. Triggers whenever Claude is about to produce HTML, CSS, React, JSX, Vue, SVG with text, email templates, mockups, landing pages, or any output referencing a named font or a named web project."
---

# Monotype Brand Kit

## Why this exists

Users on the Monotype Enterprice MCP platform may only ship fonts from their enterprise **My Library**, and the only legitimate web CSS for those fonts is what the connector returns — a WebProject's published CSS (from `get_asset_details` or `create_web_project`), or the CDN URL(s) from `validate_font_for_design`. Reaching for a Google Fonts link, a system fallback, or an invented `@font-face` breaks the license. So the connector is the single source of truth: match each font against My Library, confirm production, provision a web project, and emit only the connector's CSS — never fabricate a project, kit, or URL, and never substitute a font the user didn't approve. Colors and component patterns from the guideline are the user's own tokens (not licensed), so they bypass the connector and are applied directly to keep the output on-brand.

## The connector's tools

- **`start_font_workflow(fontNames, outputType?, hasBrandGuideline?, route?)`** — the **entry step**. Call it first (no `route`) to ask the user which path they want; after they answer, call it again with `route` = `check_library` or `create_new_project` and follow that branch. This choice decides the whole run — it is not auto-detected.
- **`validate_font_for_design(fontName, guidelineFontName?, assetId?, assetType?)`** — the licensing check for **Route A only**, keyed on **family** name. One call returns availability, whether the match is **exact** or a licensed **alternative**, the **`inProduction`** flag, the **web project(s)** that contain the font, the **`fontId`** (MD5), **`styleId`**, and **CDN URL(s)**. Do **not** call it on Route B.
- **`font_search_by_name(name, pageSize?, pageNumber?, partial?)`** — catalog search by **name** (partial match) for **Route B**. Returns matching fonts with `fontId` + `styleId`; used to find each requested font and its closest match.
- **`get_production_fonts_with_filters(styleIds|pimStyleIds|searchTerm, ...)`** — production verifier; read each row's `inProduction`. Use it to confirm production for the fonts chosen in the non-happy-path branch (or whenever validation left production unverified). Batch all styleIds into one call.
- **`get_asset_details(assetId, assetType:"WebProject")`** — the **id → name resolver**, and the CSS reader. One call on a project id returns its **`name`**, its **`publishedCSSPath`**, and its child fonts. Use it to turn every project **id** that validation returns into the **name** the user recognises, and to pull the stylesheet when reusing a project. Never display an id where a name belongs.
- **`get_web_projects_by_name(projectName?, assetTypes?, page?, pageSize?)`** — browses/searches the user's projects when they ask to see their library ("show my web projects"). It is **not** the resolution path for the status table — resolve ids with `get_asset_details`.
- **`create_web_project(name, fontIds, description?)`** — creates a new web project from a font list, adds the fonts, and returns the project's **published CSS path / CDN URL(s)**. (If unavailable in the session, reuse an existing project instead.)
- **`font_search(fontId)`** — fallback id resolver; takes **only** an MD5 `fontId`, never a name. Reach for it only if you need a `styleId` that validation didn't surface.

## Pipeline

```
0. Direct project → if the user names an EXISTING web project as the font source ("using fonts in my
   reference        [ProjectName] web project"), skip extraction/routing entirely: resolve the project,
                     check EVERY font's production status, then go straight to Stage 5. See Stage 0.
1. Brand input   → PDF/DOCX/image upload, font name in prompt, brand-guideline skill, or your own default picks
2. Extraction    → fonts (family+weight+style) AND non-font tokens (colors per role/component, component/layout patterns)
                   └─ fonts come from names in the guideline/prompt, else the file's embedded font metadata — never a visual guess
                   └─ colors/patterns are NOT licensed: skip the connector, apply at stage 5
3. Route         → call start_font_workflow with the detected fonts; the user picks check_library (A) or create_new_project (B)
4. Provision     → ROUTE A: validate_font_for_design per font → resolve project ids to names (get_asset_details per id)
                   → status table → if all exact + production + one project, reuse it (else offer B).
                   ROUTE B: font_search_by_name per font → consent on closest → verify production → status table → create_web_project.
                   Each route reports a per-font status table before the user is asked to decide.
                   Whenever a project is created, suggest a name for the user to keep or change.
5. Output        → emit only the connector's CSS, and apply the extracted colors + patterns so it's on-brand
```

## When this applies

Trigger on any output that contains, references, or implies a named font, and on any request to turn a brand guideline into web typography: HTML/CSS/style blocks; React/JSX/TSX/Vue/Svelte or any styled-text component; SVG text; email templates, landing/marketing pages; frontend artifacts, mockups, and rendered visuals where text styling matters; posters, social cards, and other deliverables that name a typeface; and a brand guideline (PDF/DOCX/image) whose typography must become licensed CSS. If the user names no font and gives no guideline ("make me a clean blog layout"), the fonts you'd otherwise pick still enter the pipeline at stage 2; if a guideline *is* present, its fonts come from names-then-metadata, not from default picks. **Also trigger when the user names an existing web project as the font source instead of naming fonts** ("create a landing page using the fonts in my Acme Marketing Fonts project", "build this with fonts from my 10Aug web project") — this goes to Stage 0, not Stage 1.

## Stage 0 — Direct project reference (skip extraction/routing, always verify production first)

Some requests don't name fonts or a brand guideline at all — they name an **existing web project** and ask you to build with whatever fonts are already in it ("create a HTML page using fonts present in my web project 10Aug"). This is a distinct, fast path: there's nothing to extract and no A/B route to choose, because the font source is already decided. But "already in a project" does **not** mean "safe to ship" — a project can contain fonts that were removed from production, or that never finished onboarding. So this path still gates on production status; it just gates before generating anything, never after.

1. **Resolve the project.** Call `get_web_projects_by_name(projectName)` with the name the user gave. If more than one project matches, list them and ask which one. If none match, say so and ask the user to confirm the name or pick a project from `get_web_projects_by_name()` with no filter — never guess or fall back to a different project.
2. **Read its fonts.** Call `get_asset_details(assetId, assetType: "WebProject")` on the resolved project. This returns the project's child fonts (with `styleId`s) and its **`publishedCSSPath`** — keep the CSS path for Stage 5, but don't use it yet.
3. **Check production status before generating anything.** Batch every font's `styleId` into one `get_production_fonts_with_filters` call and read each row's `inProduction`. This check happens **before** any HTML/CSS/content is written — never emit output first and check after.
4. **Show a short status table** (font / in production ✅❌), same spirit as Stage 4's tables:

       | Font in [Project Name] | In Production |
       | --- | --- |
       | Söhne Buch | ✅ |
       | Gill Sans Nova | ❌ |

   - If **every** font is production-ready, proceed straight to Stage 5 using the project's `publishedCSSPath` — no need to re-ask, since the user already told you which project to use.
   - If **any** font isn't production-ready, name it, say it isn't production-ready, and ask whether to proceed without it or wait — don't silently drop it and don't silently include it.
5. Colors/patterns still apply if the user also gave a brand guideline alongside the project reference; otherwise Stage 5 runs fonts-only.

This stage never calls `validate_font_for_design`, `font_search_by_name`, `start_font_workflow`, or `create_web_project` — the project already exists and the fonts are already chosen; the only open question is whether they're production-ready.

## Stage 1 — Brand input

Establish where the requirements come from (a run may combine several): an uploaded guideline (PDF/DOCX/image — the source for the *whole* brand, not just font names); font name(s) typed in the prompt; a separate brand-guideline skill's typography output; or your own default heading/body/mono picks when the user names none. Confirm the Monotype Enterprice MCP connector is connected first; if not, say so and offer to connect it — never silently fall back to Google/Adobe/system fonts.

## Stage 2 — Extraction (typography + colors + patterns)

Pull two kinds of thing. **Only fonts continue to stage 3**; colors, patterns, and other brand elements are the user's own tokens — never run them through the connector — capture them here and apply at stage 5.

**Typography** — build the font list only from what the source actually states, never from how rendered text *looks*. Licensing the wrong family defeats the whole skill, and a font identified by eye is a guess, so work down this priority and stop at the first that yields fonts:

1. **Named in the guideline or prompt (preferred).** Fonts the user typed, or that the guideline names in its text ("Headings: Helvetica Neue 65 Medium", a type-spec table, a `font-family` in embedded CSS). Take these as written.
2. **File metadata, only when nothing is named.** If an uploaded guideline names no fonts in its text, read the file's embedded font information instead of interpreting its appearance:
   - **PDF** — list the embedded fonts (`pdffonts <file>`, or the pdf / pdf-reading skill). Use the reported family names.
   - **DOCX** — read the font references in the document's style XML (`word/styles.xml`, `word/fontTable.xml`, run-level `w:rFonts`). These are the fonts the file actually uses.

   Metadata reflects the fonts *used in the file*, which is the best available signal when nothing is named — but a name written in the guideline's text always wins over metadata if the two disagree.
3. **Never visually guess.** Do not name a font because rendered text in a screenshot, image, or flattened PDF *looks like* it. If the only source is an image (or a flattened/scanned PDF) that names no fonts in its text and carries no embedded font metadata to read, say so and ask the user which fonts to use — do not fill the gap by eye.

Only when there is **no** guideline and the user names nothing ("make me a clean blog layout") do your own default heading/body/mono picks apply — that default-pick path is unchanged. A guideline being present means the fonts come from it (named, then metadata), never from a guess.

Treat **each weight and style as a separate entry** ("Inter 400/600/700" is three).

**Non-font tokens** (a bare hex list isn't enough — capture what each is *for*):
- **Colors** — every named brand color with value and the guideline's own name (incl. neutrals, tints/shades, opacity), mapped to **semantic roles** (primary, secondary, accent, background/surface, heading vs body text, muted text, border/divider, link default/hover/visited, states success/warning/error/info) and **per-component** where specified (button bg/text/border + hover/active/disabled; card surface/border/shadow; header/nav; form inputs bg/border/focus/placeholder/error; links, badges, footer), noting light/dark. If a color has no stated role, record it and note the role is unspecified rather than guessing.
- **Component patterns** — buttons (shape, radius, padding, border, elevation, per variant/state); cards/surfaces (radius, border, shadow, padding, bg); header/nav; forms/inputs (shape, border, focus, label/helper/error); other elements (badges, tables, lists, dividers, alerts, modals); and **global layout tokens** (spacing scale, grid/gutters, breakpoints, container max-widths, radius scale, border widths, elevation scale).
- **Other elements** — logo usage/clear-space, iconography, imagery/illustration treatment, motion; voice/tone only if it affects copy you generate.

Record all of these as design tokens (name → value + the role/component each serves).

## Stage 3 — Route the run (the user chooses; don't auto-detect)

Call `start_font_workflow` with the fonts extracted at stage 2 (no `route`) to surface the fork, then ask the user which they want:

    "How should I source these fonts — (A) check your existing Monotype library, or (B) create a new web project for them?"

When the user answers, call `start_font_workflow` again with `route` = `check_library` (→ Stage 4A) or `create_new_project` (→ Stage 4B) and follow **only** that branch. The route is the user's decision — do not infer a "happy path" and skip the question, and do not run the other branch's tools.

**If `start_font_workflow` isn't available to you** (it's restricted per user), skip the routing question and run Stage 4A directly with `validate_font_for_design` — it is self-sufficient. If `create_web_project` is also unavailable, reuse an existing web project's CSS (via `get_asset_details`) and, when a font isn't in a project, give the user the My-Library remediation instead of creating one.

## Stage 4A — Route `check_library`: validate, then reuse (or offer B)

For each font, call `validate_font_for_design` **keyed on the family name**, never the styled name — pass `Helvetica Neue`, not `Helvetica Neue Bold`; carry the specific requested face in `guidelineFontName` so a mismatch can be flagged. Family-level queries reliably surface the family and its licensed alternatives; styled queries routinely return nothing because the connector names weights differently. Capture per font: **exact vs. fuzzy**, the **`inProduction`** flag, the **web project(s)** that contain it, and the **`fontId` / `styleId` / CDN URL(s)**.

**Resolve project ids to project names before you render anything.** `validate_font_for_design` identifies the containing web project(s) by **id** (a hash/UUID), not by name. An id is meaningless to the user and truncating it (`159cb613…`) makes it worse — they cannot tell which project it is, so they cannot answer the reuse question. So after validating and before building the table:

1. Collect the **distinct** project ids across every font's validation result — one font in six projects and six fonts in one project both come down to the set of unique ids.
2. Call **`get_asset_details(assetId, assetType: "WebProject")`** once per distinct id and read its **`name`**. Cache the result for the run: an id that appears on four rows is still one call, never four.
3. Keep the ids yourself — you need the assetId to reuse a project — but render **only names**. Project names are not unique (several projects can share one), so track the id ↔ name pair internally and, if the user picks a name that maps to more than one id, ask which one before reusing.

The same call also returns **`publishedCSSPath`** and the project's child fonts, so resolving the name doubles as fetching the stylesheet — if the user then reuses that project, you already have its CSS and don't need to call again.

If a project's details can't be read, show `Unnamed project` in that cell rather than pasting the hash, and say in chat that one project's name couldn't be resolved.

Once every font has been validated and every id resolved to a name, show a **status table** before you ask anything. The user is about to choose between reusing a project and creating one, and they can only judge that if they can see, per font, what the library actually returned:

| Guideline font | Match in your library | Match WebProject Name | In Library | In Production |
| --- | --- | --- | --- | --- |
| Helvetica Neue Bold | Helvetica Neue LT Pro Bold (exact) | Acme Marketing Fonts | ✅ | ✅ |
| Gill Sans Regular | Gill Sans Nova (alternative) | Acme Marketing Fonts | ✅ | ❌ |
| TT Supermolot Neue Ext Regular | TT Supermolot Neue Extended Regular (alternative) | Acme Marketing Fonts, Campaign Microsite, Q3 Landing Page | ✅ | ❌ |
| Foundry Gridnik | — | — | ❌ | — |

Fill it from the validation results, **one row per requested face** — a family you queried once still produces three rows if the guideline named three weights:

- **Guideline font** — exactly as the guideline or the user wrote it, so they can recognise their own list.
- **Match in your library** — the family/style the connector matched, tagged `(exact)` or `(alternative)` so a substitution is visible at a glance; `—` when nothing matched.
- **Match WebProject Name** — the **name** of the web project(s) containing that match, from the `get_asset_details` lookup above: `Acme Marketing Fonts`, never `159cb613…` or a raw/truncated id. `—` when the font is licensed but sits in no project. When it's in several, list **every project name**, comma-separated (that's the "spread across projects" case below); if there are more than four, name the first four and add `+N more`, still names, never ids.
- **In Library** — ✅ when validation found the font in My Library, ❌ when it didn't.
- **In Production** — ✅ / ❌ from the `inProduction` flag; `—` when there's no match to have a status.

Keep to these five columns in this order. It's a decision aid, not a data dump — extra columns bury the thing the user is being asked about.

If **every** font is an exact match, production-ready, and present together in **one** web project (every row ✅ / ✅ against the same project name), tell the user and offer to reuse it:

    "All your fonts — `[font list]` — are an exact match, production-ready, and live together in your `[Project Name]` web project. Use that project's licensed CSS, or create a new web project instead?"

- **Reuse** → `get_asset_details(assetId, "WebProject")` → its published CSS path → Stage 5.
- **Create instead** → follow **Creating a web project** below, then Stage 5.

If the fonts aren't all present that way (missing, only fuzzy matches, not production-ready, or spread across projects), say so plainly and offer to switch to route B (`create_new_project`) — don't substitute.

## Stage 4B — Route `create_new_project`: search, consent, verify, create

Do **not** call `validate_font_for_design` on this route. Resolve each font by name, then create one clean web project for the approved set.

1. **Search each font by name.** Call `font_search_by_name(name)` for each requested font.
   - If the results contain an **exact** name match, take it (record its `fontId` / `styleId`).
   - If there's **no exact** match, present the **closest** match and get the user's explicit consent before using it — show requested vs. closest so they decide:

         "I couldn't find an exact match for `[requested font]`. The closest in the catalog is `[closest match]`. Use it, or would you prefer a different one?"

   Never silently accept a non-exact match — every closest-match substitution needs a yes.
2. **Verify production.** For the chosen set (exact + consented), confirm each is production-ready with `get_production_fonts_with_filters` (batch all `styleId`s into one call; read each row's `inProduction`). Surface any font that isn't production-ready and ask whether to proceed without it — don't add a non-production font silently.
3. **Show the status table.** Before anything is created, lay out what you resolved — this is the last look the user gets at the font set that's about to be provisioned:

   | Guideline font | Match in Inventory | In Production |
   | --- | --- | --- |
   | Söhne Buch | Söhne Buch (exact) | ✅ |
   | Gill Sans Regular | Gill Sans Nova (closest — approved) | ✅ |
   | Foundry Gridnik | — | — |

   One row per requested face, three columns, this order:
   - **Guideline font** — as the guideline or the user wrote it.
   - **Match in Inventory** — the catalog font you'll actually use, tagged `(exact)` or `(closest — approved)` so every substitution stays visible; `—` when the search found nothing.
   - **In Production** — ✅ / ❌ from `get_production_fonts_with_filters`; `—` when there's no match to have a status.

   Read it as a manifest: only rows with a match and a ✅ go into the project.
4. **Create the web project.** Follow **Creating a web project** below with the approved, production-ready `fontId`s, then Stage 5.

## Creating a web project (shared)

Whenever you create a project (either branch), suggest a name and let the user adjust it — this is the one thing to confirm:

- **Suggest a name** drawn from the brand/guideline (the brand or company name, e.g. "`[Brand]` Web Fonts"); if there's nothing to derive from, propose a sensible default (the site/domain, or "Brand Web Fonts"). Optionally set `description` to the target domain.
- **Ask one question and get the go-ahead:**

      "I'll create a web project called `[suggested name]` with these fonts: `[approved list]`. Use this name, or give me a different one?"

- On confirmation, call `create_web_project(name, fontIds, description?)`. It creates the project, adds the fonts, and returns the **published CSS path / CDN URL(s)** — the licensed source for Stage 5. Only licensed fonts (a real `fontId`) can be added.

## Stage 5 — Output (licensed fonts + on-brand styling)

Apply **both** the licensed fonts and the stage-2 brand tokens — the right fonts on an off-brand page is still a failure.

**Fonts — licensed source only:**
- Include the connector's published WebProject CSS, CDN link(s), or `@font-face` rules **exactly** as returned (from the reused project, or the one you created).
- Every `font-family` references only the approved set. No Google/Adobe/Bunny/other stylesheet links; no `@font-face` pointing at a URL the connector didn't return; no invented font names; no requested font that didn't pass the pipeline.
- Generic keywords (`sans-serif`, `serif`, `monospace`, `system-ui`) are fine as terminal fallbacks, but the primary face must come from the connector.

**Colors, patterns, other tokens — apply directly (no connector, no consent):**
- Use the extracted palette for its roles/components — brand colors for primary/accent, specified text colors for heading/body, specified surfaces, link and state colors, each component's own colors incl. hover/active/disabled/focus. Don't fall back to generic defaults where the guideline specified a color.
- Apply the component patterns (button shape/radius/padding + variants, card radius/border/shadow, header/nav, form/input styling, global spacing/radius/elevation scales) so components look like the brand's, not framework defaults.
- Define these as reusable tokens where it fits (CSS custom properties / `:root`, a theme object, or Tailwind config) and reference them rather than scattering raw hex inline. Don't invent colors/patterns the guideline didn't specify; where it's silent, use sensible neutral defaults and note that you did.

## Communicating with the user

- In chat, describe what you're doing ("checking My Library", "validating production status") and refer to "the Monotype Enterprice MCP connector" and "My Library" generically — don't recite raw tool identifiers.
- For a direct project reference (Stage 0), say plainly that you're checking production status before building anything ("Checking that the fonts in `[Project Name]` are production-ready before I build the page…") so the user sees the gate happen up front, not as an afterthought.
- Ask one specific question per decision: first the route (check library vs. create new), then within a route the reuse-or-create / closest-match consent, and finally the project name. Name requested vs. matched so the user can decide.
- Render the route's status table as a plain markdown table in chat — Stage 4A's five columns, Stage 4B's three — with the guideline's own font names in the first column. It's what makes the following question answerable, so it comes before the question, not after.
- Everything in that table is human-readable: font names and **project names**. Ids, MD5 hashes, `styleId`s, and CDN paths are internal plumbing — keep them for tool calls and out of chat. If you're about to print a hash, you skipped the id → name resolution step.
- When suggesting a project name, propose a concrete name from the brand and let the user keep or change it — one question, then create.
- When a font can't be included (not licensed, or not production-ready and the user declined it), name it and say you're proceeding without it — don't editorialize and don't substitute.

## Invariants (never break these)

- When `start_font_workflow` is available, call it first and let the **user** pick the route — never auto-decide it; follow only the chosen branch and never run the other route's tools. When it isn't available, use `validate_font_for_design` directly as the entry (it's self-sufficient).
- On the **create_new_project** route, do **not** call `validate_font_for_design`; resolve fonts with `font_search_by_name`. On the **check_library** route (and whenever the router is unavailable), `validate_font_for_design` is the tool.
- Never use a font CDN, file, or `@font-face` source the connector didn't produce; never fabricate a project, kit, or URL — the only way to provision a project is `create_web_project` (with the user's name confirmation).
- Never substitute a non-exact match silently — every closest-match font needs an explicit yes.
- Extract fonts only from names written in the guideline/prompt, or — when nothing is named — from the file's embedded font metadata (PDF embedded fonts, DOCX font XML). Never identify a font by how rendered text looks; if a guideline is image-only with no names and no readable metadata, ask the user rather than guess. (Default heading/body/mono picks apply only when there's no guideline and the user names nothing.)
- Always show the route's status table before the decision it informs: Stage 4A's (Guideline font / Match in your library / Match WebProject Name / In Library / In Production) before the reuse-or-create question, Stage 4B's (Guideline font / Match in Inventory / In Production) before creating the project. Report every requested face as its own row, including the ones that found nothing.
- The **Match WebProject Name** column holds project **names**, never ids. Resolve every project id returned by validation through `get_asset_details(assetId, "WebProject")` before rendering the table — a hash in that column, whole or truncated, is a failed run.
- Always confirm the project name before calling `create_web_project`.
- Never add a font that isn't production-ready without telling the user and getting the go-ahead.
- When the user names an existing web project as the font source (Stage 0), always check every font's production status via `get_production_fonts_with_filters` **before** generating any content — never emit HTML/CSS/output first and verify afterward.
- Never run a color, spacing value, or pattern through the connector or the consent gate — those are for fonts only.
- Never pass a name to `font_search` (MD5 `fontId` only); use `font_search_by_name` for name searches.
- Never recite raw tool identifiers in user-facing chat.
