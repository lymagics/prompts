# Claude Code Rules & Skills

A curated collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) rules and custom skills for software development. These configurations enforce consistent coding standards, architectural patterns, and development workflows when working with Claude Code as your AI pair programmer.

## What's Inside

### Rules (`.claude/rules/`)

Rules are project-level instructions that Claude Code automatically follows during every interaction.

| Rule | Description |
|------|-------------|
| **[backend.md](.claude/rules/backend.md)** | Python backend architecture — modular monolith with onion / ports-and-adapters layering, framework-free domain, repository pattern, unit of work, service layer, route-owned transactions, and an HTTP-only API layer |
| **[frontend.md](.claude/rules/frontend.md)** | React application development — component design, state management, routing, API client architecture, authentication, performance optimization, and deployment |
| **[testing.md](.claude/rules/testing.md)** | Testing philosophy — logic-free tests, fakes over mocks, test isolation, naming conventions, TDD workflow, and quality tools (Hypothesis, mutmut, PyHamcrest) |
| **[ci.md](.claude/rules/ci.md)** | CI/CD pipeline design — modular jobs, coverage reporting with Codecov, and local reproducibility via [act](https://github.com/nektos/act) |
| **[workflow.md](.claude/rules/workflow.md)** | Agent workflow orchestration — plan-first approach, subagent strategy, task management, self-improvement loop, and verification standards |

### Skills (`.claude/skills/`)

Skills are reusable, user-invocable capabilities that extend Claude Code with domain-specific workflows.

| Skill | Trigger | Description |
|-------|---------|-------------|
| **[pr-review-expert](.claude/skills/pr-review-expert/SKILL.md)** | `/pr-review-expert` | Structured PR/MR review with blast radius analysis, security scanning, breaking change detection, test coverage delta, and a 30+ item checklist |
| **[test-runner](.claude/skills/test-runner/SKILL.md)** | `/test-runner` | Runs unit tests, e2e tests, and linters (black, flake8, ruff) after Python code changes; enforces >= 90% coverage |

### MCP Servers (`.mcp.json`)

| Server | Purpose |
|--------|---------|
| **Playwright** | Browser automation for testing and interaction via `@playwright/mcp` |

## Usage

1. Clone this repository (or copy the `.claude/` directory) into your project root.
2. Adjust rules and skills to fit your project's stack and conventions.
3. Claude Code will automatically pick up the rules from `.claude/rules/` and make skills available via slash commands.

For more on Claude Code configuration, see the [official documentation](https://docs.anthropic.com/en/docs/claude-code).

## Acknowledgements

The following rules sections are based on ideas and patterns from these books:

- **Frontend** — [React Mega-Tutorial](https://learn.miguelgrinberg.com/product/react-mega-tutorial) by Miguel Grinberg
- **Testing** — [Angry Tests](https://www.yegor256.com/angry-tests.html) by Yegor Bugayenko

## License

[MIT](LICENSE)
