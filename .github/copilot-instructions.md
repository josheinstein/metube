# MeTube Development Guide

## Architecture Overview

MeTube is a web-based YouTube downloader split into two main components:

- **Backend** (`app/`): Python aiohttp server that manages downloads using yt-dlp, handles queue/persistence, and exposes WebSocket + REST APIs
- **Frontend** (`ui/`): Angular 21 SPA that communicates via Socket.IO for real-time download status updates

### Data Flow
1. User submits URL via Angular UI → POST to `/add` endpoint
2. Backend validates, creates `DownloadInfo`, adds to `DownloadQueue`
3. Download executes in separate process (multiprocessing), streaming status updates via queue
4. Backend emits Socket.IO events (`added`, `updated`, `completed`) to all connected clients
5. Angular `DownloadsService` receives events, updates RxJS subjects, triggers UI refresh

## Key Components

### Backend (`app/`)
- **main.py**: Entry point with aiohttp routes, Socket.IO setup, config management, and file watching
- **ytdl.py**: Download queue orchestration with three modes (`sequential`, `concurrent`, `limited`), multiprocessing downloads, shelve persistence
- **dl_formats.py**: Format selection logic (audio/video), yt-dlp postprocessor configuration

### Frontend (`ui/src/app/`)
- **app.ts**: Main component managing UI state, download actions, theme switching
- **services/downloads.service.ts**: Central service listening to Socket.IO events, maintaining `queue`/`done` Maps
- **services/metube-socket.service.ts**: Custom Socket.IO client configured for URL_PREFIX support
- **interfaces/**: TypeScript models matching Python backend (Download, Status, State, Format, etc.)

## Critical Patterns

### Environment-Driven Configuration
All behavior is controlled via environment variables (see `Config._DEFAULTS` in [main.py](app/main.py)). Never hardcode paths or settings.

```python
# Config resolves %% references: AUDIO_DOWNLOAD_DIR defaults to DOWNLOAD_DIR value
'AUDIO_DOWNLOAD_DIR': '%%DOWNLOAD_DIR'
```

### Download Modes ([ytdl.py](app/ytdl.py))
- `sequential`: Single download at a time (FIFO)
- `concurrent`: All downloads start immediately (no limit)
- `limited`: Semaphore-based concurrency (default: 3 simultaneous)

Queue persistence uses Python's `shelve` module stored in `STATE_DIR`.

### Socket.IO Event Contract
Backend emits: `all` (full state on connect), `added`, `updated`, `completed`, `canceled`, `cleared`, `configuration`, `custom_dirs`, `ytdl_options_changed`

Frontend listens via `DownloadsService`, updating reactive Maps and emitting RxJS Subjects.

### Format Selection
Format strings are built dynamically in [dl_formats.py](app/dl_formats.py#L6-L49). Example:
```python
# For MP4 best quality: "bestvideo[ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]"
# For iOS: prioritizes h264/h265 + aac codec compatibility
```

Audio downloads use FFmpeg postprocessors with embedded thumbnails.

### Angular Standalone Components
All components use Angular 21's standalone API (no NgModule). See [app.ts](ui/src/app/app.ts#L18-L32) imports array.

## Development Workflows

### Local Development
```bash
# Backend (Python 3.13+)
pip install -e .  # or: uv sync
python app/main.py

# Frontend
cd ui
pnpm install
pnpm start  # Serves on http://localhost:4200, proxies API to :8081
```

### Docker Build
Multi-stage build: Node builder → Alpine Python runtime. See [Dockerfile](Dockerfile).
```bash
docker build -t metube .
docker run -p 8081:8081 -v /downloads:/downloads metube
```

### Testing Changes
- Backend: Run `python app/main.py` and test via curl/Postman or UI
- Frontend: `pnpm build` outputs to `dist/metube/browser` (served by backend at `/`)
- Linting: `pylint app/*.py` (backend), `pnpm lint` (frontend)

## Integration Points

### yt-dlp Options
- `YTDL_OPTIONS` (env var): JSON object merged into yt-dlp config
- `YTDL_OPTIONS_FILE`: JSON file watched for changes (triggers live reload via `watchfiles`)
- Passed to `yt_dlp.YoutubeDL()` in [ytdl.py](app/ytdl.py)

### URL_PREFIX Support
All routes and static assets honor `URL_PREFIX` for reverse proxy deployments. Socket.IO path is automatically adjusted in [metube-socket.service.ts](ui/src/app/services/metube-socket.service.ts#L12-L13).

### Custom Directories
If `CUSTOM_DIRS=true`, backend scans `DOWNLOAD_DIR` recursively (excluding regex matches), emits directory tree to frontend dropdown. Created on-the-fly if `CREATE_CUSTOM_DIRS=true`.

## Common Tasks

### Adding a New Format
1. Update [dl_formats.py](app/dl_formats.py) `get_format()` with format logic
2. Add to [ui/src/app/interfaces/formats.ts](ui/src/app/interfaces/formats.ts) `Formats` array
3. Test with sample URL via UI

### Changing Download Queue Behavior
Modify [ytdl.py](app/ytdl.py) `DownloadQueue` class. Respect `DOWNLOAD_MODE` config and semaphore in `limited` mode (see `_download_next()`).

### Adding Socket.IO Events
1. Emit in backend: `await sio.emit('event_name', serializer.encode(data))`
2. Subscribe in [downloads.service.ts](ui/src/app/services/downloads.service.ts): `this.socket.fromEvent('event_name').subscribe(...)`

### Troubleshooting
- Check logs: `LOGLEVEL=DEBUG` reveals yt-dlp commands, download paths, Socket.IO connections
- Access log: `ENABLE_ACCESSLOG=true` logs HTTP requests
- Frontend console: Socket.IO events logged in browser DevTools when debug enabled
