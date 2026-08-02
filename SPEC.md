# Pi in a Tab — MVP Product and Technical Specification

Status: Draft 0.1  
Last updated: 2026-07-31

## 1. Summary

Pi in a Tab is a small, local-first coding agent that runs its workspace and tools inside the browser. A user describes a change, the agent reads and edits project files, and the result appears in a live web preview.

The first version is intentionally limited to small static web projects. It is not a Linux VM, a complete Node.js environment, or a replacement for a desktop IDE.

The architecture must make later capabilities possible without requiring a plugin framework or multiple runtimes in the MVP.

## 2. Product principles

1. **Useful before complete.** The MVP must finish a small web project end to end.
2. **Local-first.** Project files and command execution stay in the browser. Model requests are the only required remote computation.
3. **Mobile is a primary target.** The core experience must work in current mobile Safari and Chromium browsers.
4. **One workspace.** The editor, agent tools, persistence, preview, and any later runtimes operate on the same file tree.
5. **Capabilities are optional.** Shell access, JavaScript runners, Python, Git, Node.js, and remote sandboxes may be added later and loaded only when needed.
6. **Interfaces before plugins.** The MVP uses small TypeScript interfaces and static composition. It does not build a dynamic plugin marketplace, event bus, or extension loader.
7. **Generated code is untrusted.** It must not execute in the main application context.

## 3. Target user and primary job

The initial user wants to create or modify a small browser application from a phone, tablet, or desktop without installing development tools.

Primary job:

> “Give the agent an instruction, review the files it changed, and immediately see the result.”

Representative projects:

- A landing page
- A small interactive tool
- A personal page
- A UI prototype
- A simple HTML/CSS/JavaScript application

## 4. MVP scope

### 4.1 Included

- Create a project from a built-in static-web template
- Chat with a Pi-based coding agent
- Read, create, update, rename, and delete workspace files
- Search file names and text content
- Preview an `index.html` project in a sandboxed iframe
- Persist projects across browser reloads
- Create an automatic checkpoint before each agent turn
- Undo the most recent agent turn
- Import and export a project as a ZIP file
- Work in a responsive interface on desktop and mobile
- Show tool activity and errors without exposing hidden model reasoning

### 4.2 Explicitly excluded

- Arbitrary native binaries
- Linux or x86 emulation
- Docker
- Node.js and npm package installation
- Development servers
- Shell or terminal emulation
- Git clone, pull, push, and remote authentication
- Standalone JavaScript execution through QuickJS
- Python execution
- WebContainers
- Cloud execution sandboxes
- Real-time collaboration
- Dynamic third-party plugins
- Background agents
- Multi-agent orchestration
- Deployment or hosting

These exclusions are scope decisions, not architectural prohibitions.

## 5. Core user flow

1. The user opens the application.
2. The user creates a project from the static-web template or imports a ZIP.
3. The application loads the project into the browser workspace.
4. The user gives the agent an instruction.
5. The application creates a workspace checkpoint.
6. The agent inspects files and invokes constrained tools.
7. File changes appear in the editor and activity log.
8. The preview refreshes automatically.
9. The user accepts the result, continues prompting, or restores the checkpoint.
10. The user can export the project at any time.

## 6. MVP architecture

