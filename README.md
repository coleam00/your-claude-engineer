# Your Claude Engineer

**Your own AI software engineer that manages projects, writes code, and communicates progress — autonomously.**

Ever wished you could hand off a feature request and have it come back fully implemented, tested, and documented? Your Claude Engineer is a harness built on top of the Anthropic Harness for long running tasks and using the [Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python) that turns Claude into a long-running software engineer capable of tackling complex, multi-step tasks that go far beyond a single prompt.

It's a complete engineering workflow leveraging subagents to handle distinct concerns:

- **Project Management**: Creates and tracks work in Linear, breaking down features into issues and updating status as work progresses
- **Code Implementation**: Writes, tests, and iterates on code with browser-based UI verification via Playwright
- **Version Control**: Commits changes, creates branches, and opens pull requests on GitHub
- **Communication**: Keeps you informed with progress updates in Slack

The multi-agent architecture uses specialized agents (Linear, Coding, GitHub, Slack) coordinated by an orchestrator, enabling longer autonomous sessions without context window exhaustion. All external service integrations are powered by the [Arcade MCP server](https://arcade.dev), providing seamless OAuth authentication across Linear, GitHub, and Slack through a single gateway. The system also leverages Claude's tool discovery for context-optimized MCP interactions.

## Key Features

- **Long-Running Autonomy**: Harness architecture enables extended coding sessions across multiple iterations
- **Multi-Agent Orchestration**: Specialized agents handle distinct concerns (project management, coding, version control, communication)
- **Linear Integration**: Automatic issue tracking with real-time status updates and session handoff
- **GitHub Integration**: Automatic commits, branches, and PR creation
- **Slack Notifications**: Progress updates delivered to your team
- **Arcade MCP Gateway**: Single authentication flow for all external services (Linear, GitHub, Slack)
- **Browser Testing**: Playwright MCP for automated UI verification
- **Model Configuration**: Per-agent model selection (Haiku, Sonnet, or Opus)

## Prerequisites

> Note that this doesn't work on Windows because of limitations with the Claude Agent SDK and subagents. Use WSL or a Linux VM to run it!

### 0. Set Up Python Virtual Environment (Recommended)

```bash
# Create virtual environment
python3 -m venv venv

# Activate it
source venv/bin/activate  # On macOS/Linux
# or
venv\Scripts\activate  # On Windows
```

### 1. Install Claude Code CLI and Python SDK

```bash
# Install Claude Code CLI (latest version required)
npm install -g @anthropic-ai/claude-code

# Install Python dependencies
pip install -r requirements.txt
```

### 2. Set Up Authentication

```bash
# Copy the example environment file
cp .env.example .env

# Edit .env with your credentials:
# - ARCADE_API_KEY: Get from https://api.arcade.dev/dashboard/api-keys
# - ARCADE_GATEWAY_SLUG: Create at https://api.arcade.dev/dashboard/mcp-gateways
# - ARCADE_USER_ID: Your email for user tracking

# Authorize Arcade tools (run once)
python authorize_arcade.py
```

<details>
<summary><strong>Environment Variables Reference</strong></summary>

| Variable | Description | Required |
|----------|-------------|----------|
| `ARCADE_API_KEY` | Arcade API key from https://api.arcade.dev/dashboard/api-keys | Yes |
| `ARCADE_GATEWAY_SLUG` | Your Arcade MCP gateway slug | Yes |
| `ARCADE_USER_ID` | Your email for user tracking | Recommended |
| `GENERATIONS_BASE_PATH` | Base directory for generated projects (default: ./generations) | No |
| `GITHUB_REPO` | GitHub repo in format `owner/repo` for auto-push | No |
| `SLACK_CHANNEL` | Slack channel name (without #) for notifications | No |
| `ORCHESTRATOR_MODEL` | Model for orchestrator: haiku, sonnet, opus (default: haiku) | No |
| `LINEAR_AGENT_MODEL` | Model for Linear agent (default: haiku) | No |
| `CODING_AGENT_MODEL` | Model for coding agent (default: sonnet) | No |
| `GITHUB_AGENT_MODEL` | Model for GitHub agent (default: haiku) | No |
| `SLACK_AGENT_MODEL` | Model for Slack agent (default: haiku) | No |

</details>

### 3. Verify Installation

```bash
claude --version  # Should be latest version
pip show claude-agent-sdk  # Check SDK is installed
```

## Quick Start

```bash
# Basic usage - creates project in ./generations/my-app/
uv run python autonomous_agent_demo.py --project-dir my-app

# Specify custom output location
uv run python autonomous_agent_demo.py --generations-base ~/projects/ai --project-dir my-app

# Limit iterations for testing
uv run python autonomous_agent_demo.py --project-dir my-app --max-iterations 3

# Use Opus for orchestrator (more capable but higher cost)
uv run python autonomous_agent_demo.py --project-dir my-app --model opus
```

## How It Works

### Multi-Agent Orchestration

```
┌───────────────────────────────────────────────────────────────┐
│                   MULTI-AGENT ARCHITECTURE                    │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│                    ┌─────────────────┐                        │
│                    │  ORCHESTRATOR   │  (Haiku by default)    │
│                    │   Coordinates   │                        │
│                    └────────┬────────┘                        │
│                             │                                 │
│           ┌─────────────────┼─────────────────┐               │
│           │                 │                 │               │
│      ┌────▼─────┐    ┌─────▼──────┐   ┌─────▼──────┐          │
│      │  LINEAR  │    │   CODING   │   │   GITHUB   │          │
│      │  (Haiku) │    │  (Sonnet)  │   │  (Haiku)   │          │
│      └──────────┘    └────────────┘   └────────────┘          │
│           │                │                 │                │
│      ┌────▼─────┐          │                 │                │
│      │  SLACK   │          │                 │                │
│      │ (Haiku)  │          │                 │                │
│      └──────────┘          │                 │                │
│                            │                 │                │
│     ┌──────────────────────▼─────────────────▼──────┐         │
│     │         PROJECT OUTPUT (Isolated Git)         │         │
│     │      GENERATIONS_BASE_PATH/project-name/      │         │
│     └───────────────────────────────────────────────┘         │
└───────────────────────────────────────────────────────────────┘
```

### Agent Responsibilities

1. **Orchestrator Agent:**
   - Reads project state from `.linear_project.json`
   - Queries Linear for current status
   - Decides what to work on next
   - Delegates to specialized agents via Task tool
   - Coordinates handoff between agents

2. **Linear Agent:**
   - Creates and updates Linear projects and issues
   - Manages issue status transitions (Todo → In Progress → Done)
   - Adds comments with implementation details
   - Maintains META issue for session tracking

3. **Coding Agent:**
   - Implements features based on Linear issues
   - Writes and tests application code
   - Uses Playwright for browser-based UI testing
   - Validates previously completed features

4. **GitHub Agent (Optional):**
   - Commits code changes to git
   - Creates branches and pushes to remote
   - Creates pull requests when features are ready
   - Requires `GITHUB_REPO` env var

5. **Slack Agent (Optional):**
   - Posts progress updates to Slack channels
   - Notifies on feature completion
   - Requires existing Slack channel (cannot create channels)

## Command Line Options

| Option | Description | Default |
|--------|-------------|---------|
| `--project-dir` | Project name or path (relative paths go in generations base) | `./autonomous_demo_project` |
| `--generations-base` | Base directory for all generated projects | `./generations` or `GENERATIONS_BASE_PATH` |
| `--max-iterations` | Max agent iterations | Unlimited |
| `--model` | Orchestrator model: haiku, sonnet, or opus | `haiku` or `ORCHESTRATOR_MODEL` |

## Setup Guide

### 1. Arcade Gateway Setup

1. Get API key from https://api.arcade.dev/dashboard/api-keys
2. Create MCP gateway at https://api.arcade.dev/dashboard/mcp-gateways
3. Add Linear tools to your gateway (required)
4. Optionally add GitHub and Slack tools
5. Run `python authorize_arcade.py` to authorize

### 2. Linear Workspace

Ensure you have:
- A Linear workspace with at least one team
- Linear tools added to your Arcade gateway
- The orchestrator will automatically detect your team and create projects

### 3. GitHub Integration (Optional)

To enable GitHub integration:
1. Create a GitHub repository
2. Add GitHub tools to your Arcade gateway
3. Set `GITHUB_REPO=owner/repo-name` in `.env`
4. The GitHub agent will commit and push code automatically

### 4. Slack Integration (Optional)

To enable Slack notifications:
1. Create a Slack channel (agents cannot create channels)
2. Add Slack tools to your Arcade gateway
3. Set `SLACK_CHANNEL=channel-name` in `.env`

## Customization

### Changing the Application

Edit `prompts/app_spec.txt` to specify a different application to build.

### Adjusting Issue Count

Edit `prompts/initializer_task.md` to change how many issues are created during initialization.

### Modifying Allowed Commands

Edit `security.py` to add or remove commands from `ALLOWED_COMMANDS`.

## Project Structure

```
linear-agent-harness/
├── autonomous_agent_demo.py  # Main entry point
├── agent.py                  # Agent session logic
├── client.py                 # Claude SDK + MCP client configuration
├── security.py               # Bash command allowlist and validation
├── progress.py               # Progress tracking utilities
├── prompts.py                # Prompt loading utilities
├── arcade_config.py          # Arcade MCP gateway configuration
├── authorize_arcade.py       # Arcade authorization flow
├── agents/
│   ├── definitions.py        # Agent definitions with model config
│   └── orchestrator.py       # Orchestrator session runner
├── prompts/
│   ├── app_spec.txt              # Application specification
│   ├── orchestrator_prompt.md    # Orchestrator system prompt
│   ├── initializer_task.md       # Task message for first session
│   ├── continuation_task.md      # Task message for continuation sessions
│   ├── linear_agent_prompt.md    # Linear subagent prompt
│   ├── coding_agent_prompt.md    # Coding subagent prompt
│   ├── github_agent_prompt.md    # GitHub subagent prompt
│   └── slack_agent_prompt.md     # Slack subagent prompt
└── requirements.txt          # Python dependencies
```

## Generated Project Structure

Projects are created in isolated directories with their own git repos:

```
generations/my-app/           # Or GENERATIONS_BASE_PATH/my-app/
├── .linear_project.json      # Linear project state (marker file)
├── app_spec.txt              # Copied specification
├── init.sh                   # Environment setup script
├── .claude_settings.json     # Security settings
├── .git/                     # Separate git repository
└── [application files]       # Generated application code
```

## MCP Servers Used

| Server | Transport | Purpose |
|--------|-----------|---------|
| **Arcade Gateway** | HTTP | Unified access to Linear, GitHub, and Slack via Arcade MCP |
| **Playwright** | stdio | Browser automation for UI testing |

The Arcade Gateway provides access to:
- **Linear**: Project management, issues, status, comments (39 tools)
- **GitHub**: Repository operations, commits, PRs, branches (46 tools, optional)
- **Slack**: Messaging and notifications (8 tools, optional)

## Security Model

This demo uses defense-in-depth security (see `security.py` and `client.py`):

1. **OS-level Sandbox:** Bash commands run in an isolated environment
2. **Filesystem Restrictions:** File operations restricted to project directory
3. **Bash Allowlist:** Only specific commands permitted (npm, node, git, curl, rm with validation, etc.)
4. **MCP Permissions:** Tools explicitly allowed in security settings
5. **Dangerous Command Validation:** Commands like `rm` are validated to prevent system directory deletion

## Troubleshooting

**"ARCADE_API_KEY not set"**
Get your API key from https://api.arcade.dev/dashboard/api-keys and set it in `.env`

**"ARCADE_GATEWAY_SLUG not set"**
Create a gateway at https://api.arcade.dev/dashboard/mcp-gateways and add Linear tools

**"Authorization required"**
Run `python authorize_arcade.py` to complete the OAuth flow

**"Command blocked by security hook"**
The agent tried to run a disallowed command. Add it to `ALLOWED_COMMANDS` in `security.py` if needed.

**"MCP server connection failed"**
Verify your Arcade API key is valid and your gateway has the required tools configured.

**"GitHub agent requires GITHUB_REPO"**
If you want GitHub integration, set `GITHUB_REPO=owner/repo-name` in `.env`

**"Slack channel not found"**
Agents cannot create Slack channels. Create the channel manually and set `SLACK_CHANNEL` to the channel name (without #).

## Viewing Progress

**Linear Workspace:**
- View the project created by the orchestrator
- Watch real-time status changes (Todo → In Progress → Done)
- Read implementation comments on each issue
- Check session summaries on the META issue

**GitHub (if configured):**
- View commits pushed to your repository
- Review pull requests created by the GitHub agent

**Slack (if configured):**
- Receive progress updates in your configured channel
- Get notifications when features are completed

## License

MIT License - see [LICENSE](LICENSE) for details.
