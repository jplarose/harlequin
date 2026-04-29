# Keymap Refactor and Sequential Chords Design

## Goal

Implement the Textual keymap refactor and a minimal sequential key binding system in one pull request. The PR should keep a clear commit boundary between the Textual keymap refactor and the sequential binding feature so the branch can be rolled back to the Phase 1 stopping point if Phase 2 needs more work.

## Phase 1: Textual Keymap Refactor

Harlequin will keep its existing public keymap configuration format:

```toml
[[keymaps.my_keys]]
keys = "ctrl+shift+q"
action = "quit"
key_display = "^Q"
```

Internally, Harlequin will translate the active Harlequin keymaps into Textual's built-in keymap shape, where binding IDs map to key strings. Existing action names should become stable binding IDs where possible, such as `quit`, `code_editor.run_query`, and `results_viewer.cursor_down`.

The refactor will move normal single-key bindings onto Textual's binding/keymap machinery. Harlequin should still discover plugin keymaps and user-defined keymaps, preserve `keymap_name` ordering, and retain the existing fallback that binds `quit` to `ctrl+q` if no active keymap defines a quit binding.

`key_display` is Harlequin-specific and should remain supported through a small compatibility path. It should not block the move to Textual keymaps, but the implementation must preserve footer/help behavior for existing keymaps that define custom display text.

The Phase 1 completion commit should be explicitly named so the branch can be reset to it. Suggested commit message:

```text
Refactor keymaps onto Textual binding IDs
```

## Phase 2: Sequential Key Bindings MVP

Harlequin will support two-key sequential bindings in existing keymap config. A sequential binding is a whitespace-separated key sequence:

```toml
[[keymaps.my_keys]]
keys = "ctrl+k q"
action = "quit"
```

Comma-separated alternatives remain aliases:

```toml
[[keymaps.my_keys]]
keys = "ctrl+k q,ctrl+k ctrl+q"
action = "quit"
```

The MVP supports exactly two key presses. The first key is the triggering key and must be modified. Valid triggers include `ctrl+k`, `alt+x`, `shift+alt+x`, and `ctrl+shift+k`. Invalid triggers include `k`, `space`, and `enter`. The second key may be modified or unmodified.

After the triggering key is pressed, Harlequin waits up to 3 seconds for the second key. If the second key matches a configured sequence in the current scope, Harlequin dispatches that action. Pressing `escape` cancels the pending sequence. If the timer expires, Harlequin drops the pending sequence without dispatching an action. If the second key does not match any pending sequence, Harlequin cancels the pending sequence and does not dispatch the sequence action.

The MVP does not include leader key expansion, recursive sequences, mode-specific sequence maps, or sequences longer than two key presses.

The Phase 2 completion commit should be explicitly named. Suggested commit message:

```text
Add two-key sequential key bindings
```

## Architecture

Phase 1 should define all normal Harlequin bindings as Textual bindings with stable IDs. Harlequin keymaps then become configuration overlays applied through Textual's keymap API.

Phase 2 should parse sequential bindings before applying the Textual keymap. Single-key bindings continue through Textual. Sequential bindings are stored in a small Harlequin-owned resolver that listens for key events, tracks a pending trigger key, starts a 3 second timer, and dispatches the resolved Harlequin action when the second key matches.

Sequence dispatch should use the same action registry and scoping semantics as normal Harlequin key bindings. This avoids introducing a second action system.

## Validation

Config loading should validate:

- each binding action exists in Harlequin's action registry;
- single-key bindings have Textual-compatible key strings;
- sequential bindings contain exactly two whitespace-separated keys;
- sequential trigger keys include at least one supported modifier;
- sequential bindings do not use empty keys or empty sequence parts.

Invalid config should fail with a `HarlequinConfigError` that names the keymap and binding.

## UX

Existing user keymap configs should continue to work. The Keys App should continue to load and write keymaps, including sequential bindings as plain `keys` strings. Footer/help display should not claim simultaneous chords; sequential keys should be displayed as sequences.

No leader key syntax is included in this PR.

## Testing

Tests should cover:

- existing default `vscode` bindings still work;
- existing user config keymaps still work;
- unknown actions fail as config errors;
- invalid sequential trigger keys fail validation;
- a valid sequence such as `ctrl+k q` dispatches its action;
- `escape` cancels a pending sequence;
- timeout drops a pending sequence after 3 seconds;
- comma-separated sequential aliases work;
- normal single-key bindings continue to work when sequential bindings are configured.
