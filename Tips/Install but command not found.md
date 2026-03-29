The installer placed the claude binary in ~/.local/bin, but that directory isn't in your shell's PATH. Run these two commands:
```
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```
Then verify it works:
```
claude --version
```