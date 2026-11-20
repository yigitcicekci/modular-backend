# Modular Backend

[![skills.sh](https://skills.sh/b/yigitcicekci/modular-backend)](https://skills.sh/yigitcicekci/modular-backend)

An MIT-licensed Agent Skill for building and revising NestJS/Node.js backends with clear module boundaries, efficient data access, explicit Redis roles, practical identity workflows, reliable background work, and CI/CD.

## Use with an agent

Run this from your project and select your coding agent.

```sh
npx skills add yigitcicekci/modular-backend --skill modular-backend
```

Keep the `modular-backend` directory intact and add it through your agent's supported skill discovery or project-instruction mechanism. The core follows the [Agent Skills format](https://agentskills.io/specification): `SKILL.md` contains the entry instructions and relative links to task-specific references.

Claude, Codex, Cursor, and Gemini-based coding agents can read the same instructions. Native skill discovery, installation paths, and invocation syntax depend on the client/version; this package does not assume one universal installer or claim validation in every client. Where native discovery is unavailable, use a project instruction such as:

```text
For NestJS/Node.js backend work, read ./modular-backend/SKILL.md and follow
the references relevant to the task. Preserve the current project's contracts
and keep changes within the requested scope.
```

If supported by the host, invoke the skill by its name, `modular-backend`. The optional `agents/openai.yaml` provides UI metadata for clients that understand it; it is not needed by the core instructions. No MCP server, provider SDK, shell alias, or external account is required to read/use the skill. Backend implementation tools and credentials depend on the actual project.

## Scope

- New backends: connected modules, config, Redis roles, persistence and necessary behavior checks.
- Existing backends: scoped changes preserving public contracts, transaction behavior, data and deployment compatibility.
- Reference topics: architecture, data/Redis, identity, operations, background jobs and CI/CD.

Example requests:

```text
Use modular-backend to build a NestJS API with email login and mobile devices.
Use modular-backend to extract this query into a repository without changing the API.
Use modular-backend to remove N+1 reads from this endpoint and verify query count.
Use modular-backend to add CI gates and a safe release path to this backend.
```

Examples are illustrative fragments to adapt to installed versions, not a complete starter application. PostgreSQL/Prisma, BullMQ, GitHub Actions, and Docker examples do not mandate those technologies in an existing project. Redis is a baseline architecture choice of this skill; it is not permission to expand an unrelated patch. No automatic production deployment is authorized by loading the skill.

## License

[MIT](LICENSE). Linked external documentation retains its own licensing. Examples are original, generic illustrations rather than private project source.
