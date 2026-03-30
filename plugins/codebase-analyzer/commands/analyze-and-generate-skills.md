# /analyze-and-generate-skills

Analyze the codebase and generate a set of tailored Claude Code skills that will make future code generation smarter and more project-aware.

## How it works

1. **Detect** — run the `tech-stack-detector` agent first to get a structured inventory of all technologies in use.
2. **Analyze in parallel** — activate the relevant domain agents based on what was detected.
3. **Generate skills** — each agent writes a `SKILL.md` into its assigned directory under `.claude/skills/`.

---

## Step 1: Tech Stack Detection

Run `tech-stack-detector` and capture its JSON output. The output determines which agents to activate next.

Scan: `package.json`, `requirements.txt`, `Pipfile`, `Gemfile`, `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`, `composer.json`, `*.csproj`, `*.sln`, `Directory.Packages.props`, `global.json`, `NuGet.config`, `packages.lock.json`.

Expected output format:
```json
{
  "frontend": ["React", "TypeScript", "Tailwind CSS", "Vite"],
  "backend": ["ASP.NET Core", ".NET 8", "MassTransit"],
  "database": ["PostgreSQL", "Entity Framework Core"],
  "testing": ["xUnit", "Vitest", "Playwright"],
  "buildTools": ["Vite", "dotnet"],
  "other": ["RabbitMQ", "Docker", "Firebase"]
}
```

---

## Step 2: Activate Agents in Parallel

Based on the detection JSON, activate these agents simultaneously:

| Agent | Condition |
|---|---|
| `security-analyzer` | Always |
| `performance-analyzer` | Always |
| `architecture-analyzer` | Always |
| `dependency-analyzer` | Always |
| `code-quality-analyzer` | Always |
| `react-analyzer` | `frontend` array contains React |
| `frontend-analyzer` | `frontend` array has any non-React framework (Vue, Angular, Svelte, etc.) |
| `backend-analyzer` | `backend` array is non-empty |
| `database-analyzer` | `database` array is non-empty |
| `testing-analyzer` | `testing` array is non-empty |

Pass the full detection JSON to every agent so they can tailor their output and avoid duplicating each other's coverage.

---

## Step 3: Skill Generation

Each agent writes its output to `.claude/skills/<skill-dir>/SKILL.md`. If the directory already exists, overwrite the skill — the codebase may have changed since the last run.

**Directory → Agent mapping:**
- `agentic-master-security/` → security-analyzer
- `agentic-master-performance/` → performance-analyzer
- `agentic-master-architecture/` → architecture-analyzer
- `agentic-master-dependency-management/` → dependency-analyzer
- `agentic-master-code-quality/` → code-quality-analyzer
- `agentic-master-react/` → react-analyzer
- `agentic-master-frontend/` → frontend-analyzer
- `agentic-master-backend/` → backend-analyzer
- `agentic-master-database/` → database-analyzer
- `agentic-master-testing/` → testing-analyzer

**SKILL.md format — every agent must follow this exactly:**

```
---
name: agentic-master-{skill-name}
description: {one to three sentences — see guidance below}
---

# {Title}

{Skill body — see guidance below}
```

The `name` must match the subdirectory name. The frontmatter must start on line 1 with no blank lines before it.

### Writing a good description

The description is how Claude decides whether to load a skill. A weak description means the skill gets ignored even when it would help. A good description:
- Names the specific technologies ("xUnit", "Entity Framework Core") not generic terms ("database testing")
- Includes both a purpose statement and a list of trigger situations
- Is slightly "pushy" — if in doubt, Claude should load the skill

**Example:**
> Security best practices for a .NET 8 / React microservices stack. Load this when the user is implementing auth, handling secrets, configuring HTTPS, writing input validation, or mentions security, vulnerabilities, credentials, JWT, Firebase, or Azure security.

### Writing a good skill body

The body should read like advice from a senior engineer on this specific project — not generic documentation.

**Target length: 200–400 lines.** If you have more to say, use progressive disclosure:
- Keep the most-used patterns in SKILL.md
- Move deep-dive reference material to `references/<topic>.md` and link to it clearly

**Quality checklist:**
- Every guideline is grounded in this project's actual code, libraries, or conventions — not generic advice
- Code examples use the project's real types, namespaces, and patterns (read the actual source files)
- Includes "do this / not that" pairs where the distinction matters
- Explains *why* behind important rules, not just what
- Avoids duplicating content owned by another skill (trust that the other skills exist)

---

## Step 4: Summary

After all skills are written, output a summary listing:
- Which skills were generated (and which were skipped and why)
- A one-sentence description of each generated skill
- The path to each skill file
