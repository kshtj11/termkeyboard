# termkeyboard — structural reference

Single-file webapp: `index.html` (HTML + `<style>` + `<script>`). No build, no deps. Served by `python -m http.server` (see `.claude/launch.json`). Live: https://kshtj11.github.io/termkeyboard/

When prototyping a new variant, treat all dimensions/colors as soft. Only the architecture below is load-bearing.

## DOM hierarchy

```
body
├ #app                        root container, fixed size (square)
│ ├ #variant-toggle           thin top bar with .vtab × N (1, 2, 3, …)
│ ├ #terminal                 scrollable; .line divs (output + active prompt)
│ │                           active prompt is ALWAYS the last child
│ └ #keyboard
│   ├ #row-term               scrollable modifier bar
│   │ ├ #row-term-inner         pointer-drag scrollable flex row
│   │ │   .tkey × N             tab/ctrl/alt/shift/| ~ / - : * > &
│   │ └ .row-term-fade-l/r      gradient masks
│   └ #kbd-body
│     ├ #keys-layout            position:relative
│     │ ├ .row.alpha-row × 2    QWERTY / ASDFGHJKL
│     │ ├ .row.num-row × 2      hidden unless body.num-mode
│     │ ├ .row-shift.alpha-shift  ⇧ ZXCVBNM ⌫ (absolute children)
│     │ ├ .row-shift.num-shift    symbols + ⌫
│     │ └ #trackpoint           abs-positioned, lives at GHVB intersection
│     └ #row-bottom             kbd-icon, ?123, Fn, ',', space, '.', Done
└ #tp-radial                 outside #app, position:fixed pie menu (4 sectors + arrows)
```

Keys carry `data-char="x"` (types) or `data-action="name"` (dispatched to `handleAction`). A delegated click listener on `document` routes both.

## State (top of `<script>`)

```
input, cursorPos         // text + caret
selectAnchor             // null | number — selection start; cursor is the other end
shift, ctrl, alt         // sticky-toggle modifiers
numMode                  // QWERTY ↔ numpad (body.num-mode)
history, histIdx, savedInput
activeLine               // current prompt <div>, or null when frozen
variant                  // 1 | 2 | 3 (extend setVariant for more)
```

## Function map

| function | role |
|---|---|
| `getInputHTML()` | builds spans for cursor + selection (p-text / p-sel / p-cursor) |
| `render()` | `activeLine.innerHTML = PROMPT + getInputHTML()` |
| `newPrompt()` | append new active line div, render |
| `freezeLine(cmd, cls)` | static snapshot; sets activeLine = null |
| `appendOutput(text, cls)` | inserts `.line` before activeLine |
| `runCommand(cmd)` | freeze → dispatch built-in → newPrompt |
| `typeChar(ch)` | deleteSelection first, then insert |
| `handleArrow(dir)` | reads shift/ctrl/alt, rewrites `dir`, dispatches 12 cases |
| `handleAction(act)` | shift / ctrl / alt / del / enter / esc / tab / toggle-num / fn |
| `handleCtrlKey(k)` | C/L/U/W — clears ctrl + alt + selection |
| `wordLeft()` / `wordRight()` | bash-style boundaries |
| `deleteSelection()` | returns true if it removed a range |
| `clearSel()` | selectAnchor = null |
| `tpActivate(dir)` | radial: highlight sector, fire arrow, schedule auto-repeat |
| `tpStop()` | clear radial state + dragging classes + v3-dragging |
| `spStop()` | v2 spacebar: tap→space if no drag, drop v2-dragging |
| `setVariant(v)` | toggles `body.v2` / `body.v3` + active tab |

Built-in commands in `runCommand`: `clear`/`cls`, `echo`, `date`, `pwd`, `env`, `help`. Unknown → `bash: x: command not found`.

## Modifier toggle pattern (critical)

`shift`, `ctrl`, `alt` are **sticky toggles**, not held keys.

- Tap modifier → ON. Button gets `.on` class. Top-bar + keyboard-body buttons are **synced** (e.g., `btn-shift` and `btn-shift-top`).
- Persistence rules:
  - **shift**: cleared by tapping again, OR after typing a letter (auto-clear). Turning shift OFF also collapses selection.
  - **ctrl**: cleared by tapping again, OR after `handleCtrlKey` fires (C/L/U/W). Stays ON through `ctrl+⌫` (so you can repeat).
  - **alt**: cleared by tapping again, OR after typing a letter, OR via `handleCtrlKey`.
- `handleArrow` reads flags at the top and rewrites `dir`:
  - `shift` → `sel-*`
  - `ctrl || alt` → `word-*`
  - both → `sel-word-*`
- Physical keyboard uses `e.shiftKey` / `e.ctrlKey` directly in the `keydown` handler (not these flags).

When adding a new modifier button: give it `data-action="X"`, an `id="btn-X-top"` (or wherever), add `case 'X'` in `handleAction` that toggles flag + syncs both `.on` classes, then read the flag in whatever logic needs it.

## Variants

| | v1 | v2 | v3 |
|---|---|---|---|
| body class | — | `v2` | `v3` |
| trackpoint | radial pie menu on hold | hidden (`display:none`) | visible; radial suppressed; pointerdown adds `v3-dragging` |
| spacebar | types `' '` | tap=space, drag=arrows; adds `v2-dragging` | types `' '` |
| keyboard fade | — | `v2-dragging` dims keys, spacebar stays lit | `v3-dragging` dims keys |

`vN-dragging` classes apply `color: rgba(221,221,221,0.18)` + flatter backgrounds with a 0.15s CSS transition. Driven from JS in `spStop`/`tpStop` and the spacebar/trackpoint pointermove handlers.

To add v4+: add a `.vtab` to `#variant-toggle`, extend `setVariant`, add `body.v4` CSS overrides, wire any new pointer handlers behind `if (variant === 4)`.

## Touch-bar drag scroll (`#row-term-inner`)

`touch-action: none`, no `setPointerCapture` (so child key clicks still fire).

- Scroll updates at `>2px` movement (responsive)
- `tsDidDrag` only sets past `8px` (touch-slop — under this is normal tap jitter)
- Click handler: `if (tsDidDrag && inside #row-term-inner) return` — prevents key fires during scroll drag

## Shortcuts

| | physical | on-screen |
|---|---|---|
| move char | ←/→ | trackpoint / spacebar-drag (v2) |
| move word | Ctrl+←/→ | ctrl ON + trackpoint |
| select char | Shift+←/→ | shift ON + trackpoint |
| select word | Ctrl+Shift+←/→ | shift+ctrl ON + trackpoint |
| history | ↑/↓ | trackpoint up/down |
| delete prev / selection | Backspace | ⌫ |
| delete word back | Ctrl+Backspace | ctrl ON + ⌫ (ctrl stays ON) |
| cancel / clear / line / word | Ctrl+C/L/U/W | ctrl ON + C/L/U/W (ctrl cleared) |

`deleteSelection` always wins over backspace variants.

## Customization checklist (when forking a variant)

Safe to change anywhere — no logic depends on these:
- `#app { width / height }` — currently square
- `.key`, `.row`, `#keys-layout`, `#row-bottom` — sizes, gaps, padding
- `#trackpoint { top / left }` — position between GHVB if keys resize
- `#tp-radial { width / height }` + `.tp-arr-*` inset offsets
- `.tkey { height }` + matching `#row-term { height }`
- All colors / font family / font sizes

If you change key dimensions, recompute the trackpoint `top/left` so its center sits at the GHVB intersection. The math is commented above `#keys-layout` in the CSS.
