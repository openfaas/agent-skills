# OpenFaaS Agent Skills

[Agent Skills](https://agentskills.io/) for [OpenFaaS](https://www.openfaas.com) — Serverless Functions, Made Simple.

## Skills

| Skill | Description |
|-------|-------------|
| [openfaas-function-dev](skills/openfaas-function-dev/) | Creates, writes, and develops OpenFaaS functions with `faas-cli`. Covers templates, handlers, `stack.yaml`, secrets, build/deploy and local iteration. |
| [setup-openfaas](skills/setup-openfaas/) | Installs, verifies, and upgrades OpenFaaS on Kubernetes with Helm. Covers Community Edition, Standard, For Enterprises, the Pro CLI, dashboard, IAM/SSO, Function Builder with a local single-node K3s registry, and production-readiness checks. |

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
