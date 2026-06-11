# Subagent Definitions

Standalone reusable subagent definition files for use in skills and orchestrated workflows.

Each file defines a named agent with a specific role, tool access, and behavioral instructions.

## File format

```markdown
---
name: my-agent
description: What this agent does and when to use it
tools: [Read, Write, Bash, Grep]
---

# Agent Name

Instructions for the agent...
```

## Usage

Reference agent files from skill workflows using the Task tool with the agent file path.
