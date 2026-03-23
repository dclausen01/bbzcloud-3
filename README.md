# BBZCloud Desktop v3

**A unified Electron application for educational services – native APIs, no WebViews.**

## Overview

BBZCloud Desktop v3 integrates multiple educational services through native APIs:

- **Stashcat** – Native chat client (via `stashcat-api`)
- **Nextcloud** – File management (WebDAV + OCS API)
- **OnlyOffice** – Document editing (via Nextcloud integration)

### Key Features

- 🔐 **Single Login** – One authentication for all services (AD-backed)
- 💬 **Native Chat** – Real-time messaging with E2E decryption
- 📁 **File Browser** – Browse Nextcloud files directly in the app
- 📝 **Document Editor** – Edit Office documents in embedded OnlyOffice
- 🔗 **Cross-Service Links** – Attach Nextcloud files to chat messages

## Tech Stack

| Layer | Technology |
|-------|------------|
| Desktop | Electron 33+ |
| UI | React 19 + Tailwind v4 |
| Stashcat | `stashcat-api` |
| Nextcloud | WebDAV + OCS API |
| Storage | SQLite + Keytar |

## Project Structure

```
bbzcloud-3/
├── electron/           # Electron main process
├── src/               # React frontend
├── public/            # Static assets
└── docs/              # Design documentation
```

## Development

```bash
npm install           # Install dependencies
npm run dev           # Start development mode
npm run build         # Build for production
npm run dist          # Package for distribution
```

## Documentation

- [Design Document](docs/superpowers/specs/2026-03-23-bbzcloud-desktop-design.md)

## Related Projects

- [stashcat-api](https://github.com/dclausen01/stashcat-api) – Stashcat API client
- [bbzcloud-2](https://github.com/dclausen01/bbzcloud-2) – Previous version (WebView-based)

## License

MIT © Dennis Clausen