# Portable Sticky Prompt Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an explicit OMP-owned transcript viewport that paints a non-clickable sticky prompt header in terminals such as Ghostty while retaining the existing terminal-native OSC 133 mode.

**Architecture:** `tui.stickyPrompt` becomes `off | terminal | viewport`. Terminal mode keeps the existing OSC grouping; viewport mode suppresses ordinary transcript retirement, projects a scrollable semantic transcript window inside `Composer`, and derives sticky prompt chrome from turn ownership stored by `TranscriptContainer`. A tail-relative cursor plus prior row count preserves a historical window as new output arrives.

**Tech Stack:** TypeScript, Bun test, pi-tui frame provider, OSC 133, SGR mouse input, OMP settings registry.

---

## File map

- `packages/tui/src/chat/user-message.ts`: mark response-initiating user bubbles and render marker-free sticky-header rows.
- `packages/tui/src/chrome/transcript-container.ts`: retain per-entry turn ownership and project a cached scrollable transcript range.
- `packages/tui/src/prompt/composer.ts`: own viewport cursor, sticky chrome, frame allocation, navigation, and shutdown flush behavior.
- `packages/coding-agent/src/modes/settings.ts`: expose the explicit presentation enum.
- `packages/coding-agent/src/modes/utils/ui-helpers.ts`: select terminal OSC grouping only for `terminal` mode.
- `packages/coding-agent/src/modes/interactive-mode.ts`: feed the setting into `Composer`, enable wheel reporting in viewport mode, and rebuild once on mode changes.
- `packages/coding-agent/src/modes/controllers/input-controller.ts`: route wheel/Page/Home/End to the transcript viewport before editor handling.
- `packages/tui/test/transcript-container.test.ts`: prove projection, turn boundaries, anchoring, and bounded rendering.
- `packages/tui/test/composer-sticky-viewport.test.ts`: prove frame composition, no ordinary history, sticky-header switching, and flush behavior.
- `packages/coding-agent/test/modes/components/user-message-keywords.test.ts`: preserve OSC byte contracts and registry-to-renderer wiring.
- `packages/coding-agent/test/input-controller-keybindings.test.ts`: prove keyboard and mouse routing precedence.
- `packages/coding-agent/CHANGELOG.md`: document the portable presentation mode.

### Task 1: Replace the boolean setting with explicit presentation modes

**Files:**
- Modify: `packages/coding-agent/src/modes/settings.ts`
- Modify: `packages/coding-agent/src/modes/utils/ui-helpers.ts`
- Modify: `packages/coding-agent/test/modes/components/user-message-keywords.test.ts`

- [ ] **Step 1: Update the focused integration test first**

Change the existing settings wiring test to set `terminal`, and add a `viewport` assertion proving it does not emit the open OSC response envelope:

```ts
it("selects terminal-native OSC grouping from the sticky prompt presentation", () => {
	cfgTuiStickyPrompt.set(Settings.instance, "terminal");
	const raw = renderUserThroughUiHelpers("configured prompt");
	expect(raw.indexOf("\x1b]133;D;0\x07")).toBeLessThan(raw.indexOf("\x1b]133;A\x07"));
	expect(raw.endsWith("\x1b]133;C\x07")).toBe(true);
});

it("keeps viewport presentation out of the terminal-native OSC command zone", () => {
	cfgTuiStickyPrompt.set(Settings.instance, "viewport");
	const raw = renderUserThroughUiHelpers("configured prompt");
	expect(raw.startsWith("\x1b]133;A\x07")).toBe(true);
	expect(raw.endsWith("\x1b]133;B\x07\x1b]133;C\x07\x1b]133;D;0\x07")).toBe(true);
});
```

Extract `renderUserThroughUiHelpers(text)` from the existing integration setup so both tests exercise the real registry and construction path. Change `afterEach` to restore `"off"`.

- [ ] **Step 2: Run the test and verify the enum contract fails**

Run:

```bash
bun test packages/coding-agent/test/modes/components/user-message-keywords.test.ts
```

