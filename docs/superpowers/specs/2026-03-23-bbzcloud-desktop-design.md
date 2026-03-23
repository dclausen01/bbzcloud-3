# BBZCloud Desktop v3 – Design Document

**Date:** 2026-03-23
**Author:** Dennis Clausen
**Status:** Approved

---

## Overview

BBZCloud Desktop v3 is a unified Electron application that integrates multiple educational services through native APIs rather than WebViews. The MVP focuses on **Stashcat** (chat), **Nextcloud** (files), and **OnlyOffice** (document editing).

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Electron + React + Tailwind | Reuse UI from stashcat-chat, modern stack |
| Native API integration | Better UX than WebViews, deep linking between services |
| Single login | All services authenticate against the same AD |
| SQLite + Keytar | Offline-capable, secure credential storage |
| OnlyOffice via Nextcloud | Leverages existing integration, single API surface |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   BBZCloud Desktop (v3)                             │
│                   Electron + React + Tailwind                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  UI Layer (React + Tailwind v4)                                    │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐││
│  │  │   Sidebar    │ │   Content    │ │   Panels (optional)      │││
│  │  │              │ │              │ │                          │││
│  │  │ • Stashcat   │ │ • Chat View  │ │ • File Browser          │││
│  │  │ • Nextcloud  │ │ • Files View │ │ • OnlyOffice Editor     │││
│  │  │ • Settings   │ │ • Search     │ │ • Member Panel          │││
│  │  └──────────────┘ └──────────────┘ └──────────────────────────┘││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  Electron Main Process (Node.js)                                   │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │  API Layer                                                      ││
│  │  ┌─────────────────┐ ┌─────────────────┐ ┌──────────────────┐ ││
│  │  │ stashcat-api    │ │ Nextcloud Client│ │ OnlyOffice       │ ││
│  │  │ (existing lib)  │ │ (WebDAV + OCS)  │ │ (via Nextcloud)  │ ││
│  │  └─────────────────┘ └─────────────────┘ └──────────────────┘ ││
│  │                                                                 ││
│  │  Storage                                                        ││
│  │  ┌─────────────────┐ ┌─────────────────┐                       ││
│  │  │ SQLite          │ │ Keytar          │                       ││
│  │  │ (sessions, data)│ │ (credentials)   │                       ││
│  │  └─────────────────┘ └─────────────────┘                       ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Services

### 1. Stashcat (Native Chat)

**Integration:** Direct via `stashcat-api` library (file:../stashcat-api)

**Features:**
- Real-time chat via Socket.io → SSE bridge
- E2E decryption (server-side in Electron main process)
- File attachments (from Stashcat storage or Nextcloud)
- Channel/Conversation management
- Broadcast messages
- Calendar events

**Data Flow:**
```
React Component (ChatView)
    ↓ IPC
Electron Main Process
    ↓ API calls
stashcat-api → api.stashcat.com / api.schul.cloud
```

### 2. Nextcloud (Files)

**Integration:** WebDAV + OCS API

**Endpoints:**
- `cloud.bbz-rd-eck.de`
- WebDAV: `/remote.php/dav/files/{user}/`
- OCS API: `/ocs/v2.php/apps/files_sharing/api/v1/shares`

**Features:**
- Browse files/folders
- Upload/download files
- Create/rename/delete files and folders
- Share files (for attaching to chat messages)
- Get OnlyOffice editor URL for documents

**WebDAV Client Options:**
- `webdav` npm package (recommended)
- Custom fetch-based implementation

### 3. OnlyOffice (Document Editing)

**Integration:** Via Nextcloud API (existing integration)

**Flow:**
```
1. User clicks on .docx/.xlsx/.pptx file in Nextcloud browser
2. App calls Nextcloud OCS API to get editor URL
3. Open WebView with editor URL (authenticated via Nextcloud session)
4. User edits document
5. Changes saved automatically to Nextcloud
```

