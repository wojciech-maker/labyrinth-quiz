# LABIRYNT QUIZ — Kontekst dla AI

## Co to jest
Przeglądarkowa gra "Labirynt Quiz" — 3 graczy (Bartek/Wojtek/Sławek) poruszają się po labiryntach z fog of war, odpowiadają na pytania ABCD i używają mocy (zamrożenie/wyzwanie/korona).

## Stack
- Flask (Python) serwuje statyczny HTML przez `send_from_directory()` (NIE `render_template` — Jinja2 koliduje z JS)
- React 18 + Babel standalone z CDN (bez build stepa)
- Hosting: Railway.app, URL: https://web-production-19039.up.railway.app
- Cały kod gry w jednym pliku: `templates/index.html` (~1240 linii)

## Struktura plików
```
app.py                  → Flask serwer (5 linii, send_from_directory)
templates/index.html    → Cała gra React (jedyny plik do edycji przy zmianach gry)
requirements.txt        → flask, gunicorn
Procfile                → web: gunicorn app:app
DOCS.md                 → Pełna dokumentacja
```

## Architektura kodu (index.html)

### Sekcje kodu (po liniach)
- 29-93: Pule pytań (QUESTIONS_MEDIUM: 40szt, QUESTIONS_HARD: 20szt)
- 96-109: Stałe (PLAYERS, MAZE_SIZES, POINTS, AI_COMMENTS)
- 112-186: Generacja labiryntu (DFS recursive backtracker)
- 189-231: Pathfinding (BFS)
- 234-1000: Komponent App (stan gry, logika, render)
- 1004-1091: Komponent MazeView (renderowanie labiryntu jako CSS Grid)
- 1095-1145: Komponent QuestionPanel (pytanie, timer, ABCD)
- 1149-1221: Komponent PowerPanel (wybór mocy)

### Kluczowe stany (w App)
- `screen`: menu | modeSelect | game | gameOver
- `gameMode`: 'multi' | 'solo'
- `mazes[3]`: {cells[][], size, start[r,c], goal[r,c]}
- `positions[3]`: [r,c] per gracz
- `visitedCells[3]`: Set<"r,c"> per gracz
- `scores[3]`, `streaks[3]`, `frozen[3]`, `challenged[3]`, `crowns[3]`
- `activePlayer`: 0-2
- `questionPhase`: null | 'reading' | 'answering' | 'result'
- `powerPhase`: boolean
- `hints[3]`: Set<"r,c"> per gracz

### Kluczowe funkcje
- `generateMaze(size)` → {cells, size, start, goal}
- `bfsPath(maze, from, to)` → [[r,c], ...]
- `handleCellClick(r,c)` → ruch gracza
- `triggerQuestion(pid)` → uruchamia quiz
- `handleAnswer(ansIdx)` → obsługa odpowiedzi
- `usePower(power, targetId)` → zamrożenie/wyzwanie/korona
- `runAiTurn(pid)` → pełna tura AI
- `nextPlayer()` → zmiana tury
- `initLevel(lvl)` → reset poziomu
- `endLevel()` → koniec poziomu → punkty

## Znane bugi do naprawienia

### KRYTYCZNE
1. **Odpowiedź A zawsze poprawna** — Wszystkie pytania mają `c:0`. Trzeba tasować odpowiedzi przy wyświetlaniu.
2. **Podwójne naliczanie punktów** — `movePlayer()` i `endLevel()` oba dodają punkty za ukończenie labiryntu.

### WAŻNE
3. **"Twoja tura" zawsze widoczne** — Linia 952: `activePlayer === activePlayer` (tautologia). Powinno sprawdzać czy to gracz ludzki.
4. **Gracz nie może kliknąć na nowe pole przez odwiedzone** — handleCellClick sprawdza tylko bezpośrednich sąsiadów dla nowych pól.

### ULEPSZENIA
5. AI nie wyświetla pytań — odpowiada "w tle"
6. Brak obsługi klawiatury (WASD/strzałki)
7. `levelScores` zbierany ale nie wyświetlany

## Ważne zasady przy edycji
- NIGDY nie używaj `render_template()` — Jinja2 psuje JavaScript
- Poprawna odpowiedź to `question.a[question.c]` — indeks `c`
- Labirynt to siatka obiektów z `walls: {top, right, bottom, left}` (boolean)
- Fog of war sprawdzany w `MazeView.isVisible()` — pole widoczne jeśli odwiedzone LUB sąsiad odwiedzonego przez otwarty przejazd
- Pozycje i visited używają formatu string "r,c" jako klucz Set
- AI kontrolowane przez `isAI(pid)` = gameMode==='solo' && pid !== soloPlayer
