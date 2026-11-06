tv-dashboard-router Lightweight HTTP coordinator for Samsung Tizen NOC dashboards, replacing per-TV mini PCs.

## Overview

This project solves a signage problem at Triade: 8 TVs displaying internal Grafana dashboards, originally driven by small PCs that overheat, get noisy, and restart under sustained rendering load. Instead of patching the PCs, the dashboards move onto the TVs themselves through a kiosk style web app, and a small central server tells every TV which dashboard it should be showing right now.

## The Problem

- Mini PCs rendering continuously refreshing dashboards generate sustained CPU/GPU load that cheap signage hardware is not built to dissipate for 24/7 operation.
- Thermal protection kicks in, causing random restarts and audible fan noise.
- Keeping a PC per TV in the loop does not remove the root cause, it only delays it.

## Architecture

Three roles, one source of truth:

- **TV**: polls the server for its assigned URL and renders it. Never receives pushes, never gets told what to show by anyone directly.
- **Server**: holds the only authoritative mapping of TV ID to URL. Answers reads from TVs and writes from PCs.
- **PC** (any machine on the network): assigns a URL to a given TV ID by writing to the server.

Centralizing the mapping removes a real race condition. With no single source of truth, two PCs could assign conflicting URLs to the same TV with no way to resolve which one wins.

## How It Works

1. Every 60 seconds, each TV calls `fetchURL(id)` on the server.
2. The server returns the URL currently assigned to that TV ID.
3. The TV compares the returned URL against what it is currently showing.
   - Same URL: discard, do nothing.
   - Different URL: call `renderPage(url)` and update local state.
4. A PC assigns a dashboard by calling `setURL(id, url)` on the server at any time.

## Tech Stack

| Component | Choice | Why |
| --- | --- | --- |
| Server | FastAPI + Python + SQLite | Read heavy, write rare, single digit record count. A full database server would be pure overhead here. |
| TV client | Tizen Web App (HTML/CSS/JS) | TVs run Samsung Tizen. A Web App runs inside a privileged WebView that already does exactly what this project needs: fetch and render. No native (C/C++) or managed (C#) layer is required for this use case. |
| Dashboards | Grafana, anonymous Viewer access, network restricted | Avoids login prompts in a kiosk context without weakening security, since access stays scoped to the internal network already trusted by the rest of the system. |

## API

`GET /tv/{id}/url` Returns the URL currently assigned to the TV with that ID.

`POST /tv/{id}/url` Body: `{ "url": "http://172.x.x.x/macro" }` Assigns a new URL to the TV with that ID.

## Project Status

Architecture decided and validated. Currently confirming whether Tizen TVs can render these dashboards directly without a PC in between, since that assumption is the one the whole project stands on. Not yet in implementation.
