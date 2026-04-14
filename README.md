<div align="center">

<a href="https://arc-web.github.io/clawlaunch/">
  <img src="https://img.shields.io/badge/🎬_Interactive_Presentation-View_Live-7B2FBE?style=for-the-badge&labelColor=0F0F1A&color=7B2FBE" alt="View Interactive Presentation" />
</a>

</div>

---

# ClawLaunch

### Stop lighting money on fire. Start launching OpenClaw the right way.

---

You built an always-on AI agent. It lives on a VPS. It responds to Discord. It orchestrates sub-agents. It's *beautiful*.

Then your Anthropic invoice shows up.

**$847. For a month. Of an agent that mostly says "got it" and runs shell commands.**

Meanwhile, Claude Max exists at $200/month flat. Unlimited Sonnet. Massive Opus allocation. And Anthropic *just shipped* Claude Code Channels - their official "yes, you can run a Discord bot on your subscription" feature.

The problem? Nobody documented how to actually do this properly. So people either:

1. Burn API credits like it's 2024 and tokens grow on trees
2. Hack together OAuth token extraction that violates the TOS and gets them banned
3. Give up and run GPT-4o (lol)

**ClawLaunch is option 4.** A clean, one-shot setup that gets your OpenClaw instance running on Claude Max with a 1-year token. TOS compliant. No hacks. No token extraction. No ban risk.

---

## What's in the box

| File | What it does |
|------|-------------|
| [`docs/runbook.md`](docs/runbook.md) | Full step-by-step runbook. Every command explained. |
| [`docs/OpenClaw-Setup-Token-Runbook.docx`](docs/OpenClaw-Setup-Token-Runbook.docx) | Same runbook in Word format for the paper trail crowd. |
| [`docs/tos-compliance.md`](docs/tos-compliance.md) | Detailed TOS analysis with citations. What's allowed, what's not, where the lines are. |

---

## Quick Start

### Prerequisites

- Claude Code CLI installed locally (`claude` in your PATH)
- Active **Claude Max** subscription ($100 or $200/month)
- SSH access to your VPS where OpenClaw runs
- A browser (for the one-time OAuth flow)

### One command

```bash
# Generate your 1-year token (opens browser for auth)
claude setup-token
```

Copy the token (starts with `sk-ant-oat01-`), then follow the [runbook](docs/runbook.md) to inject it into your VPS.

---

## The Math

| Setup | Monthly Cost | Annual Cost |
|-------|-------------|-------------|
| API keys (moderate agent usage) | $400-1,200 | $4,800-14,400 |
| API keys (heavy agent usage) | $800-2,500+ | $9,600-30,000+ |
| **Claude Max 20x + ClawLaunch** | **$200** | **$2,400** |

That's not a typo. The Max plan is a flat rate. Your agent can make thousands of calls per day and the bill doesn't change.

*The only limit is the 5-hour rolling window and weekly usage caps. For most agent workloads, Max 20x is more than enough. If you're consistently hitting limits, you're probably doing something wrong (or something very right).*

---

## How It Works

```
Local Machine                          VPS (OpenClaw)
+------------------+                   +---------------------------+
|                  |                   |                           |
| claude setup-token|                   |  .env                     |
|   |              |    SSH            |    CLAUDE_CODE_OAUTH_TOKEN |
|   v              | ----------------> |    = sk-ant-oat01-...     |
| Browser auth     |                   |                           |
|   |              |                   |  Claude CLI shim          |
|   v              |                   |    unsets API key          |
| sk-ant-oat01-... |                   |    forces OAuth            |
|                  |                   |                           |
+------------------+                   |  Container restart         |
                                       |    vps-ops.sh oc-restart  |
                                       |                           |
                                       |  Verification              |
                                       |    3 checks, all green    |
                                       +---------------------------+
```

1. **Generate** a 1-year OAuth token locally via `claude setup-token`
2. **Inject** it into your VPS container's `.env` file
3. **Restart** the container so it picks up the new token
4. **Verify** end-to-end: token visible, Claude responds, shim intact
5. **Cleanup** shell history on both machines

The Claude CLI shim (`/data/.agents/bin/claude`) unsets `ANTHROPIC_API_KEY` on every call, forcing the CLI to authenticate via the OAuth token instead. This means your agent bills against your Max subscription, not per-token API pricing.

---

## Anthropic TOS Compliance

This is the part most people skip and then regret. **Read this.**

### The Rules (TL;DR)

Anthropic's Consumer Terms of Service (Section 3.7) prohibit automated access to Claude **except**:

1. Via an Anthropic **API Key** (Commercial Terms)
2. Where Anthropic **"otherwise explicitly permits it"**

