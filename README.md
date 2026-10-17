<p align="right">
  <strong>English</strong> · <a href="README.zh_CN.md">简体中文</a>
</p>

# Coopa game template

Make a game for the Coopa card. This template is a small playable game: Kuba pops out of three holes; move the slipper with ▲▼ and swat with ●.

## Start

1. Click **Use this template** on GitHub to create your own repository.
2. In your repository, open **Code → Codespaces → Create codespace**. The first start takes a few minutes (it installs the SDK).
3. Pick a game id (lowercase letters and digits, starting with a letter, at most 15) and run `coopa rename <your-id>` in the terminal.
4. Run `coopa run`. A browser tab opens with the card screen: ↑↓ are ▲▼, Space is ●.
5. Edit the code and run `coopa run` again.
6. Run `coopa check` for the same check the store runs (boot, screenshots, 5 minutes of random key presses, save size).

Every push also runs the **self-check** workflow (Actions), which additionally builds the card firmware and checks the game size.

You can also open the repository in VS Code with Dev Containers on your own computer (needs Docker).

When a new SDK comes out (`coopa version` shows the installed one; the latest is under [coopa-sdk Releases](https://github.com/coopa-store-dev/coopa-sdk/releases)):
run **Codespaces: Rebuild Container** from the command palette (**Dev Containers: Rebuild Container** on your own computer) to install the latest SDK.

## Files

| File | What it is |
|---|---|
| `coopa.toml` | The game's ID card: id, name, version, SDK version, entry point, milestones |
| `<id>_texts.h` | All on-screen text (the font is built from these strings) |
| `src/<id>_logic.*` | Game rules: plain C, no screen and no saves |
| `src/<id>_save.*` | Save format and milestones |
| `src/<id>_ui.*` | Drawing (LVGL and `ui_kit.h`) |
| `src/<id>_app.c` | The `kit_app_t` table the card shell calls |

## Rules

The rules are listed in [README.zh_CN.md](README.zh_CN.md); `coopa check` tells you which one a game breaks.

## Two-card play (shell interface 2)

For two players facing each other with one card each, use `kit_link.h` (set `kit_api = 2` in `coopa.toml`). On the card
it runs over Bluetooth; the shell finds the other card, connects, resends and reports disconnects. Your game only sends and
receives messages of up to 64 bytes: call `kit_link_start(<your game number>)` on both cards, then poll
`kit_link_state()` and `kit_link_recv()` in `frame()`.

- Within one connection, messages arrive once and in order. **Anything not yet delivered when the link drops is lost**:
  when `kit_link_session()` changes, compare progress with the other card.
- `kit_link_send` returning `false` (not connected, or 8 messages already queued) means it was not sent: keep it and
  try again next frame.
- `kit_link_initiator()` is true on exactly one of the two cards (use it to decide who goes first).
- The screen stays on while connected (it still turns off while searching), and the shell closes the link when the player
  leaves your game. Not reading messages for a while is fine: the other card waits instead of dropping the link. Bluetooth
  uses about 68 KB of RAM on the card.
- A message takes about 60 ms per 16-byte chunk: good for turn-based games, not for syncing every frame.
- In the browser, `coopa run` prints a `duo.html` address: two cards on one page.

## Submitting

When the Coopa website opens, sign in with GitHub, apply to become a developer, and submit your repository and commit there.
