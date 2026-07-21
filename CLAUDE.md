# WW Discord Stats — project guide

Static, client-side stats website for Filippo's Discord-run Werewolf ("Lupus in Tabula") games. No backend: the whole site is one `index.html` that loads `data.sqlite` via sql.js (SQLite compiled to WASM) and queries it in-browser. Live at https://filippomart.github.io/WW-Discord/, deployed via GitHub Pages from this repo (`https://github.com/FilippoMart/WW-Discord.git`, branch `main`).

## Files in this repo

- `index.html` — the entire site (HTML + inline CSS + inline JS, sql.js from CDN).
- `data.sqlite` — the database the site reads. Must be regenerated and copied here after every data change (see pipeline below).
- `scripts/xlsx_to_sqlite.py` — rebuilds the sqlite (and a `.sql` dump) from the xlsx source of truth. Re-run after every xlsx edit.
- `docs/` — rulebook PDFs (RegoleDueLune, RegoleTreLune, RuoliDueLune, RuoliTreLune), just reference material, not part of the pipeline.

## The one thing NOT in this repo

**`WWDiscordtemplate.xlsx`** — the source-of-truth spreadsheet — lives only at `C:\Users\filip\Documents\_Filippo stuff\Giochi\ww\WWDiscordtemplate.xlsx` on Filippo's machine. It is not version-controlled or backed up anywhere. **If you're setting this project up on a new computer, you must bring this file over yourself** (cloud sync, USB, etc.) — nothing here does it for you. Once it's in place at the same path (or you adjust `SRC` in the script), everything else works from any machine that can clone this repo.

## Pipeline: how to add or change data