Expected: type/runtime failure because `cfgTuiStickyPrompt` still accepts a boolean.

- [ ] **Step 3: Register the enum and map only terminal mode to OSC grouping**

Use this registry shape in `settings.ts`:

```ts
export const cfgTuiStickyPrompt = register({
	id: "tui.stickyPrompt",
	type: "enum",
	values: ["off", "terminal", "viewport"] as const,
	default: "off",
	ui: {
		tab: "appearance",
		group: "Display",
		label: "Sticky Prompt Headers",
		description:
			"Keep the initiating prompt visible while reading its response. Terminal-native uses OSC 133 and native scrollback but requires terminal support; OMP viewport works in terminals such as Ghostty by using OMP-owned scrolling.",
		options: [
			{ value: "off", label: "Off", description: "Use native scrollback without sticky prompt grouping" },
			{
				value: "terminal",
				label: "Terminal-native",
				description: "Use OSC 133 grouping and let a supporting terminal pin the prompt",
			},
			{
				value: "viewport",
				label: "OMP viewport",
				description: "Use OMP-owned transcript scrolling and a portable non-clickable prompt header",
			},
		],
	},
});
```

In `UiHelpers.addMessageToChat`, map the setting exactly:

```ts
semanticResponseGrouping: cfgTuiStickyPrompt.get(this.ctx.settings) === "terminal",
```

Do not add terminal detection or migrate unknown values in rendering code; registry validation owns persisted-value fallback.

- [ ] **Step 4: Run the focused test**

Run the same Bun test. Expected: all tests pass, including exact off/terminal OSC sequences and viewport's self-contained sequence.

- [ ] **Step 5: Commit**

```bash
git add packages/coding-agent/src/modes/settings.ts \
  packages/coding-agent/src/modes/utils/ui-helpers.ts \
  packages/coding-agent/test/modes/components/user-message-keywords.test.ts
git commit -m "feat(settings): add sticky prompt presentation modes"
```

### Task 2: Give user bubbles a semantic sticky-header surface

**Files:**
- Modify: `packages/tui/src/chat/user-message.ts`
- Modify: `packages/coding-agent/test/modes/components/user-message-keywords.test.ts`

- [ ] **Step 1: Add failing marker-free header tests**

Add tests that construct a normal and synthetic user bubble:

```ts
it("renders bounded sticky-header rows without OSC markers", () => {
	const component = new UserMessageComponent("first line\nsecond line");
	const rows = component.renderStickyPrompt(80, 2);
	expect(component.initiatesResponseTurn).toBe(true);
	expect(rows).toHaveLength(2);
	expect(Bun.stripANSI(rows.join("\n"))).toContain("first line");
	expect(rows.join("\n")).not.toContain("\x1b]133;");
});

it("does not mark synthetic user bubbles as response initiators", () => {
	const component = new UserMessageComponent("agent disclosure", { synthetic: true });
	expect(component.initiatesResponseTurn).toBe(false);
});
```

- [ ] **Step 2: Run the test and observe missing members**

Run the focused user-message test. Expected: failure because `initiatesResponseTurn` and `renderStickyPrompt` do not exist.

- [ ] **Step 3: Implement the semantic surface without changing normal render memoization**

Expose the marker on the class:

```ts
readonly initiatesResponseTurn: boolean;
```

Immediately after the constructor's existing `ensureThemeSync()` call, assign:

```ts
this.initiatesResponseTurn = options.synthetic !== true;
```

Extract the existing badge-adjusted source-row logic into a private method used by both render paths. `renderStickyPrompt` must never add OSC bytes and must return at most `maxRows`:

```ts
renderStickyPrompt(width: number, maxRows: number): readonly string[] {
	const limit = Math.max(0, Math.trunc(maxRows));
	if (limit === 0) return [];
	const lines = this.#bubbleRows(width);
	if (lines.length <= limit) return lines;
	const clipped = lines.slice(0, limit);
	const source = clipped[limit - 1] ?? "";
	const marker = theme.fgOnBg("dim", "userMessageBg", "…");
	clipped[limit - 1] = `${truncateToWidth(source, Math.max(0, width - 1))}${marker}`;
	return clipped;
}
```

