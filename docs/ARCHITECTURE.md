# Fantasy Football MCP Server - Architecture Guide

This document explains how the MCP service is structured and how all components work together to provide the functionality demonstrated in `examples/example_client_llm_usage.py`.

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [Entry Points](#2-entry-points)
3. [Core Components](#3-core-components)
4. [Data Flow](#4-data-flow)
5. [Tool Registration & Handling](#5-tool-registration--handling)
6. [Enhancement Layer](#6-enhancement-layer)
7. [External Data Sources](#7-external-data-sources)
8. [Example Client LLM Usage](#8-example-client-llm-usage)

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MCP CLIENTS                                     │
│                    (Cursor, Claude Desktop, Claude CLI)                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ENTRY POINTS                                       │
│  ┌─────────────────────────┐    ┌──────────────────────────────────────┐   │
│  │  fastmcp_server.py      │    │  fantasy_football_multi_league.py    │   │
│  │  (FastMCP/HTTP)         │    │  (MCP stdio - local)                 │   │
│  │  - Cloud deployment     │    │  - Local deployment                  │   │
│  │  - @server.tool deco    │    │  - @server.call_tool                 │   │
│  └─────────────────────────┘    └──────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           HANDLERS LAYER                                     │
│                         src/handlers/                                        │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│  │ league_      │ │ roster_      │ │ matchup_     │ │ player_      │       │
│  │ handlers.py  │ │ handlers.py  │ │ handlers.py  │ │ handlers.py  │       │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                        │
│  │ draft_       │ │ admin_       │ │ analytics_   │                        │
│  │ handlers.py  │ │ handlers.py  │ │ handlers.py  │                        │
│  └──────────────┘ └──────────────┘ └──────────────┘                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SERVICES & UTILITIES                                 │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐  │
│  │ src/services/        │  │ src/api/             │  │ src/parsers/     │  │
│  │ - player_enhancement │  │ - yahoo_client.py    │  │ - yahoo_parsers  │  │
│  │ - reddit_service     │  │ - yahoo_utils.py     │  │                  │  │
│  └──────────────────────┘  └──────────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL DATA SOURCES                                 │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐  │
│  │ Yahoo Fantasy API    │  │ Sleeper API          │  │ Reddit API       │  │
│  │ (Primary data)       │  │ (Projections/IDs)    │  │ (Sentiment)      │  │
│  └──────────────────────┘  └──────────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Entry Points

### 2.1 `fastmcp_server.py` (Cloud/HTTP)

**Purpose:** FastMCP-compatible server for cloud deployment (fastmcp.cloud)

**Key Features:**
- Uses `@server.tool` decorators for tool registration
- Wraps legacy `fantasy_football_multi_league.call_tool` 
- Includes prompts (`@server.prompt`) and resources (`@server.resource`)
- HTTP transport support

```python
# Tool registration example
@server.tool(
    name="ff_get_roster",
    description="Get YOUR TEAM'S current roster...",
)
async def ff_get_roster(ctx: Context, league_key: str, ...) -> Dict[str, Any]:
    return await _call_legacy_tool("ff_get_roster", ctx=ctx, league_key=league_key, ...)
```

### 2.2 `fantasy_football_multi_league.py` (Local/stdio)

**Purpose:** Primary MCP server for local deployment via stdio

**Key Features:**
- Uses `mcp.server.Server` for tool registration
- Direct handler function calls via `TOOL_HANDLERS` dict
- Dependency injection for handlers
- League discovery and caching

```python
# Tool handler mapping
TOOL_HANDLERS = {
    "ff_get_leagues": handle_ff_get_leagues,
    "ff_get_roster": handle_ff_get_roster,
    # ... more handlers
}

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    handler = TOOL_HANDLERS.get(name)
    result = await handler(arguments)
    return [TextContent(type="text", text=json.dumps(result))]
```

---

## 3. Core Components

### 3.1 Directory Structure

```
fantasy-football-mcp-public/
├── fastmcp_server.py          # FastMCP entry point (cloud)
├── fantasy_football_multi_league.py  # MCP entry point (local)
│
├── src/
│   ├── api/                   # Yahoo API client
│   │   ├── yahoo_client.py    # API calls, token management
│   │   └── yahoo_utils.py     # Rate limiting, caching
│   │
│   ├── handlers/              # Tool handlers (business logic)
│   │   ├── __init__.py        # Exports + dependency injection
│   │   ├── league_handlers.py # ff_get_leagues, ff_get_standings
│   │   ├── roster_handlers.py # ff_get_roster
│   │   ├── matchup_handlers.py # ff_get_matchup, ff_build_lineup
│   │   ├── player_handlers.py  # ff_get_players, ff_get_waiver_wire
│   │   ├── draft_handlers.py   # ff_get_draft_*
│   │   ├── admin_handlers.py   # ff_refresh_token, ff_clear_cache
│   │   └── analytics_handlers.py # ff_analyze_reddit_sentiment
│   │
│   ├── services/              # Enhancement services
│   │   ├── player_enhancement.py  # Projections, bye weeks, trends
│   │   └── reddit_service.py      # Reddit sentiment analysis
│   │
│   ├── parsers/               # Data parsing
│   │   └── yahoo_parsers.py   # Parse Yahoo API responses
│   │
│   ├── models/                # Data models (Pydantic)
│   │   ├── player.py          # Player, Position, InjuryStatus
│   │   ├── lineup.py          # Lineup, RosterSlot
│   │   ├── matchup.py         # Matchup models
│   │   └── draft.py           # DraftState, DraftRecommendation
│   │
│   ├── strategies/            # Lineup optimization strategies
│   │   ├── base.py            # BaseStrategy interface
│   │   ├── conservative.py    # Risk-averse optimization
│   │   ├── balanced.py        # Default optimization
│   │   └── aggressive.py      # High-upside optimization
│   │
│   ├── agents/                # Advanced agents (optional)
│   │   ├── draft_evaluator.py # AI draft recommendations
│   │   ├── hybrid_optimizer.py # Advanced lineup optimization
│   │   └── reddit_analyzer.py  # Reddit sentiment agent
│   │
│   └── utils/                 # Utilities
│       ├── bye_weeks.py       # Bye week lookup
│       ├── constants.py       # NFL team mappings
│       └── roster_configs.py  # League roster configurations
│
├── sleeper_api.py             # Sleeper API integration
└── lineup_optimizer.py        # Core lineup optimization logic
```

---

## 4. Data Flow

### 4.1 Simple Tool Call (ff_get_leagues)

```
Client Request
    │
    ▼
fastmcp_server.py::ff_get_leagues()
    │
    ▼
_call_legacy_tool("ff_get_leagues")
    │
    ▼
fantasy_football_multi_league.call_tool()
    │
    ▼
TOOL_HANDLERS["ff_get_leagues"] → handle_ff_get_leagues()
    │
    ▼
src/handlers/league_handlers.py::handle_ff_get_leagues()
    │
    ▼
discover_leagues()
    │
    ▼
yahoo_api_call("users;use_login=1/games;game_keys=nfl/leagues")
    │
    ▼
Yahoo Fantasy API
    │
    ▼
Parse Response → Cache → Return JSON
```

### 4.2 Enhanced Tool Call (ff_get_roster with projections)

```
Client Request (league_key, include_projections=True, data_level="enhanced")
    │
    ▼
fastmcp_server.py::ff_get_roster()
    │
    ▼
src/handlers/roster_handlers.py::handle_ff_get_roster()
    │
    ├──────────────────────────────────────────────────────────┐
    │                                                          │
    ▼                                                          ▼
yahoo_api_call(f"team/{team_key}/roster")         src/services/player_enhancement.py
    │                                                          │
    ▼                                                          ▼
Parse basic roster data                           Enhance with:
    │                                             - Sleeper projections (sleeper_api.py)
    │                                             - Bye week detection
    │                                             - Matchup analysis
    │                                             - Expert tiers/recommendations
    │                                             - Trending data
    │                                                          │
    └──────────────────────────────────────────────────────────┘
                            │
                            ▼
                    Merge enhanced data
                            │
                            ▼
                    Return comprehensive JSON
```

---

## 5. Tool Registration & Handling

### 5.1 Dependency Injection Pattern

The codebase uses dependency injection to keep handlers testable and decoupled:

```python
# In fantasy_football_multi_league.py

# Inject dependencies for roster handlers
inject_roster_dependencies(
    get_user_team_info=get_user_team_info,
    yahoo_api_call=yahoo_api_call,
    parse_team_roster=parse_team_roster,
)

# In src/handlers/__init__.py
def inject_roster_dependencies(**deps):
    import src.handlers.roster_handlers as roster_mod
    for name, func in deps.items():
        setattr(roster_mod, name, func)
```

### 5.2 Handler Structure

Each handler follows this pattern:

```python
# src/handlers/roster_handlers.py

# Placeholders for injected dependencies
get_user_team_info = None
yahoo_api_call = None
parse_team_roster = None

async def handle_ff_get_roster(arguments: dict) -> dict:
    """Get roster with configurable enhancement levels."""
    
    league_key = arguments.get("league_key")
    data_level = arguments.get("data_level", "standard")
    
    # 1. Get basic data from Yahoo
    team_info = await get_user_team_info(league_key)
    data = await yahoo_api_call(f"team/{team_key}/roster")
    roster = parse_team_roster(data)
    
    # 2. Enhance if requested
    if data_level in ["standard", "enhanced", "full"]:
        roster = await enhance_roster_data(roster, ...)
    
    # 3. Return structured result
    return {
        "status": "success",
        "roster": roster,
        "team_name": team_info.get("team_name"),
        # ... more fields
    }
```

---

## 6. Enhancement Layer

### 6.1 Player Enhancement Service

`src/services/player_enhancement.py` adds rich context to player data:

```python
@dataclass
class EnhancedPlayerData:
    on_bye: bool                         # Is player on bye this week?
    recent_performance: RecentPerformance # Last 1-3 weeks stats
    performance_flags: List[str]         # ["trending_up", "injury_concern"]
    adjusted_projection: float           # Modified based on context
    recommendation_override: str         # "must_start", "bench", etc.
    context_message: str                 # Human-readable insight
```

### 6.2 Sleeper API Integration

`sleeper_api.py` provides additional data:

```python
# Get player projections from Sleeper
async def get_sleeper_projections(week: int) -> Dict[str, Dict]:
    """Returns projections keyed by player name."""
    
# Get trending players
async def get_trending_players(sport: str = "nfl") -> List[Dict]:
    """Returns players being added/dropped the most."""
    
# Match Yahoo player to Sleeper ID
async def match_player_to_sleeper(player_name: str, team: str) -> Optional[str]:
    """Cross-reference Yahoo player with Sleeper database."""
```

---

## 7. External Data Sources

### 7.1 Yahoo Fantasy API

**Primary data source** for:
- League information
- Rosters
- Matchups
- Standings
- Player availability
- Draft results

**Rate Limiting:** `src/api/yahoo_utils.py`
```python
class RateLimiter:
    # 60 requests per minute default
    async def acquire(self):
        """Wait if rate limit would be exceeded."""
```

**Caching:** Responses cached to reduce API calls

### 7.2 Sleeper API

**Secondary data source** for:
- Player projections (more accurate than Yahoo)
- Player IDs for cross-referencing
- Trending add/drop data
- Injury updates

### 7.3 Reddit API (Optional)

**Sentiment analysis** via `src/services/reddit_service.py`:
- Scans fantasy football subreddits
- Analyzes player mentions
- Returns sentiment scores

---

## 8. Example Client LLM Usage

### 8.1 How `example_client_llm_usage.py` Works

The example demonstrates the **intended usage pattern** where:

1. **Client LLM** (the AI assistant) calls MCP tools
2. **MCP Server** returns rich, structured data
3. **Client LLM** analyzes data and makes decisions
4. **No backend LLM** needed - the client does the thinking

```python
class FantasyFootballClientLLM:
    async def optimize_lineup_for_week(self, league_key, team_key, week):
        # Step 1: Get data from MCP server
        analysis = await self.mcp_server.call_tool(
            "ff_get_comprehensive_analysis",  # or ff_get_roster with full data
            {"league_key": league_key, "week": week, ...}
        )
        
        # Step 2: Client LLM analyzes the rich data
        client_analysis = self._analyze_comprehensive_data(
            roster_data, lineup_analyses, decision_context, week
        )
        
        # Step 3: Client LLM makes decisions
        decisions = self._make_intelligent_decisions(client_analysis, week)
        
        # Step 4: Generate recommendations
        recommendations = self._generate_recommendations(decisions, client_analysis)
        
        return {"decisions": decisions, "recommendations": recommendations}
```

### 8.2 Data Levels

The MCP tools support different detail levels:

| Level | Data Included | Use Case |
|-------|--------------|----------|
| `basic` | Names, positions, teams | Quick lookups |
| `standard` | + Projections | Weekly lineup |
| `enhanced`/`full` | + Sleeper data, trends, matchup analysis, expert tiers | Comprehensive decisions |

### 8.3 Real Usage (via Cursor/Claude)

When you ask me "Get my roster", I:

1. Call `ff_get_roster` with appropriate parameters
2. Receive enhanced data including projections, tiers, recommendations
3. Analyze and present insights to you
4. Make suggestions based on the data

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FANTASY FOOTBALL MCP                                 │
│                                                                              │
│  Entry Points:                                                              │
│  • fastmcp_server.py (HTTP/Cloud)                                           │
│  • fantasy_football_multi_league.py (stdio/Local)                           │
│                                                                              │
│  Core Flow:                                                                  │
│  Client → Entry Point → Handler → Yahoo API → Parse → Enhance → Response   │
│                                                                              │
│  Key Features:                                                              │
│  • 16+ MCP tools for fantasy football                                       │
│  • Multi-source data (Yahoo + Sleeper + Reddit)                             │
│  • Configurable enhancement levels                                          │
│  • Lineup optimization with 3 strategies                                    │
│  • Dependency injection for testability                                     │
│  • Rate limiting and caching                                                │
│                                                                              │
│  Designed for:                                                              │
│  • Client LLMs to receive rich data                                         │
│  • Client LLMs to make intelligent decisions                                │
│  • No backend LLM required                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*This architecture enables powerful AI-assisted fantasy football management by providing comprehensive, well-structured data to client LLMs.*

