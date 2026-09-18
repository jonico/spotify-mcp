# Testing V12

Working workspace for the [spotify-mcp](https://github.com/jonico/spotify-mcp) server — a
FastMCP-based MCP server for the Spotify Web API. Six of the seven collections here are
git-synced from that repo; edit them in the repo, not in the UI.

## Collections

### Spotify — 46 MCP requests + 4 AI workflow requests

Exercises the MCP server itself. Most requests are `mcp-request` items speaking JSON-RPC to
`uvx spotify-mcp-jamiew` over stdio, and together they cover the server's whole surface:
**28/28 tools, 6/6 resources, 5/5 prompts**. Each request records its purpose and the
response it actually returns.

Four `llm-request` items - **Queue Similar Tracks**, **Build Discovery Playlist**, **Resume
Liked Songs** and **Clean Up Duplicate Tracks AI Request** - are genuine AI requests: a
model, this server configured as an MCP tool via `mcpConfig`, and a natural-language goal
that needs several tools chained together. They run against OpenAI (`gpt-5`) and need
`OPENAI_API_KEY` in addition to the usual Spotify credentials.

Requests whose description says "Not executed" would mutate the live Spotify account —
create, rename, reorder, unfollow, unlike, queue, device transfer. They are documented on
purpose and deliberately never run.

> **The `mcp-request` items cannot run in the CLI or a monitor.** `postman collection run`
> skips them as unsupported ("Skipping N unsupported item(s)") and executes nothing. Run
> them from the Postman app, or drive the equivalent tools through a connected MCP client.

### Spotify REST Workflow - * — 4 collections, 21 HTTP requests

One collection per AI workflow request above, each pinning down the literal Spotify Web API
calls that workflow implies - no LLM, no MCP server, no mocks. Every request maps to a
specific `spotify_api.py`/`spotipy` call and chains into the next via collection variables
(access token → read → search/list → write), the same order an agent following the matching
prompt would need. Regime notes (restricted vs. legacy path) are called out per request
where `with_fallback` picks between them.

Needs the same `SPOTIFY_CLIENT_ID` / `SPOTIFY_CLIENT_SECRET` as the Spotify MCP environment,
plus `SPOTIFY_REFRESH_TOKEN` to mint a user access token without repeating the OAuth consent
flow. The playlist- and playback-mutating requests are real writes against the signed-in
account when run.

### Spotify API Conformance — 8 HTTP requests, 17 assertions

Live checks straight against `api.spotify.com`, with no MCP server and no mocks in the path.
Built to run unattended and fail when Spotify changes what this app is allowed to do.

Four checks assert that something is **currently broken** — the search page cap above 10,
artist top-tracks, batch track reads, and the stripped `followers`/`popularity` fields. That
is deliberate: a monitor is only useful if it fires on *change*, in either direction, and
each failure message says what to do in the code when it flips.

This is not a port of the repo's Python test suite. Those 169 tests are offline and mocked —
they assert the server's own transformation logic and cannot notice an upstream change.

### Internal API Catalog — 15 HTTP requests

Unrelated to the Spotify work and **not** git-synced. A scaffold of CRUD requests across
User, Order and Inventory services against `https://api.internal.local`, which does not
resolve. Sample/demo content.

## Environments

| Environment | Git-synced | Credentials resolve from |
|---|---|---|
| **Spotify MCP** | yes | Postman Vault: `spotify-mcp-client-id`, `spotify-mcp-client-secret`, `spotify-mcp-refresh-token`, `spotify-mcp-openai-key` |
| **Spotify API Conformance** | **no — cloud only** | values stored in the cloud environment |

The asymmetry is deliberate and load-bearing. Vault values are local and never sync, so they
cannot resolve for anything running in Postman's cloud. Conversely a git-tracked environment
file cannot hold a secret, and its empty values **overwrite the cloud values with blanks on
every `postman workspace push`** — verified, it silently breaks the monitor. So the
monitor's environment is kept out of git sync entirely: there is no file for it under
`postman/environments/` and no entry in `.postman/resources.yaml`. Edit that one in the UI or
via the API, never from the repo.

Pick **Spotify MCP** for the AI/MCP requests and the four `Spotify REST Workflow - *`
collections, **Spotify API Conformance** for the conformance collection and its monitor.

## Monitor

**Spotify API conformance (daily)** — 07:00 Europe/Berlin, `eu-central`, running the
conformance collection against its cloud environment. Read-only: nothing is played, saved or
modified, so it is safe to run repeatedly.

Each run costs 8 requests against the monitor allowance, which since July 2026 is counted per
**developer account** rather than per app. Check headroom before raising the frequency.

## Working on this from the repo

```bash
# run the conformance collection locally
postman collection run "postman/collections/Spotify API Conformance" \
  --env-var "SPOTIFY_CLIENT_ID=$SPOTIFY_CLIENT_ID" \
  --env-var "SPOTIFY_CLIENT_SECRET=$SPOTIFY_CLIENT_SECRET"

# validate, then sync everything back up
postman environment lint "postman/environments/Spotify MCP.environment.yaml"
postman workspace push -y
```

Two things worth knowing before you trust a sync:

- `Environment X: Updated successfully` means "no error", **not** "the cloud matches your
  file". When there is no local diff the planner sends nothing and still reports success.
  Read back with the API when it matters.
- A pull may rewrite every collection file with byte-identical content, and on another run
  replace an environment file with an older cloud copy. Check `git diff`, not just whether
  files were touched.

## Server behaviour this workspace documents

Found while executing the requests, and recorded in the collection descriptions.

Known limits:

- `list_playlists` returns `total_tracks: null` for every entry, so playlist size is not
  available from the list — page each playlist and count the rows returned.
- `get_playlist_tracks` reports `total` as the returned page size, not the playlist length.
- `search_music` can return `total: 0` alongside populated `items`. Read `items`; never
  branch on `total`.
- `search_music` rejects `limit` above 10 on the wire; the tool retries at the cap rather
  than failing.
- Artist and user objects come back with `genres: []` and `popularity`/`followers` null.
- `check_following_artists` fails with `playback_restricted` for any input. The route is
  withheld from this app; `check_saved_tracks` and `check_saved_albums` work.

Five defects this workspace originally documented — `get_artist` dead for every artist,
`get_tracks` unable to batch, the `search_music` limit rejection surfacing as an error,
`get_playlist` reporting no count, and `control_playback` returning an empty state after
waking an idle device — were fixed upstream and released.