**Editor URL Pattern:**
```
https://cloud.bbz-rd-eck.de/apps/onlyoffice/{fileId}
```

---

## Authentication

### Unified Login Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     Login Screen                                │
│                                                                 │
│  Email:    [________________]                                   │
│  Password: [________________]                                   │
│                                                                 │
│  [Login]                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  Electron Main Process                                          │
│                                                                 │
│  1. Stashcat.login(email, password)                            │
│     → Success: Store session in SQLite                          │
│                                                                 │
│  2. Nextcloud.login(email, password)                           │
│     → WebDAV PROPFIND with Basic Auth                           │
│     → Success: Store session token                              │
│                                                                 │
│  3. Store credentials in Keytar (encrypted system keychain)     │
│                                                                 │
│  4. Auto-login on next start                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Credential Storage

| Storage | Purpose |
|---------|---------|
| **Keytar** | Email + Password (encrypted, system keychain) |
| **SQLite** | Session tokens, user preferences, cached data |

---

## Cross-Service Integration (Level 2)

### File Attachment from Nextcloud

```
┌────────────────────────────────────────────────────────────────┐
│ Chat View                                                       │
│                                                                 │
│ [📎 Attach] → Opens file picker                                │
│              ┌─────────────────────────────┐                   │
│              │ • Stashcat Files            │                   │
│              │ • Nextcloud Files    ←───── │── Default         │
│              │   ├─ 📁 Documents           │                   │
│              │   ├─ 📁 Projects            │                   │
│              │   └─ 📄 report.docx         │                   │
│              └─────────────────────────────┘                   │
│                                                                 │
│ User selects file → Share via Nextcloud API → Send link       │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Open Document in OnlyOffice

```
┌────────────────────────────────────────────────────────────────┐
│ Split View                                                      │
│                                                                 │
│ ┌───────────────────┬────────────────────────────────────────┐ │
│ │ Chat              │ OnlyOffice Editor (WebView)            │ │
│ │                   │                                        │ │
│ │ "Hier ist das     │ ┌────────────────────────────────────┐ │ │
│ │  Protokoll..."    │ │ report.docx                        │ │ │
│ │                   │ │                                    │ │ │
│ │ [📄 report.docx]  │ │ [Live editing in WebView]          │ │ │
│ │      ↓            │ │                                    │ │ │
│ │ Click opens →     │ └────────────────────────────────────┘ │ │
│ └───────────────────┴────────────────────────────────────────┘ │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

### Frontend

| Technology | Version | Purpose |
|------------|---------|---------|
| **Electron** | 33+ | Desktop framework |
| **React** | 19 | UI framework |
| **Tailwind CSS** | v4 | Styling |
| **React Router** | v7 | Navigation |
| **Lucide React** | Latest | Icons |

### Backend (Electron Main Process)

| Technology | Purpose |
|------------|---------|
| **stashcat-api** | Stashcat integration |
| **webdav** | Nextcloud WebDAV client |
| **SQLite3** | Local database |
| **Keytar** | Credential storage |
| **Electron Store** | Settings |

### Build

| Tool | Purpose |
|------|---------|
| **Vite** | Frontend build |
| **Electron Builder** | Packaging |
| **GitHub Actions** | CI/CD |

---

## Project Structure

