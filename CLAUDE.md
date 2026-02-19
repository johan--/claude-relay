# claude-relay

Fork of [chadbyte/claude-relay](https://github.com/chadbyte/claude-relay) — Web UI for Claude Code.
Used as reference for an Elixir/Phoenix rewrite.

## Run Locally

```bash
npm install
node bin/cli.js -p 5000
```

Requires `build-essential` and `python3` for native `node-pty` module.

## Plans & Handovers

- Plans: `.claude/plans/`
- Handovers: `.claude/handovers/`
- **Copy plan mode files** from `~/.claude/plans/` to `.claude/plans/` — plan mode writes to the global directory by default.

## Migration

Active migration plan: `.claude/plans/elixir-phoenix-migration.md`

Target: Elixir/Phoenix with Direct Anthropic API, LiveView, SQLite (Ecto).
New repo (not in this folder). This repo is reference only.

## Codebase

- Node.js, no build step, no TypeScript
- Entry point: `bin/cli.js`
- Server: `lib/server.js`, `lib/daemon.js`, `lib/project.js`
- Claude integration: `lib/sdk-bridge.js`
- Client: `lib/public/app.js` + modules in `lib/public/modules/`
- Codeindex: `.codeindex/output/overview.md`
