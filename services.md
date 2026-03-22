# React + Backend MVP Architecture

## 1. Product architecture

The original two-project model (**Telegram bot + backend**) should be generalized into a three-layer model:

* **Web frontend (primary product UI)** — React + TypeScript, IDE-like workspace
* **Backend service (system of record for logic/orchestration)** — Node.js service with layered architecture
* **Telegram bot (optional control plane)** — auxiliary notifications and remote commands, not the main UX

### Core principle

**AI is not a standalone chat surface. AI is an action layer over the active document.**

That means:

* the active file and current selection are primary context;
* user issues an action from editor actions, command palette, slash command, or task flow;
* backend runs orchestration and returns a **proposed change**;
* UI presents the result as a diff/proposal;
* user can **accept**, **reject**, or **export to a new file**.

### UX decisions frozen for MVP

* **Dark theme is the default visual theme for the whole application.**
* The editor supports **Markdown edit mode** and **Markdown preview mode**.
* There is **no persistent AI assistant panel on the right**.
* There is **no floating AI bar overlaying the editor workspace**.
* AI actions are invoked contextually: from selection actions, document toolbar actions, command palette, or task creation flows.
* Files from the workspace tree can be **dragged into the editor** and **attached into tasks as structured file objects**.

---

## 2. Project split

### apps/web

Primary UX.

Responsibilities:

* render workspace shell;
* show Git/file tree;
* manage tabs and editor state;
* support Markdown edit and preview modes;
* collect document context for AI actions;
* render proposal diff/review UX;
* support drag-and-drop insertion of workspace file objects into editor and task flows;
* call backend APIs only.

### apps/api

Business logic and orchestration.

Responsibilities:

* workspace discovery;
* file read/write abstraction;
* Git adapter;
* AI orchestration;
* prompt construction;
* patch/diff generation;
* metadata persistence in SQLite;
* API contract for web and Telegram control plane.

### apps/tg-bot (optional)

Secondary interface.

Responsibilities:

* trigger limited commands;
* show job status / alerts / quick actions;
* never handle inline editing or primary review UX.

---

## 3. Monorepo structure

```text
repo/
├── apps/
│   ├── web/
│   │   ├── src/
│   │   │   ├── app/
│   │   │   ├── pages/
│   │   │   ├── widgets/
│   │   │   ├── features/
│   │   │   ├── entities/
│   │   │   ├── shared/
│   │   │   └── api/
│   │   ├── package.json
│   │   └── vite.config.ts
│   │
│   ├── api/
│   │   ├── src/
│   │   │   ├── core/
│   │   │   ├── module/
│   │   │   │   ├── workspace/
│   │   │   │   ├── document/
│   │   │   │   ├── ai/
│   │   │   │   ├── change/
│   │   │   │   ├── git/
│   │   │   │   ├── task/
│   │   │   │   └── settings/
│   │   │   ├── infra/
│   │   │   │   ├── db/
│   │   │   │   ├── filesystem/
│   │   │   │   ├── git/
│   │   │   │   └── llm/
│   │   │   └── main.ts
│   │   ├── prisma/
│   │   └── package.json
│   │
│   └── tg-bot/
│       ├── src/
│       └── package.json
│
├── packages/
│   ├── shared-types/
│   ├── prompt-kit/
│   ├── diff-kit/
│   └── eslint-config/
│
└── prd.json
```

---

## 4. Architecture rule by layer

### Frontend layering

```text
UI / Widgets → Features → Entities → Shared API client
```

* **Widgets**: app shell, file tree panel, tab strip, editor area, command palette, task composer, diff review drawer.
* **Features**: open file, save file, switch editor mode, run AI command, review proposal, create new file from proposal, attach file object to task, insert file object into editor.
* **Entities**: workspace, document, proposal, git status, prompt preset, file object reference, task draft.
* **Shared API client**: HTTP calls to backend.

### Backend layering

```text
API → Application Service → Domain / Policies → Infrastructure adapters
```

