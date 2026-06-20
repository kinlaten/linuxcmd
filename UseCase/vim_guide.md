# In Shell
`set -o vi` turn vim mode in shell/bash, visual mode is powerful

# File explorer
`:Explore` or `:E` + location

Open Explorer in split pane: `Vex` or `Hex`

Keymap:

| Key | Action |
| --- | --- |
| `Enter` | open file/dir |
| `-` | go up parent dir |
| `%` | create file |
| `d` | create directory |
| `D` | delete |
| `R` | rename |
| `mf` | mark file |
| `mc` | copy marked |
| `mm` | move marked |
| `q` | quit explorer |

To open a file in split pane instead current buffer: `v` (vertical), `o` (horizontal) or `t` (new tab)

To navigate between 
- pane: `Ctrl w + hjkl`
- tab: cmd `tabnext` / `tabn` or `tabp` or `tabn<number>`

# Read file Markdown
## View outline
Cmd `/^#` find headers

# Command
# Visual Block mode
Assume want to add `left` and `right` to side of  
```
middle
middle
```

:<block>normal I<contentAtHead> <Ctrl+v then Escape> A<contentAtTail>

<Ctrl+v then Escape> is `^[` Escape in command mode

