<script lang="ts">
  function getUKDateString() {
    return new Date().toLocaleDateString("en-GB", { timeZone: "Europe/London" });
  }

  function dailyNumberUK() {
    const dateStr = getUKDateString();
    let hash = 0;
    for (let i = 0; i < dateStr.length; i++) {
      hash = (hash * 31 + dateStr.charCodeAt(i)) % 10000;
    }
    return hash.toString().padStart(4, "0");
  }

  const answer = dailyNumberUK();
  const todayUK = getUKDateString();

  const DIGITS = 4;
  const MAX_GUESSES = 4;

  type CellStatus = "empty" | "typed" | "correct" | "present" | "absent";
  type Cell = { digit: string; status: CellStatus };

  let grid: Cell[][] = Array.from({ length: MAX_GUESSES }, () =>
    Array.from({ length: DIGITS }, () => ({ digit: "", status: "empty" as CellStatus }))
  );

  let activeRow = 0;
  let solved = false;
  let current = "";
  let hiddenInput: HTMLInputElement | null = null;

  let gameOver = false;
  let resultMessage = "Enter 4 digits to make your first guess.";

  // storage keys
  const PLAY_KEY = "numberGuess_lastPlayedUK";
  const SNAPSHOT_KEY = "numberGuess_snapshot_v1";

  type Snapshot = {
    date: string;
    grid: Cell[][];
    activeRow: number;
    solved: boolean;
    gameOver: boolean;
    resultMessage: string;
  };

  function saveSnapshot() {
    if (typeof window === "undefined") return;
    const snap: Snapshot = {
      date: todayUK,
      grid,
      activeRow,
      solved,
      gameOver,
      resultMessage
    };
    localStorage.setItem(SNAPSHOT_KEY, JSON.stringify(snap));
  }

  function restoreSnapshotIfToday() {
    if (typeof window === "undefined") return;

    const raw = localStorage.getItem(SNAPSHOT_KEY);
    if (!raw) return;

    try {
      const snap = JSON.parse(raw) as Snapshot;
      if (snap.date !== todayUK) return;

      grid = snap.grid;
      activeRow = snap.activeRow;
      solved = snap.solved;
      gameOver = snap.gameOver;
      resultMessage = snap.resultMessage;
    } catch {
      // ignore bad data
    }
  }

  // restore board first
  restoreSnapshotIfToday();

  // then enforce lock message if needed
  if (typeof window !== "undefined") {
    const lastPlayed = localStorage.getItem(PLAY_KEY);
    if (lastPlayed === todayUK) {
      gameOver = true;
      if (!resultMessage || resultMessage === "Enter 4 digits to make your first guess.") {
        resultMessage = "You already played today. Come back tomorrow.";
      }
    }
  }

  function focusInput() {
    if (!gameOver) hiddenInput?.focus();
  }

  function scoreGuess(rawGuess: string, answerStr: string): CellStatus[] {
    const g = rawGuess.split("");
    const a = answerStr.split("");
    const statuses: CellStatus[] = Array(DIGITS).fill("absent");
    const used = Array(DIGITS).fill(false);

    for (let i = 0; i < DIGITS; i++) {
      if (g[i] === a[i]) {
        statuses[i] = "correct";
        used[i] = true;
      }
    }

    for (let i = 0; i < DIGITS; i++) {
      if (statuses[i] === "correct") continue;
      const idx = a.findIndex((ad, j) => !used[j] && ad === g[i]);
      if (idx !== -1) {
        statuses[i] = "present";
        used[idx] = true;
      }
    }

    return statuses;
  }

  function renderTypedRow() {
    for (let col = 0; col < DIGITS; col++) {
      const ch = current[col] ?? "";
      grid[activeRow][col] = { digit: ch, status: ch ? "typed" : "empty" };
    }
    grid = grid.map((r) => r.map((c) => ({ ...c })));
    saveSnapshot();
  }

  function commitRowIfComplete() {
    if (current.length !== DIGITS) return;

    const statuses = scoreGuess(current, answer);

    for (let col = 0; col < DIGITS; col++) {
      grid[activeRow][col] = { digit: current[col], status: statuses[col] };
    }
    grid = grid.map((r) => r.map((c) => ({ ...c })));

    if (statuses.every((s) => s === "correct")) {
      solved = true;
      gameOver = true;
      resultMessage = `🎉 You got it in ${activeRow + 1}/${MAX_GUESSES} guesses!`;
      localStorage.setItem(PLAY_KEY, todayUK);
      saveSnapshot();
      return;
    }

    activeRow += 1;
    current = "";

    if (activeRow >= MAX_GUESSES) {
      activeRow = MAX_GUESSES;
      gameOver = true;
      resultMessage = `Unlucky — out of guesses. Today's number was ${answer}.`;
      localStorage.setItem(PLAY_KEY, todayUK);
      saveSnapshot();
      return;
    }

    resultMessage = `Guess ${activeRow + 1} of ${MAX_GUESSES}`;
    renderTypedRow();
  }

  function onHiddenInput(e: Event) {
    if (gameOver) return;
    if (activeRow >= MAX_GUESSES) return;

    const value = (e.currentTarget as HTMLInputElement).value;
    current = value.replace(/\D/g, "").slice(0, DIGITS);

    if (hiddenInput) hiddenInput.value = current;
    renderTypedRow();

    if (current.length === DIGITS) {
      commitRowIfComplete();
      if (hiddenInput) hiddenInput.value = "";
    }
  }

  function onKeyDown(e: KeyboardEvent) {
    if (e.key === "Backspace") return;
  }
