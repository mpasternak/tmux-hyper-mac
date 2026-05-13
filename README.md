# tmux-hyper-mac

Konfiguracja klawiatury pod macOS która sprawia że tmuxem da się sterować **bezpośrednio** klawiszem Hyper (mapowanym na Caps Lock) — bez prefixa. `Caps+N` otwiera nowe okno, `Caps+←/→` przełącza okna, `Caps+H/J/K/L` chodzi po panelach.

## Architektura

```
Caps + N (klawiatura fizyczna)
  │
  ▼  Karabiner Elements (na laptopie)
     · Caps trzymane → modyfikator Hyper (⌘⌃⌥⇧)
     · Hyper+N → Shift+F1 (emitowane bezpośrednio)
  │
  ▼  Terminal (iTerm2 / Terminal.app)
     · Shift+F1 → escape sequence `\e[1;2P`
  │
  ▼  SSH (jeśli pracujesz zdalnie) → tmux na docelowej maszynie
  │
  ▼  tmux
     · `\e[1;2P` widzi jako `S-F1` → `bind -n S-F1 new-window`
```

Karabiner musi być **lokalnie** (tam gdzie fizyczna klawiatura). tmux może być lokalnie albo na zdalnym serwerze przez SSH — klawisze przechodzą bez problemu.

> Dlaczego Shift+Fx a nie F13–F24: macOS koduje F13–F19 jako Shift+F1..Shift+F7, ale **F20–F24 nie mają wbudowanego mapowania** — terminale ich nie wysyłają. Emitując Shift+Fx bezpośrednio omijamy ten problem dla wszystkich 12 slotów.

## Wymagania

- macOS z [Karabiner Elements](https://karabiner-elements.pqrs.org/) na laptopie
- tmux ≥ 3.0 tam gdzie ma działać multipleksowanie

## Instalacja

### 1. Reguły Karabinera (jeden plik, 13 reguł)

```bash
mkdir -p ~/.config/karabiner/assets/complex_modifications
cp karabiner-hyper-tmux.json ~/.config/karabiner/assets/complex_modifications/
```

W aplikacji: **Karabiner-Elements → Settings → Complex Modifications → Add predefined rule** → znajdź `Caps Lock → Hyper → tmux direct bindings` → **Enable All**.

Plik zawiera:
- **1 regułę** Caps Lock → Hyper (Escape if alone): Caps trzymane = Hyper (⌘⌃⌥⇧), Caps pyknięte solo = Escape
- **12 reguł** Hyper+klawisz → Shift+F1..Shift+F12 (kolejne bindy tmuxa)

> macOS 26+ (Tahoe): wariant "Caps Lock if alone" (toggle Caps Lock) jest zepsuty przez regresję `hidutil`. Stąd "Escape if alone" — przydatne i tak dla vim/emacs.

### 2. Config tmuxa

Na maszynie gdzie chodzi tmux (lokalnej lub zdalnej):

```bash
cp tmux.conf ~/.tmux.conf
tmux source-file ~/.tmux.conf    # jeśli tmux już działa
```

Jeśli masz własny `.tmux.conf` — przekopiuj z naszego pliku tylko sekcje "Hyper-direct bindy" oraz odpowiadające im ustawienia ogólne.

### 3. (jeśli zdalna maszyna) — config tmuxa na serwerze przez SSH

```bash
scp tmux.conf user@server:~/.tmux.conf
ssh user@server 'tmux source-file ~/.tmux.conf'
```

## Bindy bez prefixa

**Okna i sesje**

| Skrót | Akcja |
|---|---|
| `Caps+N` | nowe okno (dziedziczy bieżący katalog) |
| `Caps+→` | następne okno |
| `Caps+←` | poprzednie okno |
| `Caps+Space` | last-window (toggle do poprzedniego okna) |
| `Caps+T` | wybierz okno z listy (interaktywny picker) |
| `Caps+S` | wybierz sesję z listy |
| `Caps+R` | rename bieżącego okna |
| `Caps+D` | detach (sesja żyje dalej w tle) |

**Panele — nawigacja**

| Skrót | Akcja |
|---|---|
| `Caps+H` | panel w lewo |
| `Caps+J` | panel w dół |
| `Caps+K` | panel w górę |
| `Caps+L` | panel w prawo |

**Panele — splity (4 kierunki)**

| Skrót | Akcja |
|---|---|
| `Caps+↑` | nowy panel **above** |
| `Caps+-` | nowy panel **below** |
| `Caps+/` | nowy panel **left** |
| `Caps+\` | nowy panel **right** |

**Panele — pozostałe**

| Skrót | Akcja |
|---|---|
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

Dostępne "kanały emisji" (Karabiner → tmux):

| Kanał | Karabiner emit | tmux key | Zajęte sloty |
|---|---|---|---|
| podstawowy | `Shift+F1..Shift+F12` | `S-F1..S-F12` | 12/12 |
| rozszerzenie 1 | `Ctrl+Shift+F1..Ctrl+Shift+F12` | `C-S-F1..C-S-F12` | 5/12 (F1-F4, F6) |
| rozszerzenie 2 | `Alt+Shift+F1..F12` | `M-S-F1..M-S-F12` | 1/12 (F1 = Hyper+R) |

Aby dodać nową akcję bez prefixa:

1. W `~/.config/karabiner/assets/complex_modifications/karabiner-hyper-tmux.json` dopisz regułę `Hyper+klawisz → <wolny F-key combo>`
2. W `~/.tmux.conf` dopisz odpowiadający `bind -n <ten F-key combo> <akcja>`
3. Reload: w Karabinerze re-enable rule, w tmuxie `` ` r ``

## Troubleshooting

### `unknown key: F13` przy `tmux source-file`

Twój config ma `bind -n F13 ...` zamiast `bind -n S-F1 ...`. Karabiner emituje `Shift+F1..Shift+F12`, tmux łapie je jako `S-F1..S-F12`.

### `Caps+D`, `Caps+X` (i inne) nic nie robią mimo że `Caps+N` działa

Karabiner emituje `F20..F24` zamiast `Shift+F8..Shift+F12` — macOS nie ma mapowania dla F20+ → terminal wysyła nic. Sprawdź czy `karabiner-hyper-tmux.json` ma w polach `to` zapisy typu `{"key_code": "f8", "modifiers": ["left_shift"]}`, **NIE** `{"key_code": "f20"}`. Jeśli masz stary plik z F13-F24, podmień na nowy z tego repo.

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

### `Caps+R` (lub inny pojedynczy binding) nic nie robi, ale reszta działa

Konkretnie `Hyper+R` z kanału 2 (`Ctrl+Shift+F5/F7/...`) okazał się padać u nas — coś (macOS keyboard navigation, albo apka typu Raycast/BTT/Hammerspoon słuchająca globalnie `Cmd+Ctrl+Opt+Shift+R`) zjadało zdarzenie zanim docierało do iTerm2. Diagnostyka:

```sh
# w iTerm2, BEZ tmuxa:
cat
# wciśnij Caps+R — jeśli nic się nie pokazuje, coś zjada event
# wciśnij Caps+T dla porównania — powinno pokazać sekwencję escape
```

Karabiner-EventViewer.app rozstrzyga jednoznacznie: jeśli EventViewer pokazuje wyemitowany F-key z modyfikatorami, Karabiner robi swoje — winowajca siedzi między macOS a iTerm2. Jeśli EventViewer nie pokazuje wyjścia — reguła Karabinera nie aplikuje się (najczęstsza przyczyna: nie przeładowana w GUI po edycji JSON).

W tym repo `Hyper+R` siedzi już na kanale 3 (`M-S-F1`) właśnie z tego powodu — kanały macOS-owe (Ctrl+F1..F8 dla nawigacji klawiaturą) ani potencjalne globalne shortcut-listenery nie kolidują z `Alt+Shift+F1`.

### Karabiner-EventViewer.app

Najlepszy debug tool: pokazuje co dokładnie Karabiner emituje. Tap `Caps+N` z otwartym EventViewerem — powinieneś zobaczyć modyfikatory + zdarzenie `f13`. Jeśli widzisz tylko modyfikatory (bez F-keya) → reguła "Hyper → F-key" jest niewłączona.

## Pliki w repo

- `tmux.conf` — gotowy `.tmux.conf` (prefix backtick + bindy bez prefixa dla Hyper)
- `karabiner-hyper-tmux.json` — wszystkie reguły Karabinera: Caps→Hyper (Escape if alone) + 12 Hyper+klawisz → Shift+F1..Shift+F12
- `LICENSE` — MIT

## License

MIT — zobacz `LICENSE`.
