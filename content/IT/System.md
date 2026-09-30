---
title: "System"
org_id: "72def127-0901-47b3-b8b8-d9d56575ea4a"
---

# System

## Zellij

Default mode is `locked`. Press `Alt a` to enter `normal` mode, then switch to other modes.

### Locked Mode

| Key           | Action               |
|---------------|----------------------|
| `Alt a`       | → Normal mode        |
| `Alt n`       | NewPane              |
| `Alt h`       | MoveFocusOrTab left  |
| `Alt j`       | MoveFocus down       |
| `Alt k`       | MoveFocus up         |
| `Alt l`       | MoveFocusOrTab right |
| `Alt left`    | MoveFocusOrTab left  |
| `Alt down`    | MoveFocus down       |
| `Alt up`      | MoveFocus up         |
| `Alt right`   | MoveFocusOrTab right |
| `Alt f`       | ToggleFloatingPanes  |
| `Alt +`       | Resize Increase      |
| `Alt -`       | Resize Decrease      |
| `Alt =`       | Resize Increase      |
| `Alt [`       | PreviousSwapLayout   |
| `Alt ]`       | NextSwapLayout       |
| `Alt i`       | MoveTab left         |
| `Alt o`       | MoveTab right        |
| `Alt p`       | TogglePaneInGroup    |
| `Alt Shift p` | ToggleGroupMarking   |

### Normal Mode

Enter from locked via `Alt a`.

#### Mode Switching

| Key      | Action         |
|----------|----------------|
| `p`      | → Pane mode    |
| `t`      | → Tab mode     |
| `r`      | → Resize mode  |
| `s`      | → Scroll mode  |
| `m`      | → Move mode    |
| `o`      | → Session mode |
| `enter`  | → Locked mode  |
| `esc`    | → Locked mode  |
| `Ctrl q` | Quit           |

#### Alt Shortcuts (same as Locked)

| Key           | Action               |
|---------------|----------------------|
| `Alt h`       | MoveFocusOrTab left  |
| `Alt j`       | MoveFocus down       |
| `Alt k`       | MoveFocus up         |
| `Alt l`       | MoveFocusOrTab right |
| `Alt left`    | MoveFocusOrTab left  |
| `Alt down`    | MoveFocus down       |
| `Alt up`      | MoveFocus up         |
| `Alt right`   | MoveFocusOrTab right |
| `Alt n`       | NewPane              |
| `Alt f`       | ToggleFloatingPanes  |
| `Alt +`       | Resize Increase      |
| `Alt -`       | Resize Decrease      |
| `Alt =`       | Resize Increase      |
| `Alt [`       | PreviousSwapLayout   |
| `Alt ]`       | NextSwapLayout       |
| `Alt i`       | MoveTab left         |
| `Alt o`       | MoveTab right        |
| `Alt p`       | TogglePaneInGroup    |
| `Alt Shift p` | ToggleGroupMarking   |

### Pane Mode

Enter from normal via `p`.

#### Pane Actions

| Key | Action                             |
|-----|------------------------------------|
| `n` | NewPane → locked                   |
| `d` | NewPane down → locked              |
| `r` | NewPane right → locked             |
| `s` | NewPane stacked → locked           |
| `x` | CloseFocus → locked                |
| `f` | ToggleFocusFullscreen → locked     |
| `e` | TogglePaneEmbedOrFloating → locked |
| `w` | ToggleFloatingPanes → locked       |
| `z` | TogglePaneFrames → locked          |
| `i` | TogglePanePinned → locked          |
| `c` | → RenamePane mode                  |

#### Navigation

| Key     | Action          |
|---------|-----------------|
| `h`     | MoveFocus left  |
| `j`     | MoveFocus down  |
| `k`     | MoveFocus up    |
| `l`     | MoveFocus right |
| `left`  | MoveFocus left  |
| `down`  | MoveFocus down  |
| `up`    | MoveFocus up    |
| `right` | MoveFocus right |
| `tab`   | SwitchFocus     |

#### Mode Switching

| Key      | Action         |
|----------|----------------|
| `p`      | → Normal mode  |
| `t`      | → Tab mode     |
| `m`      | → Move mode    |
| `o`      | → Session mode |
| `enter`  | → Locked mode  |
| `esc`    | → Locked mode  |
| `Ctrl q` | Quit           |

