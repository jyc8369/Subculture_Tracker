# Subculture Tracker Development Guide

> Combined Markdown index: [RAEDME.md](RAEDME.md)


[English](README_DEV.md) | [한국어](README_DEV_KO.md)

This document explains the development environment, project structure, how to run the app, and the internal behavior of Subculture Tracker.

## Overview

Subculture Tracker automatically collects gacha history from the following two games, converts it into a shared JSON format, and displays the results through a web UI.

- Wuthering Waves
- Arknights Endfield

The project runs on Windows. It reads local game logs and cache files to extract the required token or URL, then fetches history data through the game server APIs.

## Development Environment

- Python 3.7 or later
- Packages defined in `requirements.txt`
  - `Flask>=3.0`
  - `requests>=2.0`
  - `beautifulsoup4>=4.0`
  - `pywebview>=3.0`

### Create a Virtual Environment

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

## Running the App

### Standard Run

```powershell
python run.py
```

In standard mode, `run.py` starts the internal Flask server and opens a native GUI window if `pywebview` is installed.

### Browser Development Mode

```powershell
python run.py -dev
```

With the `-dev` option, the UI opens in the default web browser instead of a webview. This mode also works in environments where `pywebview` is not installed.

## `run.py` Behavior Summary

- If the app is not running with administrator privileges on Windows, it attempts to relaunch itself with elevation.
- It may relaunch through `pythonw.exe` on Windows to hide the console window.
- It creates `lastlog.log` at startup and records application logs there.
- It calls `backend.web.run(use_browser=dev_mode)` to start the Flask server and web UI.

## Structure and Responsibilities

### Root Files

- `run.py`
  - Entry point
  - Administrator privilege check
  - Windows console hiding
  - Flask server startup
- `requirements.txt`
  - Project dependencies
- `SCT.spec`
  - PyInstaller build configuration
- `build-exe.ps1`
  - Script for building the executable with PyInstaller
- `README.md`
  - End-user documentation
- `README_KO.md`
  - Korean end-user documentation
- `README_DEV.md`
  - Developer documentation
- `README_DEV_KO.md`
  - Korean developer documentation

### Backend Package (`backend/`)

- `backend/web.py`
  - Flask app initialization, API endpoints, job queue, background worker
- `backend/endfield.py`
  - Endfield data extraction, token detection, API calls, JSON export
- `backend/wuwa.py`
  - Wuthering Waves log search, URL extraction, server requests, JSON export
- `backend/webview.py`
  - Native window creation based on `pywebview`

### Static Resources

- `web/`
  - Static HTML/CSS/JS UI files
  - Game-specific tracker pages in `web/game/`
- `data/`
  - Directory for collected JSON results and configuration files
  - Examples: `wuwa_<profilename>.json`, `endfield_<profilename>.json`, `web-setting.json`

## Backend Details

### `backend/web.py`

#### Runtime Setup

- Uses different paths depending on whether the app is running from source or from a PyInstaller build.
- Uses the `web/` directory for templates and static files.
- Creates the `data/` folder automatically at runtime.

#### Job Queue and Status

- `job_queue`: Queue for background JSON generation tasks
- `job_statuses`: Stores job state, output file path, and error messages
- `exporter_worker`: Pulls tasks from the queue and processes `wuwa` or `endfield` data

#### Main API Endpoints

- `/`
  - Renders `web/main.html`
- `/game/<path:filename>`
  - Serves resources under `web/game/`
- `/web/<path:filename>`
  - Serves static resources under `web/`
- `/data/list`
  - Returns a JSON file list using `game=wuwa`, `game=endfield`, or `game=all`
- `/data/<path:filename>`
  - Returns the actual file if it exists
  - For `wuwa_<profilename>.json`, it can fall back to an internal profile-name lookup
- `/exporter` (POST)
  - Registers a JSON generation task
  - Request examples:
    - `game_name=wuwa`, `profilename=USER`, `url=...`, `wait=true`, `timeout=60`
    - `game_name=endfield`, `profilename=USER`, `wait=true`
  - `game_name` is required
  - `profilename` is required
  - `url` is optional for `wuwa`, but required if log lookup fails
- `/exporter/status/<job_id>`
  - Returns job status

#### `process_export_task`

- `game_name == 'wuwa'`
  - If `url` is missing, call `wuwa.find_latest_url()` to search the log for a URL
  - Use `profilename` to generate the output filename
  - Call `backend.wuwa.process_url_to_data`
- `game_name == 'endfield'`
  - Use `profilename` to generate the output filename
  - Call `backend.endfield.fetch_endfield_data`

#### Server Startup Flow

- Find a free local port with `find_free_port()`
- Start Flask in a background thread
- In `-dev` mode, open the system browser after a 1 second delay
- If `pywebview` is installed, open a native window
- If `pywebview` is missing and `-dev` is enabled, run in browser mode only
- If `pywebview` is missing in normal mode, raise an exception

