
# spotify-to-jellyfin

Download Spotify playlists (audio + metadata) into a Jellyfin-ready music library, with proper artist/album tagging and auto-generated `.m3u` playlists that show up natively in Jellyfin. Includes bash shortcuts to download once, download-and-track-for-syncing, and update all synced playlists later.

> **Note on how this works:** this does not rip audio from Spotify (that's DRM-protected). It uses [spotDL](https://github.com/spotDL/spotify-downloader), which pulls track/album/artist metadata from Spotify's API and matches/downloads the actual audio from YouTube Music, then embeds the Spotify metadata into the downloaded files. Only use this for content you have the rights to.

## What you get

- A `library/` folder organized as `Artist/Album/Track.mp3`, tagged with title, artist, album, track number, and embedded album art, everything Jellyfin needs to scan it properly.
- A `playlists/` folder with one subfolder + `.m3u` file per Spotify playlist, which Jellyfin picks up automatically as real playlists.
- Three shell commands: one-off download, download-with-sync-tracking, and update-all-synced.

## Prerequisites

- Jellyfin already installed and running
- `ffmpeg`
- Python 3 + `pip`

```bash
sudo apt install ffmpeg -y
pip install spotdl --break-system-packages    # or: pipx install spotdl
spotdl --generate-config
```

This creates `~/.config/spotdl/config.json`. Setting up your own free [Spotify Developer app](https://developer.spotify.com/dashboard) client ID/secret in that config is optional but more reliable than spotDL's shared default.

**Check your spotDL version's `--m3u` flag before relying on the scripts below**, this has changed across versions:

```bash
spotdl sync --help | grep -A6 "\-\-m3u"
```

If it shows `--m3u [M3U]` with placeholder syntax like `{list[0]}`, you're set. This is what the commands below assume. If your version instead shows a bare `--m3u` switch with no template support, see [Troubleshooting](#troubleshooting).

## Folder layout

```
<JF_ROOT>/
├── library/        ← point a Jellyfin "Music" library here
│   └── Artist/Album/01 - Track.mp3
├── playlists/       ← point a Jellyfin "Playlists" library here (separate library!)
│   └── Playlist Name/Playlist Name.m3u
└── .sync/           ← NOT a Jellyfin library - spotDL's sync-state files live here
    └── short-name.spotdl
```

## Jellyfin setup

1. **Dashboard → Libraries → Add Library**
   - Content type: **Music** → folder: `<JF_ROOT>/library`
   - Metadata downloaders: MusicBrainz + TheAudioDB
2. **Add a second library**
   - Content type: **Playlists** → folder: `<JF_ROOT>/playlists`
   - Jellyfin auto-imports any `.m3u` file inside as a playlist. No manual "create playlist" step needed.
3. **Artist profile pictures** come from Jellyfin's own TheAudioDB/Fanart.tv lookups (not from Spotify). Install the **Fanart.tv** and **TheAudioDB** plugins (Dashboard → Plugins → Catalog) and add free API keys if artist photos aren't showing up.
4. **Get an API key** for the scan-trigger in these scripts: Dashboard → API Keys → **+**.

## Install the shortcuts

These commands support **multiple music libraries** (e.g. one per family member, or "instrumental" vs "podcasts", etc.) through a short key you choose for each one. `i`, `p`, or anything you like. Add or remove libraries anytime by editing the `JF_ROOTS` array; the commands themselves never need to change.

Append this to `~/.bashrc`, then `source ~/.bashrc`:

```bash
# ── Jellyfin / spotDL config ─────────────────────────────
declare -A JF_ROOTS=(
  [i]="$HOME/path/to/your/first/music/root"    # <-- change this
  [p]="$HOME/path/to/your/second/music/root"   # <-- change this
  # [x]="$HOME/path/to/another/music/root"     # <-- add more like this
)

JELLYFIN_URL="http://localhost:8096"
JELLYFIN_API_KEY="PASTE_YOUR_KEY_HERE"    # <-- change this

# Fall back to plain YouTube search when YT Music has no usable match
_JF_PROVIDERS=(--audio youtube-music youtube)

# Use Firefox's logged-in session to avoid YouTube's bot-detection wall.
# Re-export if this ever goes stale, or after switching Firefox's logged-in account:
#   yt-dlp --cookies-from-browser firefox --cookies "$HOME/.config/spotdl/cookies.txt" --skip-download "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
# For best results, export from a Private/Incognito window: log in, export immediately,
# then close it, a cookie jar that's never reused live in the browser won't get rotated
# out from under the exported file.
_JF_COOKIES=(--cookie-file "$HOME/.config/spotdl/cookies.txt")

# Limit concurrent downloads so requests don't look like a bot hammering YouTube
_JF_THROTTLE=(--threads 2)

# Toggle for spotDL's match-confidence filter.
# 1 (default) = spotDL's normal filtering. Safer, but rejects some real matches
#   (e.g. accented titles, unusual formatting) even when a correct result exists.
# 0 = --dont-filter-results - trusts top search result, more matches but more risk
#   of an occasional wrong/mismatched song slipping through.
# NOTE: this resets to 1 (strict) every time you open a new shell. It is not
# persisted between sessions.
JF_STRICT_MATCH=1
jf-strict() { JF_STRICT_MATCH=1; echo "Strict matching ON (spotDL's normal filter)."; }
jf-loose()  { JF_STRICT_MATCH=0; echo "Strict matching OFF (--dont-filter-results). spot-check downloads after this."; }
_jf_matching() {
  if [ "$JF_STRICT_MATCH" = "0" ]; then
    echo "--dont-filter-results"
  fi
}

_jf_root() {
  local key="$1"
  if [ -z "${JF_ROOTS[$key]}" ]; then
    echo "Unknown library key '$key'. Available: ${!JF_ROOTS[@]}" >&2
    return 1
  fi
  echo "${JF_ROOTS[$key]}"
}

_jf_scan() {
  [ -n "$JELLYFIN_API_KEY" ] && curl -s -X POST \
    "$JELLYFIN_URL/Library/Refresh" -H "X-Emby-Token: $JELLYFIN_API_KEY" >/dev/null
}

jf-list() {
  echo "Configured libraries:"
  for k in "${!JF_ROOTS[@]}"; do echo "  $k → ${JF_ROOTS[$k]}"; done
}

# Review (and optionally delete) files added/changed in a library's last run
jf-undo-last() {
  local key="$1"
  local root; root=$(_jf_root "$key") || return 1
  local lib="$root/library" marker="$root/.last-run-marker"
  if [ ! -f "$marker" ]; then
    echo "No recorded last run for '$key'."
    return 1
  fi
  echo "Files added/changed in the last run for '$key':"
  find "$lib" -type f -newer "$marker" -print
  echo
  read -p "Delete these files? [y/N] " confirm
  if [[ "$confirm" =~ ^[Yy]$ ]]; then
    find "$lib" -type f -newer "$marker" -delete
    echo "Deleted. Run jf-strict then jf-dl/jf-dl-sync again on the same URL to redownload with the stricter filter."
  else
    echo "Cancelled: nothing deleted."
  fi
}

# One-off download, no future updates tracked.
# Safe to rerun: already-downloaded tracks are skipped, only missing/failed ones retry.
jf-dl() {
  local key="$1" url="$2"
  [ -z "$url" ] && { echo "Usage: jf-dl <key> <spotify-url>"; return 1; }
  local root; root=$(_jf_root "$key") || return 1
  local lib="$root/library"
  touch "$root/.last-run-marker"; sleep 1
  local matching; matching=$(_jf_matching)
  ( cd "$root" && spotdl download "$url" \
      --output "$lib/{artist}/{album}/{track-number} - {title}.{output-ext}" \
      --m3u "playlists/{list[0]}/{list[0]}.m3u" \
      --overwrite skip \
      "${_JF_PROVIDERS[@]}" "${_JF_COOKIES[@]}" "${_JF_THROTTLE[@]}" $matching )
  _jf_scan
}

# Download AND track it for future syncing
jf-dl-sync() {
  local key="$1" url="$2" name="$3"
  [ -z "$name" ] && { echo "Usage: jf-dl-sync <key> <spotify-url> <short-name>"; return 1; }
  local root; root=$(_jf_root "$key") || return 1
  local lib="$root/library" sync="$root/.sync"
  mkdir -p "$sync"
  touch "$root/.last-run-marker"; sleep 1
  local matching; matching=$(_jf_matching)
  ( cd "$root" && spotdl sync "$url" --save-file "$sync/$name.spotdl" \
      --output "$lib/{artist}/{album}/{track-number} - {title}.{output-ext}" \
      --m3u "playlists/{list[0]}/{list[0]}.m3u" \
      --overwrite skip \
      --sync-without-deleting \
      "${_JF_PROVIDERS[@]}" "${_JF_COOKIES[@]}" "${_JF_THROTTLE[@]}" $matching )
  _jf_scan
}

# Update synced playlists, one library (key given) or all of them (no key)
jf-update-synced() {
  local key="$1"
  local keys=()
  if [ -n "$key" ]; then
    _jf_root "$key" >/dev/null || return 1
    keys=("$key")
  else
    keys=("${!JF_ROOTS[@]}")
  fi

  shopt -s nullglob
  for k in "${keys[@]}"; do
    local root="${JF_ROOTS[$k]}"
    local lib="$root/library" sync="$root/.sync"
    touch "$root/.last-run-marker"; sleep 1
    local matching; matching=$(_jf_matching)
    for f in "$sync"/*.spotdl; do
      echo "→ [$k] Updating $(basename "$f" .spotdl)"
      ( cd "$root" && spotdl sync "$f" \
          --output "$lib/{artist}/{album}/{track-number} - {title}.{output-ext}" \
          --m3u "playlists/{list[0]}/{list[0]}.m3u" \
          --overwrite skip \
          --sync-without-deleting \
          "${_JF_PROVIDERS[@]}" "${_JF_COOKIES[@]}" "${_JF_THROTTLE[@]}" $matching )
    done
  done
  _jf_scan
}
```

**Before using it:**
- Fill in `JF_ROOTS` with your real music root folders, keyed by whatever short letters/words you like (quote paths that contain spaces).
- Set `JELLYFIN_API_KEY`, or leave it blank if you'd rather trigger library scans manually.
- Set up a cookie file (see [Authentication](#authentication-cookies) below) so `_JF_COOKIES` has something real to point at.
- Each root needs its own pair of Jellyfin libraries set up as described above (one **Music** library at `<root>/library`, one **Playlists** library at `<root>/playlists`).

## Authentication (cookies)

YouTube increasingly blocks anonymous/automated requests ("Sign in to confirm you're not a bot"). Exporting a logged-in browser session as cookies works around this:

```bash
# Close Firefox first. Its cookie DB is locked while the browser is running
yt-dlp --cookies-from-browser firefox --cookies "$HOME/.config/spotdl/cookies.txt" \
  --skip-download "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
```

Two things worth knowing:
- **Cookies rotate.** YouTube periodically refreshes session cookies while you're actively browsing, which can invalidate a static export. The most reliable method: open a **Private/Incognito** Firefox window, log in, export **immediately**, then close the window without browsing further in it, since those cookies are never reused live, they can't be rotated out from under the file.
- **Consider a disposable account.** Large automated batches can occasionally get an account or IP temporarily rate-limited/flagged by YouTube. Using a throwaway Google account's session (rather than your daily-driver account) for this cookie file limits the blast radius if that happens.

## Usage

Every command now takes the library key as its **first** argument:

```bash
jf-list                                                        # see configured library keys

# One-time download, not tracked for future updates
jf-dl i "https://open.spotify.com/playlist/XXXXXXXXXXXX"

# Download and track it for future syncing (pick any short name, no spaces)
jf-dl-sync p "https://open.spotify.com/playlist/XXXXXXXXXXXX" roadtrip

# Refresh every synced playlist across ALL libraries
jf-update-synced

# Refresh only one library's synced playlists
jf-update-synced p

# Loosen spotDL's match filtering for upcoming downloads (more matches, more risk)
jf-loose
jf-dl p "https://open.spotify.com/playlist/XXXXXXXXXXXX"
jf-strict   # back to the safe default

# Review (and optionally delete) everything a library's last run touched
jf-undo-last p
```

`jf-update-synced` (with no key, to cover every library) is a good candidate for a cron job if you want playlists to stay current automatically:

```bash
# crontab -e runs every day at 4am
0 4 * * * /bin/bash -lc 'jf-update-synced' >> ~/jf-sync.log 2>&1
```

## Troubleshooting

**Playlist folder / `.m3u` never gets created:**
Your spotDL version likely handles `--m3u` differently than assumed here. Check:
```bash
spotdl sync --help | grep -A6 "\-\-m3u"
```
- If it takes a template (`--m3u [M3U]` with placeholder syntax like `{list[0]}` or `{list-name}`), match the placeholder name shown in your help output.
- If it's a bare switch with no argument, it writes to your *current directory* using the playlist's own name — the `( cd "$root" && ... )` wrapper already handles this correctly since the working directory is set before the call; just drop the `--m3u "playlists/{list[0]}/{list[0]}.m3u"` argument to a bare `--m3u`.

**Paths with spaces:** always keep every path variable (`$root`, `$lib`, `$sync`, and the entries inside `JF_ROOTS`) double-quoted. Don't remove the quotes when editing the functions.

**Unknown library key error:** run `jf-list` to see exactly which keys are configured — it must match a key in `JF_ROOTS` exactly (case-sensitive).

**Nothing shows up under Jellyfin → Playlists:** confirm you created a *separate* Jellyfin library with content type **Playlists** pointing at `<that library's root>/playlists`. It won't show up if it's just a subfolder inside the Music library.

**A lot of `YouTube Music returned no usable results for X after 3 attempts` / `LookupError: No results found`, even though manually searching for the song on YouTube finds it instantly:** spotDL runs every search result through a confidence-scoring filter before accepting it, and this filter sometimes rejects a genuinely correct result (common with accented characters, unusual title formatting, or regional releases). Run `jf-loose` before downloading to disable that filter (`--dont-filter-results`), then `jf-strict` afterward to go back to the safer default. Spot-check a few of the newly-downloaded tracks afterward, since loose matching does raise the odds of an occasional wrong match on ambiguous titles.

**A synced playlist has files silently disappearing:** `spotdl sync` deletes local files it believes were removed from the source playlist. But a transient API error (`Could not get artist by ID`, `Could not get client token`, etc.) can be misread as "track no longer in playlist," deleting a perfectly good file as a side effect. The functions above already include `--sync-without-deleting` to prevent this entirely; if you're running spotDL commands manually outside these functions, add that flag yourself. The trade-off: genuine removals from the source playlist no longer auto-delete the local file, you'll need to remove those by hand.

**Accidentally downloaded something wrong and want to undo it:** run `jf-undo-last <key>` right after the run in question — it lists (and, with confirmation, deletes) every file added or changed in that library since its last run started. Note it only tracks the single most recent run per library; running anything else overwrites that marker.

**`M3U file name contains '{list}' but no lists were provided. Specify a filename.`:** expected and harmless when using `jf-dl`/`jf-dl-sync` on a single bare track URL rather than a playlist or album. There's no playlist context for `{list[0]}` to fill in, so no m3u gets written, but the track itself still downloads normally.

**No artist photos:** these come from TheAudioDB/Fanart.tv plugins, not from Spotify or spotDL. Install and configure those plugins with free API keys in Jellyfin.

## Disclaimer

This tooling relies on spotDL, which sources audio from YouTube rather than Spotify directly. You're responsible for how you use it — only download content you have the right to.