* **API**: REST endpoints, DTO validation
* **Application service**: use cases and orchestration
* **Domain / Policies**: business rules, patch policy, file safety, command rules, file object normalization
* **Infrastructure**: SQLite, filesystem, Git, LLM providers

### Cross-cutting rule

* Frontend never touches filesystem, Git, or LLM directly.
* Backend does not own the canonical document content in DB.
* **Filesystem + Git working tree = source of truth for files.**
* SQLite stores only metadata, runs, proposals, preferences, and task/file-object references if needed.

---

## 5. Backend module breakdown

## 5.1 workspace

Responsibilities:

* register/open a local project root;
* validate safe path;
* expose project tree;
* expose repo metadata.

Suggested entities:

* `Workspace`
* `WorkspaceIndexSnapshot` (optional)

## 5.2 document

Responsibilities:

* open file content;
* save file content;
* create file;
* support normalized file object insertion metadata for editor and task flows;
* track editor session metadata.

Suggested entities:

* `DocumentSession`
* `OpenTabSnapshot` (optional persisted UI state)

## 5.3 ai

Responsibilities:

* accept AI action request;
* construct prompt from command + active file + selection + optional instructions + attached file objects;
* call text model;
* normalize output;
* return proposal rather than direct mutation.

Suggested entities:

* `AiCommandRun`
* `PromptTemplate`
* `ModelConfig`

## 5.4 change

Responsibilities:

* store proposal payload;
* compute and persist diff;
* apply proposal to existing file;
* reject proposal;
* export proposal to new file.

Suggested entities:

* `ChangeProposal`
* `ChangeArtifact`
* `DiffHunk` (optional, can be computed on demand)

## 5.5 git

Responsibilities:

* status;
* staged/unstaged summary;
* file diff summary;
* current branch.

Suggested entities:

* `GitSnapshot` (optional cache)

## 5.6 task

Responsibilities:

* create task drafts / persisted tasks if included in MVP scope;
* accept file objects attached from workspace;
* expose attached file references as AI context.

Suggested entities:

* `TaskDraft`
* `TaskAttachment`

## 5.7 settings

Responsibilities:

* editor mode defaults;
* preferred model;
* prompt presets;
* UI preferences if persisted locally.

Suggested entities:

* `UserPreference`

---

## 6. Core business rules

1. **AI never overwrites a file immediately.**
2. Every AI action on a file returns a `ChangeProposal`.
3. A proposal must be linked to:

   * workspace id;
   * target file path;
   * command type;
   * original content hash;
   * proposed content or patch;
   * status (`pending`, `accepted`, `rejected`, `exported`).
4. Apply is allowed only if the current file version is compatible with the proposal baseline, or the system explicitly re-bases/re-generates.
5. Export-to-new-file must preserve the original file unchanged.
6. Backend must reject unsafe paths outside workspace root.
7. First-stage MVP supports **text / markdown only**.
8. Markdown viewing must support a dedicated **preview mode**.
9. There is no persistent right assistant surface in MVP.
10. Dragging a workspace file into the editor or task flow creates a **structured file object reference**, not an implicit full-content paste.
11. Attached file objects may be resolved by backend as contextual references for later AI commands.
12. Telegram bot cannot be the only path for accepting or reviewing diffs.

---

## 7. Minimal API contract (MVP)

## 7.1 Workspace

### `POST /api/v1/workspaces/open`

Open or register a local workspace.

Request:

```json
{ "root_path": "/Users/me/project" }
```

Response:

```json
{
  "workspace_id": "ws_123",
  "name": "project",
  "root_path": "/Users/me/project",
  "git_enabled": true,
  "default_branch": "main"
}
```

### `GET /api/v1/workspaces/:workspaceId/tree`

Returns Git/filesystem tree.

Response:

```json
{
  "items": [
    {
      "path": "README.md",
      "name": "README.md",
      "type": "file"
    },
    {
      "path": "docs",
      "name": "docs",
      "type": "directory"
    }
  ]
}
```

---

## 7.2 Documents

### `GET /api/v1/documents/content?workspace_id=ws_123&path=README.md`

Response:

