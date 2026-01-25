# Custom Skill-Based Pydantic AI Agent

A framework-agnostic skill system for AI agents that implements **progressive disclosure** - a powerful pattern for managing context efficiently. This project demonstrates how to extract successful patterns from proprietary AI systems (like Claude Skills) and implement them in open frameworks like Pydantic AI.

## Key Features

- **Progressive Disclosure**: Skills load instructions and resources on-demand, eliminating context window constraints
- **Framework Agnostic**: Skills work with any AI framework, not locked to a specific vendor
- **Type Safe**: Full Pydantic models and type hints throughout
- **Extensible**: Easy to add new skills by following a simple directory structure
- **Production Ready**: Includes comprehensive testing patterns and examples

## What is Progressive Disclosure?

Progressive disclosure is a technique for managing AI context efficiently. Instead of loading all possible instructions into the system prompt, skills are loaded in three levels:

```
Level 1: Metadata (~100 tokens per skill)
    - Name and brief description in system prompt
    - Agent decides which skill might be relevant

Level 2: Full Instructions (loaded on-demand)
    - Complete SKILL.md with detailed instructions
    - Only loaded when agent chooses to use the skill

Level 3: Resources (loaded on-demand)
    - Reference files, scripts, and other resources
    - Only loaded when instructions reference them
```

This pattern allows an agent to have access to potentially hundreds of skills without overwhelming the context window.

## Quick Start

### Prerequisites