```text
┌──────────────────────────────── Browser application ────────────────────────────────┐
│                                                                                      │
│  UI                                                                                  │
│  ├── Chat                                                                            │
│  ├── File list and editor                                                            │
│  ├── Tool activity                                                                   │
│  └── Sandboxed preview                                                               │
│                                                                                      │
│  Application core                                                                    │
│  ├── AgentController ───── ModelTransport                                             │
│  ├── ToolRegistry                                                                    │
│  ├── Workspace ───────── PersistenceAdapter                                           │
│  ├── CheckpointService                                                               │
│  └── PreviewRenderer ─── sandboxed iframe                                             │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

There is no general-purpose VM in the MVP.

### 6.1 Agent controller

The agent controller wraps Pi Agent Core and owns one conversation at a time.

Responsibilities:

- Maintain conversation state
- Send prompts through `ModelTransport`
- Expose approved tools to the model
- Stream user-visible events to the UI
- Cancel the current turn
- Stop after configurable tool and turn limits

The agent controller must not access browser storage or UI elements directly.

### 6.2 Model transport

Model access is behind a narrow interface:

```ts
interface ModelTransport {
  stream(request: ModelRequest, signal: AbortSignal): AsyncIterable<ModelEvent>
}
```

The default deployment uses a minimal same-origin model proxy so provider credentials are not shipped to the browser. A future bring-your-own-key transport may be added without changing the agent controller.

Only file content deliberately included in the model context leaves the browser. The application must communicate this clearly to the user.

### 6.3 Workspace

`Workspace` is the single source of truth for project files.

```ts
interface Workspace {
  readFile(path: string): Promise<Uint8Array>
  writeFile(path: string, data: Uint8Array): Promise<void>
  list(path?: string): Promise<WorkspaceEntry[]>
  mkdir(path: string): Promise<void>
  move(from: string, to: string): Promise<void>
  remove(path: string): Promise<void>
  search(query: SearchQuery): Promise<SearchResult[]>
  snapshot(): Promise<WorkspaceSnapshot>
  restore(snapshot: WorkspaceSnapshot): Promise<void>
  subscribe(listener: WorkspaceListener): () => void
}
```

Rules:

- All paths are normalized absolute workspace paths.
- Paths cannot escape the workspace root.
- File names and project size are bounded.
- Mutations emit events used by persistence, the editor, and preview.
- The UI and tools never maintain independent copies of the file tree.

### 6.4 Persistence

The MVP persists serialized workspace snapshots in IndexedDB because it has broad browser support and keeps the storage model simple.

```ts
interface PersistenceAdapter {
  load(projectId: string): Promise<PersistedProject | undefined>
  save(project: PersistedProject): Promise<void>
  remove(projectId: string): Promise<void>
}
```

Persistence occurs after workspace mutations using a short debounce. A flush is required when the page becomes hidden.

OPFS may replace or complement IndexedDB later behind the same interface. The MVP does not implement both.

### 6.5 Agent tools

The MVP exposes a deliberately small tool set:

| Tool | Purpose |
| --- | --- |
| `list_files` | List a directory or project tree |
| `read_file` | Read a bounded text file |
| `write_file` | Create or replace a text file |
| `apply_patch` | Make a targeted text edit |
| `move_file` | Rename or move a file |
| `delete_file` | Delete a file inside the workspace |
| `search_files` | Search paths or text |

Tool results must be bounded before being returned to the model. Large output is truncated with an explicit notice.

The application creates one checkpoint before the first mutating tool in an agent turn. It does not snapshot before every individual file write.

### 6.6 Preview renderer

The MVP supports static projects with an `index.html` entry point.

The renderer:

1. Reads `index.html` and referenced local CSS and classic JavaScript files.
2. Produces a preview document or equivalent virtual resource mapping.
3. Loads it into a sandboxed iframe.
4. Reloads when relevant workspace files change.
5. Captures runtime errors and displays them to the user.

Initial limitations:

- Relative local CSS and classic script references only
- No npm imports
- No TypeScript or JSX
- No development server
- No backend routes
- No service worker inside the preview

The iframe must execute with a unique origin and must not receive access to application storage, model credentials, or the parent DOM. The preview document uses a restrictive Content Security Policy that blocks network access. Communication uses a narrow `postMessage` protocol that validates `event.source`, a per-preview nonce, and the message shape. An opaque-origin iframe reports a `null` serialized origin, so origin-string comparison alone is not a valid check.

## 7. UI requirements

The MVP has four surfaces:

1. **Chat:** prompt input, streaming response, cancel action, and turn status.
2. **Files:** project tree and text editor.
3. **Preview:** rendered project plus runtime error state.
4. **Activity:** concise list of tool calls and changed files.

Desktop may show multiple panes. Mobile uses one primary pane at a time with persistent navigation among Chat, Files, and Preview.

## 8. Project data model

```ts
interface ProjectRecord {
  id: string
  name: string
  createdAt: number
  updatedAt: number
  files: Record<string, Uint8Array>
  conversation?: SerializedAgentState
  checkpoints: WorkspaceCheckpoint[]
  schemaVersion: 1
}