Use ANSI-aware truncation from the existing TUI utilities rather than string slicing. Preserve the existing `#zoneSource/#zoneLines` cache for ordinary `render()`.

- [ ] **Step 4: Run focused tests**

Expected: all user-message tests pass and disabled rendering remains byte-identical.

- [ ] **Step 5: Commit**

```bash
git add packages/tui/src/chat/user-message.ts \
  packages/coding-agent/test/modes/components/user-message-keywords.test.ts
git commit -m "feat(tui): expose sticky prompt header rows"
```

### Task 3: Add virtualized semantic transcript projection

**Files:**
- Modify: `packages/tui/src/chrome/transcript-container.ts`
- Modify: `packages/tui/test/transcript-container.test.ts`

- [ ] **Step 1: Add failing projection tests**

Use the file's existing `Block` fixture plus real `UserMessageComponent` prompts. Add these complete helpers and tests:

Update imports before adding the fixtures:

```ts
import { UserMessageComponent } from "@oh-my-pi/pi-tui/chat/user-message";
import {
	TranscriptContainer,
	type ScrollableTranscriptProjection,
	type TranscriptStableRow,
	type TranscriptViewportCursor,
} from "@oh-my-pi/pi-tui/chrome/transcript-container";
```

```ts
class MutableRows implements Component {
	constructor(readonly rows: string[]) {}
	append(row: string): void {
		this.rows.push(row);
	}
	render(): readonly string[] {
		return this.rows;
	}
}

function viewportCursor(offsetFromTail = 0, measuredRows = 0): TranscriptViewportCursor {
	return { offsetFromTail, measuredRows, width: 80 };
}

function buildTwoTurnTranscript(): {
	transcript: TranscriptContainer;
	firstPrompt: UserMessageComponent;
	secondPrompt: UserMessageComponent;
} {
	const transcript = new TranscriptContainer();
	const firstPrompt = new UserMessageComponent("first prompt");
	const secondPrompt = new UserMessageComponent("second prompt");
	transcript.addChild(firstPrompt);
	transcript.addChild(new Block(["first response 1", "first response 2", "first response 3"], true));
	transcript.addChild(secondPrompt);
	transcript.addChild(new Block(["second response 1", "second response 2", "second response 3"], true));
	return { transcript, firstPrompt, secondPrompt };
}

function projectionForPrompt(
	transcript: TranscriptContainer,
	prompt: UserMessageComponent,
): ScrollableTranscriptProjection {
	const tail = transcript.renderScrollableViewport(80, 2, frame, viewportCursor());
	for (let offset = 1; offset <= tail.maxOffset; offset++) {
		const projection = transcript.renderScrollableViewport(
			80,
			2,
			frame,
			viewportCursor(offset, tail.cursor.measuredRows),
		);
		if (projection.prompt === prompt && !projection.promptVisible) return projection;
	}
	throw new Error("Expected a response-only window owned by the requested prompt");
}

it("projects a historical response with its initiating prompt", () => {
	const { transcript, firstPrompt } = buildTwoTurnTranscript();
	const projection = projectionForPrompt(transcript, firstPrompt);
	expect(Bun.stripANSI(projection.prompt?.renderStickyPrompt(80, 1).join("\n") ?? "")).toContain(
		"first prompt",
	);
	expect(projection.rows).toHaveLength(2);
});

it("switches prompt ownership across the next response boundary", () => {
	const { transcript, firstPrompt, secondPrompt } = buildTwoTurnTranscript();
	expect(projectionForPrompt(transcript, firstPrompt).prompt).toBe(firstPrompt);
	expect(projectionForPrompt(transcript, secondPrompt).prompt).toBe(secondPrompt);
});

it("increments a suspended offset when output grows", () => {
	const transcript = new TranscriptContainer();
	const prompt = new UserMessageComponent("prompt");
	const stream = new MutableRows(["old 1", "old 2", "old 3", "old 4"]);
	transcript.addChild(prompt);
	transcript.addChild(stream);
	const tail = transcript.renderScrollableViewport(80, 2, frame, viewportCursor());
	const before = transcript.renderScrollableViewport(80, 2, frame, viewportCursor(1, tail.cursor.measuredRows));
	stream.append("new row");
	const after = transcript.renderScrollableViewport(80, 2, frame, before.cursor);
	expect(after.cursor.offsetFromTail).toBe(before.cursor.offsetFromTail + 1);
	expect(after.rows).toEqual(before.rows);
});
```

