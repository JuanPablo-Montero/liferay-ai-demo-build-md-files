# 🌸 Shirley: The Liferay Demo Specialist

**Shirley** is a specialized AI agent designed for **Liferay Solutions Engineers**. She is an expert architect capable of autonomously building high-impact Liferay demos using Fragments, Client Extensions, Objects, and Commerce APIs.

Running on either the **Gemini CLI** or **Claude Code**, with a custom **domain-based skill architecture**, Shirley understands the nuances of Liferay DXP development—from core CSS tokens to complex Headless API orchestrations.

---

## 🚀 Quick Start

### 1. Install Gemini CLI
Shirley runs on the [Gemini CLI](https://github.com/google/gemini-cli). To install the engine, run:

```bash
npm install -g @google/gemini-cli
```

### 2. "Hire" Shirley (Installation)
To give the Gemini CLI "Shirley's expertise," copy the `.gemini` folder from this repository into the root of your Liferay Workspace and also the `WHO_YOU_ARE.md` file: 

```bash
# In your workspace root
cp -r path/to/this/repo/.gemini .
cp path/to/this/repo/WHO_YOU_ARE.md .
```

> Using Claude Code instead? See [Using Shirley with Claude Code](#-using-shirley-with-claude-code).

---

## 🧠 The Skill Architecture
Shirley's brain is split into specialized domains to keep context lean and accuracy high. Each domain has its own documentation and reference guides (the same skills exist under `.claude/skills/` for Claude Code):

*   **[Liferay UI Fragments](./.gemini/skills/liferay-ui-fragments/SKILL.md)**: Master of the DOM. Handles HTML, CSS tokens, and `LFR` editable tags.
*   **[Liferay CX Frontend](./.gemini/skills/liferay-cx-frontend/SKILL.md)**: Expert in packaging. Handles React Custom Elements and Client Extension integration.
*   **[Liferay Data APIs](./.gemini/skills/liferay-data-apis/SKILL.md)**: The data engine. Handles Objects, Headless APIs, and Commerce logic.
*   **[Liferay Server CX](./.gemini/skills/liferay-server-cx/SKILL.md)**: The backend logic. Specialized in Python/Node microservices and Object Actions.
*   **[Liferay MCP](./.gemini/skills/liferay-mcp/SKILL.md)**: The live connection. Discovers and calls Liferay APIs directly through the `liferay-mcp` server.
*   **[Liferay Admin](./.gemini/skills/liferay-admin/SKILL.md)**: The orchestrator. Handles workspace setup, deployment, Liferay Cloud, environments, feature flags, and performance.
*   **[Liferay AI Hub](./.gemini/skills/liferay-ai-hub/SKILL.md)**: AI Hub, AI Assistants, and chatbot Client Extensions.
*   **[Liferay Mobile App](./.gemini/skills/liferay-mobile-app/SKILL.md)**: React Native / Expo apps connected to Liferay.

Supporting skills:

*   **[Figma to Liferay Fragment](./.gemini/skills/figma-to-liferay-fragment/SKILL.md)**: Turns Figma designs or screenshots into fragments.
*   **[Generate Images](./.gemini/skills/generate-images/SKILL.md)**: Creates demo imagery (see below).
*   **[Playwright Scraper](./.gemini/skills/playwright-scraper/SKILL.md)**: Captures the fully rendered HTML of JavaScript-heavy sites.
*   **[Liferay Demo Site Initializer](./.gemini/skills/liferay-demo-site-initializer/SKILL.md)**: Opt-in only. Packages a complete, repeatable demo site. Used only when you explicitly ask for it.

---

## ⚙️ Configuration (`settings.json`)

To enable Shirley's full capabilities, you should configure your `settings.json`. Based on our current setup, your configuration should look like this:

```json
{
  "context": {
    "fileName": [
      "WHO_YOU_ARE.md"
    ]
  },
  "experimental": { 
    "enableAgents": true 
  },
  "agents": {
    "overrides": {
      "browser_agent": {
        "enabled": true
      }
    },
    "browser": {
      "sessionMode": "persistent",
      "visualModel": "gemini-2.5-computer-use-preview-10-2025"
    }
  },
  "mcpServers": {
    "liferay-mcp": {
      "httpUrl": "$LIFERAY_MCP_URL",
      "headers": {
        "Authorization": "Basic $LIFERAY_MCP_BASIC_AUTH"
      }
    }
  }
}
```

### Key Configuration Tips:
- **`context`**: Ensure `WHO_YOU_ARE.md` is included so Shirley always remembers her persona and protocols.
- **`experimental`**: Must be `true` to allow Shirley to use sub-agents for complex tasks.
- **`browser`**: Setting `sessionMode` to `persistent` allows Shirley to stay logged into your Liferay instance across different tasks.
- **`mcpServers`**: Registers the `liferay-mcp` server. Ensure that your system environment has the following variables configured (typically in a `.env` file or global system environment):
  - `LIFERAY_MCP_URL`: Point to your Liferay MCP servlet path (e.g., `https://<your-domain>/o/mcp`).
  - `LIFERAY_MCP_BASIC_AUTH`: Your Base64-encoded `username:password` string for authentication.

---

## 🤖 Using Shirley with Claude Code

Shirley also runs on [Claude Code](https://docs.claude.com/en/docs/claude-code/overview). The repo carries a parallel setup for it, so you can use whichever CLI you prefer in the same workspace. The Gemini files are untouched and keep working.

### 1. Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Then run `claude` once and sign in.

### 2. Copy the Claude files into your workspace

Copy these into the root of your Liferay Workspace (alongside, or instead of, the Gemini files):

```bash
# In your workspace root
cp -r path/to/this/repo/.claude .
cp path/to/this/repo/.mcp.json .
```

| Gemini CLI | Claude Code equivalent | What it does |
| :--- | :--- | :--- |
| `.gemini/skills/` | `.claude/skills/` | The same skills and reference guides |
| `WHO_YOU_ARE.md` (via `context.fileName`) | `.claude/CLAUDE.md` | Shirley's persona and Strict Execution Protocol, loaded automatically |
| `mcpServers` in `.gemini/settings.json` | `.mcp.json` (workspace root) | Registers the `liferay-mcp` server |
| — | `.claude/settings.json` | Pre-approves the `liferay-mcp` server so you aren't prompted |
| `.gemini/ef-header/` | `.claude/ef-header/` | Example header fragment to reuse |

> `.mcp.json` must sit in the workspace root, not inside `.claude/`. Claude Code only reads MCP servers from there.

### 3. Set your environment variables

**Claude Code does not read `.env` files.** Every variable must be a real system environment variable. Restart your terminal after setting them.

| Variable | Needed for |
| :--- | :--- |
| `LIFERAY_MCP_URL` | `liferay-mcp` server (e.g., `https://<your-domain>/o/mcp`) |
| `LIFERAY_MCP_BASIC_AUTH` | `liferay-mcp` server (Base64 `username:password`) |
| `GEMINI_API_KEY` | `generate-images` skill. Image generation still uses Google's API, because Claude cannot create images itself. |
| `FIGMA_TOKEN` | `figma-to-liferay-fragment` skill (see its [README](./.claude/skills/figma-to-liferay-fragment/README.md)) |

Windows (PowerShell):
```powershell
[System.Environment]::SetEnvironmentVariable('LIFERAY_MCP_URL', 'https://<your-domain>/o/mcp', 'User')
```

macOS / Linux: add `export LIFERAY_MCP_URL="https://<your-domain>/o/mcp"` (and the others) to `~/.zshrc` or `~/.bashrc`.

### 4. Check the other tools the skills call

The skills run these through the shell, the same as under Gemini:
- **Node.js** for the Figma fetch script.
- **Python 3** with `requests` for image generation and the API scripts, and `Pillow` for image cropping.
- **Playwright** (`pip install playwright` then `playwright install`) for the `playwright-scraper` skill.
- **Blade CLI** and the **`lcp` CLI** for deployments.

### 5. Start Claude Code and verify

```bash
cd <your-workspace>
claude
```

- Run `/mcp` and check that `liferay-mcp` shows as connected.
- Type `/` to see Shirley's skills. You can run one directly (e.g., `/liferay-ui-fragments`) or just describe the task and Shirley picks the right skill.

### What works differently in Claude Code

- **No built-in browser agent.** Gemini's `browser_agent` (persistent session, computer-use model) has no direct equivalent. Use the **Claude in Chrome** extension (`/chrome`), which drives your real Chrome so you stay logged into Liferay, or the `playwright-scraper` skill for capturing rendered pages.
- **No `--image` flag.** Put the image path in your prompt (e.g., `claude "Build a fragment from ./mockup.png"`), or paste or drag the image into the session.
- **Model and checkpointing settings don't carry over.** Use `/model` to pick a model. Claude Code's rewind is always available.
- **Sub-agents are built in.** No `enableAgents` setting is needed.

### Keeping both setups in sync

The two skill folders are separate copies. When you change a skill or guide, make the same change in both `.gemini/skills/` and `.claude/skills/`, and in both `WHO_YOU_ARE.md` and `.claude/CLAUDE.md`. They should only differ in:
- **Tool names:** `read_file` ↔ `Read`, `mcp_liferay-mcp_…` ↔ `mcp__liferay-mcp__…`, `web_fetch` ↔ `WebFetch`.
- **Paths:** `.gemini/` ↔ `.claude/`.
- **The Figma README:** `gemini "…"` ↔ `claude "…"` commands.
- **The Commerce guide:** Gemini calls `gemini-2.5-flash` for product copy, while Claude writes the copy itself.

To compare them, run `diff -rq .gemini/skills .claude/skills`.

---

## 🎨 Image Generation Skill

Shirley uses the [generate-images](./.gemini/skills/generate-images/SKILL.md) skill to create professional-grade assets (Hero backgrounds, product thumbnails, avatars) so your demos never look like "empty templates."

### How the Key Works:
1.  **API Key**: This skill requires a Google API Key with access to the **Gemini Flash 3.1** model.
2.  **Environment Variable**: Store your key in your system's environment variables as `GEMINI_API_KEY` (this exact name; the script doesn't check any other).
3.  **Automatic Detection**: Shirley's Python scripts are configured to automatically pick up this key from your environment to securely generate thumbnails for new Fragments.

---

## 🛠️ Usage Protocol

Shirley follows a **Strict Execution Protocol**. When you ask her to build something, she will:
1.  **Research**: Activate the relevant domain skill (e.g., `liferay-ui-fragments`).
2.  **Strategy**: Read the `.md` reference files to ensure the latest Liferay standards are met.
3.  **Execution**: Write surgical, idiomatic code.
4.  **Validation**: Verify the implementation against Liferay's best practices.

**"My job is to make you look like a hero in front of the customer." — Shirley**
