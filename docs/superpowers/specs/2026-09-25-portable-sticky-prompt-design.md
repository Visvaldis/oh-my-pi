# Portable Sticky Prompt Design

## Goal

Provide a sticky user-prompt header in terminals that do not implement sticky presentation for OSC 133, including Ghostty. Preserve the existing terminal-native implementation for VS Code and other supporting terminals. Do not infer support from terminal identity because terminals do not advertise sticky-scroll capability and multiplexers obscure the outer terminal.

## User-facing setting

Replace the boolean `tui.stickyPrompt` setting with an enum containing three explicit presentation modes:

- `off`: preserve the existing self-contained OSC 133 lifecycle and native terminal scrollback.
- `terminal`: group prompt and response with OSC 133 and leave sticky presentation to the terminal.
- `viewport`: keep transcript navigation inside OMP and paint a fixed, non-clickable prompt header.

The default remains functionally off. Appearance → Display describes that `terminal` preserves native scrollback and depends on terminal support, while `viewport` works portably by replacing native scrollback navigation with OMP-owned scrolling.

Changing modes applies immediately. OMP performs one display reset and transcript reconstruction so retained rows and active components all use one presentation model.

## Renderer modes

### Off

`UserMessageComponent` keeps the current self-contained byte envelope:

`A → visible prompt → B → C → D;0`

No later assistant or tool output belongs to the prompt's command zone.

### Terminal-native

`UserMessageComponent` keeps the implemented grouped envelope:

`D;0 → A → B → visible prompt → C → response output`

The next grouped prompt emits `D;0` to close the preceding response. Every `B` has a `C` in the same component render, preserving Ghostty's input-semantic invariant.

### OMP viewport

The viewport mode uses the same semantic prompt/response association in OMP's transcript model. It does not rely on terminal-native sticky presentation. OSC markers may remain self-contained because OMP paints the header itself; terminal-native grouping is not needed to implement the visual behavior.

## OMP-owned transcript viewport

In viewport mode, ordinary frames do not retire transcript rows into native terminal history. The frame provider instead allocates a bounded transcript region between a fixed prompt header and existing composer/status chrome.

The transcript container remains the source of semantic block order and component identity. It gains a viewport projection API that:

1. Renders only the logical row range required for the current viewport.
2. Reports block spans and the initiating user-message identity for the first visible response row.
3. Maintains a logical vertical offset from the transcript tail plus the previously measured total row count.
4. While follow mode is suspended, increases that offset by newly appended row count so the selected historical window stays fixed; width changes reflow and clamp it.
5. Follows the live tail while the offset is zero.

Committed entries remain available through the existing transcript ledger. The viewport path must avoid a full-history render on every frame: cache per-entry row counts by width and walk only enough entries to fill the requested window. Mutable/live entries invalidate their cached geometry; settled historical entries reuse it. This preserves responsiveness for long sessions.

Viewport mode adds an internal `archived` transcript-entry state distinct from terminal-`committed`. Finalized ordered prefixes move to `archived` without producing a `HistoryBatch`; this removes them from live-capacity accounting while retaining their components and semantic rows for OMP projection. Before graceful shutdown, the container converts archived entries back into a flushable finalized prefix and the existing acknowledged history path emits the complete transcript exactly once. Switching presentation modes reconstructs a fresh container after clearing display, so archived and terminal-committed state never mix across modes.

The fixed header is separate viewport chrome, not a duplicate transcript block. It renders the complete initiating prompt up to a bounded header-height policy; excess rows are clipped with an explicit continuation indicator. Header height is included before transcript allocation so the total frame remains bounded. When the first visible row has no response-initiating prompt, no sticky header is painted.

## Prompt association

Association is semantic, not inferred from rendered text or OSC bytes:

- A non-synthetic `UserMessageComponent` begins a response turn, matching the current OSC-grouping call path.
- Following assistant reasoning, prose, and tool activity belong to that turn until the next such user component.
- Developer and collapsed synthetic messages do not begin turns.
- Live-steered user bubbles retain the current behavior of the shared user-message construction path rather than introducing a fallback-only classification.

The transcript container records the initiating prompt component or stable turn identity on entries when they are added. Reconstructed/resumed transcripts derive the same association through the shared transcript builder.

## Input and navigation

Viewport mode keeps the composer focused for normal typing.

- Mouse wheel: scroll transcript by the existing wheel step.
- Page Up / Page Down: move by one transcript viewport page.
- Home: move to the oldest available transcript row.
- End: return to the live tail and resume follow mode.

Scrolling upward sets a nonzero tail-relative offset and suspends automatic following. As new rows arrive, OMP increases the offset by the same amount so the selected historical window does not move. Returning to offset zero resumes live following.

Mouse events are consumed only when they map to transcript scrolling. Prompt headers are display-only and have no click target. Existing overlays and fullscreen views retain input priority over transcript navigation.

## Native scrollback and lifecycle

OMP viewport mode deliberately replaces native terminal scrollback while active. Since ordinary transcript rows are not retired, the terminal cannot independently scroll to older OMP output. OMP owns navigation and can therefore know which response is visible and which prompt to pin.

On graceful shutdown or switching away from viewport mode, OMP flushes the semantic transcript once into native history before returning control to the shell or entering a native mode. The transaction must not duplicate rows already emitted before viewport mode was enabled; mode switching begins with a display reset and fresh retirement ledger.

Fullscreen overlays continue using the alternate buffer. Resizes repaint the OMP-owned viewport directly and do not replay native history while viewport mode is active.

## Error and compatibility behavior

- No terminal allowlist or sticky-support auto-detection.
- Unsupported OSC behavior cannot expose raw control bytes; viewport presentation is plain TUI rendering.
- If the terminal is too short for header, transcript, composer, and status chrome, the header yields space first and may collapse to one truncated row.
- Existing terminal-native and off modes retain native scrollback semantics.
- Ghostty's `B → C` safety invariant remains covered in both OSC-emitting modes.

## Testing

Permanent tests cover consumer-visible behavior:

1. Setting values select off, terminal-native, and OMP viewport behavior.
2. Existing off and terminal-native OSC byte contracts remain exact.
3. Viewport projection maps visible response rows to the correct initiating prompt across two or more turns.
4. Scrolling across a turn boundary switches the fixed header.
5. A visible prompt is not duplicated as sticky chrome until its transcript copy leaves the viewport.
6. Upward scrolling suspends follow; End restores live following.
7. New output preserves a historical tail-relative selection.
8. Resize clamps/reflows the projection without duplicate or missing semantic rows.
9. Synthetic/developer messages do not become sticky prompt headers.
10. Mode changes trigger one transcript rebuild/display reset.
11. Long-history projection renders only the required entry range rather than the entire transcript.
12. More than 256 finalized viewport entries archive without blocking admission, then flush exactly once on shutdown.

Smoke verification runs the built CLI in Ghostty using OMP viewport mode and in VS Code using terminal-native mode. Ghostty must show a non-clickable prompt header while OMP scrolls with the mouse wheel/Page keys; VS Code must retain its native sticky-scroll behavior.

## Non-goals

- Detecting whether a terminal implements sticky scroll.
- Making the fallback header clickable.
- Painting over terminal-owned historical scrollback.
- Replacing terminal-native mode where it already provides a better integration.
- Forking behavior by Ghostty, VS Code, tmux, SSH, or platform identity.