```json
{
  "workspace_id": "ws_123",
  "path": "README.md",
  "language": "markdown",
  "content": "# Hello",
  "sha256": "abc123"
}
```

### `PUT /api/v1/documents/content`

Saves a file.

Request:

```json
{
  "workspace_id": "ws_123",
  "path": "README.md",
  "content": "# Updated",
  "base_hash": "abc123"
}
```

Response:

```json
{
  "path": "README.md",
  "saved": true,
  "sha256": "def456"
}
```

### `POST /api/v1/documents/create`

Request:

```json
{
  "workspace_id": "ws_123",
  "path": "docs/new-file.md",
  "content": "# New file"
}
```

### `POST /api/v1/documents/file-object/resolve`

Normalizes a dragged workspace file into a structured object for editor/task insertion.

Request:

```json
{
  "workspace_id": "ws_123",
  "path": "docs/spec.md",
  "target": "editor"
}
```

Response:

```json
{
  "file_object": {
    "workspace_id": "ws_123",
    "path": "docs/spec.md",
    "display_name": "spec.md",
    "file_type": "markdown",
    "sha256": "abc123"
  },
  "editor_representation": "[[file:docs/spec.md]]"
}
```

---

## 7.3 AI commands

### `POST /api/v1/ai/commands`

Runs an AI action against the active document.

Request:

```json
{
  "workspace_id": "ws_123",
  "path": "README.md",
  "command": "rewrite_selected",
  "instruction": "Make this clearer for CTO audience",
  "selection": {
    "start_line": 10,
    "end_line": 22,
    "selected_text": "..."
  },
  "context": {
    "open_tabs": ["README.md", "docs/spec.md"],
    "language": "markdown",
    "attached_file_objects": [
      {
        "workspace_id": "ws_123",
        "path": "docs/spec.md",
        "display_name": "spec.md",
        "file_type": "markdown"
      }
    ]
  }
}
```

Response:

```json
{
  "run_id": "run_001",
  "proposal": {
    "proposal_id": "cp_001",
    "status": "pending",
    "target_path": "README.md",
    "mode": "patch",
    "summary": "Clarified selected section for technical audience",
    "original_hash": "abc123",
    "proposed_content": "...",
    "diff": {
      "unified": "@@ ...",
      "stats": { "additions": 12, "deletions": 4 }
    }
  }
}
```

---

## 7.4 Proposal review

### `POST /api/v1/proposals/:proposalId/apply`

Request:

```json
{ "apply_mode": "replace_target" }
```

Response:

```json
{
  "proposal_id": "cp_001",
  "status": "accepted",
  "saved_path": "README.md",
  "sha256": "ghi789"
}
```

### `POST /api/v1/proposals/:proposalId/reject`

Response:

```json
{
  "proposal_id": "cp_001",
  "status": "rejected"
}
```

### `POST /api/v1/proposals/:proposalId/export`

Request:

```json
{ "new_path": "README.ai.md" }
```

Response:

```json
{
  "proposal_id": "cp_001",
  "status": "exported",
  "saved_path": "README.ai.md"
}
```

---

## 7.5 Git

### `GET /api/v1/git/status?workspace_id=ws_123`

Response:

```json
{
  "branch": "main",
  "ahead": 0,
  "behind": 0,
  "files": [
    {
      "path": "README.md",
      "status": "modified"
    }
  ]
}
```

### `GET /api/v1/git/diff?workspace_id=ws_123&path=README.md`

Response:

```json
{
  "path": "README.md",
  "diff": "@@ ..."
}
```

---

## 7.6 Tasks

### `POST /api/v1/tasks/draft`

Creates or updates a task draft with attached file objects.

Request:

```json
{
  "workspace_id": "ws_123",
  "title": "Revise PRD section",
  "description": "Use attached files as context",
  "attachments": [
    {
      "workspace_id": "ws_123",
      "path": "docs/spec.md",
      "display_name": "spec.md",
      "file_type": "markdown"
    }
  ]
}
```

---

## 8. ERD (MVP)

