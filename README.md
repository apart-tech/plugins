# Apart Tech Plugins

Claude Code plugins and skills from [Apart Tech](https://github.com/apart-tech).

## Skills

### evaluating-dependencies

Evaluate npm packages for build-vs-buy decisions before adding them to your project.

**What it does:** Gathers package metadata, analyzes source complexity, checks security and maintenance health, and produces a structured **USE / EXTRACT / BUILD** verdict.

**Works two ways:**
- **On demand:** `/apart:evaluating-dependencies lodash.get`
- **Auto-gate:** Triggers when Claude is about to add a new dependency

## Install

### From the Apart marketplace

Add the Apart marketplace to your Claude Code config, then install:

```bash
# The plugin will be available after adding the marketplace
/plugin install apart@apart-marketplace
```

### From GitHub directly

```bash
npx skills add apart-tech/plugins@evaluating-dependencies -g -y
```

## Contributing

See [SKILL-AUTHORING-GUIDE.md](SKILL-AUTHORING-GUIDE.md) for best practices on creating new skills.

## License

MIT