Also assert that a prompt component present in `rows` is returned as `promptVisible: true`, and instrument old committed blocks so a second nearby projection does not rerender entries outside the requested range.

Add lifecycle tests that append 300 finalized entries, call `archiveFinalizedForViewport`, and prove `canAdmit` remains true for a nonzero viewport. Then call `releaseViewportArchiveForFlush`, drain `peekFlushBatch` with acknowledgements, and assert every fixture row appears exactly once.

- [ ] **Step 2: Run the transcript-container suite and confirm RED**

```bash
bun test packages/tui/test/transcript-container.test.ts
```

Expected: failures for the missing projection API.

- [ ] **Step 3: Add projection types and entry metadata**

Add these exported contracts:

```ts
export interface TranscriptViewportCursor {
	readonly offsetFromTail: number;
	readonly measuredRows: number;
	readonly width: number;
}

export interface ScrollableTranscriptProjection {
	readonly rows: readonly string[];
	readonly spans: readonly TranscriptViewportSpan[];
	readonly cursor: TranscriptViewportCursor;
	readonly maxOffset: number;
	readonly prompt?: UserMessageComponent;
	readonly promptVisible: boolean;
}
```

Extend `TranscriptEntry` with:

```ts
turnPrompt?: UserMessageComponent;
viewportRowsByWidth: Map<number, readonly string[]>;
```

During `#syncEntries`, recompute turn ownership in transcript order. A component starts a turn only when it is a `UserMessageComponent` with `initiatesResponseTurn === true`; later entries inherit that prompt until the next initiator.

Add an internal `archived` member to `BlockState`. Implement two explicit lifecycle methods:

```ts
archiveFinalizedForViewport(): void;
releaseViewportArchiveForFlush(): void;
```

`archiveFinalizedForViewport` settles and advances the ordered frontier across finalized entries by marking them `archived`, without creating or acknowledging a `HistoryBatch`. Archived entries remain in `#entries`, remain renderable by the projection API, cannot be removed, and are excluded from `#liveCount()`/admission pressure. `releaseViewportArchiveForFlush` changes archived entries back to `settled`, resets the frontier to the first non-committed entry, and clears stale offers so the existing `peekFlushBatch`/acknowledgement path emits every row once.

- [ ] **Step 4: Implement lazy row geometry and projection**

Add `renderScrollableViewport(width, rows, frame, cursor)`. Its invariants:

1. Clamp width and capacity.
2. Walk entries from the tail, using `viewportRowsByWidth` for settled/archived/committed entries and rerendering active entries.
3. Include one separator row between nonempty blocks.
4. Compute `totalRows`, then update suspended offset by `max(0, totalRows - cursor.measuredRows)` only when `offsetFromTail > 0` and width is unchanged.
5. Clamp offset to `maxOffset = max(0, totalRows - capacity)`.
6. Slice exactly `[totalRows - capacity - offset, totalRows - offset)` without materializing unrelated cached row arrays into a combined full transcript.
7. Produce spans in output coordinates and derive `prompt` from the first owned visible response row.
8. Set `promptVisible` when the initiating prompt's own component appears in the projected spans.

Clear `viewportRowsByWidth` alongside existing render caches in `resetStableEmission`; active entries never reuse a prior frame's cached rows. Bound settled cache widths to the same small width epoch policy used by existing render caches.

- [ ] **Step 5: Run the transcript-container suite**