Claude Code CLI is that explicit permission. It's Anthropic's own product, built for scripted and automated use. The `-p` flag, piping, CI/CD integration - all documented and encouraged.

### What ClawLaunch Does (Allowed)

- Runs the **official `claude` CLI binary** on your VPS
- Authenticates via OAuth token obtained through `claude setup-token` (Anthropic's own command)
- Uses the CLI's built-in `-p` flag for programmatic calls
- Same architecture as **Claude Code Channels** (Anthropic's official Discord/Telegram bot feature, launched March 2026)

### What ClawLaunch Does NOT Do (Would Be Prohibited)

- Does not extract OAuth tokens for use in third-party API clients
- Does not use subscription tokens with the Agent SDK
- Does not resell, share, or redistribute tokens
- Does not bypass rate limits or authentication mechanisms
- Does not use any tool other than the official `claude` binary

### The Bright Line

```
  ALLOWED                              PROHIBITED
  --------                             ----------
  claude -p "do thing"                 curl api.anthropic.com -H "Bearer $OAUTH_TOKEN"
  claude setup-token                   Extracting tokens for Cursor/OpenCode/Roo
  Claude Code Channels                 Agent SDK with subscription OAuth tokens
  CLI on VPS in tmux                   Token reselling or sharing
```

### Enforcement History

- **January 2026:** Anthropic blocked third-party tools (OpenCode, etc.) from using subscription OAuth tokens. Server-side enforcement, no warning.
- **February 2026:** Anthropic engineer Thariq confirmed targets were "token resellers and third-party harnesses," not subscribers using Claude Code.
- **March 2026:** Claude Code Channels launched - Anthropic's official answer to the "always-on agent" use case, explicitly designed for Pro/Max subscribers.

### Our Position

ClawLaunch uses the Claude CLI exactly as Anthropic designed it. We invoke the binary. We don't extract tokens. We don't wrap it in a third-party client. The architecture is identical to Claude Code Channels - we're just doing it with OpenClaw as the orchestration layer instead of Anthropic's built-in channel plugins.

If Anthropic decides to restrict CLI usage on VPS environments, that would also break their own Channels feature, their own CI/CD documentation, and their own headless deployment guides. We're standing on solid ground.

For the full analysis with citations, see [`docs/tos-compliance.md`](docs/tos-compliance.md).

---

## Token Lifecycle

| Event | Action |
|-------|--------|
| Token generated | Valid for ~1 year |
| 11 months later | Run ClawLaunch again to rotate |
| OpenClaw starts getting 401s | Token expired - time to rotate |
| Max subscription lapses | Token stops working immediately |
| Run `setup-token` again | Previous token is invalidated |

Set a calendar reminder for 11 months out. That's it.

---

## FAQ

**Q: Will I get banned for this?**
No. You're using Anthropic's own CLI binary with Anthropic's own setup-token command. This is the documented path. The bans in early 2026 targeted people extracting tokens for third-party tools (Cursor, OpenCode, etc.), not people using Claude Code.

**Q: What about rate limits?**
Max 20x gives you 20x the Pro plan limits within a 5-hour rolling window, plus weekly caps. For most agent workloads this is plenty. If you hit limits, the CLI will return rate limit errors - your agent should handle these gracefully (back off, retry after the window).

**Q: Can I run multiple agents on one Max subscription?**
Technically yes - they'd share the same rate limits. Practically, one heavy agent on Max 20x is the sweet spot. If you need multiple heavy agents, consider separate subscriptions or API keys for the overflow.

**Q: What if Anthropic changes the TOS?**
Then everyone using Claude Code on a VPS is affected, including Anthropic's own Channels users. We'll update ClawLaunch if the rules change. Follow this repo for updates.

**Q: Why not just use API keys?**
You can! And for some workloads (high-volume, bursty, commercial) API keys are the right choice. But if your agent makes a steady stream of calls throughout the day, Max at $200/month vs. $800+/month in API costs is a no-brainer.

---

## Recommended: Cut Token Usage Too

Flat-rate billing doesn't mean you should waste tokens. [lean-ctx](https://github.com/arc-web/lean-ctx) is a compression layer that sits between your agent and shell/file operations - 34-99% token reduction on command output. Fewer tokens per call = more calls before you hit Max rate limits.

---

## Contributing

Found a bug? TOS changed? Better way to handle token rotation? Open an issue or PR.

---

## License

MIT

---

*Built by the [Advertising Report Card](https://advertisingreportcard.com) team. OpenClaw is our always-on AI agent. We got tired of the API bill.*
