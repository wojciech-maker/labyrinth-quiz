# LABIRYNT QUIZ - Dokumentacja

## Przegląd
Przeglądarkowa gra quizowo-labiryntowa dla 3 graczy. Gracze poruszają się po losowo generowanych labiryntach, odpowiadają na pytania quizowe ABCD i używają mocy specjalnych. Hostowana na Railway.app przez Flask.

## Stack technologiczny
- **Backend:** Flask (Python) — serwuje statyczny HTML
- **Frontend:** React 18 + Babel standalone (z CDN, bez build stepa)
- **Hosting:** Railway.app (Procfile: `gunicorn app:app`)
- **Czcionka:** Fira Code (Google Fonts CDN)

## Struktura plików
```
labyrinth-quiz/
├── app.py                  # Serwer Flask — jedna trasa "/", serwuje index.html przez send_from_directory
├── templates/
│   └── index.html          # Cała gra — React SPA (~1240 linii)
├── requirements.txt        # flask==3.1.0, gunicorn==23.0.0
├── Procfile                # web: gunicorn app:app
├── .gitignore              # venv, __pycache__, .env
└── DOCS.md                 # Ten plik
```

## Ważna uwaga architektoniczna
`app.py` używa `send_from_directory()` zamiast `render_template()`. Flask domyślnie parsuje szablony przez Jinja2, który koliduje ze składnią JavaScript (destrukturyzacja obiektów, ternary operator z `:`, itp.). `send_from_directory()` serwuje plik surowo.

---

## Gracze
| ID | Imię   | Kolor     | Hex       | Emoji |
|----|--------|-----------|-----------|-------|
| 0  | Bartek | Czerwony  | `#ff4757` | 🔴    |
| 1  | Wojtek | Niebieski | `#1e90ff` | 🔵    |
| 2  | Sławek | Zielony   | `#2ed573` | 🟢    |

Zdefiniowane w stałej `PLAYERS` (linia 96-100).

---

## Ekrany gry (stan `screen`)

### 1. `menu` — Ekran startowy
- Tytuł "LABIRYNT QUIZ" z gradientowym tekstem
- Dwa przyciski: "3 GRACZY" (hot-seat) / "1 GRACZ" (solo z AI)
- Renderowany w liniach 842-866

### 2. `modeSelect` — Wybór postaci (tylko tryb solo)
- 3 przyciski z kolorami graczy
- Po wyborze: `startGame('solo', wybranyId)`
- Renderowany w liniach 869-887

### 3. `game` — Główny ekran gry
- **Górny pasek:** wyniki, statusy (streak, zamrożony, wyzwanie, korona)
- **Lewa strona (3/4):** duży labirynt aktywnego gracza
- **Prawa strona (1/4):** 2 mini-mapki przeciwników
- **Dół:** panel pytań lub panel mocy
- Renderowany w liniach 918-1000

### 4. `gameOver` — Ekran końcowy
- Ranking 3 graczy z medalami (🥇🥈🥉)
- Przycisk "Nowa gra"
- Renderowany w liniach 890-911

---

## Mechanika gry

### Labirynty
- **3 poziomy:** 7x7 → 8x8 → 9x9 (stała `MAZE_SIZES`, linia 101)
- **Generacja:** DFS Recursive Backtracker (funkcja `generateMaze`, linie 112-186)
- **Start:** Środek siatki (`Math.floor(size/2)`)
- **Centrum:** Gwarantowane min. 3 otwarte kierunki (linie 159-174)
- **Meta:** Losowy narożnik (linie 177-178)
- **Fog of War:** Gracz widzi tylko odwiedzone pola + sąsiedów przez otwarte przejścia (funkcja `isVisible` w `MazeView`, linie 1025-1038)
- Każdy gracz ma **osobny** labirynt

### Ruch
- **Kliknięcie** na sąsiednie pole = ruch (funkcja `handleCellClick`, linie 422-453)
- **Odwiedzone pola:** darmowe przechodzenie bez pytania
- **Nowe pole:** wywołuje pytanie quiz
- Ruch przez odwiedzone pola: BFS szukający ścieżki przez visited cells (funkcja `canReachThroughVisited`, linie 455-477)
- Każde wejście na pole = pole zdobyte (nawet przy złej odpowiedzi)

