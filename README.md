# OfficeRoster

Internal fantasy football tracking for a private company league.

OfficeRoster reads Yahoo Fantasy Sports data for a single private league and presents it in two
internal-only ways: as structured tools for a desktop AI assistant used by the league operator,
and as a read-only dashboard that other members of the same league can view on the local network.

It is an internal tool. It is not a published product, is not reachable from the internet, and is
not distributed to end users.

---

## Purpose and use case

Our company runs a private Yahoo fantasy football league among coworkers. Reviewing rosters,
matchups, standings, and the waiver wire each week means clicking through many separate Yahoo
screens and manually comparing them.

OfficeRoster reads that league data through the Yahoo Fantasy Sports API and does two things
with it:

1. **Analysis (operator only).** Exposes the data as read-only tools to a locally-running AI
   assistant (Anthropic's Claude Desktop), so questions like "who on my roster is on a bye this
   week" or "how does my roster compare to my week 4 opponent" can be answered against real
   league data instead of by manual cross-referencing.
2. **Shared visibility (league members).** Renders a periodically-refreshed snapshot of standings,
   rosters, and weekly matchups as a simple dashboard, so other members of the same league can
   glance at the league state without each person clicking through Yahoo separately.

The value is convenience and summarization for one private league. There is no commercial use, no
redistribution of Yahoo data outside that league, and no public-facing service.

## Who uses it

- **One operator.** A single league member installs and runs OfficeRoster on their own machine and
  authorizes it against their own Yahoo account. This is the only Yahoo authorization involved.
- **Other members of the same league**, as passive viewers of the dashboard on the internal
  network. They do not authenticate to Yahoo, do not have Yahoo accounts linked to this software,
  and cannot trigger Yahoo API calls. They see only a cached snapshot.

There is no multi-tenant mode, no user accounts, no sign-up flow, and no mechanism for a second
person to connect their own Yahoo account.

Data shown on the dashboard is limited to the league those viewers are already members of — that
is, information Yahoo already displays to them inside the league. Nothing is shown to anyone
outside the league.

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

**Call volume is independent of how many people view the dashboard.** A single scheduled refresh
job makes the Yahoo calls on a fixed interval — on the order of once per hour during the NFL
season — and writes the result to a local snapshot. Dashboard page loads read that snapshot and
never reach Yahoo. Ad-hoc calls from the operator's AI assistant are human-paced, a handful per
question asked.

The client enforces a self-imposed rate limit of 900 requests/hour (below Yahoo's published limit)
and caches responses with short per-endpoint TTLs (60s for live matchups, 300s for rosters and
standings, 3600s for league metadata) to avoid redundant calls.

## Architecture and deployment

```
                         Yahoo Fantasy API
                                 ▲
                                 │ HTTPS, read-only
                                 │ (holds the OAuth credentials)
                        ┌────────┴────────┐
                        │  Refresh job    │   scheduled, ~hourly
                        └────────┬────────┘
                                 │ writes
                                 ▼
                        ┌─────────────────┐
                        │ Local snapshot  │   JSON on a local volume
                        └────────┬────────┘
                     reads       │       reads
              ┌──────────────────┴──────────────────┐
              ▼                                     ▼
     ┌──────────────────┐                  ┌──────────────────┐
     │   MCP server     │                  │    Dashboard     │
     │  stdio, operator │                  │  HTTP, LAN-only  │
     └────────┬─────────┘                  └────────┬─────────┘
              │                                     │
       Claude Desktop                        League members
       (same machine)                        (read-only view)
```

- Runs as Docker containers on a single workstation inside the company network.
- The MCP server communicates with Claude Desktop over stdio on the same machine.
- The dashboard listens on the local network only. **It is not exposed to the internet**, has no
  public URL or DNS name, is not behind a public reverse proxy, and is not deployed to any cloud
  host, app store, marketplace, or MCP server directory.
- **The dashboard has no access to Yahoo credentials and no code path to the Yahoo API.** It reads
  the local snapshot and nothing else. Yahoo tokens exist only in the refresh job's process.

## Credential and token handling

- Yahoo Client ID, Client Secret, and OAuth tokens are stored in a local `.env` file on the
  operator's workstation, outside version control. `.env` and all token artifacts are excluded by
  both `.gitignore` and `.dockerignore`, and are never baked into a container image.
- No credentials are committed to this repository.
- Tokens are passed to the refresh job at runtime via `--env-file`. They are never passed to the
  dashboard container.
- Access tokens are refreshed against `api.login.yahoo.com` using the standard OAuth 2.0
  refresh-token grant. Raw tokens are never returned in any tool response or rendered in any view.
- The Client Secret never leaves the operator's machine.

## Data handling

- The snapshot is stored on a local Docker volume on the operator's workstation. It contains only
  current league state — standings, rosters, matchups, and the free agent pool — and each refresh
  overwrites the previous one. No historical archive is accumulated.
- No Yahoo data is transmitted to any third party, sold, published, or used for advertising,
  model training, or analytics.
- No personal information about league members is collected beyond what Yahoo returns for a league
  the authorized account belongs to (team names, rosters, and standings within that league).
- Dashboard access is controlled by network reachability: the service is only routable from inside
  the company network, and its contents are limited to the private league the viewers belong to.

## Third-party data sources

Supplementary, non-Yahoo context is read from public sources and kept clearly separate from Yahoo
data:

- **Sleeper API** (`api.sleeper.app`) — public, unauthenticated NFL player metadata and weekly
  projections, used to enrich roster entries with projection context.

No Yahoo data is ever sent to these services; the flow is read-only and inbound.

## Status

Pre-deployment.

- Yahoo Fantasy Sports API access has been applied for and is pending review. The application is
  non-functional against live data until that approval is granted.
- The MCP server component is adapted from the upstream project below.
- The scheduled refresh job and the viewer dashboard are not yet implemented.

## Attribution

Built on [fantasy-football-mcp-public](https://github.com/derekrbreese/fantasy-football-mcp-public)
(MIT), an open-source Yahoo Fantasy MCP server, adapted for internal single-league use.

## License

MIT
