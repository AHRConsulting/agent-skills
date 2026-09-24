# Contributing

Keep each skill self-contained and compliant with the Agent Skills specification.

- Put canonical instructions in `plugins/<plugin>/skills/<skill>/SKILL.md`.
- Do not duplicate skill bodies into tool-specific adapters.
- Add only the minimum metadata needed by a tool-specific manifest.
- Update a plugin version whenever its installed content changes.
- Validate JSON, Markdown links, and skill frontmatter before opening a pull request.
- Do not include credentials, private repository context, conversation transcripts, or reasoning
  logs.
