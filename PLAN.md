# OfficeRoster — Build Plan

Target: running for the 2026/2027 NFL season.

## Scope

OfficeRoster is built on [fantasy-football-mcp-public](https://github.com/derekrbreese/fantasy-football-mcp-public).
That project works, but it carries defects that matter for a 2026 season and it has no dashboard
at all. This plan covers what must be true before the league relies on it.

Two surfaces, deliberately separated:

| Surface | Transport | Audience | Yahoo credentials |
| --- | --- | --- | --- |
| MCP server | stdio → Claude Desktop | Operator only | Yes |
| Refresh job | outbound HTTPS | — (scheduled) | Yes |
| Dashboard | HTTP, LAN-only | League members | **No** |

The dashboard container must never receive `--env-file`. If it cannot see the credentials, no
dashboard bug can leak them. This is the property that makes unauthenticated viewer access
acceptable, and it is what the README tells Yahoo we do.

File references below point at paths in the upstream tree as vendored into this repo.

---

## Phase 0 — Yahoo API access (external, blocking)

Creating a Yahoo developer app is no longer sufficient. Every call returns
`401 additional_authorization_required` until a human at Yahoo approves the access application.
No published turnaround time. Nothing else works until this lands.

- [ ] Create a Yahoo developer app at <https://developer.yahoo.com/apps/> — Web Application,
      redirect URI `oob`
- [ ] Apply at <https://sports.yahoo.com/developer/access/>, attaching the Client ID from above
- [ ] Decide repo visibility — this repo is currently **private**, so a reviewer following the
      link gets a 404
- [ ] Verify nothing sensitive is in this repo's history before making it public

**Do not fork upstream's git history into this repo.** It contains live Yahoo OAuth tokens and a
client secret in `.env` and `.yahoo_token.json` blobs (commits `1521313`, `951f632`, `b150560`).
Vendor the working tree as a clean initial commit instead. Those are the upstream author's
credentials, not ours — nothing for us to rotate — but they must not end up in a repo we hand to
Yahoo.

Also: upstream's `INSTALLATION.md` instructs committing `.env` "for backup purposes." That is how
those secrets leaked. Ignore it. `.gitignore` and `.dockerignore` are already correct.

---

## Phase 1 — Season correctness

Do these while Yahoo approval is pending; none require working credentials. These rank above the
analysis features because wrong data on a dashboard coworkers read is more visible than wrong data
in a private chat window.

- [ ] **Bye weeks are hardcoded to 2025 and override the live API.**
      `src/utils/bye_weeks.py:83` treats `src/data/bye_weeks_2025.json` as authoritative and
      discards Yahoo's value on conflict, with no season check. Every bye week displayed this
      season will be wrong.
      *Fix:* invert precedence — trust the Yahoo API value, fall back to static data only when the
      API returns nothing. Do not simply swap in 2026 data; the override design is the bug.
      *~30 min.*

- [ ] **Fabricated defensive rankings feed matchup scores.**
      `sleeper_api.py:553` returns a hand-typed mock table labelled 2025, carrying its own
      `TODO: Replace with real-time defensive stats`. It reaches output via
      `get_player_advice` → `MatchupAnalyzer.get_matchup_score` → the `confidence` value surfaced
      as `matchup_score` in roster and lineup responses.
      *Fix:* return empty so the code falls through to its neutral-50 default. Neutral and honest
      beats confidently wrong. *~15 min.*

- [ ] **Sleeper player cache never expires.**
      `sleeper_api.py:80` uses `timedelta.seconds` (sub-day remainder, 0–86399) instead of
      `.total_seconds()`. At 25 hours it reads 3600, passes the `< 86400` check, and the player
      database is never refreshed again — injuries, team changes and callups go stale for the life
      of the process. Critical once the refresh job is long-running. *One line.*

- [ ] **Repair the test suite** so there is a regression net before touching the optimizer.
      `tests/test_live_api.py:109` uses `Tuple` without importing it, which fails collection — so
      the documented `pytest` command errors out on a clean checkout. `tests/conftest.py:17` sets
      `YAHOO_CONSUMER_KEY`/`SECRET` but `credentials_from_env()` reads `YAHOO_CLIENT_ID`/`SECRET`,
      failing 5 tests. Baseline is 123 passing of 128. *~15 min for all 128.*

- [ ] **Verify league discovery returns 2026 leagues** (needs credentials — defer to Phase 4).
      `fantasy_football_multi_league.py:85` calls `game_keys=nfl`, which should resolve to the
      current season despite a stale comment saying "461 for 2025". Two risks: it reads only
      `games["0"]`, so multiple returned seasons would silently pick the wrong one; and the whole
      parse is wrapped in `except Exception: pass`, so a schema change yields zero leagues with no
      error. If `ff_get_leagues` comes back empty, start at that bare `except`.

---

## Phase 2 — Snapshot refresh job

Everything downstream depends on this. The current cache is in-memory (`ResponseCache`) and dies
with the process; there is no persistence layer.

- [ ] Scheduled process that calls the existing handlers and serializes standings, rosters,
      matchups and the free-agent pool to JSON on a Docker volume
- [ ] **Atomic writes** — temp file plus rename, so the dashboard never reads a half-written
      snapshot
- [ ] Each refresh overwrites the previous one; no historical archive (as stated in the README)
- [ ] Refresh roughly hourly during the season. Well inside the client's self-imposed 900 req/hr
      limit, and keeps Yahoo call volume flat regardless of viewer count
- [ ] Record a refresh timestamp in the snapshot so the dashboard can show data age
- [ ] Handle refresh failure without destroying the last good snapshot — stale data beats no data
- [ ] Testable against stubbed handlers without Yahoo credentials

*~1 day.*

---

## Phase 3 — Viewer dashboard

Net-new. Upstream has no frontend — no HTML, templates, static assets or web framework beyond
FastMCP's own transport.

- [ ] Read-only web app over the snapshot: standings, rosters, current-week matchups
- [ ] **Separate container, no `--env-file`, no import path to `src/api/`**
- [ ] Publish on `-p 8080:8080` (LAN reachability is intended here, unlike the operator path)
- [ ] Show snapshot age prominently so viewers know whether they are looking at stale data
- [ ] No write actions, no forms, no Yahoo passthrough endpoints

*~1–2 days depending on polish.*

**Before deploying:** an unauthenticated listener on a corporate network is more likely to be a
policy problem than a data problem. The data is fantasy rosters; the exposure is an unsanctioned
service that security scanning will flag. Worth clearing with IT, particularly since the Yahoo
application states in writing that this runs on a company network.

---

## Phase 4 — Operator path (needs credentials)

- [ ] **Unguarded `print()` calls corrupt the stdio JSON-RPC stream.** Specific to the stdio
      transport, which is why upstream (HTTP-first) never hit it. Five offenders on live paths:
      - `sleeper_api.py:67`, `:70` — Sleeper HTTP errors
      - `sleeper_api.py:351` — "No real projections available", very likely to fire in preseason
        and off-weeks when Sleeper has no weekly projections
      - `src/api/yahoo_utils.py:37` — rate-limit wait notice
      - `src/parsers/yahoo_parsers.py:31` — a `DEBUG:` print firing during normal roster parsing

      *Fix:* route to `logging` (stderr). The noisy per-player matcher prints are already gated
      behind `SLEEPER_DEBUG`; leave those. *~30 min.*

- [ ] **Persist refreshed tokens.** `update_current_credentials`
      (`src/api/yahoo_credentials.py:95`) writes to `os.environ` only; nothing writes back to
      `.env`. With `--rm` containers every restart re-refreshes from the `.env` refresh token —
      fine *unless Yahoo ever rotates the refresh token*, at which point the replacement exists
      only in a dead container's memory and re-authentication is required mid-season.
      *Fix:* mount a token file and persist on refresh. *~1–2 hrs.*

- [ ] **Keep tokens in exactly one place.** `load_dotenv` does not override existing environment
      variables, so tokens in a Claude Desktop `env` block silently win over `.env`. Use
      `--env-file` and nothing else, or expect to chase a stale-token ghost.

- [ ] Claude Desktop config (`-i` required for stdin, no `-t`):

      ```json
      {
        "mcpServers": {
          "officeroster": {
            "command": "docker",
            "args": ["run", "--rm", "-i",
                     "--env-file", "C:\\path\\to\\.env",
                     "officeroster",
                     "python", "fantasy_football_multi_league.py"]
          }
        }
      }
      ```

- [ ] Note: ephemeral containers re-download Sleeper's full player database (~10MB) on every
      Claude Desktop restart — `get_all_players` (`sleeper_api.py:73`) caches in memory only.
      Mount a cache volume if the cold start becomes annoying.

---

## Phase 5 — Lineup optimizer (decision required)

`ff_build_lineup` does not work. `lineup_optimizer.py:687` groups starters by each player's
*current Yahoo roster slot* and keeps one player per slot label. Demonstrated against a synthetic
roster:

```
STARTERS: QB(20), RB One(15), WR One(13), W/R/T(11)
BENCH:    RB Two(14), WR Two(12), Stud On Bench(99)
```

Three defects: a second RB or WR is silently dropped, so the returned "optimal lineup" is
structurally invalid; anything in a `BN` slot can never be promoted, so it cannot produce a
start/sit recommendation at all; and `strategy` is accepted, echoed back, and never read.
Meanwhile `src/handlers/matchup_handlers.py:181` labels the result `optimal_lineup` and reports
`data_sources: ["Yahoo projections", "Sleeper rankings", "Matchup analysis", "Trending data"]`
unconditionally.

Pick one:

- **Fix properly.** Read the league's real slot structure from `roster_positions` in league
  settings (2×RB, 3×WR, 1×FLEX…), then fill greedily by projection with FLEX eligibility.
  *~half a day plus tests.*
- **Remove the tool.** Let Claude reason over `ff_get_roster` output directly. Defensible — the
  data is the valuable part.

Leaning toward fixing it: slot-constraint satisfaction is the one piece an LLM cannot reliably do
by eye. But do not ship it as-is. A tool confidently labelling an invalid lineup `optimal_lineup`
is worse than no tool.

Operator-facing only — the dashboard can ship while this is unresolved.

---

## Phase 6 — Hygiene

- [ ] **Delete the dead `src/` tree.** Static import analysis from the live entry point:
      24 files / 9.4k LOC reachable, 43 files / 21k LOC not. Excluding ~1.4k LOC of legitimate
      standalone `utils/` scripts, roughly 19.6k LOC is dead:
      - `src/agents/` except `reddit_analyzer` (~9k), `src/strategies/` (~2.3k),
        `src/models/` (~1.7k), `src/utils/{constants,roster_configs,scoring}`
      - Five modules duplicated between root and `src/` (`lineup_optimizer`, `matchup_analyzer`,
        `position_normalizer`, `sleeper_api`, `yahoo_api_utils`) — the `src/` copies are dead and
        have drifted from the live root copies
      - `src/mcp_server.py` (819 LOC) is dead *and* broken: imports symbols that do not exist in
        `mcp` 1.14 and modules that do not resolve. It is nonetheless the declared console script
        in `pyproject.toml:64`

      Not urgent, but at 11pm in week 9 you do not want to be editing `src/strategies/balanced.py`
      wondering why nothing changes.

- [ ] Add a `LICENSE` file — the README claims MIT with nothing backing it
- [ ] Fix `pyproject.toml`: placeholder author/URLs, dependency set contradicting
      `requirements.txt`, entry point pointing at the broken module
- [ ] Correct the stale Python floor — `requires-python = ">=3.9"` and INSTALLATION's "3.8 or
      higher" are both wrong; pinned `numpy==2.3.3` requires ≥3.11. The 3.11 Docker base image is
      fine, so this only bites outside Docker

---

## Out of scope

Deliberately not doing:

- Request-scoped credential plumbing and per-user cache namespacing (`src/api/yahoo_credentials.py`)
  — well-built, exercised only by tests, and irrelevant to a single-operator deployment
- HTTP auth / OAuth for the MCP server — the operator path is stdio
- ChatGPT app store readiness
- Reddit sentiment analysis — optional, needs separate API credentials, low value until the core works
- The `pyproject.toml` console script — running via Docker, not `pip install`

---

## Verification before the league relies on it

- [ ] `pytest` — all 128 green
- [ ] `ff_get_leagues` returns the 2026 league
- [ ] Bye weeks match Yahoo's own display for a known player
- [ ] Matchup scores read as neutral rather than invented
- [ ] Roster call through Claude Desktop returns clean JSON with no stdout contamination
- [ ] Dashboard renders from a snapshot with the container started **without** `--env-file`
- [ ] `docker exec` into the dashboard container confirms no Yahoo token in its environment
- [ ] Snapshot survives a failed refresh — last good data still served
- [ ] Restart Claude Desktop, confirm token refresh still works from a cold container

---

## Sequencing

| When | Work |
| --- | --- |
| Today | Phase 0 — apply to Yahoo; decide repo visibility |
| While waiting | Phase 1 (~half day), then Phase 2 refresher (~1 day) |
| While waiting | Phase 3 dashboard (~1–2 days) |
| On approval | Phase 4 — verify discovery, fix prints, token persistence |
| After | Phase 5 optimizer, Phase 6 cleanup |

Roughly 3–4 days of work, plus the unknown Yahoo approval wait. The approval is the long pole;
everything in Phases 1–3 can be built and tested against stubs before it lands.
