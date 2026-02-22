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
│   └── index.html          # Cała gra — React SPA (~1316 linii)
├── requirements.txt        # flask==3.1.0, gunicorn==23.0.0
├── Procfile                # web: gunicorn app:app
├── .gitignore              # venv, __pycache__, .env
├── CONTEXT.md              # Skrócony kontekst dla AI
└── docs.md                 # Ten plik
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

### 2. `modeSelect` — Wybór postaci (tylko tryb solo)
- 3 przyciski z kolorami graczy
- Po wyborze: `startGame('solo', wybranyId)`

### 3. `game` — Główny ekran gry
- **Górny pasek:** wyniki, statusy (streak, zamrożony, wyzwanie, korona)
- **Lewa strona (3/4):** duży labirynt aktywnego gracza
- **Prawa strona (1/4):** 2 mini-mapki przeciwników
- **Dół:** panel pytań lub panel mocy
- **Nagłówek:** Imię gracza + "Twoja tura" (tylko dla gracza ludzkiego)

### 4. `gameOver` — Ekran końcowy
- Ranking 3 graczy z medalami (🥇🥈🥉)
- Tabela punktów per-poziom (levelScores)
- Przycisk "Nowa gra"

---

## Mechanika gry

### Labirynty
- **3 poziomy:** 7x7 → 8x8 → 9x9 (stała `MAZE_SIZES`, linia 101)
- **Generacja:** DFS Recursive Backtracker (funkcja `generateMaze`, linie 112-186)
- **Start:** Środek siatki (`Math.floor(size/2)`)
- **Centrum:** Gwarantowane min. 3 otwarte kierunki
- **Meta:** Losowy narożnik
- **Fog of War:** Gracz widzi tylko odwiedzone pola + sąsiedów przez otwarte przejścia (funkcja `isVisible` w `MazeView`)
- Każdy gracz ma **osobny** labirynt

### Ruch
- **Kliknięcie** na sąsiednie pole = ruch (funkcja `handleCellClick`, linia 432)
- **Odwiedzone pola:** darmowe przechodzenie bez pytania (bezpośrednio lub przez BFS visited path)
- **Nowe pole:** wywołuje pytanie quiz
- **Klik na nowe pole przez odwiedzone:** dozwolone — jeśli nowe pole sąsiaduje z odwiedzonym polem osiągalnym przez visited path
- Ruch przez odwiedzone pola: BFS szukający ścieżki przez visited cells (`canReachThroughVisited`, linia 476)
- Każde wejście na pole = pole zdobyte (nawet przy złej odpowiedzi)
- **Interakcja zablokowana** gdy: questionPhase aktywne, powerPhase aktywne, isAiTurn, turnMessage wyświetlany, gracz ukończył labirynt

### Pytania
- **Pula:** 40 medium + 20 hard, pre-generowane (linie 29-93)
- **Tematy:** Tool, Black Sabbath, Metallica, SOAD, RATM, motoryzacja, lingwistyka, psychologia, teoria muzyki, geografia
- **Format:** ABCD, odpowiedzi tasowane losowo przy każdym wyświetleniu (`drawQuestion` tasuje tablicę `a[]` i aktualizuje indeks `c`)
- **Timer:** 10s czytanie + 10s odpowiedź (useEffect, linia 344)
- **Przycisk "Wiem → odpowiadam":** skipuje fazę czytania (ukryty podczas tury AI)
- **Po odpowiedzi:** pokazuje zieloną (prawidłowa) i czerwoną (błędna)
- **Wyczerpanie puli:** przetasowanie i restart (`drawQuestion`, linia 275)

### Wynik pytania
- **Dobrze (medium):** +podpowiedź (1 nieodwiedzone pole na prawidłowej ścieżce) + wybór mocy
- **Dobrze ze STREAK (2+):** +2 podpowiedzi zamiast 1 + wybór mocy
- **Dobrze na hard (wyzwanie):** brak bonusu, streak=0, koniec tury
- **Źle:** streak=0, podpowiedzi znikają, koniec tury
- Logika w `handleAnswer` (linia 554)

### Streak
- Licznik kolejnych poprawnych odpowiedzi (per gracz)
- Stan: `streaks[3]` — tablica, wartość per gracz
- Streak >= 2: podwójna podpowiedź
- Resetowany przy: błędnej odpowiedzi, pytaniu hard (wyzwanie)
- Wyświetlany jako 🔥 w pasku wyników

