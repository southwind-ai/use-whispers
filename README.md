# use-whispers

An agent skill for building products and integrations on top of [Southwind Whispers](https://app.southwind.ai) — an AI-powered analytics and report generation platform.

## What's in here

```
skills/use-whispers/
├── SKILL.md       # Agent skill — integration guide and workflows
└── reference.md   # Curated API endpoint catalog
```

## What it does

When this skill is installed, the agent becomes a Whispers integration specialist. It can guide you through:

- **Connecting data** — uploading files (CSV, XLSX, PDF) or registering external REST APIs as data sources
- **Generating reports** — running AI agents (`custom_report`, `rover_report`, `blank_report`) against your data
- **Rendering results** — interpreting the report output, including chart blocks
- **Advanced workflows** — real-time SSE streaming, chat over reports, public sharing, and export (PDF/PPTX/DOCX)

## API

**Base URL:** `https://app.southwind.ai/api`  
**Auth:** `X-API-Key: ak_...` on every request  
**Docs:** `https://app.southwind.ai/api/docs`

## Getting started

1. Get an API key at `https://app.southwind.ai/settings/api-keys`
2. Install the skill in your workspace
3. Ask the agent to integrate with Whispers

## License

MIT — see [LICENSE](LICENSE)
