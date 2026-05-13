# tmux-hyper-mac

Konfiguracja klawiatury pod macOS która sprawia że tmuxem da się sterować **bezpośrednio** klawiszem Hyper (mapowanym na Caps Lock) — bez prefixa. `Caps+N` otwiera nowe okno, `Caps+←/→` przełącza okna, `Caps+H/J/K/L` chodzi po panelach.

## Architektura

```
Caps + N (klawiatura fizyczna)
  │
  ▼  Karabiner Elements (na laptopie)
     · Caps trzymane → modyfikator Hyper (⌘⌃⌥⇧)
     · Hyper+N → wirtualny klawisz F13
  │
  ▼  Terminal (iTerm2 / Terminal.app)
     · F13 → escape sequence `\e[1;2P` (=Shift+F1, bo macOS nie ma fizycznego F13)
  │
  ▼  SSH (jeśli pracujesz zdalnie) → tmux na docelowej maszynie
  │
  ▼  tmux
     · `\e[1;2P` widzi jako `S-F1` → `bind -n S-F1 new-window`
```

Karabiner musi być **lokalnie** (tam gdzie fizyczna klawiatura). tmux może być lokalnie albo na zdalnym serwerze przez SSH — klawisze przechodzą bez problemu.

## Wymagania

- macOS z [Karabiner Elements](https://karabiner-elements.pqrs.org/) na laptopie
- tmux ≥ 3.0 tam gdzie ma działać multipleksowanie

## Instalacja

### 1. Reguła Karabinera: Caps Lock → Hyper (Escape if alone)

```bash
mkdir -p ~/.config/karabiner/assets/complex_modifications
cp karabiner-caps-to-hyper.json ~/.config/karabiner/assets/complex_modifications/
```

W aplikacji: **Karabiner-Elements → Settings → Complex Modifications → Add predefined rule** → znajdź `Caps Lock → Hyper Key (⌃⌥⇧⌘) (Escape if alone)` → **Enable**.

Daje to:
- **Caps trzymane** + cokolwiek = Hyper modifier (⌘⌃⌥⇧)
- **Caps pyknięte solo** = Escape (przydatne dla vim/emacs)

> macOS 26+ (Tahoe): wariant "Caps Lock if alone" (toggle Caps Lock) jest zepsuty przez regresję `hidutil`. Używamy wariantu "Escape if alone".

### 2. Reguła Karabinera: Hyper+klawisz → F-key

```bash
cp karabiner-hyper-tmux.json ~/.config/karabiner/assets/complex_modifications/
```

W aplikacji: **Add predefined rule** → `Hyper → tmux direct bindings (F13-F24)` → **Enable All** (12 reguł).

### 3. Config tmuxa

Na maszynie gdzie chodzi tmux (lokalnej lub zdalnej):

```bash
cp tmux.conf ~/.tmux.conf
tmux source-file ~/.tmux.conf    # jeśli tmux już działa
```

Jeśli masz własny `.tmux.conf` — przekopiuj z naszego pliku tylko sekcje "Hyper-direct bindy" oraz odpowiadające im ustawienia ogólne.

## Bindy bez prefixa

| Skrót | Akcja |
|---|---|
| `Caps+N` | nowe okno (dziedziczy bieżący katalog) |
| `Caps+→` | następne okno |
| `Caps+←` | poprzednie okno |
| `Caps+H` | panel w lewo |
| `Caps+J` | panel w dół |
| `Caps+K` | panel w górę |
| `Caps+L` | panel w prawo |
| `Caps+\` | split poziomy (nowy panel po prawej) |
| `Caps+-` | split pionowy (nowy panel pod) |
| `Caps+D` | detach (sesja żyje dalej w tle) |
| `Caps+Z` | zoom panela (toggle) |
| `Caps+X` | kill panel (pyta y/n) |

## Bindy z prefixem (`` ` ``, backtick)

Dla rzadszych akcji `tmux.conf` zostawia klasyczny prefix backtick. `` ` ? `` w tmuxie zawsze pokazuje pełną listę. Najważniejsze:

| Skrót | Akcja |
|---|---|
| `` ` r `` | reload `.tmux.conf` |
| `` ` , `` | rename window |
| `` ` $ `` | rename session |
| `` ` [ `` | wejście w copy-mode (vi-style: `v` selekcja, `y` kopiuj do `pbcopy`) |
| `` ` s `` | wybierz sesję z listy |
| `` ` w `` | wybierz okno z listy |
| `` ` : `` | command mode tmuxa |

Żeby wstawić literalny backtick — wciśnij `` ` `` dwa razy.

## Dodawanie własnych bindów

12 slotów F13–F24 jest zajętych. Aby dodać nową akcję bez prefixa:

1. W `~/.config/karabiner/assets/complex_modifications/karabiner-hyper-tmux.json` zamień jedną z reguł na nową kombinację `Hyper+klawisz → Fxx`
2. W `~/.tmux.conf` zamień odpowiadający `bind -n S-Fy <stara akcja>` na `bind -n S-Fy <nowa>`
3. Reload: w Karabinerze re-enable rule, w tmuxie `` ` r ``

Mapowanie F-keys → tmux key names:

| Karabiner emit | tmux widzi |
|---|---|
| F13 | S-F1 |
| F14 | S-F2 |
| F15 | S-F3 |
| F16 | S-F4 |
| F17 | S-F5 |
| F18 | S-F6 |
| F19 | S-F7 |
| F20 | S-F8 |
| F21 | S-F9 |
| F22 | S-F10 |
| F23 | S-F11 |
| F24 | S-F12 |

(macOS koduje wirtualne F13+ jako Shift+F1+, bo fizyczne klawisze F13+ nie istnieją na klawiaturach macOS.)

## Troubleshooting

### `unknown key: F13` przy `tmux source-file`

Twój config ma `bind -n F13 ...` zamiast `bind -n S-F1 ...`. Patrz tabela mapowania wyżej.

### `Caps+N` nic nie robi

Diagnostyka warstwa po warstwie:

```bash
# 1. Test na laptopie (lokalnie, nie SSH):
cat
```
Wciśnij `Caps+N`. Powinno pokazać `^[[1;2P`. Pusto = reguły Karabinera nie aktywne. Cmd+N effect = reguła "Hyper → F-key" nie zaimportowana.

```bash
# 2. Test przez SSH (jeśli używasz):
ssh user@server
cat
```
Powinno pokazać to samo `^[[1;2P`. Co innego = profil terminala filtruje F-keys.

```bash
# 3. Test w tmuxie:
tmux new -s test
```
`Caps+N` powinno otworzyć drugie okno (status bar na dole pokaże `2:zsh`). Jeśli `cat` w tmuxie pokazuje `^[[1;2P` — tmux dostaje klawisz, ale nie ma bindy → reload: `tmux source-file ~/.tmux.conf`.

### Caps Lock LED nie świeci

To celowe — wariant "Escape if alone" zamienia toggle Caps Lock na Esc. Aby pisać DUŻYMI LITERAMI używaj Shift jak w normalnym tekście.

### Karabiner-EventViewer.app

Najlepszy debug tool: pokazuje co dokładnie Karabiner emituje. Tap `Caps+N` z otwartym EventViewerem — powinieneś zobaczyć modyfikatory + zdarzenie `f13`. Jeśli widzisz tylko modyfikatory (bez F-keya) → reguła "Hyper → F-key" jest niewłączona.

## Pliki w repo

- `tmux.conf` — gotowy `.tmux.conf` (prefix backtick + bindy bez prefixa dla Hyper)
- `karabiner-caps-to-hyper.json` — Karabiner: Caps→Hyper (wariant Escape if alone)
- `karabiner-hyper-tmux.json` — Karabiner: Hyper+klawisz → F13..F24
- `LICENSE` — MIT

## License

MIT — zobacz `LICENSE`.