### Podpowiedzi (Hints)
- Podświetlone **nieodwiedzone** pola na prawidłowej ścieżce do mety
- BFS od aktualnej pozycji do mety (`bfsPath`, linia 189)
- Stare hinty czyszczone przy każdym wywołaniu `showHints()` — zawsze aktualne
- Wizualizacja: zielony glow + pulsacja CSS (`pulse` animacja)
- Widoczne do końca tury lub do złej odpowiedzi
- Stan: `hints[3]` — tablica Set-ów z kluczami "r,c"

---

## Moce (Powers)

Po poprawnej odpowiedzi (medium) gracz **musi** wybrać jedną moc.
Komponent: `PowerPanel` (linia 1228)

| Moc | Emoji | Efekt | Ograniczenie |
|-----|-------|-------|--------------|
| Zamrożenie | 🧊 | Cel traci następną turę | Nie można zamrozić tego samego gracza 2x z rzędu (stan `lastFrozenBy`) |
| Wyzwanie | ⚡ | Cel dostaje pytanie HARD (brak bonusu nawet za poprawną) | - |
| Korona | 👑 | +1 punkt bonusowy | - |

Logika użycia (gracz ludzki): `usePower()` (linia 660)

---

## Punktacja
- **Labirynt:** 1. miejsce = 3 pkt, 2. = 2 pkt, 3. = 1 pkt (stała `POINTS`, linia 102)
- **Korona:** +1 pkt za każde użycie (dodawane natychmiast)
- **Punkty za metę:** dodawane TYLKO w `movePlayer()`/`movePlayerAfterQuestion()`, NIE w `endLevel()`
- **`endLevel()`:** zapisuje `levelScores` do tabeli per-poziom, ale nie modyfikuje `scores`
- **Koniec gry:** po 3 labiryntach → ranking końcowy z tabelą levelScores
- Stan: `scores[3]`, `finishOrder[]`, `levelScores[]`

---

## AI (tryb solo)

Aktywowane gdy `gameMode === 'solo'` i gracz nie jest `soloPlayer`.
Sprawdzenie: `isAI(pid)` (linia 271)

### Zachowanie AI
- **Ruch:** 75% szans na optymalną ścieżkę (BFS do mety), 25% losowy ruch
- **Odpowiedzi:** 50% poprawnych (medium), 30% (hard)
- **Wybór mocy:** 45% zamrożenie, 35% wyzwanie, 20% korona
- **Pytania widoczne:** AI wyświetla pytania w QuestionPanel z pełną wizualizacją (reading → answering → result)

### Architektura tury AI (`async function runAiTurn`, linia 709)
Tura AI jest zaimplementowana jako `async function` z `await wait()` — flat, sekwencyjna logika zamiast zagnieżdżonych setTimeout.

**Sekwencja tury:**
```
[+0ms]     Ruch na pole (500ms visited, 1000ms new)
           → while loop: rusza się przez odwiedzone, zatrzymuje na nowym
[+1000ms]  Pytanie — reading phase (2s)
[+3000ms]  Pytanie — answering phase, AI wybiera odpowiedź (2s)
[+5000ms]  Pytanie — result phase (2s)
[+7000ms]  Jeśli poprawna medium → moc (2.5s) → nextPlayer()
           Jeśli zła → nextPlayer()
           Jeśli challenge correct → nextPlayer()
```

**Functional state updaters:** AI używa `setState(prev => ...)` zamiast `const new = [...state]` aby uniknąć stale closure issues między `await` pointami.

**Blokada interakcji:** Podczas tury AI przyciski odpowiedzi są disabled (`isAiTurn` prop w QuestionPanel), przycisk "Wiem→odpowiadam" ukryty, labirynt nie-interaktywny.

### Zamrożenie AI
Obsługiwane w `nextPlayer()` (linia 369):
- NIE zmienia `activePlayer` na zamrożonego gracza (zapobiega triggerowaniu AI useEffect)
- Pokazuje komunikat 1.5s, potem skipuje do następnego gracza

### Komentarze AI
Losowe, wyświetlane pod mini-mapkami. Stała `AI_COMMENTS` (linie 104-109):
- **Poprawna:** "Ha! Łatwizna! 😎", "Jestem maszyną! 🤖", ...
- **Błędna:** "Jasna cholera! 😤", "Mój kot by lepiej odpowiedział...", ...
- **Zamrożony:** "AAAA! Zamrożony?! 🥶", ...
- **Użycie mocy:** "Hehehehe... 😈", "Smacznego! 💥", ...
- Auto-clear po 4 sekundach (useEffect, linia 872)

---

## Komponenty React

### `App` (linia 239)
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
- `isAiTurn` — blokuje interakcję labiryntu i pytań
- `turnMessage` — komunikat blokujący interakcję (zamrożenie, moc, meta)

