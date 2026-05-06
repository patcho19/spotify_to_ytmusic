# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`spotify2ytmusic` is a Python tool that copies Spotify "liked" songs, liked albums, and playlists into YouTube Music. It exposes both a CLI (entry points prefixed `s2yt_*`) and a Tkinter GUI. Migration is a two-step flow: first dump Spotify data to `playlists.json` (via Spotify's OAuth-via-localhost flow in `spotify_backup.py`), then read that JSON and replay it into YTMusic via `ytmusicapi`.

## Commands

```bash
# Install (editable, with deps)
python3 -m pip install -e .

# Run the CLI without installing entry points (dispatch via __main__)
python3 -m spotify2ytmusic <command> [args]   # e.g. list_playlists, load_liked, copy_all_playlists

# Run the GUI
python3 -m spotify2ytmusic gui                # or: s2yt_gui

# Tests (pytest is the CI choice; unittest also works)
pytest tests/                                 # all
pytest tests/test_basics.py::TestCopier::test_copier_success   # single test
python3 -m unittest tests.test_basics

# Lint (matches CI in .github/workflows/main.yml)
flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

# Formatter (enforced via pre-commit)
black .

# Generate YTMusic credentials (writes oauth.json from raw_headers.txt)
python3 spotify2ytmusic/ytmusic_credentials.py
# Alternative interactive OAuth flow:
s2yt_ytoauth                                  # wraps `ytmusicapi oauth`

# Backup Spotify data into playlists.json (interactive OAuth)
python3 spotify2ytmusic/spotify_backup.py playlists.json --dump=liked,playlists --format=json
```

CI (`.github/workflows/main.yml`) runs flake8 across Python 3.10–3.12. The pytest step is currently commented out, so tests are not gated by CI — run them locally.

## Required Files in CWD at Runtime

The backend reads/writes these files **from the current working directory**, not from package paths. Always run commands from the repo root (or a directory containing these files):

- `oauth.json` — YTMusic credentials. `backend.get_ytmusic()` hard-exits if missing.
- `playlists.json` — Spotify dump produced by `spotify_backup.py`. Loaded by `load_playlists_json()` and every iterator.
- `raw_headers.txt` — Only used by `ytmusic_credentials.py` to bootstrap `oauth.json`.

## Architecture

The package is small but the layering matters:

- **`backend.py`** — All business logic. Pure functions / generators that the CLI and GUI both call. Key pieces:
  - `iter_spotify_playlist(src_pl_id=None, …)` — yields `SongInfo(title, artist, album)` namedtuples; `src_pl_id=None` selects the "Liked Songs" playlist by name. Reverses track order by default so YTMusic ends up matching Spotify order.
  - `iter_spotify_liked_albums(...)` — yields tracks from Spotify liked **albums** (stored separately from liked songs in the dump).
  - `lookup_song(yt, track, artist, album, yt_search_algo, details=None)` — the matching heuristic. Always tries album-first lookup (search `"<album> by <artist>"`, then walks the first 3 albums for an exact track-name match). Falls back to song search; behavior of the fallback depends on `yt_search_algo`:
    - `0` — return the first song hit (default).
    - `1` — strict: require exact match on title, first artist, and album; raise `ValueError` otherwise.
    - `2` — fuzzy: strip `[...]`/`(...)` from titles, allow substring matches; if still no match, retry with `filter="videos"` to catch user-uploaded reposts.
    - Optional `ResearchDetails` dataclass collects the query/suggestions/song list for diagnostics (used by `s2yt_search`).
  - `copier(src_tracks, dst_pl_id=None, …)` — the workhorse. Iterates `SongInfo` from a generator, looks each up, then either `add_playlist_items` (when `dst_pl_id` given) or `rate_song(..., "LIKE")` (when `None`). Wraps the YTMusic call in a 10-attempt exponential backoff (5s → doubling) for rate-limit resilience. De-duplicates by `videoId` within a run. `dst_pl_id=None` means "like the songs" rather than "add to playlist".
  - `copy_playlist` / `copy_all_playlists` — orchestrate `iter_spotify_playlist` → `copier`. The "+name" convention for `ytmusic_playlist_id` (looks up by title; creates if missing) is implemented here, not in the CLI layer.
  - `_ytmusic_create_playlist` — also has its own retry-with-backoff loop and a 1s post-create sleep to dodge a race in the YTMusic API where the new ID isn't immediately visible.

