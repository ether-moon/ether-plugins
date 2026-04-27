# ether-plugins

A Claude Code plugin marketplace aggregating [ether-moon](https://github.com/ether-moon)'s plugin collection. Each plugin lives in its own repository; this marketplace is a thin index that lets you discover and install all of them from a single entry point.

## Install the marketplace

```sh
claude plugin marketplace add ether-moon/ether-plugins
```

## Plugins

| Plugin | Description | Source |
|---|---|---|
| `skill-set` | Comprehensive productivity skills and development tools for Claude Code — git workflow automation, code context understanding, peer LLM consulting, PR review feedback. | [ether-moon/skill-set](https://github.com/ether-moon/skill-set) |
| `agent-atelier` | Autonomous product development loop — AI agent team that cycles through spec, implement, validate, and self-correct. | [ether-moon/agent-atelier](https://github.com/ether-moon/agent-atelier) |
| `knowledge-distillery` | A knowledge distillation system that delivers only verified knowledge to AI coding agents. 3-layer architecture with convention-based air gap. | [ether-moon/knowledge-distillery](https://github.com/ether-moon/knowledge-distillery) |
| `hotwire-frontend-skills` | 7 skills (1 gateway + 6 specialists) for building Rails frontend with Hotwire — Turbo Drive, Turbo Frames, Turbo Streams, Stimulus, view transitions, forms, media, and native bridge. | [ether-moon/hotwire-frontend-skills](https://github.com/ether-moon/hotwire-frontend-skills) |
| `herb-lsp-plugin` | Plugin to support [herb-lsp](https://github.com/marcoroth/herb) in Claude Code. | [ether-moon/herb-lsp-plugin](https://github.com/ether-moon/herb-lsp-plugin) |

## Install individual plugins

```sh
claude plugin install skill-set@ether-plugins
claude plugin install agent-atelier@ether-plugins
claude plugin install knowledge-distillery@ether-plugins
claude plugin install hotwire-frontend-skills@ether-plugins
claude plugin install herb-lsp-plugin@ether-plugins
```

## How it works

Each entry in `.claude-plugin/marketplace.json` uses the `git-subdir` source type to reference the plugin directory inside its source repository, tracking the `main` branch:

```json
{
  "source": {
    "source": "git-subdir",
    "url": "https://github.com/ether-moon/<plugin>.git",
    "path": "plugins/<plugin>",
    "ref": "main"
  }
}
```

This means:
- **Plugin code stays in its own repository.** No duplication, full git history preserved per plugin.
- **Each plugin releases on its own schedule.** Updates to a plugin's `main` branch are picked up automatically by `claude plugin marketplace update`.
- **The hub repo stays tiny.** It only contains `marketplace.json` and this README; no plugin code lives here.

## Contributing

Issues and pull requests for individual plugins should go to **the plugin's source repository** (linked in the table above), not this marketplace repo.

This repo accepts:
- Marketplace metadata fixes (descriptions, ordering)
- New plugin additions
- Documentation improvements

## Backward compatibility

Each plugin's source repository also ships its own `marketplace.json` for direct, single-plugin installation. Existing users who installed a plugin via its source repo's marketplace remain unaffected. New users are encouraged to use this hub for unified discovery.