### Pytania
- **Pula:** 40 medium + 20 hard, pre-generowane (linie 29-93)
- **Tematy:** Tool, Black Sabbath, Metallica, SOAD, RATM, motoryzacja, lingwistyka, psychologia, teoria muzyki, geografia
- **Format:** ABCD, poprawna odpowiedź zawsze pod indeksem `c:0` (odpowiedź A) — TODO: tasowanie odpowiedzi przy wyświetlaniu
- **Timer:** 10s czytanie + 10s odpowiedź (useEffect, linie 331-351)
- **Przycisk "Wiem → odpowiadam":** skipuje fazę czytania
- **Po odpowiedzi:** pokazuje zieloną (prawidłowa) i czerwoną (błędna)
- **Wyczerpanie puli:** przetasowanie i restart (funkcja `drawQuestion`, linie 270-281)

### Wynik pytania
- **Dobrze (medium):** +podpowiedź (1 pole na prawidłowej ścieżce, zielone pulsujące) + wybór mocy
- **Dobrze ze STREAK (2+):** +2 podpowiedzi zamiast 1 + wybór mocy
- **Dobrze na hard (wyzwanie):** brak bonusu, streak=0, koniec tury
- **Źle:** streak=0, podpowiedzi znikają, koniec tury
- Logika w `handleAnswer` (linie 533-594)

### Streak
- Licznik kolejnych poprawnych odpowiedzi (per gracz)
- Stan: `streaks[3]` — tablica, wartość per gracz
- Streak >= 2: podwójna podpowiedź
- Resetowany przy: błędnej odpowiedzi, pytaniu hard (wyzwanie)
- Wyświetlany jako 🔥 w pasku wyników

### Podpowiedzi (Hints)
- Podświetlone pola na prawidłowej ścieżce do mety
- BFS od aktualnej pozycji do mety (funkcja `bfsPath`, linie 189-217)
- Wizualizacja: zielony glow + pulsacja CSS (`pulse` animacja)
- Widoczne do końca tury lub do złej odpowiedzi
- Stan: `hints[3]` — tablica Set-ów z kluczami "r,c"

---

## Moce (Powers)

Po poprawnej odpowiedzi (medium) gracz **musi** wybrać jedną moc.
Komponent: `PowerPanel` (linie 1149-1221)

| Moc | Emoji | Efekt | Ograniczenie |
|-----|-------|-------|--------------|
| Zamrożenie | 🧊 | Cel traci następną turę | Nie można zamrozić tego samego gracza 2x z rzędu (stan `lastFrozenBy`) |
| Wyzwanie | ⚡ | Cel dostaje pytanie HARD (brak bonusu nawet za poprawną) | - |
| Korona | 👑 | +1 punkt bonusowy | - |

Logika użycia: `usePower()` (linie 634-664)

---

## Punktacja
- **Labirynt:** 1. miejsce = 3 pkt, 2. = 2 pkt, 3. = 1 pkt (stała `POINTS`, linia 102)
- **Korona:** +1 pkt za każde użycie (dodawane natychmiast)
- **Koniec gry:** po 3 labiryntach → ranking końcowy
- Stan: `scores[3]`, `finishOrder[]`

---

## AI (tryb solo)

Aktywowane gdy `gameMode === 'solo'` i gracz nie jest `soloPlayer`.
Sprawdzenie: `isAI(pid)` (linia 267)

### Zachowanie AI
- **Ruch:** 75% szans na optymalną ścieżkę (BFS do mety), 25% losowy ruch (linia 692)
- **Odpowiedzi:** 50% poprawnych (medium), 30% (hard) (linie 741-742)
- **Wybór mocy:** 45% zamrożenie, 35% wyzwanie, 20% korona (linie 767-781)
- **Tura:** ~1-2s z widocznym ruchem (setTimeout w `runAiTurn`, linie 683-798)