- **`cli.py`** — Thin wrappers around backend functions. Each command defines its own nested `parse_arguments()` (no shared parser). Common flags: `--track-sleep`, `--dry-run`, `--algo`, `--spotify-playlists-encoding`, `--privacy`, `--no-reverse-playlist`. Entry points are wired in `pyproject.toml` under `[tool.poetry.scripts]` (e.g. `s2yt_load_liked` → `cli:load_liked`).

- **`__main__.py`** — Discovers public functions in `cli` via `inspect.getmembers` and dispatches `python -m spotify2ytmusic <name>` to them. This means **any new CLI command must be a top-level function in `cli.py`** to be reachable via `python -m`; helpers should go in `backend.py` or be nested inside the command.

- **`gui.py`** — Tkinter UI organized as a `ttk.Notebook` of tabs (Login, Backup, Liked, List, Copy All, Copy One, Search). It runs backend functions on background threads and pipes `stdout` into a log widget so the user sees the same output as the CLI. The GUI auto-detects an existing `oauth.json` and skips the login tab.

- **`spotify_backup.py`** — Vendored from <https://github.com/caseychu/spotify-backup> (MIT-licensed, separate from the project's CC0). Spins up a localhost HTTP server to capture the Spotify OAuth redirect, then paginates the Spotify Web API into `playlists.json`. Treat as third-party and avoid restructuring it.

- **`ytmusic_credentials.py`** — Reads `raw_headers.txt` (raw HTTP headers copied from a logged-in YouTube Music tab in Firefox dev-tools) and feeds them to `ytmusicapi.setup` to produce `oauth.json`. This is the headers-based path; `s2yt_ytoauth` is the alternative interactive OAuth path.

- **`reverse_playlist.py`** — Standalone utility that rewrites a `playlists.json` with track lists reversed. Mostly historical: track reversal is also handled inline by `iter_spotify_playlist(reverse_playlist=True)`.

## Conventions and Gotchas

- **Python 3.10+ required** — `lookup_song` uses structural `match`/`case`. Don't silently lower the floor.
- **Black** is the formatter (pinned in `.pre-commit-config.yaml`); run `pre-commit install` to get it on commit.
- **Tests in `tests/test_basics.py` are stale.** They patch `spotify2ytmusic.cli.YTMusic` and call `spotify2ytmusic.cli.copier` / `cli.iter_spotify_playlist`, but those names live in `backend.py` after a refactor and are not re-exported through `cli`. The tests will raise `AttributeError` as written. If you touch tests, fix the imports to target `spotify2ytmusic.backend` (and update the corresponding fixture `tests/playliststest.json` if the schema changes).
- **Errors print and `sys.exit(1)`** in many backend paths (missing `oauth.json`, malformed playlists, failed `create_playlist`). Don't convert these to exceptions without checking the GUI thread that consumes stdout.
- **Default `track_sleep=0.1`** is intentional throttling for the YTMusic API. The README documents bumping to `--track-sleep=3` when users hit HTTP 400s.
- **`copy_playlist` accepts `+<name>` as the destination** to mean "find by title or create" — this convention is in the backend, surfaced through both CLI and GUI.
- **`raw_headers.txt` is in `.gitignore`-territory by intent** (it contains session cookies). The empty file checked in is a placeholder; do not commit a populated one.
- **Two licenses coexist**: project is CC0; `spotify_backup.py` is MIT. Preserve its header.
