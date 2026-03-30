# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a **Claude Code plugin marketplace** that provides automated codebase analysis and skill generation. The repository contains a single plugin (`agentic-master-skills`) that uses 11 specialized agents to detect tech stacks and generate tailored Claude Code skills.

## Development Commands

### Plugin Validation

```bash
# Validate marketplace structure
/plugin validate .

# Using CLI (if available)
claude plugin validate .
```

### Local Testing

```bash
# Add the marketplace locally
/plugin marketplace add agentic-master/marketplace

# Install the plugin
/plugin install agentic-master-skills@agentic-master

# Run the analysis command
/agentic-master-skills:analyze-and-generate-skills
```

## Architecture

### Marketplace Structure

```
.
├── .claude-plugin/
│   └── marketplace.json           # Marketplace metadata and plugin registry
├── plugins/
│   └── codebase-analyzer/
│       ├── .claude-plugin/
│       │   └── plugin.json        # Plugin configuration and entry points
│       ├── commands/
│       │   └── analyze-and-generate-skills.md    # Main slash command
│       └── agents/
│           ├── tech-stack-detector.md           # Detects all technologies
│           ├── security-analyzer.md             # Security best practices
│           ├── performance-analyzer.md          # Performance optimization
│           ├── react-analyzer.md                # React patterns (conditional)
│           ├── backend-analyzer.md              # Backend patterns (conditional)
│           ├── frontend-analyzer.md             # Frontend patterns (conditional)
│           ├── database-analyzer.md             # Database patterns (conditional)
│           ├── testing-analyzer.md              # Testing strategies
│           ├── architecture-analyzer.md         # Code structure analysis
│           ├── dependency-analyzer.md           # Dependency management
│           └── code-quality-analyzer.md         # Quality standards
├── README.md
└── LICENSE
```

### Key Components

1. **Marketplace Definition** (`.claude-plugin/marketplace.json`)
   - Defines marketplace metadata (name, owner, description, version)
   - Registers available plugins with their source paths and metadata

2. **Plugin Definition** (`plugins/codebase-analyzer/.claude-plugin/plugin.json`)
   - Plugin metadata (name, description, version, author)
   - Lists all commands and agents the plugin provides
   - Acts as the plugin's entry point and manifest

3. **Slash Command** (`commands/analyze-and-generate-skills.md`)
   - Main user-facing command: `/agentic-master-skills:analyze-and-generate-skills`
   - Orchestrates the entire analysis workflow
   - Controls agent activation based on tech stack detection
   - Manages skill generation lifecycle

4. **Agents** (`agents/*.md`)
   - **tech-stack-detector**: Always runs first to identify technologies
   - **Always-on agents**: security, performance, architecture, dependency, code-quality
   - **Conditional agents**: react, backend, frontend, database, testing (only run if relevant tech detected)

### Workflow Architecture

The plugin follows a **detect-then-analyze** pattern:

1. **Detection Phase**
   - Tech stack detector scans dependency files (package.json, requirements.txt, *.csproj, etc.)
   - Identifies frontend frameworks, backend frameworks, databases, testing tools, build systems
   - Supports: Node.js, Python, Ruby, Rust, Go, Java, PHP, .NET/.NET Core

2. **Analysis Phase**
   - Activates relevant agents based on detection results
   - Agents run in parallel where possible
   - Each agent generates domain-specific insights

3. **Generation Phase**
   - Creates `.claude/skills/` directory structure
   - Each skill gets its own subdirectory (prefixed with "agentic-master-")
   - Generates `SKILL.md` files (case-sensitive, uppercase) with:
     - YAML frontmatter (name, description) starting on line 1
     - Markdown instructions for applying the skill
   - Skills are model-invoked (Claude automatically uses them when relevant)

### Tech Stack Support

The plugin comprehensively detects and supports:

