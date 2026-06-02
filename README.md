# Model Serving Skills

Claude Code skills for the **RHOAI Model Serving team**.

This repository is registered in the [opendatahub-io skills-registry](https://github.com/VedantMahabaleshwarkar/skills-registry) as a plugin marketplace entry.

## Skills

| Skill | Description |
|-------|-------------|
| [jira-hygiene](skills/jira-hygiene/SKILL.md) | Enforce JIRA hygiene standards — story points, team, components, activity type, fix version, and labels |

## Installation

### Via the skills-registry marketplace

```bash
# Add the marketplace (one-time)
/plugin marketplace add https://github.com/VedantMahabaleshwarkar/skills-registry

# Install the plugin
/plugin install model-serving-skills@opendatahub-skills
```

### Direct install

```bash
/plugin marketplace add https://github.com/VedantMahabaleshwarkar/model-serving-skills
/plugin install model-serving-skills
```

## Prerequisites

Skills in this repo require JIRA API access via environment variables:

| Variable | Purpose |
|----------|----------|
| `JIRA_BASE_URL` | Atlassian instance URL (e.g. `https://redhat.atlassian.net`) |
| `JIRA_EMAIL` | User's email for basic auth |
| `JIRA_API_TOKEN` | Atlassian API token |

## Contributing

To add a new skill for the Model Serving team:

1. Create a directory under `skills/<skill-name>/`
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`)
3. Open a PR