Expected: projection tests and all existing retirement tests pass.

- [ ] **Step 6: Commit**

```bash
git add packages/tui/src/chrome/transcript-container.ts packages/tui/test/transcript-container.test.ts
git commit -m "feat(tui): project scrollable semantic transcripts"
```

### Task 4: Compose the OMP-owned viewport and sticky header

**Files:**
- Modify: `packages/tui/src/prompt/composer.ts`
- Create: `packages/tui/test/composer-sticky-viewport.test.ts`

- [ ] **Step 1: Add frame-level failing tests**

Build a `Composer` with an in-memory terminal, `stickyPrompt: "viewport"`, and mounted runtime children. Assert:

```ts
expect(frame.history).toBeUndefined();
expect(Bun.stripANSI(frame.viewport[0] ?? "")).toContain("first prompt");
```

Then scroll across a turn boundary and assert the first row changes to the second prompt. Add tests that:

- no header is painted while the prompt component itself is visible;
- a terminal-native composer still offers history under pressure;
- viewport mode follows new tail output at offset zero;
- scrolling up and appending output preserves the same visible historical rows;
- `beginHistoryFlush()` bypasses viewport suppression and offers each finalized row exactly once.

- [ ] **Step 2: Run the new test and confirm RED**

```bash
bun test packages/tui/test/composer-sticky-viewport.test.ts
```

Expected: missing `stickyPrompt` preference and navigation methods.

- [ ] **Step 3: Add the shared presentation type and composer state**

In `composer.ts` export:

```ts
export type StickyPromptPresentation = "off" | "terminal" | "viewport";
```

Add `stickyPrompt: StickyPromptPresentation` to `ComposerPreferences` with default `"off"`. Add state:

```ts
#transcriptCursor: TranscriptViewportCursor = { offsetFromTail: 0, measuredRows: 0, width: 0 };
#transcriptViewportRows = 0;
#transcriptMaxOffset = 0;
```

Reset the cursor when entering or leaving viewport mode in `setPreferences`.

- [ ] **Step 4: Add navigation methods**

Expose methods used by the coding-agent input controller:

```ts
get stickyPromptPresentation(): StickyPromptPresentation {
	return this.#preferences.stickyPrompt;
}

scrollTranscriptRows(delta: number): boolean;
scrollTranscriptPage(direction: -1 | 1): boolean;
scrollTranscriptToStart(): boolean;
scrollTranscriptToEnd(): boolean;
```

Positive row deltas move toward newer content; negative deltas move older. Clamp against `#transcriptMaxOffset`. Any nonzero offset suspends follow; `scrollTranscriptToEnd` restores offset zero.

- [ ] **Step 5: Add the viewport frame branch**

When `stickyPrompt === "viewport"` and `#historyFlush === false`:

1. Call `transcript.archiveFinalizedForViewport()` so finalized prefixes leave live-capacity accounting without entering terminal history.
2. Do not call `#offerHistory`.
3. Render fixed before/after chrome.
4. Project transcript rows for available capacity.
5. If projection has a prompt not already visible, render at most `min(4, max(1, floor(availableRows / 3)))` sticky rows.
6. Reproject using capacity reduced by the actual header height; repeat prompt selection once if the first visible turn changes at the boundary.
7. Paint sticky rows before transcript rows and keep existing click spans shifted by the header height.
8. Save the returned cursor, viewport row count, and max offset.

The viewport branch must preserve existing `renderFrame` behavior for `off` and `terminal` modes. When history flushing begins, call `transcript.releaseViewportArchiveForFlush()` once and deliberately take the existing native history path so graceful shutdown leaves a complete transcript for the shell.

- [ ] **Step 6: Handle resize and mode changes**

`renderResizeFrame` uses the same OMP projection while viewport mode is active instead of borrowing/replaying native history. A width change resets `measuredRows` and lets projection clamp the offset; it must not emit history. `setPreferences` requests one render after resetting cursor state.

- [ ] **Step 7: Run composer and transcript tests**