### `MazeView` (linia 1083)
Renderuje labirynt jako CSS Grid.
**Props:** maze, position, visited, player, hints, onCellClick, isActive, isInteractive
- `isActive=true` → duży (3/4 ekranu), `false` → mini-mapka
- Responsywny rozmiar komórek (pomiar containerRef)
- Fog of war w `isVisible()`
- Gracz = biała kropka z glow w kolorze gracza
- Meta = 🏁 emoji

### `QuestionPanel` (linia 1174)
Wyświetla pytanie, timer, przyciski ABCD.
**Props:** question, phase, timer, selectedAnswer, answerResult, onSkipReading, onAnswer, player, isAiTurn
- Fazy: reading → answering → result
- Wyzwanie: żółty banner "WYZWANIE"
- `isAiTurn`: ukrywa przycisk "Wiem→odpowiadam", blokuje klikanie odpowiedzi

### `PowerPanel` (linia 1228)
Panel wyboru mocy po poprawnej odpowiedzi.
**Props:** player, others, onUsePower, frozen, lastFrozenBy, activePlayer
- Dwuetapowy: wybór mocy → wybór celu (freeze/challenge)
- Korona: jednoetapowa (bez celu)

---

## Funkcje pomocnicze

| Funkcja | Linia | Opis |
|---------|-------|------|
| `wait(ms)` | 234 | Promise-based delay helper dla async AI |
| `generateMaze(size)` | 112 | DFS recursive backtracker, zwraca {cells, size, start, goal} |
| `bfsPath(maze, from, to)` | 189 | BFS szukający najkrótszej ścieżki, zwraca tablicę [r,c] |
| `getAccessibleNeighbors(maze, r, c)` | 219 | Zwraca sąsiednie pola dostępne przez otwarte przejścia |
| `canReachThroughVisited(maze, pid, from, to)` | 476 | BFS po odwiedzonych polach — sprawdza czy gracz może dojść |
| `drawQuestion(type)` | 275 | Losuje pytanie z puli, **tasuje odpowiedzi**, aktualizuje indeks `c` |
| `initLevel(lvl)` | 297 | Inicjuje nowy poziom (labirynty, pozycje, stany) |
| `startGame(mode, solo)` | 327 | Rozpoczyna grę |
| `nextPlayer()` | 369 | Przechodzi do następnego gracza (obsługuje zamrożenie bez triggera AI) |
| `endLevel()` | 405 | Kończy poziom, zapisuje levelScores, NIE dodaje punktów |
| `handleCellClick(r, c)` | 432 | Obsługa kliknięcia (visited/new/through-visited) |
| `movePlayer(pid, r, c)` | 500 | Przesuwa gracza, sprawdza metę, dodaje punkty |
| `movePlayerAfterQuestion(pid, r, c)` | 618 | j.w., wywoływana po pytaniu |
| `triggerQuestion(pid)` | 536 | Uruchamia pytanie quiz |
| `handleAnswer(ansIdx)` | 554 | Obsługa odpowiedzi (poprawna/błędna, streak, hints, power) |
| `showHints(pid, count)` | 641 | Czyści stare, pokazuje N nieodwiedzonych pól na ścieżce |
| `usePower(power, targetId)` | 660 | Użycie mocy przez gracza ludzkiego |
| `runAiTurn(pid)` | 709 | Async — pełna tura AI (ruch + pytanie + moc) |
| `menuBtnStyle(color)` | 1303 | Styl przycisków menu |

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

## Przepływ tury (diagram)

```
nextPlayer() → setActivePlayer(next)
    │
    ├── Gracz zamrożony? → Komunikat 1.5s (bez zmiany activePlayer) → skip
    │
    ├── Gracz AI? → useEffect → setIsAiTurn(true) → runAiTurn() [async]
    │       ├── while(visited): ruch 500ms/krok
    │       ├── Nowe pole → pytanie wizualne (reading 2s → answering 2s → result 2s)
    │       │       ├── Dobrze medium → komentarz + moc 2.5s → nextPlayer()
    │       │       ├── Dobrze hard → nextPlayer()
    │       │       └── Źle → komentarz → nextPlayer()
    │       └── Meta? → finishOrder + punkty → nextPlayer()
    │
    └── Gracz ludzki? → Czeka na kliknięcie (isInteractive=true)
            ├── Klik na odwiedzone → movePlayer() (darmowy)
            ├── Klik na nowe (bezpośrednie) → triggerQuestion()
            ├── Klik na nowe (przez visited) → triggerQuestion()
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

## Znane problemy / TODO
- Brak obsługi klawiatury (WASD/strzałki)
- Timer stale closure — timer w useEffect przechwytuje `questionPhase` w closure (zabezpieczony przez `isAiTurn` check)

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
