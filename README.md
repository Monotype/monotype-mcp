# Monotype Fonts

The **Monotype Fonts connector** brings your organisation's approved brand fonts and their production status into AI-native content creation. It is powered by the hosted **Monotype Fonts MCP server**, which connects your Monotype Fonts inventory to any MCP-compatible AI platform, so AI-assisted content starts from your licensed, production-ready fonts instead of generic defaults or substitute typefaces.

Starting from a brand guideline, a campaign kit or a plain prompt, the connector identifies the relevant fonts, checks their availability in your inventory and verifies their production status. Then:

- **Web and HTML content:** finds the fonts in an existing web project or creates a new one, and returns the CSS / CDN URL that renders content with your approved brand fonts.
- **Desktop workflows:** creates font lists and syncs fonts to the Monotype Fonts desktop app.

This repository is the setup page for the connector. It contains:

- Connection details for the hosted Monotype Fonts MCP server
- A **skill** that teaches agents how to use the server correctly
- **Plugin manifests** that install the server and the skill together in supported clients

| | |
|---|---|
| **Server URL** | `https://mcp.monotype.com/mcp` |
| **Transport** | Streamable HTTP |
| **Authentication** | OAuth 2.1 with PKCE. Each user signs in with their own Monotype Fonts account. |
| **Client registration** | Monotype's authorization server supports [Client ID Metadata Documents](https://modelcontextprotocol.io/specification/draft/basic/authorization) (CIMD), so clients that implement CIMD need no client ID, secret or API key. Clients that require a pre-registered client ID can request one from [Monotype support](https://support.monotype.com/). |

---

## Contents

