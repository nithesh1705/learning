# IPRU NexGen — Project Specification

**Audience:** This document is written for the coding agent (Copilot) that will implement this project. It is the single source of truth for scope, architecture, and behavior. Read this fully before writing any code. Phase-by-phase build prompts will be issued separately and will always reference back to this document.

---

## 1. What We're Building

**IPRU NexGen** is a new, unified, one-stop AI assistant page for ICICI Prudential AMC employees, replacing the placeholder page currently named **"IPRU Assist New"** inside the existing multi-page `IpruAssist` frontend application.

It combines:
- Multi-model AI chat (Gemini + GPT), carried over and reworked from the existing `IPRU GPT` page.
- **Assists** — routes to existing, separately-hosted backend systems (PMS Assist, PST Assist, etc.) via `/assist-name` triggers.
- **Tools/Features** — small, self-contained Python capabilities (chat-with-doc, create-doc, chat-with-pdf, create-pdf, chat-with-excel, create-excel, translate-document, send-mail) invoked via `@tool-name` triggers, callable by the AI model itself through tool-calling, not just by explicit user selection.

Nothing about any other existing page (Dashboard, IPRU GPT, PMS Assist, PST Assist, Investment Assist, Sales Assist, Compliance Assist, Finance Assist) changes. Only the IPRU NexGen page is built/modified.

---

## 2. Hard Constraints

1. **Never edit, delete, or write inside `Current_IPRUGPT_Backend/`.** It is read-only reference material for table schemas, model config patterns, env/credential handling, and general backend conventions.
2. **All new backend code lives in `IPRU_NexGen/`.** Nothing is copy-pasted from the old backend — everything is written fresh, with a clean folder structure, typing/annotations, and docstrings. The old backend is studied for *patterns*, not reused as code.
3. **No other existing frontend page is touched.** Only the page currently called "IPRU Assist New" (to be renamed "IPRU NexGen") is modified, inside `IpruAssistFrontend/frontend/src/projects/`.
4. Existing assist backends (PMS Assist backend, PST Assist backend, the existing translate-document backend) are **not rewritten** — IPRU NexGen calls them as-is. Their existing frontend integration code is studied as the reference for how to call them correctly.
5. Config-driven, not hardcoded: which assists exist, which tools exist, their trigger keywords, icons, and descriptions must live in config (`assist_config`, `tools_config`) on both frontend and backend, so adding a new assist/tool later doesn't require touching core logic.

---

## 3. High-Level Architecture

```
NewIpruAssist/
├── Current_IPRUGPT_Backend/        # READ-ONLY reference. Do not touch.
├── IpruAssistFrontend/
│   └── frontend/
│       └── src/projects/
│           └── IPRU_NexGen/         # renamed from "IPRU Assist New"
│               ├── components/
│               │   ├── Composer/            # +, input, model dropdown, web search toggle
│               │   ├── TriggerAutocomplete/  # / and @ dropdown, keyboard nav
│               │   ├── ChatWindow/
│               │   ├── ChatBubble/
│               │   ├── DocumentPreviewPane/  # right-side split preview/edit
│               │   └── AssistBanner/         # "You are chatting with PST Assist"
│               ├── config/
│               │   ├── assistConfig.ts
│               │   └── toolsConfig.ts
│               ├── hooks/
│               ├── services/                 # API clients (chat, assist-router, files)
│               ├── types/
│               └── IPRUNexGenPage.tsx
└── IPRU_NexGen/                     # NEW backend, built from scratch
    ├── app/
    │   ├── api/
    │   │   ├── chat.py               # multi-model chat endpoint
    │   │   ├── assist_router.py      # proxies to PMS/PST/etc backends
    │   │   ├── files.py               # upload / preview / download / expiry
    │   │   └── tools/                 # one route module per tool if needed
    │   ├── core/
    │   │   ├── config.py               # env, credentials, urls
    │   │   ├── model_providers/
    │   │   │   ├── gemini_provider.py
    │   │   │   └── gpt_provider.py
    │   │   └── tool_registry.py        # maps @tool-name -> python callable + schema
    │   ├── tools/                       # each tool = one python module
    │   │   ├── chat_with_doc.py
    │   │   ├── create_doc.py
    │   │   ├── chat_with_pdf.py
    │   │   ├── create_pdf.py
    │   │   ├── chat_with_excel.py
    │   │   ├── create_excel.py
    │   │   ├── translate_document.py    # thin wrapper calling existing translate backend
    │   │   └── send_mail.py             # based on testsendmail.py reference
    │   ├── models/                       # DB models (tables), pydantic schemas
    │   ├── services/
    │   │   ├── storage_service.py        # local/GCS storage + 24h expiry cleanup job
    │   │   └── assist_client.py          # HTTP clients to PMS/PST backends
    │   ├── db/
    │   └── main.py
    ├── requirements.txt
    └── .env.example
```

