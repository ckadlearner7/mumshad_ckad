## Vim cheatsheet

### Examples

Adjust indentation of a text block:

1. Set shift width `:set shiftwidth=<number>`
1. Select text block (`Vjjj...` or `V<number>j`)
1. Indent (`>>`) or unindent (`<<`)

Move a block of text

1. Select text block (`Vjjj...` or `V<number>j`)
2. Kill the block (`d`) to cut it, or yank the block (`y`) to copy it
3. Move the cursor above the line where you want to paste
4. Paste (`p`)