## 8.1 Tables

### `workspaces`

* `id` (pk)
* `name`
* `root_path` (unique)
* `git_enabled`
* `default_branch`
* `created_at`
* `updated_at`

### `document_sessions`

* `id` (pk)
* `workspace_id` (fk -> workspaces.id)
* `path`
* `last_opened_at`
* `last_known_hash`
* `is_pinned` (optional)

### `ai_command_runs`

* `id` (pk)
* `workspace_id` (fk)
* `path`
* `command`
* `instruction`
* `selection_json`
* `context_json`
* `model_name`
* `status` (`queued`, `completed`, `failed`)
* `error_message` (nullable)
* `created_at`

### `change_proposals`

* `id` (pk)
* `run_id` (fk -> ai_command_runs.id)
* `workspace_id` (fk)
* `target_path`
* `mode` (`patch`, `replace`, `new_file`)
* `summary`
* `original_hash`
* `proposed_content`
* `diff_unified`
* `status` (`pending`, `accepted`, `rejected`, `exported`)
* `exported_path` (nullable)
* `created_at`
* `updated_at`

### `task_drafts`

* `id` (pk)
* `workspace_id` (fk)
* `title`
* `description`
* `status` (`draft`, `ready`, `archived`)
* `created_at`
* `updated_at`

### `task_attachments`

* `id` (pk)
* `task_id` (fk -> task_drafts.id)
* `workspace_id` (fk)
* `path`
* `display_name`
* `file_type`
* `file_hash` (nullable)
* `created_at`

### `prompt_templates`

* `id` (pk)
* `command`
* `template_text`
* `is_active`
* `created_at`

### `user_preferences`

* `id` (pk)
* `workspace_id` (nullable fk)
* `key`
* `value_json`
* `updated_at`

---

## 8.2 Relations

* `workspaces` 1 → many `document_sessions`
* `workspaces` 1 → many `ai_command_runs`
* `ai_command_runs` 1 → many `change_proposals` (or 1 → 1 for MVP if you want simplicity)
* `workspaces` 1 → many `change_proposals`
* `workspaces` 1 → many `task_drafts`
* `task_drafts` 1 → many `task_attachments`

---

## 9. Frontend screen architecture

## 9.1 App shell

```text
┌──────────────────────────────────────────────────────────────────────┐
│ Top bar: workspace / branch / mode / save state / command access    │
├───────────────┬──────────────────────────────────────────────────────┤
│ File tree     │ Tabs + editor / Markdown preview                    │
│ Git tree      │ Monaco / CodeMirror                                 │
│ status icons  │ contextual actions / proposal review                │
│ drag source   │ drag target for file objects                        │
├───────────────┴──────────────────────────────────────────────────────┤
│ optional bottom review / diff drawer / task composer                │
└──────────────────────────────────────────────────────────────────────┘
```

### Theme and visual rules

* dark theme by default across shell, tree, editor, controls, and review states;
* contrast must be sufficient for long writing/editing sessions;
* proposal review and file object chips must remain visually distinct from normal document content.

## 9.2 Key UI components

### Left panel

* `WorkspaceTree`
* `GitStatusBadge`
* `FileTreeNode`
* `DragPreviewGhost`

### Center

* `EditorTabs`
* `DocumentEditor`
* `MarkdownPreview`
* `EditorModeSwitch`
* `InlineDiffOverlay` or `ReviewDrawer`
* `SaveIndicator`
* `ContextActionBar`

### Bottom / modal surfaces

* `ProposalReviewDrawer`
* `TaskComposer`
* `CommandPalette`
* `ExportToFileModal`

### Explicitly excluded from MVP UX

* persistent right-side AI assistant panel;
* floating AI command bar above the editor canvas;
* separate AI chat workspace disconnected from the active file.

## 9.3 Markdown interaction modes

The editor must support:

* **Edit mode** — direct text editing of Markdown;
* **Preview mode** — rendered Markdown reading/viewing state.

Optional split-view can be considered later, but is **not required** unless promoted into scope.

## 9.4 Drag-and-drop file objects

