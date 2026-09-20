# Jev Browser (Hardened Edition)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Upstream: jkudish/jev-browser](https://img.shields.io/badge/Upstream-jkudish%2Fjev--browser-6366f1.svg)](https://github.com/jkudish/jev-browser)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.6-3178c6.svg)](https://www.typescriptlang.org)

Fast, reliable, and cost-effective browser automation powered by **TypeSafe's Jev** non-autoregressive decision model, running as an MCP server, CLI, or library.

> **Notice on Upstream Attribution**:
> This project is a hardened, production-oriented edition of [jkudish/jev-browser](https://github.com/jkudish/jev-browser) by Joey Kudish. It preserves the complete Jev-driven browser architecture while introducing crucial stability, proxy connectivity, selector coverage, and local browser detection improvements for real-world enterprise and cross-platform desktop environments.

---

## Demo

Searches GitHub for a repository, opens Releases, and answers questions about page content. Every Jev judgment is tracked in real-time on the right: the action chosen, its confidence, and the goal/stuck probabilities for each step:

![jev-browser searching GitHub and opening its own Releases page](assets/github-demo.gif)

Full-resolution demo video: [assets/github-demo.mp4](assets/github-demo.mp4).

---

## Key Enhancements Over Upstream

While the original `jev-browser` demonstrates an elegant decision architecture, practical deployment in varied developer environments (particularly Windows and proxied networks) reveals several failure modes. This fork incorporates the following production enhancements:

### 1. System / Local Chrome Detection (`--chrome` / `JEV_BROWSER_USE_CHROME=1`)
- **Problem**: Upstream exclusively relies on Playwright downloading its bundled Chromium build (~300MB), which frequently hangs or fails in restricted networks and ignores installed browsers.
- **Enhancement**: Automatic detection of existing Google Chrome installations across Windows (`%LOCALAPPDATA%`, `Program Files`), macOS (`/Applications/Google Chrome.app`), and Linux (`/usr/bin/google-chrome`, `/usr/bin/chromium`). Added `--chrome` CLI flag to run instantly without downloading browser binaries.

### 2. Native HTTP / HTTPS Proxy Support
- **Problem**: Upstream fails to route browser network requests through standard system proxies, breaking completely in environments utilizing corporate firewalls or local proxy tools (e.g., Clash, v2ray on port 7890/7897).
- **Enhancement**: Automatically reads `HTTPS_PROXY` and `HTTP_PROXY` environment variables and injects them directly into Playwright's `chromium.launch({ proxy: { server } })`.

### 3. Expanded Interactive Element Selectors
- **Problem**: Upstream's candidate selectors miss essential modern UI elements such as details/summary disclosures, dropdown options, and tab interfaces.
- **Enhancement**: Expanded discovery to index `<summary>`, `[role="menuitem"]`, `[role="menuitemradio"]`, `[role="menuitemcheckbox"]`, `[role="option"]`, and `[role="tab"]`, preventing Jev from missing vital navigation controls.

### 4. Synthetic Click Fallback & Anchor Navigation
- **Problem**: Playwright's standard `page.click` frequently throws timeout errors on elements that are partially obscured, re-rendering, or detached in dynamic single-page applications.
- **Enhancement**: Implemented synthetic DOM dispatch click fallback (`$eval((el) => el.click())`) and explicit anchor tag triggering (`anchor.click()`) when standard clicks time out, preventing agent deadlock.

### 5. Zero-Config Environment Loading (`process.loadEnvFile`)
- **Problem**: Invoking Jev via CLI or MCP required manual exports or external shell scripts to populate `OPENROUTER_API_KEY` or `TYPESAFE_API_KEY`.
- **Enhancement**: Native, backward-compatible auto-loading of `.env` files from both the current working directory and the package root.

### 6. Sensitive Page Mutation Detection
- **Problem**: Upstream requires a 50-character text delta to acknowledge page state changes, causing false "stuck" alarms during search autocomplete or tab filtering.
- **Enhancement**: Lowered text delta threshold to 10 characters and integrated visible text excerpt comparison to reliably track subtle DOM updates.

---

## Comparison Matrix

| Feature | Upstream (`jkudish/jev-browser`) | This Fork (`connectedGraph/jev-browser`) |
| :--- | :--- | :--- |
| **Jev Decision Engine** | TypeSafe direct / OpenRouter / Cloudflare | Identical (`{state, questions}` contract) |
| **System Chrome Detection** | Unsupported (bundled Chromium only) | **Full Auto-Discovery + `--chrome` flag** |
| **Proxy (`HTTP_PROXY`)** | Ignored | **Automatic Playwright Proxy Routing** |
| **Candidate Selectors** | Links, buttons, basic inputs | **Added `<summary>`, menuitems, tabs, options** |
| **Click Resilience** | Standard `page.click` only (times out) | **Synthetic DOM fallback on timeout** |
| **Zero-Config `.env`** | Manual environment setup required | **Native `.env` loading from CWD / package** |
| **Page Change Sensitivity** | 50-char delta | **10-char delta + excerpt comparison** |

---

## Installation & Quick Start

### 1. Clone & Build
```bash
git clone https://github.com/connectedGraph/jev-browser.git
cd jev-browser
npm install
npm run build
```

### 2. Configure Environment
Create a `.env` file in the project root:
```bash
# TypeSafe direct API key (Recommended)
TYPESAFE_API_KEY=ts_your_key_here

# OR use OpenRouter (single key powers both Jev judgments and typing model)
OPENROUTER_API_KEY=sk-or-v1-your-key-here

# Optional: set proxy if behind a corporate network or firewall
HTTPS_PROXY=http://127.0.0.1:7890
```

### 3. Command Line Usage

Run a task with local Google Chrome:
```bash
# Run using system Chrome
node dist/index.js run "Search Wikipedia for Espresso and find its origin" https://en.wikipedia.org/wiki/Main_Page --chrome
```

Watch the browser in headed mode:
```bash
JEV_BROWSER_HEADED=1 node dist/index.js run "Search DuckDuckGo for TypeSafe Jev" https://duckduckgo.com --chrome
```

Using a local LLM (e.g. Ollama) for text input generation:
```bash
JEV_BROWSER_TYPE_BASE_URL=http://localhost:11434/v1 JEV_BROWSER_TYPE_MODEL=qwen2.5:7b \
  node dist/index.js run "Search Wikipedia for Ristretto" https://en.wikipedia.org/wiki/Main_Page --chrome
```

---

## MCP Server Registration

Register Jev Browser as an MCP tool in your favorite AI agent:

### Claude Code
```bash
claude mcp add jev-browser -- node /path/to/jev-browser/dist/index.js
```

### Cursor / Codex (`~/.codex/config.toml`)
```toml
[mcp_servers.jev-browser]
command = "node"
args = ["/path/to/jev-browser/dist/index.js"]
```

### Generic MCP Client (`mcpServers` JSON)
```json
{
  "mcpServers": {
    "jev-browser": {
      "command": "node",
      "args": ["/path/to/jev-browser/dist/index.js"],
      "env": {
        "OPENROUTER_API_KEY": "sk-or-...",
        "HTTPS_PROXY": "http://127.0.0.1:7890"
      }
    }
  }
}
```

---

## How It Works

Jev Browser separates **control flow** from **judgment**:
- **Code owns control flow**: time and step budgets, DOM stamping with `data-jev-id`, recovery fallback, network timeouts, and screenshot capture.
- **Jev owns the judgments**: at each step, TypeSafe's Jev model evaluates the current page state, selects the single best interactive element from the indexed candidates, and computes probabilities for whether the goal is met or the execution is stuck.
- **Lightweight typing model**: Jev does not generate freeform text; whenever a form field requires string input, a small typing model (e.g., Haiku, GPT-5-Luna, or local Ollama) is called on-demand (~48 tokens).

---

## Upstream & Acknowledgements

- Original Author: [Joey Kudish (@jkudish)](https://github.com/jkudish)
- Upstream Repository: [https://github.com/jkudish/jev-browser](https://github.com/jkudish/jev-browser)
- Decision Model: [TypeSafe Jev](https://typesafe.ai)

---

## License

MIT License. See [LICENSE](LICENSE) for details.
