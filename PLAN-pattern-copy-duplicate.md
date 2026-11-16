# Plan: pattern copy, paste and duplicate in orb-ui

This file is the brief for the coding agent that builds the feature. It is a working file of the
branch `feature/pattern-copy-duplicate` and must not be part of the pull request (step 9 removes
it).

- Base: `upstream/main` at `4e3d22b` (2026-11-16). File and line references below are of that
  commit.
- Remotes: `upstream` is the maintainer's repository, `origin` is the contributor's mirror. Push
  to `origin` only.
- The work is two slices. This plan specifies slice 1. Slice 2 (drag and drop) is outlined at
  the end and is not built now.

## 1. Why

- In pattern mode a musician builds a song from pattern variations: bass drum in pattern 1,
  bass drum plus snare in pattern 2, plus hi-hats in pattern 3.
- Today every variation is clicked in again from zero. orb-ui has no copy, paste or duplicate
  for patterns.
- The maintainer reduced the menu on 2026-10-26 (docs/ui_spec.md section 18) and left the
  original's Edit pattern functions out of scope (section 12). This contribution proposes to
  add pattern commands anyway, as orb's own extension. The pull request says so and leaves the
  choice to the maintainer.

## 2. Before writing code

- Read `AGENTS.md` (standing rules, commit format, build and test commands).
- Do not create or change anything under `backlog/`. An outside contributor has no task; the
  maintainer adds one at merge, as for pull requests #4 and #11.
- Read these in full before planning the edits: `crates/orb-ui/src/link.rs`,
  `crates/orb-ui/src/menu.rs`, `crates/orb-ui/src/shortcuts.rs`,
  `crates/orb-ui/src/key_help.rs`, and in `crates/orb-ui/src/app.rs` the function `show`
  (the block at lines 821-847) and `control` (line 1103).