- Python 3.11 or higher
- [UV](https://docs.astral.sh/uv/) - Fast Python package manager
- An API key for an LLM provider (OpenRouter, OpenAI, Anthropic, etc.)

### Installation

1. **Install UV** (if not already installed):

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

2. **Clone the repository**:

```bash
git clone https://github.com/dynamous-community/workshops.git
cd workshops/custom-skill-agent
```

3. **Sync dependencies**:

```bash
uv sync
```

This creates a virtual environment and installs all dependencies automatically.

4. **Configure environment variables**:

Copy `.env.example` to `.env` and set your API key:

```bash
cp .env.example .env
```

Edit `.env` with your API credentials:

```bash
# LLM Provider Configuration
LLM_PROVIDER=openrouter
LLM_API_KEY=sk-or-v1-your-api-key-here
LLM_MODEL=anthropic/claude-haiku-4.5
LLM_BASE_URL=https://openrouter.ai/api/v1
```

### Running the Agent

Start the CLI agent:

```bash
uv run python -m src.cli
```

You'll see a Rich-based interface where you can interact with the agent. Try asking:
- "What's the weather in New York?"
- "Can you review this code for best practices?"

## Included Skills

### Weather Skill

A simple skill demonstrating Level 2 and Level 3 progressive disclosure:

- **Level 2**: Full instructions for getting weather data
- **Level 3**: API reference documentation for Open-Meteo

Example interaction:
```
User: What's the weather in Tokyo?
Agent: [loads weather skill] [loads API reference] The current weather in Tokyo is...
```

### Code Review Skill

An advanced skill demonstrating extensive Level 3 resources (~45KB total):

- Multi-step review workflow
- Three reference files:
  - `best_practices.md` (~18KB) - Coding standards
  - `security_checklist.md` (~24KB) - OWASP-based security review
  - `common_antipatterns.md` (~34KB) - Code smell detection

Example interaction:
```
User: Review this authentication code for security issues
Agent: [loads code_review skill] [loads security_checklist]
       I'll analyze this against security best practices...
```

## Architecture

```
custom-skill-agent/
|-- src/                      # Core agent implementation
|   |-- agent.py              # Pydantic AI agent with skill tools
|   |-- skill_loader.py       # Skill discovery and metadata parsing
|   |-- skill_tools.py        # Progressive disclosure tools
|   |-- http_tools.py         # HTTP request tools
|   |-- dependencies.py       # Agent dependencies
|   |-- providers.py          # LLM provider configuration
|   |-- settings.py           # Pydantic Settings configuration
|   |-- prompts.py            # System prompt templates
|   +-- cli.py                # Rich-based CLI interface
|
|-- skills/                   # Skill library
|   |-- weather/              # Simple weather skill
|   |   |-- SKILL.md          # Skill instructions
|   |   +-- references/       # API documentation
|   |       +-- api_reference.md
|   |
|   +-- code_review/          # Advanced code review skill
|       |-- SKILL.md          # Multi-step review workflow
|       |-- references/       # Extensive documentation (~45KB)
|       |   |-- best_practices.md
|       |   |-- security_checklist.md
|       |   +-- common_antipatterns.md
|       +-- scripts/          # Helper scripts
|           +-- lint_patterns.py
|
|-- tests/                    # Test suite
|   |-- test_skill_loader.py  # Skill loader tests
|   |-- test_skill_tools.py   # Tool tests
|   +-- test_agent.py         # Agent integration tests
|
+-- examples/                 # Reference implementations (MongoDB RAG)
```

## Creating Your Own Skill

### 1. Create the Skill Directory

```bash
mkdir -p skills/my_skill/references
```

### 2. Create SKILL.md

Every skill must have a `SKILL.md` file with YAML frontmatter:

```markdown
---
name: my_skill
description: Brief description for agent discovery (1-2 sentences)
version: 1.0.0
author: Your Name
---

# My Skill

Detailed description of what this skill does.

## When to Use

- Scenario 1
- Scenario 2

## Available Operations

1. Operation 1: Description
2. Operation 2: Description

## Instructions

Step-by-step instructions for using this skill...

## Resources

- `references/guide.md` - Detailed guide
- `scripts/helper.py` - Helper script

## Examples

### Example 1
User asks: "..."
Response approach: ...
```

### 3. Add Reference Files (Optional)

Place any supporting documentation in the `references/` directory:

```markdown
# skills/my_skill/references/guide.md

Detailed documentation that the agent can load when needed...
```

### 4. Test Your Skill

Verify your skill is discovered:

```python
from src.skill_loader import SkillLoader
from pathlib import Path

loader = SkillLoader(Path("skills"))
skills = loader.discover_skills()

for skill in skills:
    print(f"- {skill.name}: {skill.description}")
```

## How the Agent Uses Skills

### System Prompt (Level 1)

The agent's system prompt includes a summary of all available skills:

```
Available Skills:
- **weather**: Get weather information for locations using Open-Meteo API.
- **code_review**: Review code for quality, security, and best practices.
```

### Loading Skills (Level 2)

When the agent decides a skill is relevant, it calls `load_skill_tool`:

```python
# Agent automatically calls this when needed
instructions = await load_skill_tool(ctx, skill_name="weather")
# Returns: Full SKILL.md content (without frontmatter)
```

### Loading Resources (Level 3)

When instructions reference a file, the agent calls `read_skill_file_tool`:

```python
# Agent loads specific resources as needed
api_docs = await read_skill_file_tool(
    ctx,
    skill_name="weather",
    file_path="references/api_reference.md"
)
```

## Available Tools

The agent has access to these tools:

| Tool | Description |
|------|-------------|
| `load_skill_tool` | Load full instructions for a skill (Level 2) |
| `read_skill_file_tool` | Read a file from a skill's directory (Level 3) |
| `list_skill_files_tool` | List available files in a skill |
| `http_get_tool` | Make HTTP GET requests |
| `http_post_tool` | Make HTTP POST requests |

## Switching LLM Providers

The agent supports three LLM providers. Configure via `.env`:

### OpenRouter (Recommended)

```bash
LLM_PROVIDER=openrouter
LLM_API_KEY=sk-or-v1-your-key
LLM_MODEL=anthropic/claude-haiku-4.5
LLM_BASE_URL=https://openrouter.ai/api/v1

# Optional: App attribution
OPENROUTER_APP_URL=https://yourdomain.com
OPENROUTER_APP_TITLE=Your App Name
```

### OpenAI

```bash
LLM_PROVIDER=openai
LLM_API_KEY=sk-proj-your-openai-api-key
LLM_MODEL=gpt-4o-mini
# LLM_BASE_URL not needed for OpenAI
```

### Ollama (Local)

```bash
LLM_PROVIDER=ollama
LLM_API_KEY=ollama
LLM_MODEL=llama3.2
LLM_BASE_URL=http://localhost:11434/v1
```

## Testing & Validation

### Running Tests

```bash
# Unit tests
uv run pytest tests/test_skill_loader.py -v
uv run pytest tests/test_skill_tools.py -v

# Integration tests
uv run pytest tests/test_agent.py -v

# All tests
uv run pytest tests/ -v
```

### Running Evaluations

The project includes a comprehensive evaluation system with YAML-based datasets and custom evaluators:

```bash
# Run all evals
uv run python -m tests.evals.run_evals

# Run specific dataset
uv run python -m tests.evals.run_evals --dataset skill_loading

# Verbose output with reasons
uv run python -m tests.evals.run_evals --verbose
```

### Validation Scripts

```bash
# Test agent interactively with predefined queries
uv run python -m scripts.test_agent

# Validate skill structure and content
uv run python -m scripts.validate_skills

# Run full validation pipeline
uv run python -m scripts.run_full_validation
```

## Observability with Logfire (Optional)

Enable Logfire for production monitoring and debugging:

1. Get Logfire token: `logfire auth`
2. Set in `.env`: `LOGFIRE_TOKEN=your-token`
3. Run agent - traces appear at https://logfire.pydantic.dev

Without token, Logfire is disabled and agent works normally.