Users can drag files from the workspace tree into:

1. **The editor**

   * dropping a file inserts a structured file reference object;
   * the object may render as a smart chip, embed block, or markdown reference token depending on file type and target context;
   * for MVP text/Markdown focus, insertion may resolve to a normalized markdown reference syntax plus metadata.

2. **Task composer / task input**

   * dropping a file attaches it as a task object rather than pasting raw text;
   * the task stores a structured reference to the workspace file;
   * backend can later resolve these attached objects as AI context.

A file object should minimally include:

* workspace id;
* relative path;
* display name;
* file type;
* optional content hash.

---

## 10. Suggested frontend structure

```text
apps/web/src/
├── app/
│   ├── providers/
│   ├── router/
│   └── store/
├── pages/
│   └── workspace-page/
├── widgets/
│   ├── app-shell/
│   ├── file-tree-panel/
│   ├── editor-workspace/
│   ├── command-palette/
│   ├── task-composer/
│   └── proposal-review-drawer/
├── features/
│   ├── open-workspace/
│   ├── open-document/
│   ├── save-document/
│   ├── switch-editor-mode/
│   ├── run-ai-command/
│   ├── apply-proposal/
│   ├── reject-proposal/
│   ├── export-proposal/
│   ├── attach-file-object/
│   └── insert-file-object-into-editor/
├── entities/
│   ├── workspace/
│   ├── document/
│   ├── proposal/
│   ├── git/
│   ├── file-object/
│   └── task/
└── shared/
    ├── api/
    ├── lib/
    ├── ui/
    └── config/
```

Recommended stack:

* React + TypeScript
* Vite
* Zustand or Redux Toolkit for local editor/workspace state
* TanStack Query for server state
* Monaco Editor for IDE-like UX
* React Router
* shadcn/ui or minimal custom component system
* Markdown renderer for preview mode
* dnd-kit or React DnD for workspace-to-editor/task drag-and-drop

Additional frontend state concerns:

* editor mode state (`edit` / `preview`);
* drag payload state for file object insertion;
* task attachment state based on workspace file references;
* command palette state instead of assistant panel state.

---

## 11. Use-case mapping to layers

### UC01 Open workspace

Frontend:

* workspace picker
* tree render

Backend:

* workspace validation
* tree indexing
* git detection

### UC02 Open file in tab

Frontend:

* open in tab
* activate editor

Backend:

* read file content
* detect language

### UC03 Edit and save file

Frontend:

* dirty state
* save shortcut/button

Backend:

* optimistic save with base hash check

### UC04 Run AI command on selected text

Frontend:

* collect active file + selection + instruction
* show loading state

Backend:

* build prompt
* call model
* create proposal
* compute diff

### UC05 Accept proposal

Frontend:

* preview diff
* user clicks accept

Backend:

* validate baseline
* write file
* mark proposal accepted

### UC06 Reject proposal

Frontend:

* remove pending proposal from review state

Backend:

* mark proposal rejected

### UC07 Export proposal to new file

Frontend:

* input target filename

Backend:

* create new file
* mark proposal exported

### UC08 Switch Markdown mode

Frontend:

* toggle between edit and preview mode
* preserve active tab and document state

Backend:

* none required

### UC09 Drag workspace file into editor

Frontend:

* drag file node from workspace tree
* drop into editor and insert file object representation

Backend:

* optional validation/normalization of inserted file reference object

### UC10 Drag workspace file into task

Frontend:

* drag file node into task composer
* render attached file object chip/card

Backend:

* accept structured file reference in task/AI context payloads

---

## 12. Development backlog by epics

## Epic A. Workspace shell

1. Bootstrap monorepo
2. Create React app shell layout
3. Implement workspace open flow
4. Implement file tree API + UI
5. Detect Git repo and show branch/status
6. Apply default dark theme tokens and shell styling

## Epic B. Editor core

7. Integrate Monaco
8. Open file in tabs
9. Dirty state and save flow
10. Basic markdown/text language support
11. Markdown preview mode
12. Editor mode switch (edit / preview)
13. Drag-and-drop target support inside editor

