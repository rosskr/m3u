# CLAUDE.md

## Project Overview

This is a browser-based M3U playlist player — a single-page web application that loads, parses, and plays streaming media (video and audio) from M3U playlist files. It supports HLS streams and includes preset playlists for Russian-language movies, TV, and radio.

The project is entirely static: no build system, no package manager, no server-side code. Everything runs in the browser.

---

## Repository Structure

```
/
├── index.html       # The entire application (HTML + CSS + JS in one file)
├── rusmovie.m3u     # Preset playlist: Russian movies/TV series (HLS streams)
├── rusradio.m3u     # Preset playlist: Russian radio stations (audio streams)
└── CLAUDE.md        # This file
```

---

## Technology Stack

| Concern         | Technology                                                   |
|-----------------|--------------------------------------------------------------|
| UI / Logic      | Vanilla HTML5, CSS3, JavaScript (ES6+)                       |
| Video playback  | [HLS.js](https://cdn.jsdelivr.net/npm/hls.js) (via CDN)      |
| Player UI       | [Plyr 3](https://unpkg.com/plyr@3) (via CDN)                 |
| Player styling  | Plyr CSS from `https://cdn.plyr.io/3.6.7/plyr.css`           |
| Playlist format | M3U / Extended M3U                                           |
| Deployment      | Static file hosting (e.g., GitHub Pages)                     |

There is **no Node.js, no npm, no bundler, no framework**. CDN dependencies are loaded at runtime via `<script>` and `<link>` tags.

---

## Application Architecture

The entire application lives in `index.html` as three collocated sections:

### 1. CSS (lines 6–147)

Embedded in a `<style>` block. Key layout decisions:
- `#topContainer`: fixed-height header bar with hamburger menu, URL input, and Load button
- `#mainContainer`: flexbox row split into two columns
  - `#streamListContainer`: 25% width, scrollable sidebar listing stream entries
  - `#playerContainer`: 73% width, holds the sticky video player and stream description
- `.selected-stream` / `.group-description`: highlight colors (gray `#888888`)
- `a:link`, `a:visited`: global anchor styling with green border — affects all `<a>` tags

### 2. HTML (lines 149–178)

- Hamburger button (`#hamburger`) triggers a dropdown submenu with preset playlist links
- `#subMenu` (absolutely positioned, hidden by default): VOD, Russian TV, US TV options
- `#streamList`: a `<ul>` populated dynamically by JavaScript
- `<video id="player">`: the Plyr-enhanced video element

### 3. JavaScript (lines 182–321)

All JS is in a `<script>` block at the bottom of `<body>`.

---

## Core Functions

| Function | Lines | Purpose |
|---|---|---|
| `toggleSubMenu()` | 184–193 | Show/hide the hamburger dropdown menu |
| `loadPlaylist(url)` | 196–207 | Fetch M3U from a URL, parse it, render the list |
| `parseM3U(data)` | 253–273 | Parse raw M3U text into an array of stream objects |
| `displayStreamList(streams)` | 275–303 | Render stream objects as clickable list items, grouped by category |
| `playStream(url, description)` | 305–320 | Load and play a stream URL using HLS.js or native HLS |

### Data flow

```
User action
  → loadPlaylist(url)
      → fetch(url) → raw M3U text
      → parseM3U(data) → streams[]
      → displayStreamList(streams) → DOM list items
          → click → playStream(url, description)
                       → Hls.loadSource / video.src
                       → video.play()
```

### Stream object shape

`parseM3U` produces objects of this shape:

```js
{
  duration: "100",        // from #EXTINF duration field
  description: "string",  // display name (after the comma in #EXTINF)
  url: "https://...",     // the stream URL on the next line
  group: "Group Name"     // from the most recent #EXTGRP directive
}
```

---

## Preset Playlist URLs

| Menu Item   | Source URL |
|-------------|------------|
| VOD         | `https://raw.githubusercontent.com/rosskr/m3u/main/rusmovie.m3u` |
| Russian TV  | `<ENTER_RUSSIAN_TV_PLAYLIST_URL>` — **placeholder, not yet configured** (index.html line 225) |
| US TV       | `https://iptv-org.github.io/iptv/countries/us.m3u` |

---

## M3U File Format

Both `.m3u` files use Extended M3U format. Directives used:

| Directive | Purpose | Example |
|-----------|---------|---------|
| `#EXTM3U` | Required header, first line of file | `#EXTM3U` |
| `#EXTGRP:<name>` | Sets the group/category for following entries | `#EXTGRP:Radio` |
| `#EXTINF:<duration>,<title>` | Metadata for the next stream URL | `#EXTINF:100,Station Name` |
| `#EXTIMG:<url>` | Thumbnail image URL (parsed but not displayed by current player) | `#EXTIMG:https://...` |

The line immediately following `#EXTINF` must be the stream URL. `parseM3U` reads `lines[i+1]` for the URL whenever it encounters `#EXTINF`.

---

## Stream Playback Logic (`playStream`, line 305)

```
if (Hls.isSupported())          → use HLS.js (most desktop browsers)
else if (canPlayType mpegurl)   → use native HLS (Safari, iOS)
```

HLS.js instance is stored on `window.hls`. **Known issue:** a new `Hls` instance is created on every stream selection without destroying the previous one. This can cause resource leaks if many streams are played in sequence.

---

## Development Workflow

There is no build step. To develop:

1. Edit `index.html` directly in any text editor
2. Open `index.html` in a browser (or serve locally with `python3 -m http.server`)
3. Test by loading a playlist URL or clicking a preset menu item

### Local serving (avoids CORS issues with `fetch`)

```sh
python3 -m http.server 8080
# Then open http://localhost:8080
```

### To add a new preset playlist

1. Add a `<li><a href="#" id="myMenu">Label</a></li>` entry to `#subMenu` in the HTML (around line 163)
2. Wire up an event listener in the JS section following the pattern at lines 218–231:
   ```js
   document.getElementById("myMenu").addEventListener("click", function () {
     toggleSubMenu();
     loadPlaylist("https://example.com/playlist.m3u");
   });
   ```

### To add entries to the M3U playlists

Follow the Extended M3U format:

```
#EXTGRP:Group Name
#EXTINF:100,Stream Title
https://stream-url.example.com/stream.m3u8
```

---

## Known Issues / TODOs

- **Russian TV placeholder** (index.html:225): `russianTVMenu` calls `loadPlaylist("<ENTER_RUSSIAN_TV_PLAYLIST_URL>")` — this will fail until a real URL is substituted.
- **HLS instance leak**: `playStream` creates a new `Hls()` on every call without calling `hls.destroy()` on the previous instance.
- **No CORS handling**: `fetch` requests for external M3U files will fail if the host does not serve appropriate CORS headers.
- **`#EXTIMG` ignored**: Thumbnail URLs are present in `rusmovie.m3u` but `parseM3U` does not extract them and `displayStreamList` does not render them.
- **YouTube iframe in M3U**: `rusmovie.m3u` line 7 contains an iframe embed as a stream URL. The current `playStream` function passes this to HLS.js, which will fail. YouTube embeds require a separate iframe rendering path.
- **`keyCode` deprecated**: The Enter-key handler (line 247) uses `event.keyCode` — prefer `event.key === "Enter"`.

---

## Conventions

- **One file**: all application code stays in `index.html`. Do not split into separate `.css` or `.js` files unless the project structure is intentionally being expanded.
- **No framework**: keep the implementation in vanilla JS. Do not introduce React, Vue, or any other framework.
- **No build tools**: do not add npm, webpack, vite, etc. unless explicitly requested.
- **Camelcase** for all JS identifiers.
- **Version string in `<title>`**: the title `M3U Playlist Player - v6_13_6_58` appears to encode a date (June 13, version 6:58). Update it when making significant changes.
- **CDN versions**: Plyr CSS is pinned to `3.6.7`; HLS.js and Plyr JS use unpinned CDN URLs (`latest`). Be mindful of breaking changes when unpinned packages update.
