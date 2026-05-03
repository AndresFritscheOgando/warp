# Plan Title Rename — Tech Spec

See [`PRODUCT.md`](./PRODUCT.md) for user-facing behavior.

## Context

Plan titles live in `AIDocument::title: String`
(`app/src/ai/document/ai_document_model.rs:119`). The header bar is
rendered in `AIDocumentView::render_plan_header`
(`app/src/ai/ai_document_view.rs:715`), which reads the title from
`AIDocumentModel`, then delegates to `render_pane_header_title_text` for
the static text label.

Relevant existing infrastructure:

- `AIDocumentModel::update_title` (`ai_document_model.rs:771`) — sets
  `doc.title` and emits `DocumentUpdated { source: User }`, but does **not**
  call `enqueue_save` or update the cloud.
- `AIDocumentModel::enqueue_save` (`ai_document_model.rs:977`) — marks the
  document dirty and sends it through the throttled save channel, which
  ultimately calls `persist_content_to_sqlite` (saves title + content to
  SQLite) and `maybe_update_cloud_notebook_data` (updates cloud content).
- `UpdateManager::update_notebook_title` (`update_manager.rs:2001`) — sends
  a title-only patch to Warp Drive for cloud-synced plans.
- `AIDocumentAction` enum (`ai_document_view.rs:106`) — typed actions
  handled by `AIDocumentView::handle_action`.
- Tab/pane rename pattern (`workspace/view.rs`) — inline `EditorView`
  created once and reused; focus is given to it when rename starts; events
  `Enter`, `Blurred`, and `Escape` drive commit/cancel.

Streaming guard: `AIDocumentModel::is_document_creation_streaming` and the
`is_earlier_version` flag already gate `is_read_only` for the content
editor; the same conditions should gate the rename affordance.

## Proposed changes

### 1. `AIDocumentModel` — add `rename_document_title`

Add a public method `rename_document_title` that wraps the existing
`update_title` and adds the two persistence steps that are missing from it:

```rust
pub fn rename_document_title(
    &mut self,
    id: &AIDocumentId,
    new_title: impl Into<String>,
    ctx: &mut ModelContext<Self>,
) {
    let title = new_title.into();
    self.update_title(id, &title, AIDocumentUpdateSource::User, ctx);
    self.enqueue_save(id);  // triggers SQLite + content cloud sync
    // title-specific cloud update if synced
    self.maybe_update_cloud_notebook_title(id, title, ctx);
}
```

Add a private `maybe_update_cloud_notebook_title` method alongside the
existing `maybe_update_cloud_notebook_data` (`ai_document_model.rs:1028`).
It reads `doc.sync_id`, and if set, calls
`UpdateManager::update_notebook_title` with the new title.

`update_title` itself does not need to change — it stays as the low-level
setter used by agent streaming paths that handle their own persistence.

### 2. `AIDocumentView` — rename editor and state

Add to `AIDocumentView`:

```rust
title_rename_editor: ViewHandle<EditorView>,
is_renaming_title: bool,
title_hover_state: MouseStateHandle,
```

Create the editor in `AIDocumentView::new` mirroring how workspace creates
`tab_rename_editor` (`workspace/view.rs:1252`):

```rust
fn build_title_rename_editor(ctx: &mut ViewContext<Self>) -> ViewHandle<EditorView> {
    let editor = ctx.add_typed_action_view(|ctx| {
        let appearance = Appearance::as_ref(ctx);
        let options = SingleLineEditorOptions {
            text: TextOptions::ui_text(Some(appearance.ui_font_size()), appearance),
            select_all_on_focus: true,
            ..Default::default()
        };
        EditorView::single_line(options, ctx)
    });
    ctx.subscribe_to_view(&editor, |me, _, event, ctx| {
        me.handle_title_rename_editor_event(event, ctx);
    });
    editor
}
```

Handle `EditorEvent::Enter | Blurred → commit_title_rename` and
`EditorEvent::Escape → cancel_title_rename`:

- **`start_title_rename`**: populate the editor with the current title
  (`editor.insert_selected_text`), set `is_renaming_title = true`,
  `ctx.focus(&self.title_rename_editor)`, `ctx.notify()`.
- **`commit_title_rename`**: read `editor.buffer_text(ctx)`, trim it; if
  non-empty call `AIDocumentModel::rename_document_title`; if empty, do
  nothing (preserve the previous title). Set `is_renaming_title = false`,
  `ctx.notify()`.  Send `TelemetryEvent::PlanTitleRenamed`.
- **`cancel_title_rename`**: set `is_renaming_title = false`, `ctx.notify()`.

Add `AIDocumentAction::RenameTitle` variant and handle it in
`handle_action` by calling `start_title_rename`.

### 3. `render_plan_header` — conditional title element

Change the center argument of `render_three_column_header` from the static
`render_pane_header_title_text(…)` to a helper
`render_title_area(…)` that:

- Returns the static text label when not renaming and not hovered.
- Returns the static text label **plus** a pencil icon (using `Icon::Edit`
  or equivalent) when hovered and not renaming. Wrap in a `Hoverable` +
  `MouseStateHandle` for the hover detection. The title text itself should
  be `Clickable` to dispatch `AIDocumentAction::RenameTitle`.
- Returns `ChildView::new(&self.title_rename_editor)` (wrapped in a
  `ConstrainedBox` matching the header height) when `is_renaming_title`.
- Hides the affordance entirely when streaming or viewing an earlier
  version (checked the same way `is_read_only` is set in `refresh`).

### 4. Telemetry

Add a `PlanTitleRenamed` event to `TelemetryEvent` in
`app/src/server/telemetry/events.rs`, following the pattern of
`TabRenamed(TabRenameEvent)`. A single event variant without sub-types is
sufficient for now — we just need to know a rename happened.

## Testing and validation

**Unit test** (add to `ai_document_model_tests.rs`):

- Call `rename_document_title` and assert `get_current_document` returns the
  new title (Behavior §14).
- Verify `DocumentUpdated { source: User }` is emitted.
- Verify `enqueue_save` is triggered (check `content_dirty_flags`).
- Verify that a subsequent `apply_streamed_agent_update` does **not** change
  a user-renamed title — this requires making `rename_document_title` set a
  `user_title_override: bool` flag on `AIDocument`, and having
  `apply_streamed_agent_update` skip the title update when the flag is set
  (Behavior §15).

> **Note on Behavior §15:** Protecting a user-set title from agent overwrites
> requires a small model change: add `user_title_locked: bool` to
> `AIDocument` (default `false`). `rename_document_title` sets it `true`;
> `apply_streamed_agent_update` skips `doc.title = …` when it is `true`.

**Manual validation checklist:**

| Behavior | Step |
|----------|------|
| §1–2  | Open a plan, hover the title — pencil icon appears; click title → input appears pre-populated |
| §3    | Trigger rename while plan is streaming — affordance is absent |
| §4    | View an earlier version — affordance is absent |
| §9    | Type a new title, press Enter — header and plan picker show new title |
| §10   | Enter edit mode, press Escape — original title is unchanged |
| §11   | Enter edit mode, click elsewhere — title commits |
| §12   | Clear the input, press Enter — previous title is preserved |
| §14   | Restart app — custom title survives |
| §15   | Set a custom title; trigger an agent re-plan — agent update does not overwrite the title |
| §16   | Rename a Warp Drive–synced plan — Warp Drive reflects new title |
| §20   | Enter a very long title — header clips it; plan picker shows full title |

## Risks and mitigations

- **Cloud title sync race**: if two clients rename the same plan simultaneously, `update_notebook_title` will use the standard optimistic-update / conflict-resolution path already in `UpdateManager`. No special handling needed.
- **`select_all_on_focus` regression**: `EditorView` with `select_all_on_focus: true` should make all existing text selected on focus entry. Verify that pressing a letter immediately replaces the old title rather than appending to it.