```bash
bun test packages/tui/test/composer-sticky-viewport.test.ts packages/tui/test/transcript-container.test.ts
```

Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git add packages/tui/src/prompt/composer.ts packages/tui/test/composer-sticky-viewport.test.ts
git commit -m "feat(tui): render portable sticky prompt viewport"
```

### Task 5: Route keyboard and mouse navigation from interactive mode

**Files:**
- Modify: `packages/coding-agent/src/modes/interactive-mode.ts`
- Modify: `packages/coding-agent/src/modes/controllers/input-controller.ts`
- Modify: `packages/coding-agent/test/input-controller-keybindings.test.ts`

- [ ] **Step 1: Add failing input-routing tests**

Extend the input harness with composer spies:

```ts
const scrollTranscriptRows = vi.fn(() => true);
const scrollTranscriptPage = vi.fn(() => true);
const scrollTranscriptToStart = vi.fn(() => true);
const scrollTranscriptToEnd = vi.fn(() => true);
```

Set `stickyPromptPresentation: "viewport"`, dispatch Page Up/Down, Home, End, and an SGR wheel report. Assert each is consumed and invokes only its matching method. Add negative cases for `terminal` mode, active overlays, and non-editor focus.

- [ ] **Step 2: Run the input test and confirm RED**

```bash
bun test packages/coding-agent/test/input-controller-keybindings.test.ts
```

Expected: navigation reaches the editor or existing mouse path instead of transcript methods.

- [ ] **Step 3: Feed the preference into Composer**

Add the mode to `#liveComposerPreferences()`:

```ts
stickyPrompt: cfgTuiStickyPrompt.get(this.settings),
```

Update the inline mouse tracking provider so viewport mode receives wheel reports even when clickable transcript mouse support is off:

```ts
const viewportMouse = cfgTuiStickyPrompt.get(this.settings) === "viewport";
const on = viewportMouse || this.#isMouseCaptureEnabled();
```

Keep hover-band cleanup tied to clickable mouse capture, not viewport wheel capture.

- [ ] **Step 4: Install keyboard routing before editor handling**

In `InputController.setupKeyHandlers`, add one listener that defers to overlays and non-editor focus, then maps:

```ts
if (matchesKey(data, "pageUp")) handled = this.ctx.composer.scrollTranscriptPage(-1);
else if (matchesKey(data, "pageDown")) handled = this.ctx.composer.scrollTranscriptPage(1);
else if (matchesKey(data, "home")) handled = this.ctx.composer.scrollTranscriptToStart();
else if (matchesKey(data, "end")) handled = this.ctx.composer.scrollTranscriptToEnd();
```

Return `{ consume: true }` only when viewport mode is active and the composer reports handling the action.

- [ ] **Step 5: Route wheel before clickable-mouse gating**

In `#handleInlineMouse`, parse SGR input first. If viewport mode is active and `event.wheel !== null`, call:

```ts
this.ctx.composer.scrollTranscriptRows(event.wheel * 3);
this.ctx.ui.requestRender();
return { consume: true };
```

Only after that branch apply `cfgTuiMouse` to hover/click behavior. This keeps the fallback non-clickable when `tui.mouse` is off while still supporting wheel scrolling.

- [ ] **Step 6: Run the input tests**

Expected: viewport navigation is consumed, overlays retain priority, and existing click-to-focus tests remain unchanged.

- [ ] **Step 7: Commit**

```bash
git add packages/coding-agent/src/modes/interactive-mode.ts \
  packages/coding-agent/src/modes/controllers/input-controller.ts \
  packages/coding-agent/test/input-controller-keybindings.test.ts
git commit -m "feat(coding-agent): route sticky viewport navigation"
```

### Task 6: Apply presentation changes live and preserve shutdown/resume behavior

**Files:**
- Modify: `packages/coding-agent/src/modes/interactive-mode.ts`
- Modify: `packages/coding-agent/test/issue-2372-repro.test.ts`
- Modify: `packages/tui/test/composer-sticky-viewport.test.ts`