## Epic C. AI actions surface

14. Command palette UI
15. Contextual document actions
16. Selection-aware command dispatch
17. Task composer with file object attachments
18. Remove dependency on right-side assistant panel UX

## Epic D. AI orchestration

19. AI command endpoint
20. Prompt builder
21. LLM adapter
22. Proposal generation and diff creation
23. Run status/error handling
24. File object references as AI context inputs

## Epic E. Proposal review

25. Proposal review drawer/panel
26. Accept proposal flow
27. Reject proposal flow
28. Export proposal to new file flow

## Epic F. Local persistence

29. SQLite schema + migrations
30. Store workspaces
31. Store AI runs and proposals
32. Persist preferences
33. Persist task attachments / object references if tasks are stored locally

## Epic G. Git layer

34. Git status endpoint
35. Git diff endpoint
36. File badges and modified indicators in UI

## Epic H. Optional Telegram control plane

37. Bot skeleton
38. Notification hooks for completed AI jobs
39. Limited commands: run preset, fetch status, open link back to app

---

## 13. Sprint slicing

## Sprint 0 — foundation

* monorepo bootstrap
* shared types
* workspace API skeleton
* React shell layout
* SQLite setup
* dark theme base tokens

## Sprint 1 — usable editor

* open workspace
* file tree
* open tabs
* Monaco integration
* save file
* markdown preview mode
* edit / preview switch
* drag file into editor

## Sprint 2 — first AI loop

* command palette / contextual actions
* run command on selection
* store proposal
* show diff
* accept/reject/export
* task composer with attached file objects

## Sprint 3 — operational polish

* git status
* error states
* preference persistence
* dark-theme refinement
* basic tests

---

## 14. Tests by layer

### Backend

* open workspace with valid path
* reject unsafe path traversal
* read text file
* save file with matching base hash
* reject save on stale hash
* create proposal from AI run
* apply proposal to target file
* export proposal to new file
* reject proposal state transition
* git status retrieval
* resolve file object from workspace path
* accept task draft with attached file objects

### Frontend

* tree renders workspace items
* file click opens editor tab
* dirty state appears after edit
* save triggers document API
* markdown preview renders current document
* mode switch toggles between edit and preview
* dragging a file into the editor inserts a file object/reference
* dragging a file into a task attaches it as an object chip/card
* AI command sends active file + selection + attached object context
* proposal review panel renders diff
* accept calls apply endpoint and refreshes editor
* reject clears review state
* export opens filename flow and creates tab/file
* no persistent right-side assistant panel is rendered in MVP layout

---

## 15. Main architectural decisions to freeze now

1. **Primary interface = web app, not Telegram.**
2. **Backend owns orchestration and proposal lifecycle.**
3. **Filesystem/Git is the source of truth for file contents.**
4. **SQLite stores metadata, not canonical document source.**
5. **AI output is always a proposal first.**
6. **Text/Markdown only in MVP.**
7. **Dark theme is the default application theme.**
8. **Markdown must support both edit and preview modes.**
9. **There is no persistent AI assistant panel in MVP.**
10. **Workspace files can be dragged into the editor and tasks as structured objects.**
11. **Telegram is optional control plane only.**

---

## 16. What should be produced next

1. Low-fidelity wireframes for:

   * workspace shell
   * editor in dark theme
   * Markdown edit mode
   * Markdown preview mode
   * command palette flow
   * proposal review state
   * task composer with attached file objects
   * export-to-new-file modal
2. OpenAPI-style API contract for each endpoint
3. ERD diagram from the MVP tables above
4. Delivery backlog with task IDs, owners, estimates, and dependencies

---

## 17. Implementation recommendation

For this product, treat the former instruction:

* **Bot = UI layer**
* **Backend = logic layer**

as replaced by:

* **React Web = UI layer**
* **Backend = logic/orchestration layer**
* **Telegram = optional control plane**

This preserves the original architectural discipline while making the product fit the actual UX requirements of an IDE-like AI editor.
