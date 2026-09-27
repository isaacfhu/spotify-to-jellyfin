# spotify-to-jellyfin

Download Spotify playlists (audio + metadata) into a Jellyfin-ready music library, with proper artist/album tagging and auto-generated `.m3u` playlists that show up natively in Jellyfin. Includes bash shortcuts to download once, download-and-track-for-syncing, and update all synced playlists later.

> **Note on how this works:** this does not rip audio from Spotify (that's DRM-protected). It uses [spotDL](https://github.com/spotDL/spotify-downloader), which pulls track/album/artist metadata from Spotify's API and matches/downloads the actual audio from YouTube Music, then embeds the Spotify metadata into the downloaded files. Only use this for content you have the rights to.

## What you get

- A `library/` folder organized as `Artist/Album/Track.mp3`, tagged with title, artist, album, track number, and embedded album art — everything Jellyfin needs to scan it properly.
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

Append this to `~/.bashrc`, then `source ~/.bashrc`:

```bash
# ── Jellyfin / spotDL config ─────────────────────────────
JF_ROOT="$HOME/path/to/your/music/root"   # <-- change this
JF_LIB="$JF_ROOT/library"
JF_PL="$JF_ROOT/playlists"
JF_SYNC="$JF_ROOT/.sync"
JELLYFIN_URL="http://localhost:8096"
JELLYFIN_API_KEY="PASTE_YOUR_KEY_HERE"    # <-- change this

_JF_OUT="$JF_LIB/{artist}/{album}/{track-number} - {title}.{output-ext}"
_JF_M3U="playlists/{list[0]}/{list[0]}.m3u"   # relative — must run from $JF_ROOT

_jf_scan() {
  [ -n "$JELLYFIN_API_KEY" ] && curl -s -X POST \
    "$JELLYFIN_URL/Library/Refresh" -H "X-Emby-Token: $JELLYFIN_API_KEY" >/dev/null
}

# One-off download of a playlist/album — no future updates tracked
jf-dl() {
  [ -z "$1" ] && { echo "Usage: jf-dl <spotify-url>"; return 1; }
  ( cd "$JF_ROOT" && spotdl download "$1" --output "$_JF_OUT" --m3u "$_JF_M3U" )
  _jf_scan
}

# Download AND set it up to be kept in sync later
jf-dl-sync() {
  [ -z "$2" ] && { echo "Usage: jf-dl-sync <spotify-url> <short-name>"; return 1; }
  mkdir -p "$JF_SYNC"
  ( cd "$JF_ROOT" && spotdl sync "$1" --save-file "$JF_SYNC/$2.spotdl" --output "$_JF_OUT" --m3u "$_JF_M3U" )
  _jf_scan
}

# Update every playlist that's being synced
jf-update-synced() {
  shopt -s nullglob
  for f in "$JF_SYNC"/*.spotdl; do
    echo "→ Updating $(basename "$f" .spotdl)"
    ( cd "$JF_ROOT" && spotdl sync "$f" --output "$_JF_OUT" --m3u "$_JF_M3U" )
  done
  _jf_scan
}
```

**Before using it:**
- Set `JF_ROOT` to your actual music root folder (quote it if the path has spaces).
- Set `JELLYFIN_API_KEY`, or leave it blank if you'd rather trigger library scans manually.

## Usage

```bash
# One-time download, not tracked for future updates
jf-dl "https://open.spotify.com/playlist/XXXXXXXXXXXX"

# Download and track it for future syncing (pick any short name, no spaces)
jf-dl-sync "https://open.spotify.com/playlist/XXXXXXXXXXXX" roadtrip

# Later, refresh every playlist you've set up with jf-dl-sync
jf-update-synced
```

`jf-update-synced` is a good candidate for a cron job if you want playlists to stay current automatically:

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
- If it's a bare switch with no argument, it writes to your *current directory* using the playlist's own name — the `( cd "$JF_ROOT" && ... )` wrapper already handles this correctly since the working directory is set before the call; just change `_JF_M3U="playlists/{list[0]}/{list[0]}.m3u"` to `_JF_M3U=""` and call `--m3u` with no value.

**Paths with spaces:** always keep `JF_ROOT` (and anything built from it) double-quoted. If you edit the functions, don't remove the quotes around `"$JF_ROOT"`, `"$_JF_OUT"`, etc.

**Nothing shows up under Jellyfin → Playlists:** confirm you created a *separate* Jellyfin library with content type **Playlists** pointing at `<JF_ROOT>/playlists` — it won't show up if it's just a subfolder inside your Music library.

**No artist photos:** these come from TheAudioDB/Fanart.tv plugins, not from Spotify or spotDL. Install and configure those plugins with free API keys in Jellyfin.

## Disclaimer

This tooling relies on spotDL, which sources audio from YouTube rather than Spotify directly. You're responsible for how you use it. Only download content you have the right to.
