# Claude Code: Agent Memory Frontmatter

Persistent memory for subagents — enabling agents to learn, remember, and build knowledge across sessions.

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## Table of Contents

1. [Overview](#overview)
2. [Syntax](#syntax)
3. [Memory Scopes](#memory-scopes)
4. [How It Works Under the Hood](#how-it-works-under-the-hood)
5. [Agent Memory vs Other Memory Systems](#agent-memory-vs-other-memory-systems)
6. [Practical Patterns](#practical-patterns)
7. [Tips for Effective Agent Memory](#tips-for-effective-agent-memory)
8. [Sources](#sources)

---

## Overview

Introduced in **Claude Code v2.1.33** (February 2026), the `memory` frontmatter field transforms subagents from stateless tools into **context-aware assistants** that persist knowledge across conversations.

Before this feature, every agent invocation started from scratch — a code review agent couldn't remember patterns it flagged last week, and a debugging agent couldn't recall the architecture it mapped yesterday. The `memory` field fixes this by giving each agent its own persistent markdown-based knowledge store.

---

## Syntax

Add the `memory` field to the YAML frontmatter of any agent file in `.claude/agents/`:

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Write, Edit, Bash
model: sonnet
memory: user
---

You are a code reviewer. As you review code, update your agent memory with
patterns, conventions, and recurring issues you discover.
```

The `memory` field accepts one of three scope values: `user`, `project`, or `local`.

---

## Memory Scopes

| Scope | Storage Location | Version Controlled | Shared With Team | Best For |
|-------|-----------------|-------------------|-----------------|----------|
| `user` | `~/.claude/agent-memory/<agent-name>/` | No | No | Cross-project knowledge (recommended default) |
| `project` | `.claude/agent-memory/<agent-name>/` | Yes | Yes | Project-specific knowledge the team should share |
| `local` | `.claude/agent-memory-local/<agent-name>/` | No (git-ignored) | No | Project-specific knowledge that's personal |

### Scope Selection Guide

**Use `user` when** the agent's knowledge applies across projects — coding style preferences, common anti-patterns, general best practices. This is the recommended default scope.

**Use `project` when** the agent's knowledge is codebase-specific AND should be shared — architectural decisions, project conventions, known quirks. This gets committed to version control so teammates benefit.

**Use `local` when** the agent's knowledge is codebase-specific but personal — your local environment setup, debugging notes, personal workflow preferences for this project.

### Scope Hierarchy Parallel

These scopes mirror the existing settings hierarchy:

| Agent Memory Scope | Settings Equivalent | Philosophy |
|-------------------|---------------------|------------|
| `user` | `~/.claude/settings.json` | Global personal defaults |
| `project` | `.claude/settings.json` | Team-shared project config |
| `local` | `.claude/settings.local.json` | Personal project overrides |

---

## How It Works Under the Hood

When an agent has `memory` configured:

1. **On startup**: The first 200 lines of `MEMORY.md` from the agent's memory directory are injected into the agent's system prompt
2. **Tool access**: `Read`, `Write`, and `Edit` tools are automatically enabled (regardless of the `tools` frontmatter) so the agent can manage its memory files
3. **During execution**: The agent can read from and write to its memory directory at any time
4. **Memory structure**: The agent maintains a `MEMORY.md` file and can create additional topic-specific files as needed

### Memory Directory Structure

```
~/.claude/agent-memory/code-reviewer/     # user scope example
├── MEMORY.md                              # Primary memory file (first 200 lines loaded)
├── react-patterns.md                      # Topic-specific file
└── security-checklist.md                  # Topic-specific file
```

**Important**: If `MEMORY.md` exceeds 200 lines, the agent is instructed to curate it — moving detailed notes into separate topic files and keeping the main file as a high-level index.

---

## Agent Memory vs Other Memory Systems

Claude Code has multiple memory mechanisms. Here's how they compare:

| System | Who Writes It | Who Reads It | Scope | Location |
|--------|--------------|-------------|-------|----------|
| **CLAUDE.md** | You (manually) | Main Claude + all agents | Project | `./CLAUDE.md` |
| **Auto-memory** | Main Claude (automatically) | Main Claude only | Per-project per-user | `~/.claude/projects/<hash>/memory/` |
| **`/memory` command** | You (via editor) | Main Claude only | Per-project per-user | Opens auto-memory files |
| **Agent memory** ✨ | The agent itself | That specific agent only | Configurable (user/project/local) | Scope-dependent (see above) |

### Key Distinctions

- **CLAUDE.md** is human-authored project instructions — shared with everyone, loaded everywhere
- **Auto-memory** is what the main Claude conversation learns about your project over time — personal, automatic
- **Agent memory** is what a specific subagent learns over repeated invocations — scoped to that agent, self-maintained
- These systems are **complementary, not overlapping** — an agent reads both CLAUDE.md (for project context) and its own memory (for agent-specific knowledge)

---

## Practical Patterns

### Pattern 1: Code Review Agent with Learning

```yaml
---
name: code-reviewer
description: Reviews PRs for quality, patterns, and conventions
memory: project
---

Review the code changes. Check your memory for known patterns and recurring
issues in this codebase. After reviewing, update your memory with any new
patterns or conventions you discovered.
```

Over time this agent builds a project-specific knowledge base of code conventions, common mistakes, and architectural patterns — shared with the whole team via `project` scope.

### Pattern 2: Architecture Explorer

```yaml
---
name: arch-explorer
description: Maps and remembers codebase architecture
memory: user
---

Explore the codebase to answer architecture questions. Consult your memory
first for previously mapped codepaths. Update your memory as you discover
new architectural decisions, module boundaries, and key integration points.
```

Using `user` scope means this agent's architectural knowledge carries across all your projects.

### Pattern 3: Debugging Assistant

```yaml
---
name: debugger
description: Investigates bugs with context from past debugging sessions
memory: local
---

Investigate the reported issue. Check your memory for similar bugs or
known problematic areas. Document your findings and resolution in memory
for future reference.
```

Using `local` scope keeps personal debugging notes private and project-specific.

### Pattern 4: Complete Agent with Skills and Memory

```yaml
---
name: api-developer
description: Implement API endpoints following team conventions
tools: Read, Write, Edit, Bash
model: sonnet
memory: project
skills:
  - api-conventions
  - error-handling-patterns
---

Implement API endpoints. Follow the conventions from your preloaded skills.
As you work, save architectural decisions and patterns to your memory.
```

This combines **skills** (static knowledge loaded at startup) with **memory** (dynamic knowledge built over time).

---

## Tips for Effective Agent Memory

### 1. Prompt the Agent to Use Its Memory

Include explicit memory instructions in the agent's markdown body:

```markdown
Before starting work, review your memory for relevant context.
After completing the task, update your memory with what you learned.
```

### 2. Request Memory Consultation

When invoking agents, you can ask them to check their memory:

> "Review this PR, and check your memory for patterns you've seen before."

### 3. Request Memory Updates

After task completion, prompt the agent to save learnings:

> "Now that you're done, save what you learned to your memory."

### 4. Include Proactive Memory Instructions in the Agent Body

For agents that should always maintain their memory:

```markdown
Update your agent memory as you discover codepaths, patterns, library
locations, and key architectural decisions. This builds institutional
knowledge across conversations. Write concise notes about what you found
and where.
```

### 5. Choose the Right Scope

| If the knowledge is... | Use scope |
|------------------------|-----------|
| Useful across all your projects | `user` |
| Codebase-specific, useful for the team | `project` |
| Codebase-specific, personal to you | `local` |

---

## Sources

- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Manage Claude's memory — Claude Code Docs](https://code.claude.com/docs/en/memory)
- [Claude Code v2.1.33 Release Notes](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
