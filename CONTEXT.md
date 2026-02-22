# LABIRYNT QUIZ — Kontekst dla AI

## Co to jest
Przeglądarkowa gra "Labirynt Quiz" — 3 graczy (Bartek/Wojtek/Sławek) poruszają się po labiryntach z fog of war, odpowiadają na pytania ABCD i używają mocy (zamrożenie/wyzwanie/korona).

## Stack
- Flask (Python) serwuje statyczny HTML przez `send_from_directory()` (NIE `render_template` — Jinja2 koliduje z JS)
- React 18 + Babel standalone z CDN (bez build stepa)
- Hosting: Railway.app, URL: https://web-production-19039.up.railway.app
- Cały kod gry w jednym pliku: `templates/index.html` (~1316 linii)

## Struktura plików
```
app.py                  → Flask serwer (5 linii, send_from_directory)
templates/index.html    → Cała gra React (jedyny plik do edycji przy zmianach gry)
requirements.txt        → flask, gunicorn
Procfile                → web: gunicorn app:app
CONTEXT.md              → Ten plik (skrócony kontekst)
docs.md                 → Pełna dokumentacja
```

## Architektura kodu (index.html)

### Sekcje kodu (po liniach)
- 29-93: Pule pytań (QUESTIONS_MEDIUM: 40szt, QUESTIONS_HARD: 20szt)
- 96-109: Stałe (PLAYERS, MAZE_SIZES, POINTS, AI_COMMENTS)
- 112-186: Generacja labiryntu (DFS recursive backtracker)
- 189-231: Pathfinding (BFS) + getAccessibleNeighbors
- 234-236: Helper `wait(ms)` — async delay dla AI
- 239-1080: Komponent App (stan gry, logika, render)
- 1083-1170: Komponent MazeView (renderowanie labiryntu jako CSS Grid)
- 1174-1224: Komponent QuestionPanel (pytanie, timer, ABCD)
- 1228-1300: Komponent PowerPanel (wybór mocy)

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
- `isAiTurn`: boolean — blokuje interakcję gracza
- `turnMessage`: string|null — komunikat blokujący interakcję

### Kluczowe funkcje
- `wait(ms)` → Promise delay helper (linia 234)
- `generateMaze(size)` → {cells, size, start, goal}
- `bfsPath(maze, from, to)` → [[r,c], ...]
- `drawQuestion(type)` → losuje pytanie, tasuje odpowiedzi, aktualizuje indeks `c`
- `handleCellClick(r,c)` → ruch gracza (obsługuje klik przez odwiedzone na nowe pola)
- `triggerQuestion(pid)` → uruchamia quiz
- `handleAnswer(ansIdx)` → obsługa odpowiedzi
- `usePower(power, targetId)` → zamrożenie/wyzwanie/korona (gracz ludzki)
- `runAiTurn(pid)` → async — pełna tura AI (ruch + pytanie + moc, sekwencyjne await)
- `nextPlayer()` → zmiana tury (zamrożenie: nie zmienia activePlayer, skip po 1.5s)
- `initLevel(lvl)` → reset poziomu
- `endLevel()` → koniec poziomu (NIE dodaje punktów, tylko zapisuje levelScores)

## Naprawione bugi (historia)
1. ✅ **Odpowiedź A zawsze poprawna** — `drawQuestion()` tasuje odpowiedzi i aktualizuje `c`
2. ✅ **Podwójne naliczanie punktów** — usunięto naliczanie z `endLevel()`, punkty tylko w `movePlayer()`/`movePlayerAfterQuestion()`
3. ✅ **"Twoja tura" zawsze widoczne** — zmieniono na `!isAI(activePlayer)`
4. ✅ **Klik na nowe pole przez odwiedzone** — `handleCellClick` sprawdza sąsiadów nowego pola osiągalnych przez visited path
5. ✅ **AI nie wyświetla pytań** — `runAiTurn()` pokazuje pytanie w QuestionPanel z fazami reading→answering→result
6. ✅ **AI zamrożenie bug** — `nextPlayer()` nie ustawia `activePlayer` na zamrożonego AI (nie triggeruje AI useEffect)
7. ✅ **Hinty na odwiedzonych polach** — `showHints()` pomija visited, czyści stare hinty
8. ✅ **Przechwycenie tury AI przez gracza** — dodano `!turnMessage` do `isInteractive`
9. ✅ **levelScores niewyświetlany** — tabela per-poziom na ekranie gameOver

## Znane problemy / TODO
- Brak obsługi klawiatury (WASD/strzałki)
- Timer stale closure — timer w useEffect przechwytuje `questionPhase` w closure (zabezpieczony przez `isAiTurn` check)

## Ważne zasady przy edycji
- NIGDY nie używaj `render_template()` — Jinja2 psuje JavaScript
- Poprawna odpowiedź to `question.a[question.c]` — indeks `c` (po tasowaniu)
- Labirynt to siatka obiektów z `walls: {top, right, bottom, left}` (boolean)
- Fog of war sprawdzany w `MazeView.isVisible()` — pole widoczne jeśli odwiedzone LUB sąsiad odwiedzonego przez otwarty przejazd
- Pozycje i visited używają formatu string "r,c" jako klucz Set
- AI kontrolowane przez `isAI(pid)` = gameMode==='solo' && pid !== soloPlayer
- `isInteractive` sprawdza 5 warunków: `!questionPhase && !powerPhase && !isAiTurn && !turnMessage && !finishOrder.includes(activePlayer)`
- AI tura jest `async function` z `await wait()` — NIE używa zagnieżdżonych setTimeout
- Punkty za labirynt dodawane TYLKO w `movePlayer()`/`movePlayerAfterQuestion()`, NIE w `endLevel()`
