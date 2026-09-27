# spotify-to-jellyfin

Download Spotify playlists (audio + metadata) into a Jellyfin-ready music library, with proper artist/album tagging and auto-generated `.m3u` playlists that show up natively in Jellyfin. Includes bash shortcuts to download once, download-and-track-for-syncing, and update all synced playlists later.

> **Note on how this works:** this does not rip audio from Spotify (that's DRM-protected). It uses [spotDL](https://github.com/spotDL/spotify-downloader), which pulls track/album/artist metadata from Spotify's API and matches/downloads the actual audio from YouTube Music, then embeds the Spotify metadata into the downloaded files. Only use this for content you have the rights to.

## What you get

- A `library/` folder organized as `Artist/Album/Track.mp3`, tagged with title, artist, album, track number, and embedded album art and everything Jellyfin needs to scan it properly.
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

**Check your spotDL version's `--m3u` flag before relying on the scripts below** — this has changed across versions:

```bash
spotdl sync --help | grep -A6 "\-\-m3u"
```

If it shows `--m3u [M3U]` with placeholder syntax like `{list[0]}`, you're set — this is what the commands below assume. If your version instead shows a bare `--m3u` switch with no template support, see [Troubleshooting](#troubleshooting).

## Folder layout

```
<JF_ROOT>/
├── library/        ← point a Jellyfin "Music" library here
│   └── Artist/Album/01 - Track.mp3
├── playlists/       ← point a Jellyfin "Playlists" library here (separate library!)
│   └── Playlist Name/Playlist Name.m3u
└── .sync/           ← NOT a Jellyfin library — spotDL's sync-state files live here
    └── short-name.spotdl
```

## Jellyfin setup

1. **Dashboard → Libraries → Add Library**
   - Content type: **Music** → folder: `<JF_ROOT>/library`
   - Metadata downloaders: MusicBrainz + TheAudioDB
2. **Add a second library**
   - Content type: **Playlists** → folder: `<JF_ROOT>/playlists`
   - Jellyfin auto-imports any `.m3u` file inside as a playlist — no manual "create playlist" step needed.
3. **Artist profile pictures** come from Jellyfin's own TheAudioDB/Fanart.tv lookups (not from Spotify). Install the **Fanart.tv** and **TheAudioDB** plugins (Dashboard → Plugins → Catalog) and add free API keys if artist photos aren't showing up.
4. **Get an API key** for the scan-trigger in these scripts: Dashboard → API Keys → **+**.

## Install the shortcuts

These commands support **multiple music libraries** (e.g. one per family member, or "instrumental" vs "podcasts", etc.) through a short key you choose for each one — `i`, `p`, or anything you like. Add or remove libraries anytime by editing the `JF_ROOTS` array; the commands themselves never need to change.

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

# One-off download — no future updates tracked
jf-dl() {
  local key="$1" url="$2"
  [ -z "$url" ] && { echo "Usage: jf-dl <key> <spotify-url>"; return 1; }
  local root; root=$(_jf_root "$key") || return 1
  local lib="$root/library"
  ( cd "$root" && spotdl download "$url" \
      --output "$lib/{artist}/{album}/{track-number} - {title}.{output-ext}" \
      --m3u "playlists/{list[0]}/{list[0]}.m3u" )
  _jf_scan
}

# Download AND track it for future syncing
jf-dl-sync() {
  local key="$1" url="$2" name="$3"
  [ -z "$name" ] && { echo "Usage: jf-dl-sync <key> <spotify-url> <short-name>"; return 1; }
  local root; root=$(_jf_root "$key") || return 1
  local lib="$root/library" sync="$root/.sync"
  mkdir -p "$sync"
  ( cd "$root" && spotdl sync "$url" --save-file "$sync/$name.spotdl" \
      --output "$lib/{artist}/{album}/{track-number} - {title}.{output-ext}" \
      --m3u "playlists/{list[0]}/{list[0]}.m3u" )
  _jf_scan
}

# Update synced playlists — one library (key given) or all of them (no key)
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
    for f in "$sync"/*.spotdl; do
      echo "→ [$k] Updating $(basename "$f" .spotdl)"
      ( cd "$root" && spotdl sync "$f" \
          --output "$lib/{artist}/{album}/{track-number} - {title}.{output-ext}" \
          --m3u "playlists/{list[0]}/{list[0]}.m3u" )
    done
  done
  _jf_scan
}
```

**Before using it:**
- Fill in `JF_ROOTS` with your real music root folders, keyed by whatever short letters/words you like (quote paths that contain spaces).
- Set `JELLYFIN_API_KEY`, or leave it blank if you'd rather trigger library scans manually.
- Each root needs its own pair of Jellyfin libraries set up as described above (one **Music** library at `<root>/library`, one **Playlists** library at `<root>/playlists`).

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
```

`jf-update-synced` (with no key, to cover every library) is a good candidate for a cron job if you want playlists to stay current automatically:

```bash
# crontab -e — runs every day at 4am
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

**Nothing shows up under Jellyfin → Playlists:** confirm you created a *separate* Jellyfin library with content type **Playlists** pointing at `<that library's root>/playlists` — it won't show up if it's just a subfolder inside the Music library.

**No artist photos:** these come from TheAudioDB/Fanart.tv plugins, not from Spotify or spotDL. Install and configure those plugins with free API keys in Jellyfin.

## Disclaimer

This tooling relies on spotDL, which sources audio from YouTube rather than Spotify directly. You're responsible for how you use it — only download content you have the right to.