### Tab Mode

Enter from normal via `t`.

#### Tab Actions

| Key   | Action                       |
|-------|------------------------------|
| `n`   | NewTab → locked              |
| `x`   | CloseTab → locked            |
| `r`   | → RenameTab mode             |
| `s`   | ToggleActiveSyncTab → locked |
| `b`   | BreakPane → locked           |
| `[`   | BreakPaneLeft → locked       |
| `]`   | BreakPaneRight → locked      |
| `tab` | ToggleTab                    |
| `1-9` | GoToTab N → locked           |

#### Navigation

| Key     | Action          |
|---------|-----------------|
| `h`     | GoToPreviousTab |
| `j`     | GoToNextTab     |
| `k`     | GoToPreviousTab |
| `l`     | GoToNextTab     |
| `left`  | GoToPreviousTab |
| `down`  | GoToNextTab     |
| `up`    | GoToPreviousTab |
| `right` | GoToNextTab     |

#### Mode Switching

| Key      | Action         |
|----------|----------------|
| `t`      | → Normal mode  |
| `p`      | → Pane mode    |
| `m`      | → Move mode    |
| `o`      | → Session mode |
| `enter`  | → Locked mode  |
| `esc`    | → Locked mode  |
| `Ctrl q` | Quit           |

### Resize Mode

Enter from normal via `r`.

#### Resize Actions

| Key     | Action                |
|---------|-----------------------|
| `h`     | Resize Increase left  |
| `j`     | Resize Increase down  |
| `k`     | Resize Increase up    |
| `l`     | Resize Increase right |
| `H`     | Resize Decrease left  |
| `J`     | Resize Decrease down  |
| `K`     | Resize Decrease up    |
| `L`     | Resize Decrease right |
| `left`  | Resize Increase left  |
| `down`  | Resize Increase down  |
| `up`    | Resize Increase up    |
| `right` | Resize Increase right |
| `+`     | Resize Increase       |
| `-`     | Resize Decrease       |
| `=`     | Resize Increase       |

#### Mode Switching

| Key      | Action         |
|----------|----------------|
| `r`      | → Normal mode  |
| `p`      | → Pane mode    |
| `t`      | → Tab mode     |
| `s`      | → Scroll mode  |
| `m`      | → Move mode    |
| `o`      | → Session mode |
| `enter`  | → Locked mode  |
| `esc`    | → Locked mode  |
| `Ctrl q` | Quit           |

### Move Mode

Enter from normal via `m`.

#### Move Actions

| Key     | Action            |
|---------|-------------------|
| `h`     | MovePane left     |
| `j`     | MovePane down     |
| `k`     | MovePane up       |
| `l`     | MovePane right    |
| `left`  | MovePane left     |
| `down`  | MovePane down     |
| `up`    | MovePane up       |
| `right` | MovePane right    |
| `n`     | MovePane (next)   |
| `p`     | MovePaneBackwards |
| `tab`   | MovePane (next)   |

#### Mode Switching

| Key      | Action         |
|----------|----------------|
| `m`      | → Normal mode  |
| `t`      | → Tab mode     |
| `s`      | → Scroll mode  |
| `o`      | → Session mode |
| `r`      | → Resize mode  |
| `enter`  | → Locked mode  |
| `esc`    | → Locked mode  |
| `Ctrl q` | Quit           |

### Scroll Mode

Enter from normal via `s`.

#### Scroll Actions

| Key        | Action                  |
|------------|-------------------------|
| `j`        | ScrollDown              |
| `k`        | ScrollUp                |
| `down`     | ScrollDown              |
| `up`       | ScrollUp                |
| `d`        | HalfPageScrollDown      |
| `u`        | HalfPageScrollUp        |
| `h`        | PageScrollUp            |
| `l`        | PageScrollDown          |
| `left`     | PageScrollUp            |
| `right`    | PageScrollDown          |
| `PageUp`   | PageScrollUp            |
| `PageDown` | PageScrollDown          |
| `Ctrl b`   | PageScrollUp            |
| `Ctrl f`   | PageScrollDown          |
| `Ctrl c`   | ScrollToBottom → locked |
| `e`        | EditScrollback → locked |
| `f`        | → EnterSearch mode      |

#### Scroll Mode Alt Shortcuts

