## Vim cheatsheet

### Examples

Adjust indentation of a text block:

1. Set shift width `:set shiftwidth=<number>`
1. Select text block (`Vjjj...` or `V<number>j`)
1. Indent (`>>`) or unindent (`<<`)

Move a block of text

1. Select text block (`Vjjj...` or `V<number>j`)
1. Kill the block (`d`) to cut it, or yank the block (`y`) to copy it
1. Move the cursor above the line where you want to paste
1. Paste (`p`)

Convert tabs to spaces

1. Set tab width (`set tabstop=<number>`)
1. Retab entire file (`:retab`)

Use spaces instead of tabs

1. set tabstop=2 shiftwidth=2 expandtab