---

## 4. Frontend Behavior

### 4.1 Composer layout
```
        |---------------------------------------------------|
        |     trigger autocomplete dropdown (slides up)      |
        |---------------------------------------------------|
| +  |  |  Input box                                     |  | > |
|----|  |-------------------------------------------------|  |---|
      Model [Dropdown]        Web Search [Toggle]
```
- The **`+` button is for adding files ONLY.** It no longer lists tools/assists (too many now to list nicely there).
- The trigger dropdown is a separate UI element anchored to the composer, sliding in above it, not a menu opened by `+`.

### 4.2 Trigger autocomplete (`/` and `@`)
- Typing `/` anywhere in the input opens a dropdown of **assists** (from `assistConfig`), typing `@` opens a dropdown of **tools/features** (from `toolsConfig`).
- As the user keeps typing after `/` or `@`, the list filters live, case-insensitive, ordered alphabetically by best-match.
- Keyboard support: Up/Down arrow to move selection, `Tab` to autocomplete the currently highlighted match inline (repeated Tab/typing narrows further), `Enter` or click to commit the selection, `Space` after a full match also commits it and returns focus to free typing.
- Assists and tools can be typed **anywhere in the prompt text**, not only at the start.
- On commit of an **assist** trigger (e.g. `/pst-assist`):
  - The literal trigger text is removed from the box (per Phase 1 spec: user shouldn't need to see it sitting inside their message) — instead an **AssistBanner** appears (e.g. "You are chatting with PST Assist") docked to the composer (top or bottom, whichever we implement) for the duration of that assist session.
  - The remaining full prompt text is what gets sent to that assist's backend.
- On commit of a **tool** trigger (e.g. `@create-doc`), the trigger text **stays inline** in the message (tools are meant to be referenced inline, e.g. "Can you `@create-doc` on FastAPI vs REST API").

### 4.3 Assist vs Tool routing logic
- If the message contains an assist trigger → route the entire request to that assist's dedicated backend (via `assist_router.py`), bypassing the general chat model.
- If no assist trigger is present → request goes to the general chat backend (`chat.py`), using the user's selected model provider (Gemini or GPT).
- Within general chat, if a `@tool-name` is present **or** the model itself decides a tool is needed based on user intent (see §4.4 on send-mail), the backend's tool-calling loop invokes the matching Python tool function.
- If neither an assist nor a tool applies: plain multi-model chat, same as current IPRU GPT.

### 4.4 Tool-decision nuance (send-mail example)
- The presence of `@send-mail` is a strong signal, but not the only signal — the model must be able to decide a tool applies purely from user intent even without the literal trigger, and conversely must be able to recognize when the user does *not* want the tool invoked.
- Example: "I need to send mail... @send-mail" → model asks clarifying questions (sender email, recipient email, subject/body if not given), then calls `send_mail.py`.
- Example: "give me the subject and body" (no trigger, wants text only, not an actual send) → model must NOT call the tool, just returns drafted subject/body as text.
- Example: "give me the subject and body @send-mail" → explicit trigger overrides ambiguity → tool IS called (i.e., it composes and actually sends).
- This means tool schemas passed to the model need clear descriptions, and the backend prompt/system instructions must explicitly cover this disambiguation.

### 4.5 Split-pane document view
- Default: chat is a single full-width column.
- The moment a generated file (doc/pdf/excel) is opened for preview/edit, the page splits: **left half = chat, right half = preview/edit pane.**
- Under the AI chat bubble that produced a file: two buttons — **Preview/Edit** and **Download**.
  - Word/Excel-type docs → "Preview/Edit" (editable, e.g. via a rich text/quill-based editor for docs).
  - PDFs → "Preview" only (no inline edit).
- The preview pane header has **Download** and **Close** actions.
- If user edited content and tries to close without saving → confirmation popup: "Changes will not be saved."
- Every generated file auto-expires **24 hours** after creation (deleted from backend storage / GCS). After expiry, any Preview/Download attempt shows: **"Expired – Unable to Preview or Download file."**

---

## 5. Backend Behavior

### 5.1 Multi-model chat
- Supports Gemini and GPT as pluggable providers behind a common interface (`model_providers/`).
- Gemini path supports web search (native grounding/tool), matching the toggle in the composer.
- Model + web-search selection from the frontend is passed through per-request, not hardcoded.

### 5.2 Tool registry
- `tool_registry.py` holds a config-driven map: tool name → Python function + JSON schema (description, parameters) that gets handed to whichever model provider is active, so both Gemini and GPT tool-calling can use the same registry.
- Each tool lives in its own file under `app/tools/`, single responsibility, fully typed, with docstrings.

### 5.3 Assist routing
- `assist_router.py` + `assist_client.py`: thin proxy layer. For each assist in `assist_config` (backend), holds the target base URL, auth/headers, and request/response shape needed to call the existing PMS Assist / PST Assist backends unchanged. Reference the existing PMS/PST frontend code only to learn the exact contract (payload shape, streaming vs non-streaming, auth headers) — do not modify those backends.

### 5.4 File lifecycle & storage
- `storage_service.py` handles saving generated files (local disk or GCS — configurable), generating preview/download URLs, and a scheduled cleanup (24h TTL) that deletes expired files and marks their DB records as expired so the frontend can render the "Expired" state instead of erroring.

### 5.5 Translate-document
- `translate_document.py` is a thin wrapper around the **existing, already-running** translate backend — same GCS URI conventions as the current implementation. This is the one tool that doesn't get built from scratch; it's integrated/re-exposed under the new backend's tool-calling interface.

### 5.6 Send-mail
- `send_mail.py` built from the reference `testsendmail.py` sample. Requires sender email and recipient email before sending; if missing, the model must ask for them conversationally before invoking the tool (see §4.4).

### 5.7 Config & credentials
- `core/config.py` centralizes env vars, model API keys, backend URLs for each assist, GCS bucket/creds — following the same conventions used in `Current_IPRUGPT_Backend` (studied, not copied) so ops/infra patterns stay consistent.

---

## 6. Data Model (indicative — refine per phase)
- `conversations`, `messages` — chat history, tagged with which model/assist/tool was used per turn.
- `generated_files` — id, owner, type (doc/pdf/xlsx), storage path/URI, created_at, expires_at, status (active/expired), edit history if applicable.
- `assist_config` / `tools_config` (backend mirror of frontend config) — name, trigger keyword, display label, description, target URL or handler ref, enabled flag.

---

## 7. Build Phases

| Phase | Scope |
|---|---|
| 1 | Rename "IPRU Assist New" → "IPRU NexGen". Build config-driven `assist_config`/`tools_config` (frontend + backend). Build the `/` and `@` trigger autocomplete UI (keyboard nav, inline Tab-complete, sliding dropdown anchored to composer). Restrict `+` button to file-adding only. Assist banner UI. |
| 2 | Working backend for existing multi-model chat (Gemini + GPT), including Gemini web search — ported from placeholder/non-functional frontend wiring to a real new backend. |
| 3 | `@chat-with-doc`, `@create-doc` tools + split-pane preview/edit (Quill-based editor), download, 24h expiry + "Expired" state. |
| 4 | `@chat-with-pdf`, `@create-pdf` tools + split-pane preview (no edit), download, 24h expiry + "Expired" state. |
| 5 | `@chat-with-excel`, `@create-excel` tools + split-pane preview/edit, download, 24h expiry + "Expired" state. |
| 6 | `@translate-document` — wrap existing running translate backend into the new backend's tool interface, reusing its GCS URI conventions. |
| 7 | `@send-mail` — build from `testsendmail.py` reference; implement the ask-for-recipient/sender flow and the tool-vs-no-tool disambiguation logic. |

Each phase will get its own detailed build prompt referencing the relevant section(s) of this document. Do not start a phase's implementation without the corresponding build prompt — this document sets scope and architecture, not step-by-step instructions.

---

## 8. Open Items To Confirm Before/During Build
- Exact list & shape of tables to reuse/mirror from `Current_IPRUGPT_Backend` (models, env keys, URLs) — to be pulled during Phase 2 backend build by inspecting that folder read-only.
- Which storage backend to standardize on for generated files (local disk vs GCS) for Phases 3–5.
- Exact editor library for the doc preview/edit pane (Quill confirmed for docs; PDFs are preview-only; Excel preview/edit library TBD during Phase 5).