### Komentarze AI
Losowe, wyświetlane pod mini-mapkami. Stała `AI_COMMENTS` (linie 104-109):
- **Poprawna:** "Ha! Łatwizna! 😎", "Jestem maszyną! 🤖", ...
- **Błędna:** "Jasna cholera! 😤", "Mój kot by lepiej odpowiedział...", ...
- **Zamrożony:** "AAAA! Zamrożony?! 🥶", ...
- **Użycie mocy:** "Hehehehe... 😈", "Smacznego! 💥", ...
- Auto-clear po 4 sekundach (useEffect, linie 834-837)

---

## Komponenty React

### `App` (linia 234)
Główny komponent. Zarządza całym stanem gry (~30 useState).
**Kluczowe stany:**
- `screen` — aktualny ekran (menu/modeSelect/game/gameOver)
- `gameMode` — 'multi' lub 'solo'
- `mazes[3]` — obiekty labiryntów (cells, size, start, goal)
- `positions[3]` — pozycje [r,c] graczy
- `visitedCells[3]` — Set-y odwiedzonych pól
- `scores[3]` — punkty
- `activePlayer` — aktualny gracz (0-2)
- `questionPhase` — null / 'reading' / 'answering' / 'result'
- `powerPhase` — boolean, czy wybieramy moc
- `hints[3]` — Set-y podpowiedzi

### `MazeView` (linia 1004)
Renderuje labirynt jako CSS Grid.
**Props:** maze, position, visited, player, hints, onCellClick, isActive, isInteractive
- `isActive=true` → duży (3/4 ekranu), `false` → mini-mapka
- Responsywny rozmiar komórek (pomiar containerRef)
- Fog of war w `isVisible()`
- Gracz = biała kropka z glow w kolorze gracza
- Meta = 🏁 emoji

### `QuestionPanel` (linia 1095)
Wyświetla pytanie, timer, przyciski ABCD.
**Props:** question, phase, timer, selectedAnswer, answerResult, onSkipReading, onAnswer, player
- Fazy: reading → answering → result
- Wyzwanie: żółty banner "WYZWANIE"

### `PowerPanel` (linia 1149)
Panel wyboru mocy po poprawnej odpowiedzi.
**Props:** player, others, onUsePower, frozen, lastFrozenBy, activePlayer
- Dwuetapowy: wybór mocy → wybór celu (freeze/challenge)
- Korona: jednoetapowa (bez celu)

---

## Funkcje pomocnicze

| Funkcja | Linia | Opis |
|---------|-------|------|
| `generateMaze(size)` | 112 | DFS recursive backtracker, zwraca {cells, size, start, goal} |
| `bfsPath(maze, from, to)` | 189 | BFS szukający najkrótszej ścieżki, zwraca tablicę [r,c] |
| `getAccessibleNeighbors(maze, r, c)` | 219 | Zwraca sąsiednie pola dostępne przez otwarte przejścia |
| `canReachThroughVisited(maze, pid, from, to)` | 455 | BFS po odwiedzonych polach — sprawdza czy gracz może dojść |
| `drawQuestion(type)` | 270 | Losuje pytanie z puli, usuwa je, resetuje pulę gdy pusta |
| `initLevel(lvl)` | 284 | Inicjuje nowy poziom (labirynty, pozycje, stany) |
| `startGame(mode, solo)` | 314 | Rozpoczyna grę |
| `nextPlayer()` | 354 | Przechodzi do następnego gracza (obsługuje zamrożenie, koniec poziomu) |
| `endLevel()` | 392 | Kończy poziom, przydziela punkty, przechodzi do następnego lub gameOver |
| `handleCellClick(r, c)` | 422 | Obsługa kliknięcia pola w labiryncie |
| `movePlayer(pid, r, c)` | 479 | Przesuwa gracza, sprawdza metę |
| `triggerQuestion(pid)` | 515 | Uruchamia pytanie quiz |
| `handleAnswer(ansIdx)` | 533 | Obsługa odpowiedzi (poprawna/błędna, streak, hints, power) |
| `showHints(pid, count)` | 620 | Pokazuje N podpowiedzi na prawidłowej ścieżce |
| `usePower(power, targetId)` | 634 | Użycie mocy przez gracza |
| `runAiTurn(pid)` | 683 | Pełna tura AI (ruch + pytanie + moc) |
| `usePowerAI(pid, power, targetId)` | 800 | Użycie mocy przez AI |
| `menuBtnStyle(color)` | 1224 | Styl przycisków menu |

