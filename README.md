# Codex CLI — Tutorial Edition

> **This is a clone of OpenAI's [openai/codex](https://github.com/openai/codex), enhanced with an in-depth tutorial documentation set.**
>
> The tutorial docs were researched and written by **Claude Fable 5**. They explain the Codex CLI as an *agent harness* — its architecture, every subsystem (with source-code references), an engineering critique, a roadmap, and a feature-by-feature competitive analysis against Anthropic's Claude Code and xAI's Grok Build.
>
> All original Codex source code is unchanged and remains under its [Apache-2.0 License](LICENSE). The only additions in this fork are the documentation under [`harness-docs/`](harness-docs/) and this README notice.

## What was added

A complete, code-referenced documentation set lives in **[`harness-docs/`](harness-docs/)**. Start at [`harness-docs/index.md`](harness-docs/index.md).

| Read this to… | File |
|---------------|------|
| Understand what an agent harness *is* | [01-what-is-a-harness.md](harness-docs/01-what-is-a-harness.md) |
| Get the Codex mental model + crate map | [02-codex-overview.md](harness-docs/02-codex-overview.md) |
| Look up any term | [03-key-concepts.md](harness-docs/03-key-concepts.md) |
| Read the engine deep-dives | [04-agent-loop](harness-docs/04-agent-loop.md) · [05-tools-and-approvals](harness-docs/05-tools-and-approvals.md) · [06-execution-and-sandboxing](harness-docs/06-execution-and-sandboxing.md) · [07-model-client](harness-docs/07-model-client.md) · [08-context-state-memory](harness-docs/08-context-state-memory.md) · [09-extensibility](harness-docs/09-extensibility.md) · [10-frontends-and-protocol](harness-docs/10-frontends-and-protocol.md) · [11-config-auth-telemetry](harness-docs/11-config-auth-telemetry.md) |
| See a critique of the harness | [12-critique.md](harness-docs/12-critique.md) |
| See proposed improvements + new features | [13-improvements-and-new-features.md](harness-docs/13-improvements-and-new-features.md) |
| Compare Codex vs Claude Code vs Grok Build | [14-competitive-analysis.md](harness-docs/14-competitive-analysis.md) |

Every claim about Codex cites `crate/src/file.rs:line` in the source tree. Claims about other tools cite public docs and are marked where a fact rests on a single or unverified source.

## Attribution

- **Original project:** [openai/codex](https://github.com/openai/codex) — the Codex CLI coding agent from OpenAI (Apache-2.0).
- **Tutorial documentation:** authored by Claude Fable 5. Educational/reference material only; not affiliated with or endorsed by OpenAI, Anthropic, or xAI.

---

The original Codex CLI README follows, preserved verbatim.

---

<p align="center"><strong>Codex CLI</strong> is a coding agent from OpenAI that runs locally on your computer.
<p align="center">
  <img src="https://github.com/openai/codex/blob/main/.github/codex-cli-splash.png" alt="Codex CLI splash" width="80%" />
</p>
</br>
If you want Codex in your code editor (VS Code, Cursor, Windsurf), <a href="https://developers.openai.com/codex/ide">install in your IDE.</a>
</br>If you want the desktop app experience, run <code>codex app</code> or visit <a href="https://chatgpt.com/codex?app-landing-page=true">the Codex App page</a>.
</br>If you are looking for the <em>cloud-based agent</em> from OpenAI, <strong>Codex Web</strong>, go to <a href="https://chatgpt.com/codex">chatgpt.com/codex</a>.</p>

---

## Quickstart

### Installing and running Codex CLI

Run the following on Mac or Linux to install Codex CLI:

```shell
curl -fsSL https://chatgpt.com/codex/install.sh | sh
```

Run the following on Windows to install Codex CLI:

```shell
powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"
```

Codex CLI can also be installed via the following package managers:

```shell
# Install using npm
npm install -g @openai/codex
```

```shell
# Install using Homebrew
brew install --cask codex
```

Then simply run `codex` to get started.

<details>
<summary>You can also go to the <a href="https://github.com/openai/codex/releases/latest">latest GitHub Release</a> and download the appropriate binary for your platform.</summary>

Each GitHub Release contains many executables, but in practice, you likely want one of these:

- macOS
  - Apple Silicon/arm64: `codex-aarch64-apple-darwin.tar.gz`
  - x86_64 (older Mac hardware): `codex-x86_64-apple-darwin.tar.gz`
- Linux
  - x86_64: `codex-x86_64-unknown-linux-musl.tar.gz`
  - arm64: `codex-aarch64-unknown-linux-musl.tar.gz`

Each archive contains a single entry with the platform baked into the name (e.g., `codex-x86_64-unknown-linux-musl`), so you likely want to rename it to `codex` after extracting it.

</details>

### Using Codex with your ChatGPT plan

Run `codex` and select **Sign in with ChatGPT**. We recommend signing into your ChatGPT account to use Codex as part of your Plus, Pro, Business, Edu, or Enterprise plan. [Learn more about what's included in your ChatGPT plan](https://help.openai.com/en/articles/11369540-codex-in-chatgpt).

You can also use Codex with an API key, but this requires [additional setup](https://developers.openai.com/codex/auth#sign-in-with-an-api-key).

## Docs

- [**Codex Documentation**](https://developers.openai.com/codex)
- [**Contributing**](./docs/contributing.md)
- [**Installing & building**](./docs/install.md)
- [**Open source fund**](./docs/open-source-fund.md)

This repository is licensed under the [Apache-2.0 License](LICENSE).