</script>

<section class="page">
  <div class="container">
    <h1 class="title">Guess the four digit number for today!</h1>
    <p class="status" aria-live="polite">{resultMessage}</p>

    <input
      class="hidden-capture"
      bind:this={hiddenInput}
      type="text"
      inputmode="numeric"
      autocomplete="one-time-code"
      aria-label="Type your guess"
      on:input={onHiddenInput}
      on:keydown={onKeyDown}
    />

    <button
      type="button"
      class="board-button"
      on:click={focusInput}
      on:focus={focusInput}
      aria-label="Guess board (click or focus to start typing)"
      disabled={gameOver}
    >
      <section class="board" aria-label="Guess board">
        {#each grid as row, r}
          <div class="row" aria-label={`Row ${r + 1}`}>
            {#each row as cell, c}
              <div
                class={[
                  "cell",
                  cell.status,
                  r === activeRow && !solved && activeRow < MAX_GUESSES ? "active-row" : ""
                ].join(" ")}
                aria-label={`Row ${r + 1}, col ${c + 1}`}
              >
                {cell.digit}
              </div>
            {/each}
          </div>
        {/each}
      </section>
    </button>
  </div>
</section>

<section class="container">
	<!-- game UI -->

	<p class="privacy-link">
		<a href="https://www.damper.dev/privacy-number-guess" target="_blank" rel="noopener noreferrer">
			Privacy
		</a>
	</p>
</section>

<style>
  :global(body) {
    margin: 0;
    background: #111;
    color: #eee;
    font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, sans-serif;
  }

  .page {
    min-height: 100vh;
    display: grid;
    place-items: center;
    padding: 24px;
  }

  .container {
    width: min(720px, 100%);
    display: grid;
    gap: 18px;
    justify-items: center;
  }

  .title {
    margin: 0;
    text-align: center;
    font-size: 28px;
    line-height: 1.15;
  }

  .status {
    margin: 0;
    min-height: 1.5em;
    text-align: center;
    color: #d8d8d8;
    font-weight: 600;
  }

  .hidden-capture {
    position: absolute;
    opacity: 0;
    pointer-events: none;
    width: 1px;
    height: 1px;
  }

  .board-button {
    background: transparent;
    border: 0;
    padding: 0;
    color: inherit;
  }

  .board-button:disabled {
    opacity: 0.7;
    cursor: not-allowed;
  }

  .board {
    display: grid;
    gap: 14px;
    user-select: none;
  }

  .row {
    display: flex;
    gap: 14px;
  }

  .cell {
    width: 60px;
    height: 60px;
    border-radius: 999px;
    display: grid;
    place-items: center;
    font-weight: 800;
    font-size: 22px;
    border: 2px solid #333;
    background: transparent;
  }

  .active-row {
    border-color: #666;
  }

  .typed {
    background: rgba(255, 255, 255, 0.06);
    border-color: #555;
  }

  .empty {
    background: transparent;
    border-color: #2a2a2a;
    color: transparent;
  }

  .absent {
    border-color: #444;
    color: #cfcfcf;
    background: rgba(255, 255, 255, 0.04);
  }

  .present {
    background: #b59f3b;
    border-color: #b59f3b;
    color: #111;
  }

  .correct {
    background: #538d4e;
    border-color: #538d4e;
    color: #111;
  }

.privacy-link {
	margin-top: 0.75rem;
	text-align: center;
	font-size: 0.85rem;
	opacity: 0.85;
}

.privacy-link a {
	color: #bdbdbd;
	text-decoration: none;
}

.privacy-link a:hover {
	text-decoration: underline;
}

</style>