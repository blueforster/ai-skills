# ai-skills

A collection of Claude Code skills and plugins.

## Install this marketplace

In Claude Code, run:

```
/plugin marketplace add gh:blueforster/ai-skills
```

## Available plugins

### `create-bat`

Generates correct Windows `.bat` launcher files from WSL/Linux.
Enforces all 24 rules: CRLF endings, ASCII-only, label syntax, `.cmd` traps, quoted paths, `cd /d`, and more.

**Install:**
```
/plugin install create-bat@ai-skills
```

**Use:**
```
/create-bat
```

## Adding more skills

1. Create a folder under `plugins/your-skill-name/`
2. Add `.claude-plugin/plugin.json` and `skills/your-skill-name/SKILL.md`
3. Register the plugin in `.claude-plugin/marketplace.json`