interface WorkspaceCheckpoint {
  id: string
  createdAt: number
  label: string
  files: Record<string, Uint8Array>
}
```

MVP limits:

- Maximum 500 files per project
- Maximum 5 MB total uncompressed project size
- Maximum five retained checkpoints per project
- Text-oriented projects; binary files may be imported and exported but are not editable by the agent

Limits must be centralized configuration rather than scattered constants.

## 9. Security model

The browser sandbox is useful but does not make every component trusted.

Required controls:

- Never execute generated JavaScript in the application window.
- Run preview code in a sandboxed, opaque-origin iframe.
- Keep provider secrets on the server-side model proxy.
- Validate every path at the `Workspace` boundary.
- Limit agent turns, tool calls, file sizes, and output sizes.
- Do not expose IndexedDB, OPFS, cookies, or authentication tokens to preview code.
- Treat imported projects and ZIP paths as untrusted input.
- Sanitize file names and reject archive path traversal.
- Make destructive workspace changes recoverable through checkpoints.

The MVP does not claim VM-grade isolation. Features requiring arbitrary code or binaries must use a purpose-built runtime or remote sandbox in a later phase.

## 10. Modularity rules

The architecture is modular through ports and adapters, not dynamic plugins.

### 10.1 Stable ports

The following interfaces are intended extension points:

- `ModelTransport`
- `Workspace`
- `PersistenceAdapter`
- `AgentTool`
- `PreviewRenderer`
- `ProjectImporter`
- `ProjectExporter`

### 10.2 Dependency direction

Core modules may depend on interfaces, never on concrete browser or provider implementations.

```text
UI ──> application services ──> core interfaces <── adapters
```

Examples:

- The agent knows about `Workspace`, not IndexedDB.
- The workspace knows about `PersistenceAdapter`, not a particular database library.
- The preview feature knows about `PreviewRenderer`, not WebContainers.
- Tools know about capabilities passed to them, not global browser objects.

### 10.3 What not to build yet

- No runtime-discovered plugins
- No extension manifest format
- No general event bus
- No dependency injection framework
- No remote procedure abstraction shared by every component
- No universal POSIX compatibility layer
- No attempt to normalize every future runtime behind one enormous interface

Add an abstraction only after a second implementation demonstrates the need for it.

## 11. Evolution path

Later capabilities are separate modules and should be lazy-loaded.

### Stage A: constrained shell

- Add `just-bash` behind a `CommandRunner` interface
- Run it in a disposable Web Worker
- Disable networking and optional language runtimes
- Synchronize mutations through `Workspace`
- Bound execution time, command count, loop count, and output

### Stage B: richer web projects

- Add `esbuild-wasm` as a new `PreviewRenderer`
- Support local TypeScript, JSX, and module graphs
- Add a controlled dependency resolver

### Stage C: Git

- Add a `VersionControl` interface
- Implement local Git with `isomorphic-git` or `wasm-git`
- Add provider-specific remote authentication separately

### Stage D: standalone code execution

- Add QuickJS as a `CodeRunner` for bounded JavaScript execution
- Keep it in a disposable Worker
- Grant filesystem and network capabilities explicitly

### Stage E: Python

- Add MicroPython for lightweight scripts
- Add Pyodide only for CPython and package compatibility
- Load neither runtime until requested

### Stage F: full environments

- Add WebContainers as an optional Node.js capability on compatible devices
- Add a remote sandbox adapter for native binaries, large builds, or unsupported devices
- Preserve the local workspace as the user-owned source of truth

None of these stages is required to validate the MVP.

## 12. Acceptance criteria

The MVP is complete when all of the following work on a supported desktop browser and at least one iOS and one Android device:

1. A user creates a project from the default template.
2. The user asks the agent to make a visible change.
3. The agent reads and edits the correct files using approved tools.
4. The changed project appears in the preview without a full page reload.
5. Refreshing or closing and reopening the application preserves the project.
6. The user can undo the most recent agent turn.
7. Malformed, oversized, or out-of-scope tool input fails without corrupting the workspace.
8. Preview code cannot read application storage, access the network, or modify the parent UI.
9. The user can export the project and import it into a new local project.

## 13. Initial implementation sequence

1. Implement the in-memory `Workspace` and tests.
2. Add IndexedDB persistence and schema migration support.
3. Build the file tree and text editor around `Workspace`.
4. Implement static preview isolation and error reporting.
5. Add model transport and Pi Agent Core with read-only tools.
6. Add mutating tools, checkpoints, and undo.
7. Add ZIP import/export.
8. Complete mobile interaction and cross-browser testing.

## 14. Decisions to make before implementation

Only three product choices remain blocking:

1. Which model provider the first proxy supports
2. Which frontend framework and editor component the application uses
3. Whether conversations persist with projects or only workspace files persist

All other optional runtime decisions are deferred until after the MVP is validated.