---

## Wygląd (CSS)

- **Tło:** `#0f0f23` (ciemny granat)
- **Panel:** `#1a1a3e`
- **Ściany labiryntu:** `#3a3a5c`
- **Pola nieodkryte:** `#0a0a1a`
- **Pola odwiedzone:** `#1a1a3e`
- **Pola widoczne (nieodwiedzone):** `#12122e`
- **Tekst:** `#e0e0e0`
- **Złoto (wyniki, timer):** `#ffd700`
- **Czcionka:** `'Fira Code', monospace`

### Animacje CSS
- `pulse` — pulsowanie podpowiedzi (opacity)
- `glow` — świecenie (box-shadow)
- `crownFloat` — unoszenie korony (translateY)
- `fadeIn` — pojawianie elementów (opacity + translateY)
- `shake` — trzęsienie (translateX)

---

## Znane problemy / TODO

1. **Odpowiedź A zawsze poprawna** — wszystkie pytania mają `c:0`, co oznacza że poprawna odpowiedź to zawsze A. Potrzebne: tasowanie kolejności odpowiedzi przy wyświetlaniu pytania.
2. **Podwójne naliczanie punktów** — `endLevel()` (linia 402-403) dodaje punkty za labirynt, ale `movePlayer()` (linia 498) też je dodaje przy osiągnięciu mety. Powoduje podwójne naliczanie.
3. **Brak walidacji kliknięcia na odległe nowe pole** — gracz może kliknąć na nowe pole tylko jeśli jest bezpośrednim sąsiadem. Nie może kliknąć na nowe pole po przejściu przez odwiedzone. Powinno: pozwalać kliknąć na nowe pole sąsiadujące z dowolnym odwiedzonym polem.
4. **Timer stale closure** — timer w `useEffect` (linie 331-351) przechwytuje `questionPhase` w closure. Przejście reading→answering działa, ale edge case'y mogą powodować problemy.
5. **Napis "Twoja tura" pokazuje się zawsze** — linia 952: `activePlayer === activePlayer` jest zawsze true.
6. **AI nie wyświetla pytań** — AI odpowiada "w tle" (losowo), bez wizualizacji pytania na ekranie.
7. **Brak obsługi klawiatury** — Brak sterowania WASD/strzałkami, tylko kliknięcia myszą.
8. **levelScores nigdy nie jest użyty** — stan zbierany ale nie wyświetlany.

---

## Przepływ tury (diagram)

```
nextPlayer() → setActivePlayer(next)
    │
    ├── Gracz zamrożony? → Pokaż komunikat → skip do następnego
    │
    ├── Gracz AI? → runAiTurn()
    │       ├── Ruch (75% optymalny / 25% losowy)
    │       ├── Pole odwiedzone? → kolejny ruch (rekurencja)
    │       ├── Nowe pole → losuj wynik (50%/30%)
    │       │       ├── Dobrze → komentarz + wybór mocy AI → nextPlayer()
    │       │       └── Źle → komentarz → nextPlayer()
    │       └── Meta? → finishOrder + punkty → nextPlayer()
    │
    └── Gracz ludzki? → Czeka na kliknięcie
            ├── Klik na odwiedzone → movePlayer() (darmowy)
            ├── Klik na nowe → triggerQuestion()
            │       ├── Faza reading (10s)
            │       ├── Faza answering (10s)
            │       └── handleAnswer()
            │           ├── Dobrze → streak++ → hints → PowerPanel
            │           │       ├── Zamrożenie → freeze cel → nextPlayer()
            │           │       ├── Wyzwanie → challenge cel → nextPlayer()
            │           │       └── Korona → score++ → nextPlayer()
            │           └── Źle → streak=0 → clear hints → nextPlayer()
            └── Meta? → finishOrder + punkty → nextPlayer()
```

---

## Deploy

```bash
# Push na GitHub
git add -A && git commit -m "update" && git push

# Railway automatycznie przebudowuje po push
# URL: https://web-production-19039.up.railway.app
```

Railway wymaga:
- `requirements.txt` — instalacja zależności
- `Procfile` — komenda startowa (`gunicorn app:app`)
- Port z `os.environ.get("PORT")` — Railway ustawia dynamicznie
