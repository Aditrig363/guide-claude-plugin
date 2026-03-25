# Sonatype Guide Plugin for Claude Code

Integrates [Sonatype Guide](https://guide.sonatype.com) MCP server to provide Claude Code with software supply chain intelligence and dependency security analysis.

## Features

With this plugin, Claude Code can:

- **Proactively check dependencies** - Automatically evaluates packages before installing or upgrading, not just when you ask
- **Analyze vulnerabilities** - Surface CVEs with severity scores, distinguishing direct vs transitive risks
- **Recommend secure versions** - Ranked upgrade paths with Developer Trust Scores and breaking change analysis
- **Audit your project** - Scan dependency manifests for security, license, and policy compliance issues
- **Compare alternatives** - Side-by-side security comparison when choosing between libraries

## Prerequisites

You need a Sonatype Guide account and API token.

### Get Your Token

1. Visit [guide.sonatype.com/settings/tokens](https://guide.sonatype.com/settings/tokens)
2. Generate a new token
3. Copy the token value

## Setup

### 1. Set Your Environment Variable

Add your Sonatype Guide token as an environment variable. Choose one method:

**Option A: Shell profile** (recommended)

Add to `~/.zshrc`, `~/.bashrc`, or `~/.profile`:

```bash
export SONATYPE_GUIDE_TOKEN="your-token-here"
```

Then reload:
```bash
source ~/.zshrc  # or ~/.bashrc
```

**Option B: Claude Code settings**

Add to `.claude/settings.json` or `~/.claude/settings.json`:

```json
{
  "env": {
    "SONATYPE_GUIDE_TOKEN": "your-token-here"
  }
}
```

### 2. Install the Plugin

```bash
claude plugin install sonatype-guide
```

### 3. Verify Installation

Check the MCP server status:

```
/mcp
```

You should see `sonatype-guide` listed as connected.

## Usage

The plugin includes a skill that activates automatically when Claude installs, adds, or upgrades dependencies — no special syntax needed. You can also ask directly:

```
Scan my package.json for vulnerable dependencies
```

```
What's the most secure version of spring-core I should use?
```

```
Should I use axios or got for HTTP requests?
```

## Troubleshooting

**MCP server not connecting:**
- Verify your token: `echo $SONATYPE_GUIDE_TOKEN`
- Ensure your token is valid at [guide.sonatype.com](https://guide.sonatype.com)
- Restart Claude Code after setting the environment variable

**Token not recognized:**
- If using shell profile, restart your terminal
- If using settings.json, check JSON syntax
- Variable name must be exactly `SONATYPE_GUIDE_TOKEN`

## Security

Never commit your token to version control. The plugin uses environment variable expansion to keep credentials secure and user-specific.

## Links

- [Sonatype Guide](https://guide.sonatype.com)
- [Sonatype](https://www.sonatype.com)
- [Claude Code Plugin Documentation](https://docs.anthropic.com/en/docs/claude-code/plugins)