1. [Prerequisites](#prerequisites)
2. [Connect to Monotype Fonts](#connect-to-monotype-fonts)
3. [Skills](#skills)
4. [Tools exposed](#tools-exposed)
5. [Example prompts](#example-prompts)
6. [Repository layout](#repository-layout)
7. [Support](#support)

---

## Prerequisites

| Requirement | Details |
|---|---|
| Monotype Fonts account | An active **Monotype Fonts** subscription. Every user who connects needs their own login. |
| Monotype Fonts desktop app | Required only for desktop workflows (syncing fonts to your computer). Install it and sign in with the same Monotype account. |
| MCP client | A client that supports **remote MCP servers** over Streamable HTTP, with OAuth sign-in using **Client ID Metadata Documents (CIMD)**. Clients without CIMD need a pre-registered client ID (see *Client registration* above). On some platforms, adding an MCP server is an admin action. |

---

## Connect to Monotype Fonts

Most AI platforms with remote MCP support follow the same flow:

1. Open the platform's tools, connectors or integrations settings.
2. Add a remote MCP server with the URL `https://mcp.monotype.com/mcp`.
3. If offered a choice, use **per-user** authentication, so each person signs in with their own Monotype account and keeps their own library permissions.
4. Connect and complete the Monotype sign-in in the browser window that opens.
5. Make the tool available to the users, teams or agents that need it.
6. Set up the [skill](#skills).

### Client notes

**Claude Code**

```bash
claude mcp add --transport http monotype https://mcp.monotype.com/mcp
```

Or install the plugin, which adds the server and the skill together:

```bash
claude plugin marketplace add Monotype/monotype-mcp
claude plugin install monotype-mcp@monotype
```

**Claude.ai / Claude Desktop** — Settings → Connectors → *Add custom connector*, paste the server URL and complete the sign-in. Organisation admins can add it once for everyone; each member then connects with their own account.

**Dust and other hosted agent platforms** — an admin adds the server from the platform's tools or connections catalogue using the URL above and chooses per-user authentication where available. Each user connects once and signs in with their Monotype account. Then add the skill below to the agents that generate content.

**Gemini CLI** — install this repository as an extension:

```bash
gemini extensions install https://github.com/Monotype/monotype-mcp
```

---

## Skills

The MCP server gives agents access to your font library. The **skill** tells agents *how* to use it correctly: match fonts only through the connector, take font names only from text (never guess them from an image), show each font's production status, use only the CSS link the connector returns, and ask before creating anything.

The skill follows the open [Agent Skills](https://agentskills.io) format, so the same `SKILL.md` file works across clients.

| Skill | Use it when | Source |
|---|---|---|
| **Monotype Fonts** | Users upload a brand guideline, style guide, brand book or type spec, name the fonts, or ask for web output for a named brand or project. Matches the fonts in your Monotype inventory, shows their production status, and reuses an existing web project, digital ad or font list, or creates a new one after you confirm. | [`skills/monotype-fonts`](skills/monotype-fonts/SKILL.md) |

### Installing the skills

**Clients with plugins or extensions** (Claude Code, Gemini CLI): the plugin installs the skill. Nothing else to do.

**Clients that read skill files directly**: copy the skill folder into your client's skills directory, or use the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add Monotype/monotype-mcp
```

**Hosted platforms** (Claude.ai projects, Dust agents and similar): paste the contents of `SKILL.md` into the agent's instructions, or upload it as a skill where the platform supports Agent Skills.

### Keeping skills up to date

The skill is updated as the MCP server evolves. Watch this repository or check the [Releases](../../releases) page. Plugin installs can be updated through your client's plugin manager.

---

## Tools exposed

| Tool | What it does |
|---|---|
| `font_search_by_name` | Searches the Monotype Fonts catalogue and your imported fonts by font name and returns each match's font ID. |
| `font_search` | Returns detailed metadata for a font by its font ID. |
| `get_production_fonts_with_filters` | Reports the production status of specific fonts, or lists your production fonts. |
| `identify_font_from_image` | Identifies the fonts in a PNG or JPEG image and returns ranked candidates with their production status. |
| `get_web_projects_by_name` | Lists or searches your web projects, digital ads and font lists. |
| `get_asset_details` | Returns the details of a web project, digital ad or font list, including its fonts and published CSS. |
| `create_web_project` | Creates a new web project or digital ad with the selected fonts and returns its CSS / CDN URL. |
| `update_web_project` | Adds fonts to an existing web project or digital ad and returns the refreshed CSS / CDN URL. |
| `create_font_list` | Creates a new font list to group and organise fonts. |
| `update_font_list` | Adds fonts to, or removes fonts from, an existing font list. |
| `sync_fonts` | Syncs fonts to the Monotype Fonts desktop app. |
| `unsync_fonts` | Removes fonts from the Monotype Fonts desktop app. They stay in your inventory. |
| `list_synced_fonts` | Lists the fonts currently synced to the Monotype Fonts desktop app. |
| `font_asset_delivery` | Delivers the font files (.ttf / .otf) for a font, family or font list name as download links. |
| `get_font_download_url` | Returns a short-lived download link for one font file, by font ID. |
| `get_font_package_download_url` | Packages several fonts into one archive and returns a short-lived download link. |

All tools act on the signed-in user's own Monotype Fonts account and respect its permissions. Tools that create or change assets (web projects, font lists, desktop sync) only run when the agent is asked to.

Some tools are available only for certain accounts or apps: `identify_font_from_image` and the three font-file tools (`font_asset_delivery`, `get_font_download_url`, `get_font_package_download_url`). If one of them doesn't appear in your AI app, it isn't enabled for your account.

---

## Example prompts

- *"Is Helvetica Now Display in our Monotype library, and is it in production?"*
- *"Create a luxury Black Friday HTML social media advertisement for VEXTOR Aeris Chrono. Use the Monotype Fonts connector to fetch the brand fonts."*
- *"Here's our brand guideline. Check whether all the brand fonts are in our Monotype library and in production, and create a web project for them."*
- *"Sync the fonts from our Spring Campaign web project to my desktop."*

---

## Repository layout

```
.
├── README.md                      # This page
├── .claude-plugin/
│   ├── plugin.json                # Claude Code plugin manifest
│   └── marketplace.json           # Lets users add this repo as a plugin marketplace
├── .mcp.json                      # Remote MCP server config bundled with the plugin
├── gemini-extension.json          # Gemini CLI extension manifest
├── server.json                    # MCP Registry manifest
└── skills/
    └── monotype-fonts/SKILL.md
```

Each [Release](../../releases) also attaches `monotype-mcp-plugin.zip` for clients that install plugins or skills by upload.

---

## Support

- **Website:** [monotype.com](https://www.monotype.com)
- **Monotype support:** [support.monotype.com](https://support.monotype.com/)
- **Privacy policy:** [monotype.com/legal/privacy-policy](https://www.monotype.com/legal/privacy-policy)
- **Terms of use:** [monotype.com/legal/terms-use](https://www.monotype.com/legal/terms-use)

---

## Changelog

| Version | Changes |
|---|---|
| 1.0.0 | Initial release: connection details, the Monotype Fonts skill, plugin manifests. |
