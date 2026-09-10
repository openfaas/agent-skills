# OpenFaaS Agent Skills

[Agent Skills](https://agentskills.io/) for [OpenFaaS](https://www.openfaas.com) — Serverless Functions, Made Simple.

## Skills

| Skill | Description |
|-------|-------------|
| [openfaas-function-dev](skills/openfaas-function-dev/) | Creates, writes, and develops OpenFaaS functions with `faas-cli`. Covers templates, handlers, `stack.yaml`, secrets, build/deploy and local iteration. |
| [setup-openfaas-edge](skills/setup-openfaas-edge/) | Installs and verifies commercial OpenFaaS Edge on a dedicated Linux host, including runtime-conflict preflight and authenticated checks. |
| [setup-openfaas](skills/setup-openfaas/) | Installs, verifies, upgrades, and troubleshoots OpenFaaS CE, Standard, or For Enterprises on Kubernetes with Helm, including IAM/SSO, license management, and Function Builder. |

## Installation

### npx (recommended — works with 40+ agents)

```bash
npx skills add openfaas/agent-skills
```

This installs the skill into whichever AI coding agents you have (Claude Code, Amp, Cursor, Codex, Gemini CLI, etc.).

### Manual

Clone and copy the skills into your agent's skills directory:

```bash
git clone https://github.com/openfaas/agent-skills.git
cp -r agent-skills/skills/* .claude/skills/   # Claude Code
cp -r agent-skills/skills/* .agents/skills/    # Amp / Codex
cp -r agent-skills/skills/* .cursor/skills/    # Cursor
```

## License

MIT — see [LICENSE](LICENSE).
