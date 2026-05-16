# CLI
`emacs -nw`: Run emacs in terminal - no window system

# Keymaps

## Core mental model

Emacs uses modifier keys heavily:

| Key | Meaning |
| --- | --- |
| `C-` | `Ctrl` |
| `M-` | `Meta`, usually `Alt` |
| `RET` | `Enter` |
| `SPC` | `Space` |
| `DEL` | `Backspace` |

Example:

`C-x C-f` means hold `Ctrl` + `x`, release, then `Ctrl` + `f`.

## Movement

| Key | Action |
| --- | --- |
| `C-f` | forward char |
| `C-b` | backward char |
| `C-n` | next line |
| `C-p` | previous line |
| `M-f` | forward word |
| `M-b` | backward word |
| `C-a` | beginning of line |
| `C-e` | end of line |
| `M-<` | top of file |
| `M->` | bottom of file |

These come from old terminal conventions and appear in many Linux shells too.

## File operations

| Key | Action |
| --- | --- |
| `C-x C-f` | open file |
| `C-x C-s` | save |
| `C-x C-w` | save as |
| `C-x k` | kill buffer |
| `C-x C-c` | quit Emacs |

Important: Emacs edits buffers, not directly files.

## Copy / cut / paste

| Key | Action |
| --- | --- |
| `C-SPC` | start selection |
| `C-w` | cut |
| `M-w` | copy |
| `C-y` | paste |
| `M-y` | cycle previous copied items |

Emacs calls clipboard history the kill ring.

## Undo / cancel

| Key | Action |
| --- | --- |
| `C-/` | undo |
| `C-g` | cancel current command |

`C-g` is extremely important. If Emacs feels stuck or confusing, press `C-g`.

## Search

| Key | Action |
| --- | --- |
| `C-s` | search forward |
| `C-r` | search backward |

Incremental search is one of Emacs' best features. Press `C-s` repeatedly to jump to the next match.

## Windows

Emacs calls panes "windows".

| Key | Action |
| --- | --- |
| `C-x 2` | split horizontal |
| `C-x 3` | split vertical |
| `C-x o` | switch pane |
| `C-x 0` | close current pane |
| `C-x 1` | keep only current pane |

## Buffers

| Key | Action |
| --- | --- |
| `C-x b` | switch buffer |
| `C-x C-b` | list buffers |

Unlike Visual Studio Code or Neovim, Emacs users usually switch buffers more than tabs.

## Help system

| Key | Action |
| --- | --- |
| `C-h k` | describe key |
| `C-h f` | describe function |
| `C-h v` | describe variable |
| `C-h t` | Emacs tutorial |
| `C-h ?` | help menu |

# Browser EWW
EWW is the built-in browser of Emacs.

## Navigation and scrolling

| Key | Action |
| --- | --- |
| `SPC` | scroll up |
| `DEL` | scroll down |
| `<` | beginning of page |
| `>` | end of page |
| `TAB` | next link |
| `M-TAB` | previous link |
| `n` | next paragraph |
| `p` | previous paragraph |
| `l` | backward in history |
| `r` | forward in history |
| `g` | reload current page |
| `G` | search URL |

## Page actions

| Key | Action |
| --- | --- |
| `w` | copy current page URL or link under cursor to kill ring |
| `d` | download file or link under cursor |
| `R` | toggle readable mode |
| `v` | view raw HTML source |
| `&` | open current page in external browser |
| `b` | add bookmark |
| `B` | view bookmarks |
| `H` | view browsing history |