1. Edit `WWDiscordtemplate.xlsx` (sheets: `Giocatori`, `Ruoli`, `GPR`, `PRuoliPoss`, `PNotes`, `PWin`, `POverrides` — see schema below). Typically done via one-off openpyxl scripts rather than by hand.
2. Run `python scripts/xlsx_to_sqlite.py`. It reads the xlsx and writes `WWDiscordtemplate.sqlite` + `.sql` into the **parent** folder (`ww/`, not this repo) — that's deliberate, keeps the repo from tracking a second copy.
3. Copy the regenerated sqlite into this repo: `cp "../WWDiscordtemplate.sqlite" data.sqlite`.
4. Verify in a browser (see "Testing changes" below) before committing.
5. `git add`, commit, push. GitHub Pages redeploys automatically (check the Actions tab if the site doesn't update — a "build succeeded, deploy failed" hiccup has happened before; an empty commit retriggers it).

## Data model (sqlite schema)

- `players(nick PK, nome)` — nick is the Discord handle, nome is the real first name. The site displays `nome (nick)`.
- `roles(ruolo PK, aura, misticismo, fazione, ombra, espansione, set_lune)` — `fazione` is a comma-separated list of win-condition tokens (e.g. `"Villaggio, Mistici, Criminali, Inquisizione"` for a composite role, or a single token like `"Capobranco"`). `ombra` is `Ombra` / `Aiutante` / `Non Ombra`.
- `games(play PK)` — play id format is `YY-MM-DD_n` (e.g. `26-07-14_1`), chosen so lexicographic sort = chronological sort. `_n` increments for multiple games the same day.
- `assignments(play, nick, ruolo)` — one row per player per game (plus a `ruolo = 'MASTER'` row identifying that game's moderator, excluded from all stats via `WHERE ruolo != 'MASTER'`).
- `game_wins(play PK, win, win_ruoli)` — `win` is the free-text winning fazione(s) for that game (e.g. `"Villaggio, Mistici"`, `"Capobranco"`); `win_ruoli` is the comma list of specific role **names** credited with the win that game.
- `game_notes(play PK, notes)` — free text, e.g. "Angelo di X, Giulietta di Y", vampirization events, Monaco's checks. This is the raw material `fazione_overrides` gets derived from.
- `game_ruoli_possibili(play PK, ruoli_possibili)` — the ruleset/role-pool description for that game. Default when unspecified: **"2 lune NO Giullare"**.
- `fazione_overrides(play, nick, fazione_finale)` — see "Condizioni Vittoria Finale" below.

## How a win gets credited (important, easy to get wrong)

For each assignment, take the **final** fazione tokens (role's `fazione`, or the override if one exists — see below) and check whether they overlap with the game's `win` tokens:
- If the role's fazione is a single atomic token (e.g. `"Villaggio"`), it's credited if that exact token string appears in `win`.
- If the role's fazione is a multi-token set that **exactly matches** one of five composite signatures (Città, Branco, Traditore, Ghoul, Megera — see `COMPOSTE` in `index.html`), it's bucketed under that composite instead of its atomic tokens, and credited simply if the role **name** appears in `win_ruoli` (no token-overlap check needed for composites).

**Do not use a blanket rule like "helper roles always win when the wolves win."** Always check the specific role's fazione tokens (visible in the site's "Ruoli e Condizioni (RC)" catalog) against what actually won. It usually agrees with the loose rule, but not always, and the token check is the one that's actually correct.

## Condizioni Vittoria Finale (fazione_overrides)

A role's fazione at game start isn't always its real win condition — mid-game events change it. When adding a new game, always re-read the notes for these five patterns and add a row to `POverrides` (play, nick, fazione_finale) for each match, before regenerating the sqlite:

| Note pattern | Affected player | fazione_finale |
|---|---|---|
| "Giulietta di X" | whoever plays role X (Giulietta's Romeo) | `Romeo` (new atomic token, not tied to any role) |
| "X gildato/a" | X | `Criminali` |
| "X vampirizzato/a" | X | `Vampiro` |
| "Contadino Lupo svegliato" | the Contadino Lupo player | `Capobranco, Lupo Reietto` (exactly the Branco composite signature) |
| "Fanatico di/del/della X" | the Fanatico player | whatever role X's own (possibly also-overridden) fazione is, looked up from that same game's assignments |

Note: "Angelo di X" does **not** trigger an override — only Giulietta's target does. Becchino has no override rule; its win condition stays genuinely variable.

In `index.html`, `atomicTokens` has `'Romeo'` added manually (since no role literally has that fazione), and `IMPLOSE_GROUPS.Amanti` includes it. `IMPLOSE_GROUPS` currently:
```
Umani:    Mistici, Villaggio, Criminali, Inquisizione, Città
Ombre:    Capobranco, Lupo Reietto, Lupo Solitario, Vampiro, Negromante, Nosferatu, Posseduto, Branco
Aiutanti: Traditore, Megera, Ghoul
Amanti:   Angelo, Giulietta, Romeo
Afazione: Pazzo, Giullare, Viaggiatore
```
Becchino is excluded from all groups (variable condition, no override rule). Fanatico is never a group member directly — once overridden, its games land in whichever group its target's fazione belongs to.

## Site-wide conventions

- **Wilson score lower bound** (95% CI, z=1.96) is the standard ranking metric everywhere, not raw win %. Tiebreak chain for every Wilson-ranked table: **Wilson score desc → numerator (wins) desc → denominator (games) asc**. See `wilsonLowerBound()` / `sortByRate()` in `index.html`.
- Leaderboards generally require a minimum sample size before a player appears (5 games for %Ombra/%Cattivo/Dark Souls, 50 opponents faced for %Underdog) — check the relevant `showX()` function for the exact threshold before adding a new one.
- New games default to `"2 lune NO Giullare"` for Ruoli Possibili unless told otherwise.
- Play IDs: `YY-MM-DD_n`, always verify lexicographic order still matches chronological order after adding a game.

## Testing changes locally

Serve the folder (e.g. `python -m http.server 8532` from inside `website/`) and open it in a browser. The page fetches `data.sqlite` once on load — after regenerating it, a normal refresh may serve a stale cached copy. To force a reload in a JS console / automated check:

```js
(async () => {
  const SQL = await initSqlJs({ locateFile: file => `https://sql.js.org/dist/${file}` });
  const resp = await fetch('data.sqlite?bust=' + Date.now(), {cache: 'no-store'});
  const buf = await resp.arrayBuffer();
  db = new SQL.Database(new Uint8Array(buf)); // bare `db`, NOT `window.db` — the page declares `let db` at script scope, and window.db is a different binding that the page's own functions won't see
})();
```

## Git push note

The first push from a fresh environment may need an interactive browser popup for GitHub auth (Git Credential Manager) — if a push hangs or fails silently in an automated context, it may need to be run manually once by Filippo; after that the cached credential works fine.
