# Serply skill pack for Aeon

A community skill pack for [Aeon](https://github.com/aeonfun/aeon) that adds Google web and news search through the [Serply](https://serply.io) API.

| Skill | What it does |
|-------|--------------|
| `serply-search` | Searches Google web or news results for a query and sends a cited digest through `./notify`. |

## Install

```bash
bin/install-skill-pack serply-inc/skills --path aeon
```

Then add `SERPLY_API_KEY` as a repository secret. Get a key at [serply.io](https://serply.io); the API reference is at [serply.io/docs](https://serply.io/docs).

The skill installs disabled. Enable it from the dashboard or in `aeon.yml`, and pass the query as the skill's `var`.

## Options

| Secret or variable | Effect |
|--------------------|--------|
| `SERPLY_API_KEY` | Required. |
| `SERPLY_NEWS=1` | Google News results instead of web results. |
| `SERPLY_GL` | Two-letter country code, for example `de`. |
| `SERPLY_FRESHNESS` | `day`, `week`, `month`, or `year`. |
