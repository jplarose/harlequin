# Keymap Refactor and Sequential Chords Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move Harlequin's existing keymap config onto Textual binding IDs and add a two-key sequential binding MVP in one PR with clear rollback commits.

**Architecture:** Phase 1 declares stable Textual bindings for Harlequin actions and translates existing Harlequin keymaps into Textual keymap overlays. Phase 2 keeps single-key bindings in Textual and routes two-key sequences through a small Harlequin-owned resolver with a 3 second pending-key timeout.

**Tech Stack:** Python 3.10, Textual 6.4.0, pytest, pytest-asyncio, Harlequin keymap plugins/config.

---

## Commit Boundaries

The PR must contain these meaningful checkpoints:

- Phase 1 completion commit: `Refactor keymaps onto Textual binding IDs`
- Phase 2 completion commit: `Add two-key sequential key bindings`

The Phase 2 commit must be after Phase 1 has passing tests, so the branch can be reset to Phase 1 if sequence support needs to be deferred.

## File Map

- Modify `src/harlequin/actions.py`: keep the Harlequin action registry as the source of action metadata and add helpers that convert actions to Textual `Binding` objects.
- Modify `src/harlequin/keymap.py`: add validation and helpers for splitting alias lists and identifying sequential bindings.
- Modify `src/harlequin/app.py`: replace direct dynamic binding with Textual keymap application in Phase 1; add sequence event handling in Phase 2.
- Modify `src/harlequin/bindings.py`: either remove usage or narrow it to compatibility helpers after direct dynamic binding is removed.
- Modify `src/harlequin/keys_app.py`: keep existing keymap editing compatible with normalized/validated bindings.
- Modify `src/harlequin_vscode/__init__.py`: keep default keymap data in the public Harlequin format.
- Add `src/harlequin/key_sequences.py`: focused parser/state machine for two-key sequential bindings.
- Add/modify `tests/unit_tests/test_keymap.py`: parser and validation tests.
- Modify `tests/functional_tests/test_keymap_from_config.py`: existing config compatibility and sequence behavior.
- Modify `tests/functional_tests/test_keymap_vscode.py`: regression coverage for default bindings after the refactor.

---

## Task 1: Baseline and Validation Tests

**Files:**
- Create: `tests/unit_tests/test_keymap.py`
- Modify: `src/harlequin/keymap.py`

- [ ] **Step 1: Write failing tests for action and sequence validation**

Create `tests/unit_tests/test_keymap.py`:

```python
from __future__ import annotations

import pytest

from harlequin.exception import HarlequinConfigError
from harlequin.keymap import HarlequinKeyMap, split_key_aliases


def test_split_key_aliases_preserves_sequence_parts() -> None:
    assert split_key_aliases("ctrl+k q, ctrl+k ctrl+q") == [
        "ctrl+k q",
        "ctrl+k ctrl+q",
    ]


def test_unknown_action_raises_config_error() -> None:
    with pytest.raises(HarlequinConfigError) as exc_info:
        HarlequinKeyMap.from_config(
            name="bad",
            bindings=[{"keys": "x", "action": "does_not_exist"}],
        )

    assert "does_not_exist" in exc_info.value.msg
    assert "bad" in exc_info.value.msg


def test_unmodified_sequence_trigger_raises_config_error() -> None:
    with pytest.raises(HarlequinConfigError) as exc_info:
        HarlequinKeyMap.from_config(
            name="bad",
            bindings=[{"keys": "k q", "action": "quit"}],
        )

    assert "k q" in exc_info.value.msg
    assert "modified" in exc_info.value.msg


def test_three_key_sequence_raises_config_error() -> None:
    with pytest.raises(HarlequinConfigError) as exc_info:
        HarlequinKeyMap.from_config(
            name="bad",
            bindings=[{"keys": "ctrl+k q x", "action": "quit"}],
        )

    assert "exactly two" in exc_info.value.msg
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run pytest tests/unit_tests/test_keymap.py -q
```

Expected: failures because `split_key_aliases` and validation do not exist yet.

- [ ] **Step 3: Add keymap parsing and validation**

Modify `src/harlequin/keymap.py`:

```python
from harlequin.actions import HARLEQUIN_ACTIONS
```

Add helpers above `HarlequinKeyBinding`:

```python
SUPPORTED_SEQUENCE_MODIFIERS = {"ctrl", "alt", "shift", "meta"}


def split_key_aliases(keys: str) -> list[str]:
    return [key.strip() for key in keys.split(",") if key.strip()]


def is_sequence_key(keys: str) -> bool:
    return " " in keys.strip()


def _sequence_trigger_is_modified(trigger: str) -> bool:
    parts = trigger.split("+")
    if len(parts) < 2:
        return False
    return any(part in SUPPORTED_SEQUENCE_MODIFIERS for part in parts[:-1])


def _validate_binding(name: str, binding: HarlequinKeyBinding) -> None:
    if binding.action not in HARLEQUIN_ACTIONS:
        raise HarlequinConfigError(
            title="Harlequin could not load your keymap.",
            msg=(
                f"Keymap {name!r} defines an unknown action "
                f"{binding.action!r}."
            ),
        )
    for key_alias in split_key_aliases(binding.keys):
        if not is_sequence_key(key_alias):
            continue
        sequence_parts = key_alias.split()
        if len(sequence_parts) != 2:
            raise HarlequinConfigError(
                title="Harlequin could not load your keymap.",
                msg=(
                    f"Keymap {name!r} defines sequence {key_alias!r}, "
                    "but sequential bindings must contain exactly two keys."
                ),
            )
        if not _sequence_trigger_is_modified(sequence_parts[0]):
            raise HarlequinConfigError(
                title="Harlequin could not load your keymap.",
                msg=(
                    f"Keymap {name!r} defines sequence {key_alias!r}, "
                    "but the triggering key must be modified."
                ),
            )
```

Call validation in `HarlequinKeyMap.from_config` after constructing `keymap`:

```python
for binding in keymap.bindings:
    _validate_binding(name=name, binding=binding)
```

Also correct the existing error text from `key_profile` to `key_display`.

- [ ] **Step 4: Run validation tests**

Run:

```bash
uv run pytest tests/unit_tests/test_keymap.py tests/unit_tests/test_config.py -q
```

Expected: all pass.

---

## Task 2: Phase 1 Textual Binding IDs

**Files:**
- Modify: `src/harlequin/actions.py`
- Modify: `src/harlequin/app.py`
- Modify: `tests/functional_tests/test_keymap_vscode.py`
- Modify: `tests/functional_tests/test_keymap_from_config.py`

- [ ] **Step 1: Write regression test for config keymap after Textual overlay**

In `tests/functional_tests/test_keymap_from_config.py`, keep `test_results_viewer_bindings` as the primary regression. Add an assertion after app setup that the active keymap names are preserved:

```python
assert app.keymap_names == profile["keymap_name"]
```

- [ ] **Step 2: Add binding factory helpers**

In `src/harlequin/actions.py`, import Textual's binding type:

```python
from textual.binding import Binding
```

Add helper functions after `HARLEQUIN_ACTIONS`:

```python
def binding_id_for_action(action_name: str) -> str:
    return action_name


def binding_for_action(action_name: str, keys: str) -> Binding:
    action = HARLEQUIN_ACTIONS[action_name]
    return Binding(
        key=keys,
        action=action.action,
        description=action.description or "",
        show=action.show,
        key_display=None,
        priority=action.priority,
        id=binding_id_for_action(action_name),
    )
```

- [ ] **Step 3: Add default Textual bindings on app/widget targets**

In `src/harlequin/app.py`, replace the direct dynamic binding path with helper methods:

```python
def _iter_active_keymap_bindings(self) -> Iterable[HarlequinKeyBinding]:
    for keymap_name in self.keymap_names:
        keymap = self._get_keymap(keymap_name=keymap_name)
        if keymap is None:
            continue
        yield from keymap.bindings


def _resolved_single_keymap(self) -> dict[str, str]:
    resolved: dict[str, str] = {}
    for binding in self._iter_active_keymap_bindings():
        single_keys = [
            key
            for key in split_key_aliases(binding.keys)
            if not is_sequence_key(key)
        ]
        if single_keys:
            resolved[binding.action] = ",".join(single_keys)
    resolved.setdefault("quit", "ctrl+q")
    return resolved
```