- [ ] **Step 1: Add failing live-change coverage**

Add an interactive-mode regression test that changes `tui.stickyPrompt` from `off` to `viewport` while an optimistic user message exists. Assert one `rebuildChatFromMessages()` call, one `ui.resetDisplay()` call, and preservation of the optimistic message.

Add composer shutdown coverage that starts in viewport mode, renders enough finalized transcript to exceed the screen, initiates flush, acknowledges offered history, and asserts every semantic row appears exactly once.

- [ ] **Step 2: Run both focused tests and confirm RED where behavior is missing**

```bash
bun test packages/coding-agent/test/issue-2372-repro.test.ts packages/tui/test/composer-sticky-viewport.test.ts
```

- [ ] **Step 3: Finalize live setting behavior**

Keep `"tui.stickyPrompt": cfgTuiStickyPrompt` in `cfgLiveUiSettings`. On change:

```ts
if (any("tui.stickyPrompt")) {
	this.composer.setPreferences(this.#liveComposerPreferences());
	rebuildChat = true;
}
```

Reuse the existing final `rebuildChatFromMessages()` and `ui.resetDisplay()` transaction. Do not add a second reset. The rebuild creates a fresh `TranscriptContainer`, preventing rows retired under the old mode from mixing with viewport state.

- [ ] **Step 4: Verify focused lifecycle tests**

Expected: optimistic/live components survive the rebuild, mode changes redraw once, and shutdown flushes one complete transcript.

- [ ] **Step 5: Commit**

```bash
git add packages/coding-agent/src/modes/interactive-mode.ts \
  packages/coding-agent/test/issue-2372-repro.test.ts \
  packages/tui/test/composer-sticky-viewport.test.ts
git commit -m "fix(coding-agent): rebuild sticky viewport safely"
```

### Task 7: Documentation and end-to-end verification

**Files:**
- Modify: `packages/coding-agent/CHANGELOG.md`

- [ ] **Step 1: Add the Unreleased changelog entry**

Under `## [Unreleased]` → `### Added`, add:

```md
- Added portable sticky prompt headers with an OMP-owned transcript viewport for terminals without native OSC 133 sticky-scroll presentation, alongside the existing terminal-native mode.
```

- [ ] **Step 2: Run focused suites**

```bash
bun test packages/coding-agent/test/modes/components/user-message-keywords.test.ts \
  packages/tui/test/transcript-container.test.ts \
  packages/tui/test/composer-sticky-viewport.test.ts \
  packages/coding-agent/test/input-controller-keybindings.test.ts \
  packages/coding-agent/test/issue-2372-repro.test.ts
```

Expected: all pass with zero failures.

- [ ] **Step 3: Run repository checks**

```bash
bun run check:ts
```

Expected: oxlint, oxfmt, and every package TypeScript check pass.

- [ ] **Step 4: Build the CLI**

```bash
bun --cwd=packages/coding-agent run build
```

Expected: `packages/coding-agent/dist/omp` is produced.

- [ ] **Step 5: Smoke the built CLI in Ghostty**

Run:

```bash
packages/coding-agent/dist/omp
```

Select Settings → Appearance → Display → Sticky Prompt Headers → OMP viewport. Submit two prompts that each request at least 120 plain-text lines. Verify:

- wheel and Page Up/Down scroll inside OMP;
- Home reaches the oldest transcript and End returns to live follow;
- the non-clickable header changes from the first prompt to the second at the response boundary;
- the prompt is not duplicated while its original bubble remains visible;
- new streaming output does not move a historical selection;
- exiting leaves one readable transcript in native terminal history.

- [ ] **Step 6: Regression-smoke terminal-native mode in VS Code**

Select Terminal-native and enable `terminal.integrated.stickyScroll.enabled`. Verify native scrollback and VS Code sticky presentation still work and OMP does not intercept Page/Home/End.

- [ ] **Step 7: Commit documentation**

```bash
git add packages/coding-agent/CHANGELOG.md
git commit -m "docs: document portable sticky prompts"
```
