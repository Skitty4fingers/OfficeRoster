# OfficeRoster

Internal fantasy football tracking for a private company league.

OfficeRoster is a locally-hosted [Model Context Protocol](https://modelcontextprotocol.io) (MCP)
server that reads Yahoo Fantasy Sports data for a single private league and makes it available
to a desktop AI assistant for analysis and summarization. It is an internal-only tool. It is not
a published product, has no public endpoint, and is not distributed to end users.

---

## Purpose and use case

Our company runs a private Yahoo fantasy football league among coworkers. Reviewing rosters,
matchups, standings, and the waiver wire each week means clicking through many separate Yahoo
screens and manually comparing them.

OfficeRoster reads that league data through the Yahoo Fantasy Sports API and exposes it as a set
of structured, read-only tools to a locally-running AI assistant (Anthropic's Claude Desktop).
This allows questions like "who on my roster is on a bye this week," "how does my roster compare
to my week 4 opponent," or "which free agents at running back are worth a waiver claim" to be
answered against real league data, instead of by manual cross-referencing.

The value is convenience and summarization for one league. There is no commercial use, no
redistribution of Yahoo data, and no public-facing service.

## Who uses it

A single instance, run locally by one operator (the league member who set it up), on that
person's own workstation. Data accessed is that operator's own Yahoo fantasy account and the
leagues that account belongs to, using that operator's own OAuth authorization.

There is no multi-tenant mode, no user accounts, no sign-up flow, and no mechanism for a second
person to authenticate. Other members of the league are not users of this software.

## Yahoo API usage

**Read-only.** OfficeRoster issues only HTTP `GET` requests to the Yahoo Fantasy Sports API. It
does not add or drop players, submit waiver claims, propose trades, set lineups, or modify any
league, team, or roster state. The `fspt-r` (read) scope is sufficient and is the only scope
requested.

The endpoints used are:

| Endpoint | Purpose |
| --- | --- |
| `users;use_login=1/games;game_keys=nfl/leagues` | Discover which NFL leagues the authorized account belongs to |
| `league/{league_key}` | League metadata and scoring settings |
| `league/{league_key}/standings` | Current standings and records |
| `league/{league_key}/teams` | Team list for the league |
| `league/{league_key}/players;status=A` | Free agent / waiver wire pool |
| `team/{team_key}/roster` | Team roster |
| `team/{team_key}/matchups` | Weekly matchup and opponent |

Request volume is low and human-paced — a handful of calls per question asked, a few times a
week during the NFL season. The client enforces a self-imposed rate limit of 900 requests/hour
(below Yahoo's published limit) and caches responses in memory with short per-endpoint TTLs
(60s for live matchups, 300s for rosters and standings, 3600s for league metadata) to avoid
redundant calls.

## Architecture and deployment

```
Claude Desktop  ──stdio──▶  Docker container (localhost)  ──HTTPS──▶  Yahoo Fantasy API
   (local app)                    OfficeRoster MCP server
```

- Runs as a Docker container on a single local workstation.
- Communicates with Claude Desktop over stdio (standard input/output on the local machine).
- **No network listener, no public URL, no cloud hosting, no reverse proxy.** The container is
  not reachable from the internet or from the local network.
- Not deployed to any app store, marketplace, or MCP server directory.

## Credential and token handling

- Yahoo Client ID, Client Secret, and OAuth tokens are stored in a local `.env` file on the
  operator's workstation, outside version control. `.env` and all token artifacts are excluded
  by both `.gitignore` and `.dockerignore`, and are never baked into the container image.
- No credentials are committed to this repository.
- Tokens are passed to the container at runtime via `--env-file` and exist only for the life of
  the container process.
- Access tokens are refreshed against `api.login.yahoo.com` using the standard OAuth 2.0
  refresh-token grant. Raw tokens are never returned in any tool response.
- The Client Secret never leaves the local machine.

## Data handling

- Yahoo data is held in process memory for the life of a session and used only to answer the
  operator's own questions. It is not written to a database, not persisted to disk, and not
  retained after the container exits.
- No Yahoo data is transmitted to any third party, sold, published, or used for advertising,
  model training, or analytics.
- No personal information about other league members is collected beyond what Yahoo already
  returns for a league the authorized account belongs to (team names and public league standings).

## Third-party data sources

Supplementary, non-Yahoo context is read from public sources and kept clearly separate from
Yahoo data:

- **Sleeper API** (`api.sleeper.app`) — public, unauthenticated NFL player metadata and weekly
  projections, used to enrich Yahoo roster entries with projection context.

No Yahoo data is ever sent to these services; the flow is read-only and inbound.

## Status

Pre-deployment. Yahoo Fantasy Sports API access has been applied for and is pending review; the
application is non-functional against live data until that approval is granted.

## Attribution

Built on [fantasy-football-mcp-public](https://github.com/derekrbreese/fantasy-football-mcp-public)
(MIT), an open-source Yahoo Fantasy MCP server, adapted for internal single-league use.

## License

MIT
