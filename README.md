# Serply Agent Skills

Skills that teach a coding agent how to use the [Serply API](https://serply.io):
which endpoint returns which kind of result, how auth works, the hosted MCP
server's fourteen tools, and the gotchas that cost credits when guessed wrong.

| Skill | What it covers |
|---|---|
| [`searching-with-serply`](skills/searching-with-serply/SKILL.md) | Google Search, Bing, News, Maps, Images, Jobs, Scholar, Video, Amazon, eBay and Reddit results, URL scraping to markdown, and the MCP server |

The skill follows the [Agent Skills](https://agentskills.io/specification)
format, so it works in Claude Code, claude.ai, the Claude API, and any other
client that reads `SKILL.md` files. It is also served at
[serply.io/skill.md](https://serply.io/skill.md).

## Install

With the [`skills`](https://skills.sh) CLI, into the current project or your
user-level skills directory:

```bash
npx skills add serply-inc/skills
```

As a Claude Code plugin:

```bash
claude plugin marketplace add serply-inc/skills
claude plugin install serply@serply-plugins
```

Or copy the file by hand:

```bash
mkdir -p .claude/skills/searching-with-serply
curl -o .claude/skills/searching-with-serply/SKILL.md https://serply.io/skill.md
```

You also need a Serply API key, set as an environment variable and passed in
the `X-Api-Key` header. New accounts get 2,500 free credits with no card at
[app.serply.io/users/sign_up](https://app.serply.io/users/sign_up).

## Verifying

Every endpoint, parameter and response shape the skill describes was checked
against the live API. To validate the skill file against the specification:

```bash
uvx --from skills-ref agentskills validate skills/searching-with-serply
```

## Related

- [Agent Skill guide](https://serply.io/docs/guides/agent-skill) on serply.io
- [Serply MCP server](https://github.com/serply-inc/mcp), the hosted server the skill points agents at
- [API reference](https://serply.io/docs)

## License

MIT. See [LICENSE](LICENSE).