Then make `action_bind_keymaps` call `self.set_keymap(self._resolved_single_keymap())`.

Keep `bind_keys` temporarily if needed for targets that do not yet have class-level bindings, but migrate each target to stable `Binding` entries. If target-level class bindings are impractical in one step, create them during widget mount with `id=action_name`, then apply the Textual keymap to override them.

- [ ] **Step 4: Preserve custom display labels**

Add a compatibility map in `Harlequin`:

```python
self.key_display_overrides: dict[str, str] = {}
```

Populate it while resolving active keymaps:

```python
if binding.key_display:
    self.key_display_overrides[binding.action] = binding.key_display
```

Use this map when constructing default `Binding` instances so existing custom footer labels still render.

- [ ] **Step 5: Run Phase 1 regression tests**

Run:

```bash
uv run pytest tests/unit_tests/test_keymap.py tests/unit_tests/test_config.py tests/functional_tests/test_keymap_from_config.py tests/functional_tests/test_keymap_vscode.py -q
```

Expected: all pass.

- [ ] **Step 6: Commit Phase 1**

Run:

```bash
git add src/harlequin/actions.py src/harlequin/app.py src/harlequin/keymap.py tests/unit_tests/test_keymap.py tests/unit_tests/test_config.py tests/functional_tests/test_keymap_from_config.py tests/functional_tests/test_keymap_vscode.py
git commit -m "Refactor keymaps onto Textual binding IDs"
```

This commit is the rollback point requested by the user.

---

## Task 3: Sequential Binding Parser and Store

**Files:**
- Create: `src/harlequin/key_sequences.py`
- Modify: `tests/unit_tests/test_keymap.py`

- [ ] **Step 1: Write unit tests for sequence resolution**

Append to `tests/unit_tests/test_keymap.py`:

```python
from harlequin.key_sequences import KeySequence, SequenceBindings


def test_sequence_bindings_match_second_key() -> None:
    bindings = SequenceBindings(
        {
            "ctrl+k": {
                "q": KeySequence(trigger="ctrl+k", key="q", action="quit"),
            }
        }
    )

    assert bindings.actions_for_trigger("ctrl+k") == ["q"]
    assert bindings.match("ctrl+k", "q").action == "quit"
    assert bindings.match("ctrl+k", "x") is None
```

- [ ] **Step 2: Add sequence store implementation**

Create `src/harlequin/key_sequences.py`:

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass(frozen=True)
class KeySequence:
    trigger: str
    key: str
    action: str


class SequenceBindings:
    def __init__(
        self,
        bindings: dict[str, dict[str, KeySequence]] | None = None,
    ) -> None:
        self._bindings = bindings or {}

    def add(self, sequence: KeySequence) -> None:
        self._bindings.setdefault(sequence.trigger, {})[sequence.key] = sequence

    def actions_for_trigger(self, trigger: str) -> list[str]:
        return list(self._bindings.get(trigger, {}))

    def has_trigger(self, trigger: str) -> bool:
        return trigger in self._bindings

    def match(self, trigger: str, key: str) -> KeySequence | None:
        return self._bindings.get(trigger, {}).get(key)
```

- [ ] **Step 3: Run parser tests**

Run:

```bash
uv run pytest tests/unit_tests/test_keymap.py -q
```

Expected: all pass.

---

## Task 4: Sequential Key Event Handling

**Files:**
- Modify: `src/harlequin/app.py`
- Modify: `src/harlequin/key_sequences.py`
- Modify: `tests/functional_tests/test_keymap_from_config.py`

- [ ] **Step 1: Add functional sequence config fixture**

In `tests/functional_tests/test_keymap_from_config.py`, add a helper keymap in test code by constructing `HarlequinKeyMap` directly:

```python
from harlequin.keymap import HarlequinKeyBinding, HarlequinKeyMap
```

Use this in a new test:

```python
@pytest.mark.asyncio
async def test_sequential_quit_binding(
    duckdb_adapter: type[HarlequinAdapter],
    wait_for_workers: Callable[[Harlequin], Awaitable[None]],
) -> None:
    app = Harlequin(
        duckdb_adapter([":memory:"], no_init=True),
        keymap_names=["sequence_test"],
        user_defined_keymaps=[
            HarlequinKeyMap(
                name="sequence_test",
                bindings=[HarlequinKeyBinding("ctrl+k q", "quit")],
            )
        ],
    )
    async with app.run_test() as pilot:
        await wait_for_workers(app)
        await pilot.press("ctrl+k")
        assert app._pending_key_sequence_trigger == "ctrl+k"
        await pilot.press("q")
        assert app.return_code == 0