- **Frontend**: React, Vue, Angular, Svelte, UI libraries (Material-UI, Tailwind, Bootstrap), state management (Redux, Zustand, MobX, Vuex)
- **Backend**: Express, Fastify, NestJS, Django, Flask, FastAPI, Rails, Sinatra, Spring Boot, Gin, Echo, Fiber, **ASP.NET Core** (MVC, Web API, Minimal APIs)
- **Database**: PostgreSQL, MySQL, SQLite, MongoDB, Redis, Cassandra, ORMs
- **Testing**: Jest, Vitest, Mocha, Cypress, Playwright, pytest, RSpec, JUnit, xUnit, NUnit, MSTest
- **Build Tools**: Webpack, Vite, Rollup, Parcel, TypeScript, ESLint, Prettier
- **.NET/.NET Core**: Full support for ASP.NET Core patterns, middleware, DI, configuration, hosted services, Serilog, Swagger/OpenAPI

### Skill File Format Requirements

Generated skills must follow this exact structure:

```markdown
---
name: agentic-master-{skill-name}
description: Clear description with specific capabilities and trigger terms that users would mention
---

# Skill Title

[Markdown instructions on how to apply this skill...]
```

**Critical Requirements**:
- Filename must be exactly `SKILL.md` (uppercase, case-sensitive)
- YAML frontmatter must start on line 1 with no blank lines before it
- `name` field must match the subdirectory name
- `description` field is critical—Claude reads descriptions to find relevant skills, so include keywords users would naturally say

## Editing Agents or Commands

### Agent Files

When modifying agent files in `plugins/codebase-analyzer/agents/`:
- Agents are written in Markdown and define specialized analysis roles
- Each agent should focus on a single domain (security, performance, testing, etc.)
- Agents specify what to analyze and what output format to generate
- Conditional agents (react, backend, frontend, database) should include activation conditions
- Tech stack detector agent should be updated when adding support for new languages/frameworks

### Command Files

When modifying `plugins/codebase-analyzer/commands/analyze-and-generate-skills.md`:
- This file defines the main slash command workflow
- It orchestrates agent activation in the correct order (detect → analyze → generate)
- It specifies the skill generation requirements (directory structure, SKILL.md format, YAML frontmatter)
- Changes here affect how users interact with the entire plugin

### Plugin Metadata

When modifying `plugins/codebase-analyzer/.claude-plugin/plugin.json`:
- Must list all commands and agents for them to be recognized
- Paths are relative to the plugin directory
- Version should follow semantic versioning
- Any new agent or command files must be added to the appropriate arrays

### Marketplace Metadata

When modifying `.claude-plugin/marketplace.json`:
- This registers the plugin in the marketplace
- Update version when releasing changes
- Source paths are relative to the marketplace root

## Best Practices for Development

### Adding New Agents

1. Create new agent file in `plugins/codebase-analyzer/agents/`
2. Define the agent's specialized role and output format
3. Add the agent path to `plugin.json` agents array
4. Update `analyze-and-generate-skills.md` to orchestrate the new agent
5. Determine if the agent should always run or be conditional

### Adding Support for New Tech Stacks

1. Update `tech-stack-detector.md` with new dependency files to scan
2. Update relevant analyzer agents (backend, frontend, etc.) with framework-specific patterns
3. Consider if a new specialized agent is needed (e.g., a GraphQL-specific agent)
4. Test detection and skill generation with representative codebases

### Skill Generation Guidelines

Generated skills should:
- Be specific to the detected tech stack (not generic advice)
- Include actionable code patterns and examples
- Reference framework-specific best practices
- Provide clear "do this, not that" guidance
- Use trigger terms in descriptions that match how users naturally ask questions

### Version Management

- Marketplace version in `.claude-plugin/marketplace.json` and plugin version in `plugin.json` can differ
- Update marketplace version when changing marketplace structure or adding/removing plugins
- Update plugin version when changing agent behavior, command workflow, or output format
- Follow semantic versioning (MAJOR.MINOR.PATCH)
