# Fantasy Football MCP Server - Quick Start Guide

A comprehensive guide to set up and use the Yahoo Fantasy Football MCP service with Cursor, Claude Code CLI, or Claude Desktop.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Initial Setup](#2-initial-setup)
3. [Yahoo API Authentication](#3-yahoo-api-authentication)
4. [MCP Client Configuration](#4-mcp-client-configuration)
5. [Using the Tools](#5-using-the-tools)
6. [Token Management](#6-token-management)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Prerequisites

- **Python 3.8+** installed
- **Yahoo Fantasy Sports account** with active leagues
- One of these MCP clients:
  - Cursor IDE
  - Claude Code CLI (`claude` command)
  - Claude Desktop app

---

## 2. Initial Setup

### 2.1 Clone and Install Dependencies

```bash
# Clone the repository
git clone https://github.com/derekrbreese/fantasy-football-mcp-public.git
cd fantasy-football-mcp-public

# Create virtual environment (recommended)
python -m venv env
source env/bin/activate  # On Windows: env\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### 2.2 Create Yahoo Developer App

1. Go to https://developer.yahoo.com/apps/
2. Click **"Create an App"**
3. Fill in:
   - **Application Name**: Fantasy Football MCP
   - **Application Type**: Web Application
   - **Redirect URI(s)**: `oob`
   - **API Permissions**: Fantasy Sports (Read)
4. Click **"Create App"**
5. Save your **Client ID** and **Client Secret**

### 2.3 Create `.env` File

Create a `.env` file in the project root:

```env
# Yahoo API Credentials (Required)
YAHOO_CLIENT_ID=your_client_id_here
YAHOO_CLIENT_SECRET=your_client_secret_here

# These will be auto-filled by authentication script
YAHOO_ACCESS_TOKEN=
YAHOO_REFRESH_TOKEN=
YAHOO_GUID=
```

---

## 3. Yahoo API Authentication

### 3.1 Run Authentication Script

```bash
cd utils
python setup_yahoo_auth.py
```

This will:
1. Open your browser for Yahoo login
2. Ask you to authorize the app
3. Save tokens to `.env` file automatically
4. Update MCP config files if they exist

### 3.2 Get Your Yahoo GUID

If GUID wasn't set automatically, run:

```bash
cd /path/to/fantasy-football-mcp-public
source env/bin/activate
python -c "
import os, requests
from dotenv import load_dotenv
load_dotenv()
token = os.getenv('YAHOO_ACCESS_TOKEN')
resp = requests.get(
    'https://fantasysports.yahooapis.com/fantasy/v2/users;use_login=1?format=json',
    headers={'Authorization': f'Bearer {token}', 'Accept': 'application/json'}
)
if resp.status_code == 200:
    data = resp.json()
    guid = data.get('fantasy_content', {}).get('users', {}).get('0', {}).get('user', [{}])[0].get('guid')
    print(f'Your YAHOO_GUID is: {guid}')
"
```

Add the GUID to your `.env` file:
```env
YAHOO_GUID=YOUR_GUID_HERE
```

### 3.3 Verify Setup

```bash
python utils/verify_setup.py
```

Expected output: All ✅ checkmarks

---

## 4. MCP Client Configuration

### 4.1 Cursor IDE

**Config file:** `~/.cursor/mcp.json`

```json
{
  "mcpServers": {
    "yahoo-fantasy-football": {
      "command": "python",
      "args": [
        "/absolute/path/to/fantasy-football-mcp-public/fantasy_football_multi_league.py"
      ],
      "env": {
        "YAHOO_CLIENT_ID": "your_client_id",
        "YAHOO_CLIENT_SECRET": "your_client_secret",
        "YAHOO_ACCESS_TOKEN": "your_access_token",
        "YAHOO_REFRESH_TOKEN": "your_refresh_token",
        "YAHOO_GUID": "your_guid"
      }
    }
  }
}
```

**After setup:** Restart Cursor

---

### 4.2 Claude Code CLI

**Config file:** `~/.claude/mcp.json` (same format as Cursor)

**Usage:**
```bash
# One-time use
claude --mcp-config ~/.claude/mcp.json

# Or create an alias in ~/.zshrc
alias claude-ff="claude --mcp-config ~/.claude/mcp.json"
```

---

### 4.3 Claude Desktop

**Config file locations:**
| OS | Path |
|----|------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

**Same JSON format as Cursor.**

**After setup:** Quit and restart Claude Desktop (Cmd+Q)

---

## 5. Using the Tools

### 5.1 Natural Language (Recommended)

Just ask in plain English:
- "Show me my fantasy football leagues"
- "Get my roster"
- "What are the standings?"
- "Find top waiver wire RBs"
- "Build my optimal lineup for week 18"
- "Who has the highest projected points on my team?"

### 5.2 Available Tools

| Tool | Description | Example |
|------|-------------|---------|
| `ff_get_leagues` | List all your leagues | "Show my leagues" |
| `ff_get_league_info` | Detailed league info | "Get info for my league" |
| `ff_get_standings` | Current standings | "Show standings" |
| `ff_get_roster` | Get team roster with projections | "Get my roster" |
| `ff_get_teams` | All teams in a league | "List all teams" |
| `ff_get_matchup` | Weekly matchup details | "Show my matchup" |
| `ff_get_players` | Browse available players | "Find available QBs" |
| `ff_get_waiver_wire` | Top waiver pickups | "Best waiver wire picks" |
| `ff_build_lineup` | Optimize your lineup | "Build optimal lineup" |
| `ff_compare_teams` | Compare two teams | "Compare my team vs Team X" |
| `ff_get_draft_results` | Draft grades | "Show draft results" |
| `ff_get_draft_rankings` | Pre-draft rankings | "Get draft rankings" |
| `ff_analyze_reddit_sentiment` | Reddit buzz analysis | "What's Reddit saying about Josh Allen?" |
| `ff_refresh_token` | Refresh expired token | "Refresh my Yahoo token" |

### 5.3 Tool Parameters

**ff_get_roster** (enhanced):
```
league_key: "461.l.123456"
week: 18  (optional)
include_projections: true
data_level: "enhanced"  (basic/standard/enhanced)
```

**ff_get_waiver_wire**:
```
league_key: "461.l.123456"
position: "RB"  (QB/RB/WR/TE/K/DEF/all)
count: 30
sort: "rank"  (rank/points/owned/trending)
```

**ff_build_lineup**:
```
league_key: "461.l.123456"
week: 18
strategy: "balanced"  (conservative/balanced/aggressive)
```

---

## 6. Token Management

### 6.1 Token Expiration

- **Access Token**: Expires after ~1 hour (auto-refreshes)
- **Refresh Token**: Expires after ~60 days if unused

### 6.2 Refresh Tokens Manually

```bash
cd utils
python refresh_yahoo_token.py
```

This updates:
- `.env` file
- All MCP config files (Cursor, Claude Desktop, etc.)

### 6.3 Full Re-authentication

If refresh token expires (after ~60 days):

```bash
cd utils
python reauth_yahoo.py
```

**After refresh/reauth:** Restart your MCP client

---

## 7. Troubleshooting

### "Token expired" or "401 Unauthorized"

```bash
cd utils
python refresh_yahoo_token.py
```
Then restart your MCP client.

### "No leagues found"

1. Verify `YAHOO_GUID` is set correctly
2. Check you have active leagues for current season
3. Run `python utils/verify_setup.py`

### "Cannot find team"

1. Ensure `YAHOO_GUID` is in both `.env` AND MCP config
2. Verify you're a member of the leagues

### MCP tools not appearing

1. Check config file path is correct
2. Verify JSON syntax is valid
3. Restart your MCP client completely (Cmd+Q)
4. Check logs:
   - Cursor: Check Output panel
   - Claude Desktop: `~/Library/Logs/Claude/mcp*.log`

### Python/Import errors

```bash
source env/bin/activate
pip install -r requirements.txt
```

---

## Quick Reference

### Config File Locations

| Client | Config Path |
|--------|-------------|
| Cursor | `~/.cursor/mcp.json` |
| Claude Code CLI | `~/.claude/mcp.json` (+ use `--mcp-config` flag) |
| Claude Desktop (macOS) | `~/Library/Application Support/Claude/claude_desktop_config.json` |

### Essential Commands

```bash
# Verify setup
python utils/verify_setup.py

# Initial auth
python utils/setup_yahoo_auth.py

# Refresh tokens
python utils/refresh_yahoo_token.py

# Full re-auth
python utils/reauth_yahoo.py

# Run Claude Code CLI with MCP
claude --mcp-config ~/.claude/mcp.json
```

### Environment Variables Required

```env
YAHOO_CLIENT_ID=dj0yJmk9...
YAHOO_CLIENT_SECRET=abc123...
YAHOO_ACCESS_TOKEN=Rv9UTty...
YAHOO_REFRESH_TOKEN=ALxgWGn...
YAHOO_GUID=ABCDEFG123456
```

---

## Support

- Check [GitHub repository](https://github.com/derekrbreese/fantasy-football-mcp)
- Review `INSTALLATION.md` for detailed setup
- Ensure Yahoo tokens are current

---

*Last updated: January 2026*