```

- [ ] **Step 2: Build sequence bindings from active keymaps**

In `src/harlequin/app.py`, add:

```python
def _resolved_sequence_bindings(self) -> SequenceBindings:
    sequences = SequenceBindings()
    for binding in self._iter_active_keymap_bindings():
        for key in split_key_aliases(binding.keys):
            if not is_sequence_key(key):
                continue
            trigger, second_key = key.split()
            sequences.add(
                KeySequence(
                    trigger=trigger,
                    key=second_key,
                    action=binding.action,
                )
            )
    return sequences
```

Store it when binding keymaps:

```python
self.sequence_bindings = self._resolved_sequence_bindings()
```

- [ ] **Step 3: Add pending sequence state**

In `Harlequin.__init__`, initialize:

```python
self.sequence_bindings = SequenceBindings()
self._pending_key_sequence_trigger: str | None = None
self._pending_key_sequence_timer: Timer | None = None
```

Import `Timer` from Textual's timer module if type checking needs it:

```python
from textual.timer import Timer
```

- [ ] **Step 4: Add event handler**

Add to `src/harlequin/app.py`:

```python
@on(events.Key)
async def handle_key_sequence(self, event: events.Key) -> None:
    key = event.key
    if self._pending_key_sequence_trigger is not None:
        if key == "escape":
            self._clear_pending_key_sequence()
            event.stop()
            return
        sequence = self.sequence_bindings.match(
            self._pending_key_sequence_trigger,
            key,
        )
        self._clear_pending_key_sequence()
        if sequence is not None:
            action = HARLEQUIN_ACTIONS[sequence.action]
            target = self._target_for_action(action)
            if target is not None:
                await target.run_action(action.action)
            event.stop()
        return

    if self.sequence_bindings.has_trigger(key):
        self._pending_key_sequence_trigger = key
        self._pending_key_sequence_timer = self.set_timer(
            3.0,
            self._clear_pending_key_sequence,
        )
        event.stop()
```

Add helpers:

```python
def _clear_pending_key_sequence(self) -> None:
    self._pending_key_sequence_trigger = None
    if self._pending_key_sequence_timer is not None:
        self._pending_key_sequence_timer.stop()
        self._pending_key_sequence_timer = None


def _target_for_action(self, action: Action) -> DOMNode | None:
    if action.target is None:
        return self
    focused = self.focused
    target: DOMNode | None = focused
    while isinstance(target, DOMNode):
        if isinstance(target, action.target):
            return target
        target = target.parent
    matches = self.query(action.target)
    return matches.first() if matches else None
```

Adjust imports for `events`, `Action`, `KeySequence`, and `SequenceBindings`.

- [ ] **Step 5: Run sequence functional test**

Run:

```bash
uv run pytest tests/functional_tests/test_keymap_from_config.py::test_sequential_quit_binding -q
```

Expected: pass.

---

## Task 5: Cancellation, Timeout, and Alias Behavior

**Files:**
- Modify: `tests/functional_tests/test_keymap_from_config.py`
- Modify: `src/harlequin/app.py`

- [ ] **Step 1: Add cancellation and alias tests**

Append tests:

```python
@pytest.mark.asyncio
async def test_escape_cancels_pending_sequence(
    duckdb_adapter: type[HarlequinAdapter],
    wait_for_workers: Callable[[Harlequin], Awaitable[None]],
) -> None:
    app = Harlequin(
        duckdb_adapter([":memory:"], no_init=True),
        keymap_names=["sequence_test"],
        user_defined_keymaps=[
            HarlequinKeyMap(
                name="sequence_test",
                bindings=[HarlequinKeyBinding("ctrl+k q", "quit")],
            )
        ],
    )
    async with app.run_test() as pilot:
        await wait_for_workers(app)
        await pilot.press("ctrl+k")
        await pilot.press("escape")
        assert app._pending_key_sequence_trigger is None
        assert app.return_code is None