```
bbzcloud-3/
├── electron/
│   ├── main.ts                 # Electron main process
│   ├── preload.ts              # IPC bridge
│   ├── services/
│   │   ├── StashcatService.ts  # stashcat-api wrapper
│   │   ├── NextcloudService.ts # WebDAV + OCS client
│   │   ├── AuthService.ts      # Unified login
│   │   └── DatabaseService.ts  # SQLite operations
│   └── utils/
│       └── ipc.ts              # IPC channel definitions
├── src/
│   ├── App.tsx                 # Root component
│   ├── main.tsx                # React entry
│   ├── components/
│   │   ├── Sidebar.tsx         # Navigation
│   │   ├── ChatView.tsx        # Stashcat chat
│   │   ├── FilesView.tsx       # Nextcloud browser
│   │   ├── EditorView.tsx      # OnlyOffice WebView
│   │   ├── FilePicker.tsx      # Cross-service file picker
│   │   └── ...
│   ├── context/
│   │   ├── AuthContext.tsx     # Auth state
│   │   └── SettingsContext.tsx # App settings
│   ├── hooks/
│   │   └── useIPC.ts           # Electron IPC hooks
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   └── MainPage.tsx
│   └── styles/
│       └── index.css           # Tailwind imports
├── public/
│   └── assets/
├── package.json
├── vite.config.ts
├── electron-builder.yml
└── README.md
```

---

## API Contracts

### IPC Channels

| Channel | Direction | Payload |
|---------|-----------|---------|
| `auth:login` | Renderer → Main | `{ email, password }` |
| `auth:logout` | Renderer → Main | `void` |
| `auth:status` | Main → Renderer | `{ authenticated, user? }` |
| `stashcat:getChannels` | Renderer → Main | `{ companyId }` |
| `stashcat:getMessages` | Renderer → Main | `{ targetId, type, limit?, offset? }` |
| `stashcat:sendMessage` | Renderer → Main | `{ targetId, type, text, files? }` |
| `nextcloud:listFiles` | Renderer → Main | `{ path? }` |
| `nextcloud:getFile` | Renderer → Main | `{ path }` |
| `nextcloud:shareFile` | Renderer → Main | `{ path }` |
| `onlyoffice:getEditorUrl` | Renderer → Main | `{ fileId }` |

### Nextcloud WebDAV

```typescript
// List files
PROPFIND /remote.php/dav/files/{user}/{path}
Headers:
  Authorization: Basic {base64(user:password)}
  Depth: 1

// Upload file
PUT /remote.php/dav/files/{user}/{path}/{filename}
Headers:
  Authorization: Basic {base64(user:password)}
  Content-Type: application/octet-stream
Body: {file content}

// Download file
GET /remote.php/dav/files/{user}/{path}/{filename}
Headers:
  Authorization: Basic {base64(user:password)}
```

---

## Future Considerations

### Phase 2 Services

| Service | Integration | Priority |
|---------|-------------|----------|
| **Moodle** | Web Services API | High |
| **WebUntis** | REST API | Medium |
| **TaskCards** | REST API | Medium |

### Potential Enhancements

- Unified inbox (all notifications in one place)
- Project workspaces (link chat + files + boards)
- Offline mode (sync when back online)
- Multi-account support
- Dark mode (inherited from bbzcloud-2)

---

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Nextcloud API changes | Version API calls, add fallbacks |
| E2E key management complexity | Reuse proven stashcat-api implementation |
| WebView performance for OnlyOffice | Test on low-end devices, add fallback to external browser |
| Session expiry handling | Implement refresh tokens, auto-reconnect |

---

## Acceptance Criteria

### MVP Success Criteria

- [ ] User can login once and access Stashcat + Nextcloud
- [ ] User can browse and participate in Stashcat chats
- [ ] User can browse Nextcloud files
- [ ] User can open documents in OnlyOffice editor
- [ ] User can attach Nextcloud files to Stashcat messages
- [ ] App persists sessions and auto-logs in on restart
- [ ] App builds for Windows, macOS, Linux

---

## References

- [stashcat-api Documentation](../stashcat-api/CLAUDE.md)
- [stashcat-chat Architecture](../stashcat-chat/CLAUDE.md)
- [bbzcloud-2 Previous Version](../bbzcloud-2/README.md)
- [Nextcloud WebDAV API](https://docs.nextcloud.com/server/latest/developer_manual/client_apis/WebDAV/)
- [OnlyOffice Document Server API](https://api.onlyoffice.com/editors/basic)