- Read docs/ui_spec.md sections 4, 10, 11, 16 and 18, and docs/song_load.md section 4.10.
- Read the merged commit `8bd0dcc` (the knob double click, pull request #11) as the model for
  the size of a contribution, its comments, its tests, its spec entry and its commit message.
- Record the baseline: run `cargo test` on the untouched branch and note the counts, so the
  result can state how many tests are new.

## 3. What slice 1 does

A new **Edit** menu sits between File and Mods. Every command acts on the selected pattern of
the row that has the focus (`Panel::focus`, the lit focus bar).

- Undo (Ctrl+Z, Cmd+Z on macOS). The item names what it undoes, for example "Undo Paste
  Pattern". It is disabled when there is nothing to undo.
- Cut Pattern (Ctrl+X, Cmd+X).
- Copy Pattern (Ctrl+C, Cmd+C).
- Paste Pattern (Ctrl+V, Cmd+V).
- Duplicate Pattern (Ctrl+D, Cmd+D).

Words used below:

- A **row** is one of the four pattern sections: TB-303 1, TB-303 2, TR-808, TR-909.
- A **slot** is one of a row's 32 patterns, written as bank and button: A1 to D8.
- A row's **kind** is TB-303, TR-808 or TR-909. The two 303 rows have the same kind.
- A pattern is the bytes the engine holds: shuffle flag, length, 16 steps (98 bytes for a 303,
  194 for a drum machine).

### 3.1 Rules

- **Copy** puts the whole pattern (steps, length, shuffle flag) on the clipboard. Nothing else
  changes and the song is not marked modified.
- **Paste** writes the clipboard's pattern into the selected slot of the focused row: steps,
  length and shuffle flag. The knobs do not change, they belong to the device.
  - It works only into a row of the same kind. A 303 pattern pastes into either 303.
  - With a pattern of another kind, or no pattern, on the clipboard nothing changes and the
    status line says why.
  - A paste that would change nothing sends nothing and does not mark the song modified (the
    rule the knob reset follows).
- **Cut** is Copy, then the slot's steps are emptied. Length and shuffle flag stay, as with the
  303's Clear button.
- **Duplicate** copies the selected pattern into the next empty slot of the same row and then
  selects that slot exactly as a click on its pattern button does.
  - "Next" is the first empty slot after the selected one, on through bank D, then from A1.
  - The shown bank follows the new selection.
  - It does not touch the clipboard.
  - With no empty slot nothing changes and the status line says so.
- **Empty** (decided by the user): all 16 steps are clear, whatever the length and the shuffle
  flag are.
  - Drum machine: all 12 bytes of all 16 steps are 0.
  - 303: every step is in the Clear state, the six bytes `0, 0, 0, 0, 0, 1` (note 0, no flags,
    rest on). That is also what a new song holds (`crates/orb-engine/src/tb303.rs:268`).
- **Undo** is one level and covers Cut, Paste and Duplicate.
  - It puts back the bytes the one changed slot held before.
  - After a Duplicate it also selects the source slot again, as a click does.
  - It is forgotten when a song is opened, on File > New, and when any other pattern edit
    follows (a step, the length, the shuffle flag, the 303's Clear).
  - There is no redo.
- **No dialogs.** A paste onto a used slot overwrites it; Undo brings it back. An "overwrite?"
  dialog is a later idea (section 11).
- **Song mode** (recommended and adopted): no special case. Copy, Cut and Paste change pattern
  data only. Duplicate's selection is a click on a pattern button, so everything a click does
  holds: in pattern mode the row changes at the next bar line; in song mode the selection is
  recorded when record is on, and without record it lasts until the song's next pattern change.
- **Playing**: an edit is heard the next time its step plays, as today. A Duplicate while
  playing switches at the next bar line to a slot that sounds the same.
- **Status line** (`App::status`): each command reports what it did, with the row and the slot
  names, for example "TR-909: A1 duplicated to A2", "TR-909: pasted into A2",
  "TR-909: no empty pattern left", "The clipboard holds a TB-303 pattern". Keep the texts
  short and in the voice of the existing ones ("Saved ...").
- **Without a song** (the demo panel has no link) the commands do nothing and the menu items
  are disabled.
- **The used marker**: a pattern button whose slot is not empty shows a small marker, in both
  its lit and unlit state, so a musician sees which slots are free and where Duplicate lands.

## 4. Facts about the code (checked)

- The pattern arrays: `Link::patterns` (`link.rs:484`) holds four arrays of 32 patterns.
  `layout(row)` (`link.rs:118`) gives the pattern length (98 or 194) and the step length (6 or
  12). Bytes 0 and 1 of a pattern are the shuffle flag and the length.
- A whole pattern goes to the engine as msg 41 sub 44 (303), 61 (808) or 78 (909) with the data
  `{p, bytes}` (docs/song_load.md section 4.10, line 466). In code that is
  `header_sub(d) + 2`; `Link::command` (`link.rs:865`) already turns it into
  `Command::Pattern` for all four rows. The 303's Clear is the existing sender
  (`Event::Clear303`, `link.rs:721`).
- `Player::send` refuses a `Command::Pattern` whose length is not the device's
  (`crates/orb-player/src/lib.rs:587`).
- `Link::fill_views` (`link.rs:626`) fills the panel from the arrays: the 32 lengths, shuffle
  flags and 303 steps of a row when `shown[r]` is `None`, and the drum step strips when the
  selected pattern differs from `shown[r]`.
- `Link::set_mod` (`link.rs:600`) is the model for a command that does not come from a panel
  control: it forwards the panel's pending events first, then sends, then `flush_marked`.
- `Link::forward` (`link.rs:897`) is where messages are logged to `Link::log`. The tests compare
  that log with the original's wire format, so the new whole-pattern messages must be logged
  too.
- A click on a pattern button is `Panel::press` -> `press_pattern` (`model.rs:749`): it
  computes the slot from the shown bank, sets the focus and emits `Event::Pattern`.
- Save writes the patterns from `Link::patterns` (`document.rs:247`). For the 808 a step byte
  is stored as "set or not", so compare engine pattern arrays after a reload, not raw file
  bytes.
- The song is marked modified by `App::note_edits` (`menu.rs:939`) from the panel's events, or
  directly (`self.doc.modified = true`, `menu.rs:882`).
- Menu: `Cmd` (`menu.rs:70`), `App::command` (`menu.rs:890`), `App::menu_bar`
  (`menu.rs:1099`), the helper `item` (`menu.rs:1271`). Shortcut constants are in
  `shortcuts.rs`; Help > Keyboard Shortcuts lists them through `menu_row`
  (`key_help.rs:292`).
- The pattern buttons draw the part `{family}on` or `{family}off` (`Panel::part_name`,
  `model.rs:1073`); the press is handled in `App::control` (`app.rs:1103`).

## 5. Traps

- **Ctrl/Cmd+X, +C and +V never arrive as key events in the real app.** egui-winit 0.37.0
  (`src/lib.rs:1128-1152` in the cargo registry) turns them into `egui::Event::Cut`,
  `Event::Copy` and `Event::Paste(text)` and returns before it pushes the key event. So
  `shortcuts::consume` cannot see them, although a headless test that injects a key event
  would pass. Listen for the three events.
- **`Event::Paste` comes only when the system clipboard holds text** (or an image). Therefore
  Copy and Cut must put a text on the system clipboard (`Context::copy_text`).
  - Recommended design: the pattern bytes stay in the app's memory. The text on the system
    clipboard is a short readable marker of that copy, for example
    "OpenRB-338 pattern: TR-909 A1" plus something that makes the copy unique (another running
    instance must not match it). A paste is accepted only when the pasted text is the marker
    of the pattern in memory.
  - This needs no parsing of foreign bytes. Do not parse pattern bytes from clipboard text in
    this slice: out-of-range bytes would reach the audio thread.
  - If the user copies other text elsewhere in between, Cmd+V pastes nothing and the status
    line says the clipboard holds no pattern. That is correct clipboard behavior.
  - The menu items can use `ViewportCommand::RequestCut`, `RequestCopy` and `RequestPaste`
    (they exist in egui 0.37.0) so that a menu click takes the same path as the shortcut.
    Check how that behaves in the headless harness before relying on it.
- **Ctrl/Cmd+D and +Z are ordinary key events**; `shortcuts::consume` works for them.
- **Text fields and windows own these keys first.** While a text field has the focus (File >
  Info) the clipboard events and Cmd+Z are the field's. The error box takes the events away
  (`error_box.rs:233`). Selected text in a window (About, Mod Info, the error box) must still
  copy as text. The pattern commands act only when the panel takes keys: use the conditions
  `App::show` already has for `self.keyboard(ui)`.
- **Order of messages in a Duplicate**: the pattern bytes first, then the selection, so the
  engine never plays the target before it holds the data.
- **Pending events**: forward the panel's pending events before reading or writing the
  arrays, as `Link::set_mod` does.
- **Pattern 32 on a 303 row** once caused a panic in `fill_views` (task-077). Cover slot D8 on
  every row.
- **The panel must show the result in the next frame**, also with no audio stream (then the
  snapshot never moves and nothing repaints by itself).
- **Nothing about the original without evidence.** The repository records that the original's
  Edit menu has pattern functions (docs/ui_spec.md section 12) but not which ones or what they
  do, and not whether its panel sends sub 61 or 78. Describe orb's behavior only. This was
  the main review finding of pull request #11.

## 6. Units of work

One agent, the units in order: they share `menu.rs`, `app.rs` and `link.rs`, so they cannot run
in parallel. Keep one commit per unit while working; step 9 squashes them.

### Unit 1: the pattern operations in the link

- A way to read a slot's bytes, to test a slot for empty, to find the next empty slot, and to
  write a whole pattern into a slot (arrays updated, whole-pattern message logged and sent,
  panel views refreshed).
- Recommended shape: methods on `Link` modeled on `set_mod`, and pure functions for "empty"
  and "next empty slot" with unit tests. An event through `Panel::events` is the alternative;
  choose after reading and say why in the commit message.
- Done when:
  - writing a pattern into any slot of each of the four rows sends exactly one message
    `{p, bytes}` with sub 44, 61 or 78, and after one engine call the engine's pattern array
    holds the bytes (tests in `tests/wired.rs`, modeled on
    `the_303_step_editor_edits_the_pattern_as_the_original` and
    `the_808_step_buttons_edit_the_selected_instrument_live`);
  - the panel shows the new steps, length and shuffle flag of the selected slot;
  - slot D8 works on every row;
  - the unit tests cover: an empty and a used pattern of each kind, a pattern that differs
    only in length or shuffle flag (still empty), the search skipping used slots, crossing a
    bank border, wrapping from D8, and no empty slot.

### Unit 2: Copy, Cut, Paste and Duplicate as commands

- New `Cmd` variants and their `App` methods, the clipboard state in `App`, the status line
  texts.
- Done when (tests in `tests/wired.rs` and `tests/menu.rs`, driven through `App::command`):
  - copy then paste works on each row, and from one 303 row to the other;
  - a clipboard of another kind, and an empty clipboard, change nothing and send nothing;
  - a paste of identical content sends nothing and leaves the song unmodified;
  - Copy alone leaves the song unmodified; Paste, Cut and Duplicate mark it modified;
  - Cut empties the steps, keeps length and shuffle flag, and its pattern can be pasted;
  - Duplicate sends the pattern message before the selection message, selects the target and
    switches the shown bank; with no empty slot it sends nothing;
  - while playing in pattern mode the engine plays the target from the next bar line (model:
    `a_new_pattern_starts_at_the_bar_line_but_the_panel_shows_it_at_once`);
  - in song mode with record on, a Duplicate's selection lands in the track as a click's does
    (model: `a_double_click_while_recording_is_recorded_as_a_drag_is`);
  - after a paste, Save and a reopen give the same engine patterns (model:
    `an_edit_of_each_kind_is_saved`);
  - on the demo panel (no link) the commands do nothing (`tests/interaction.rs`).

### Unit 3: the Edit menu and the keys

- The Edit menu in `menu_bar`, the shortcut constants in `shortcuts.rs`, the event handling
  in `App::show`, the rows in Help > Keyboard Shortcuts.
- Done when:
  - the menu bar shows Edit between File and Mods with the five items and their shortcut
    texts, and each item runs its command (model: `the_menu_bar_sits_above_the_panel_and_opens`);
  - the items are disabled without a link, Undo also when there is nothing to undo;
  - `Event::Copy`, `Event::Cut` and `Event::Paste(marker)` and the keys Ctrl/Cmd+D and +Z run
    the commands;
  - `Event::Paste` with any other text changes nothing;
  - with a text field focused, and with the error box open, the patterns stay untouched and
    the field still gets the events (models:
    `the_menu_shortcuts_and_escape_work_while_a_field_is_focused`, `tests/error_box.rs:470`);
  - selected text in a window still copies as text;
  - the keys window lists the new shortcuts and its two unit tests still pass.

### Unit 4: Undo

- Done when:
  - Undo after Paste, Cut and Duplicate restores the slot's former bytes with one
    whole-pattern message; after a Duplicate it also selects the source slot;
  - a second Undo does nothing;
  - a step edit, a length or shuffle change, the 303's Clear, opening a song and File > New
    each forget the undo;
  - the menu item's text names the command and it is disabled when nothing can be undone.

### Unit 5: the used marker

- Drawn by code over the pattern button (a small dot is the starting point), not a new art
  part, so the panel art stays untouched. It shows for a used slot in the shown bank, lit or
  not.
- Done when:
  - a test shows the marker state follows the slots (set a step, clear it, paste, cut, change
    the bank, open another song);
  - `tests/frame_cost.rs` still passes;
  - two screenshots (`cargo run --release -p orb-ui -- SONG.rbs --screenshot OUT.png`, one
    with `--size 624x510`, the native size, and one with `--size 1248x1020`) are shown to the
    user, who approves the look before the pull request.
- This unit is separable. Keep it as the last commit so it can become its own pull request if
  the user or the maintainer prefers.

### Unit 6: the documents

- docs/ui_spec.md:
  - section 18: the menu table gets the Edit rows; the sentence that says only song files, the
    mod and the Vintage flag remain of the original's menu is made true again;
  - section 4: one short **orb** bullet for the commands and the used marker, in the pattern
    of the knob class's bullet (section 2);
  - section 16: one bullet on how a paste is wired (the whole-pattern message, the clipboard
    marker);
  - section 10 only if it lists menu shortcuts.
- The module docs (`//!`) of `menu.rs` and `link.rs` where they list what the module does
  ("Nothing else of the original's menu." in `menu.rs` is no longer true).
- Search README.md and docs/ for other lists of menu items or shortcuts and update them.

## 7. House style

Code:

- Lines of Rust are at most 100 columns; format with `cargo +nightly fmt --all`. Long string
  literals are split by hand.
- Follow the surrounding code: names, `Option` handling, casts, `let ... else`, no new
  dependencies, no new abstractions for one use.
- Edit source with the editor tools, not with scripts.
- American spelling ("color").

Comments (the points the review of pull request #11 raised, plus what the code shows):

- A doc comment (`///`) is a short contract in complete sentences, present tense: what the
  item is or does, its ranges (`0..=31`), what `None` means. One to three lines is the norm.
  Example: `link.rs:417-419` (`knob_default`).
- Do not repeat a rationale in several places. Explain it once (in the spec) and point to it:
  `(docs/ui_spec.md section 4)`.
- Longer comments are for real timing or ownership constraints only (`link.rs:17-24`,
  `link.rs:924-927`).
- Inline comments say why, above the line they explain. Short ones are lowercase fragments
  without a period (`// behind what waits in the backlog`); a multi-line one is full
  sentences.
- Link related items with intra-doc links: ``[`Link::patterns`]``.
- A fact about the original carries its address or its document section. State nothing about
  the original that the repository does not record.
- Do not write task ids for this work: it has none.
- Test names are sentences in snake case that state the behavior
  (`a_double_click_on_a_knob_at_its_default_sends_nothing`). Test comments are short and
  exact.

Documents:

- orb's own behavior is labeled **orb** and kept short; implementation detail goes to
  docs/ui_spec.md section 16.

Commit (AGENTS.md "Commit messages"):

- Subject `ui,docs: <what>`, at most 72 characters.
- Body: Change, Result, Testing. No `Task:` line. The attribution line the harness gives
  goes last.
- The branch ends with exactly one commit.

## 8. Gates

All must pass before the review and again after its fixes:

- `cargo test` (state passed, failed, ignored, and how many are new; say that `ORB_CD_ISO`
  and the two third-party variables were not set if so).
- `cargo clippy --all-targets -- -D warnings`
- `cargo +nightly fmt --all --check`
- `git diff upstream/main --stat` lists nothing under `backlog/` and not this plan file.

## 9. Finishing

1. Rebase on a fresh `upstream/main` (`git fetch upstream`); upstream moves several times a
   day. Run the gates again.
2. Remove this plan file and squash to one commit with the message of section 7.
3. Get an independent review of that commit: correctness, and whether the code and comments
   match the repository's style in length and voice. Fold accepted findings into the commit by
   amending, and rerun the gates.
4. Push the branch to `origin` (`--force-with-lease` after the squash).
5. Hand over to the user:
   - what to try in the real app (`cargo run --release -p orb-ui`): each shortcut with the
     real keyboard, the menu items, a paste after copying text in another program, a copy of
     selected text in the About window, a Duplicate while a pattern plays, Undo;
   - the two marker screenshots;
   - a draft of the pull request text.
6. Do not open the pull request against `upstream`. The user opens it after trying the
   feature and approving the text.

Pull request text, in the form of #11, short bullets:

- What it does.
- How.
- For the maintainer to decide:
  - whether pattern commands are wanted, given the reduced menu of 2026-10-26;
  - what the original's Edit pattern functions do (not read here) and whether orb's should
    follow them;
  - the look of the used marker;
  - the shortcuts in the plugin editor (task-201), where the host may take them.
- Validation.

## 10. Stop and ask the user when

- a rule of section 3 turns out to contradict what the engine or the panel does;
- the clipboard design of section 5 does not work in the real app;
- the change grows well past the size of its parts as planned (the link operations, five
  commands, one menu, one marker);
- a gate fails for a reason outside this change.

## 11. Slice 2 (not now): drag and drop

- A drag of a pattern button onto another pattern button of a row of the same kind copies the
  pattern there. Hovering a bank button during the drag switches the shown bank.
- To settle when slice 2 starts:
  - a press selects a pattern today at mouse-down (`App::first_press`); a drag needs a
    movement threshold, as task-203 describes for its drag;
  - task-203 drags a pattern button out of the plugin window as a MIDI clip: a drop on a
    pattern button copies, a drop outside the window exports;
  - a drop on a used slot: overwrite with Undo, or the user's idea of an "overwrite? Yes / No"
    dialog, which would then also apply to Paste;
  - whether a modifier key moves or swaps instead of copying.
- Slice 2 reuses unit 1 (write a pattern into a slot), Undo and the used marker.

## 12. Open questions for the user

- None block slice 1. The used marker's look needs the user's approval on screenshots (unit 5).