@pytest.mark.asyncio
async def test_sequence_alias_dispatches(
    duckdb_adapter: type[HarlequinAdapter],
    wait_for_workers: Callable[[Harlequin], Awaitable[None]],
) -> None:
    app = Harlequin(
        duckdb_adapter([":memory:"], no_init=True),
        keymap_names=["sequence_test"],
        user_defined_keymaps=[
            HarlequinKeyMap(
                name="sequence_test",
                bindings=[HarlequinKeyBinding("ctrl+k x,ctrl+k q", "quit")],
            )
        ],
    )
    async with app.run_test() as pilot:
        await wait_for_workers(app)
        await pilot.press("ctrl+k")
        await pilot.press("q")
        assert app.return_code == 0
```

- [ ] **Step 2: Add timeout test**

Append:

```python
@pytest.mark.asyncio
async def test_sequence_timeout_clears_pending_trigger(
    duckdb_adapter: type[HarlequinAdapter],
    wait_for_workers: Callable[[Harlequin], Awaitable[None]],
) -> None:
    app = Harlequin(
        duckdb_adapter([":memory:"], no_init=True),
        keymap_names=["sequence_test"],
        user_defined_keymaps=[
            HarlequinKeyMap(
                name="sequence_test",
                bindings=[HarlequinKeyBinding("ctrl+k q", "quit")],
            )
        ],
    )
    async with app.run_test() as pilot:
        await wait_for_workers(app)
        await pilot.press("ctrl+k")
        await pilot.pause(3.1)
        assert app._pending_key_sequence_trigger is None
        assert app.return_code is None
```

- [ ] **Step 3: Run sequence behavior tests**

Run:

```bash
uv run pytest tests/functional_tests/test_keymap_from_config.py -q
```

Expected: all pass.

---

## Task 6: Keys App and Full Regression

**Files:**
- Modify: `src/harlequin/keys_app.py`
- Modify: `tests/functional_tests/test_keys_app.py`
- Modify: docs if keymap docs are part of this repository's generated source.

- [ ] **Step 1: Confirm Keys App reads sequence strings without splitting on whitespace**

Run:

```bash
uv run pytest tests/functional_tests/test_keys_app.py -q
```

Expected before changes: pass or reveal formatting assumptions.

- [ ] **Step 2: Preserve sequence strings in display/edit/write flow**

If the Keys App sorts comma-separated key aliases, keep each alias intact:

```python
keys = ",".join(sorted({btn.key.strip() for btn in self.query(EditButton) if btn.key}))
```

Do not split aliases by whitespace in `keys_app.py`.

- [ ] **Step 3: Run targeted tests**

Run:

```bash
uv run pytest tests/unit_tests/test_keymap.py tests/unit_tests/test_config.py tests/functional_tests/test_keys_app.py tests/functional_tests/test_keymap_from_config.py tests/functional_tests/test_keymap_vscode.py -q
```

Expected: all pass.

- [ ] **Step 4: Run broader test suite**

Run:

```bash
uv run pytest tests/unit_tests tests/functional_tests -q
```

Expected: all pass.

- [ ] **Step 5: Commit Phase 2**

Run:

```bash
git add src/harlequin src/harlequin_vscode tests docs/superpowers
git commit -m "Add two-key sequential key bindings"
```

- [ ] **Step 6: Push branch**

Run:

```bash
git push
```

Expected: branch updates on `origin/keymap-refactor-sequential-chords`.

---

## Self-Review Notes

- The plan keeps one PR and two implementation commits.
- Existing user config compatibility is covered by Phase 1 regression tests.
- Sequential bindings are scoped to exactly two key presses and no leader syntax.
- The 3 second timeout and modified trigger validation are explicit.
- No implementation task requires changing Harlequin's public keymap config shape.