| Key         | Action                        |
|-------------|-------------------------------|
| `Alt h`     | MoveFocusOrTab left → locked  |
| `Alt j`     | MoveFocus down → locked       |
| `Alt k`     | MoveFocus up → locked         |
| `Alt l`     | MoveFocusOrTab right → locked |
| `Alt left`  | MoveFocusOrTab left → locked  |
| `Alt down`  | MoveFocus down → locked       |
| `Alt up`    | MoveFocus up → locked         |
| `Alt right` | MoveFocusOrTab right → locked |

#### Mode Switching

| Key      | Action         |
|----------|----------------|
| `s`      | → Normal mode  |
| `p`      | → Pane mode    |
| `t`      | → Tab mode     |
| `m`      | → Move mode    |
| `o`      | → Session mode |
| `r`      | → Resize mode  |
| `enter`  | → Locked mode  |
| `esc`    | → Locked mode  |
| `Ctrl q` | Quit           |

### Search Mode

Enter from scroll via `f` → type query → `enter`.

#### Search Actions

| Key | Action                   |
|-----|--------------------------|
| `n` | Search down (next match) |
| `p` | Search up (prev match)   |
| `c` | Toggle CaseSensitivity   |
| `w` | Toggle Wrap              |
| `o` | Toggle WholeWord         |

#### Scroll (shared with Scroll mode)

| Key        | Action                  |
|------------|-------------------------|
| `j`        | ScrollDown              |
| `k`        | ScrollUp                |
| `down`     | ScrollDown              |
| `up`       | ScrollUp                |
| `d`        | HalfPageScrollDown      |
| `u`        | HalfPageScrollUp        |
| `h`        | PageScrollUp            |
| `l`        | PageScrollDown          |
| `left`     | PageScrollUp            |
| `right`    | PageScrollDown          |
| `PageUp`   | PageScrollUp            |
| `PageDown` | PageScrollDown          |
| `Ctrl b`   | PageScrollUp            |
| `Ctrl f`   | PageScrollDown          |
| `Ctrl c`   | ScrollToBottom → locked |

#### Mode Switching

| Key      | Action        |
|----------|---------------|
| `s`      | → Scroll mode |
| `t`      | → Tab mode    |
| `m`      | → Move mode   |
| `r`      | → Resize mode |
| `enter`  | → Locked mode |
| `esc`    | → Locked mode |
| `Ctrl q` | Quit          |

### Session Mode

Enter from normal via `o`.

#### Session Actions

| Key | Action                              |
|-----|-------------------------------------|
| `w` | Session Manager (floating) → locked |
| `c` | Configuration (floating) → locked   |
| `p` | Plugin Manager (floating) → locked  |
| `a` | About (floating) → locked           |
| `s` | Share (floating) → locked           |
| `d` | Detach                              |

#### Mode Switching

| Key      | Action        |
|----------|---------------|
| `o`      | → Normal mode |
| `t`      | → Tab mode    |
| `m`      | → Move mode   |
| `r`      | → Resize mode |
| `enter`  | → Locked mode |
| `esc`    | → Locked mode |
| `Ctrl q` | Quit          |

### EnterSearch Mode

Enter from scroll via `f`. Type search query, then `enter` to search.

| Key      | Action        |
|----------|---------------|
| `enter`  | → Search mode |
| `esc`    | → Scroll mode |
| `Ctrl c` | → Scroll mode |

### RenameTab Mode

Enter from tab via `r`. Type new name, then `enter` to confirm.

| Key      | Action                   |
|----------|--------------------------|
| `enter`  | Confirm → locked         |
| `esc`    | UndoRenameTab → Tab mode |
| `Ctrl c` | → Locked mode            |

### RenamePane Mode

Enter from pane via `c`. Type new name, then `enter` to confirm.

| Key      | Action                     |
|----------|----------------------------|
| `enter`  | Confirm → locked           |
| `esc`    | UndoRenamePane → Pane mode |
| `Ctrl c` | → Locked mode              |

### Configuration Notes

- Config file: `~/.config/zellij/config.kdl`
- Theme: `tokyo-night`
- Default layout: `compact`
- Default mode: `locked`
- Pane frames: disabled (`pane_frames false`)
- Session name in frame: hidden (`hide_session_name true`)
- Scrollback editor: `~/.local/bin/zellij-emacs`
- Startup tips: disabled
- Release notes: disabled