## Wuthering Waves (`backend/wuwa.py`)

### Goal

- Extract the latest gacha record URL from Wuthering Waves logs
- Call the server API using that URL to collect history data
- Convert it into the shared JSON schema and save it as `data/wuwa_<profilename>.json`

### Installation Path and Log Search

- Searches the Windows Registry for the Steam install location
- Uses a list of possible install paths such as `Program Files`, `SteamLibrary`, `Epic Games`, `Games`, and `Wuthering Waves Game`
- Also searches default AppData/LocalLow, AppData/Local, AppData/Roaming, Documents, Saved Games, and ProgramData paths

### Finding Log Files

- Recursively searches for `.log` files
- Searches up to depth 4
- Sorts by latest modification time and checks the newest logs first

### URL Extraction

- Searches the log text with this URL pattern:
  - `https://aki-gm-resources.../aki/gacha/index.html#/record...`
- Treats the last match as the latest URL

### Data Collection

- Parses the hash fragment (`#`) from the URL to extract query parameters such as `player_id`, `resources_id`, `svr_id`, `lang`, and `record_id`
- Uses the fixed POST endpoint: `https://gmserver-api.aki-game2.net/gacha/record/query`
- Iterates through `cardPoolType` values 1 through 9 to collect banner records
- Collects all returned records without deduplication

### JSON Conversion

- Each record is normalized with `normalize_wuwa_record()`
- Shared schema:
  - `id`: player ID
  - `banners`: list of banners
    - `bannerName`
    - `items`: list of records
      - `type`, `seqId`, `name`, `rarity`, `timestamp`
- Output file: `data/wuwa_<profilename>.json`
- Existing `wuwa_*.json` files with the same `id` are deduplicated

## Endfield (`backend/endfield.py`)

### Goal

- Extract the latest token from the Endfield installation cache
- Call the Endfield webview API to fetch character gacha history with pagination
- Convert the result into the shared JSON schema and save it as `data/endfield_<profilename>.json`

### Token Search

- Default data file path: `%LOCALAPPDATA%\\PlatformProcess\\Cache\\data_1`
- If that path does not exist, fall back to `~/AppData/Local/PlatformProcess/Cache/data_1`
- Reads the file content and extracts the `token` value
- Supports multiple patterns, including URL query strings, JSON properties, and script variables

### API Requests

- Default page URL: `https://ef-webview.gryphline.com/page/gacha_char?...`
- `build_request()` parses either the page URL or a direct `/api/record/char`-style URL
- Requests set the `token`, `server_id`, `pool_type`, and `lang` parameters

### Pagination

- `fetch_all_pages()` continues requesting more pages using the `seq_id` value
- Collection stops when the response contains no records or `hasMore` is `False`
- The request URL list and the full record list are accumulated together

### JSON Conversion

- Combines `poolId` and `poolName` values into `banners`
- Each record is stored as:
  - `type`, `seqId`, `name`, `rarity`, `timestamp`
- Overall structure:
  - `id`: token value
  - `banners`: list of per-pool entries
- Output file: `data/endfield_<profilename>.json`
- Duplicate files with the same token are removed

### Command-Line Usage

- You can run it directly with `python backend/endfield.py <profilename>`
- The `main()` function prints logs and returns a success or failure exit code

## Example Data Format

```json
{
  "id": "user-or-token",
  "banners": [
    {
      "bannerName": "Banner Name",
      "items": [
        {
          "type": 1,
          "seqId": "123456",
          "name": "Character Name",
          "rarity": "5",
          "timestamp": 1680000000000
        }
      ]
    }
  ]
}
```

## Debugging and Logs

- Backend logs are written to `lastlog.log`
- `backend/web.py` merges Flask request logs so `werkzeug` output uses the same logger name
- `backend/endfield.py` and `backend/wuwa.py` log HTTP failures, JSON parsing errors, and file I/O errors

## Packaging

- Running `build-exe.ps1` uses `SCT.spec` to build with PyInstaller
- The Windows standalone executable bundles the `run.py` entry point directly
- Build options:
  - `-Clean`: remove previous build output before rebuilding
  - `-NoUpx`: skip UPX compression during the build

## Folder Overview

- `.gitignore`: Git ignore rules
- `.venv/`: local Python virtual environment
- `SCT.spec`: PyInstaller configuration
- `backend/`: application logic
- `data/`: output JSON files and configuration storage
- `demo/`: sample assets for development and testing
- `dist/`: build output
- `lastlog.log`: latest runtime log
- `web/`: static web UI assets

## Additional Notes for Development

- `/exporter` in `backend/web.py` supports both synchronous `wait` mode and asynchronous queue-based processing, which makes it easier to separate browser requests from background tasks.
- Both `wuwa` and `endfield` use `profilename` in the output file name, so make sure the UI does not create collisions for the same name.
- The `data/` folder may store configuration files such as `web-setting.json` in addition to collected results.
