<script lang="ts">
  // --- Daily answer logic (yours) ---
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

  // --- Game config ---
  const DIGITS = 4;
  const MAX_GUESSES = 4;

  type CellStatus = "empty" | "typed" | "correct" | "present" | "absent";
  type Cell = { digit: string; status: CellStatus };

  // Grid state: 4 rows × 4 cells
  let grid: Cell[][] = Array.from({ length: MAX_GUESSES }, () =>
    Array.from({ length: DIGITS }, () => ({ digit: "", status: "empty" as CellStatus }))
  );

  // Which row is currently being typed into
  let activeRow = 0;

  // Keep track of whether game is solved
  let solved = false;

  // Hidden input value (what user is typing for the active row)
  let current = "";

  let hiddenInput: HTMLInputElement | null = null;

  function focusInput() {
    hiddenInput?.focus();
  }

  function scoreGuess(rawGuess: string, answerStr: string): CellStatus[] {
    const g = rawGuess.split("");
    const a = answerStr.split("");

    const statuses: CellStatus[] = Array(DIGITS).fill("absent");
    const used = Array(DIGITS).fill(false);

    // Pass 1: correct
    for (let i = 0; i < DIGITS; i++) {
      if (g[i] === a[i]) {
        statuses[i] = "correct";
        used[i] = true;
      }
    }

    // Pass 2: present
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
    // Update the visible circles in the active row as the user types
    for (let col = 0; col < DIGITS; col++) {
      const ch = current[col] ?? "";
      grid[activeRow][col] = {
        digit: ch,
        status: ch ? "typed" : "empty"
      };
    }
    // force reactivity by reassigning grid (because we mutate nested arrays)
    grid = grid.map((r) => r.map((c) => ({ ...c })));
  }

  function commitRowIfComplete() {
    if (current.length !== DIGITS) return;

    const statuses = scoreGuess(current, answer);

    for (let col = 0; col < DIGITS; col++) {
      grid[activeRow][col] = {
        digit: current[col],
        status: statuses[col]
      };
    }
    grid = grid.map((r) => r.map((c) => ({ ...c })));

    if (statuses.every((s) => s === "correct")) {
      solved = true;
      return;
    }

    activeRow += 1;
    current = "";

    // If out of rows, stop
    if (activeRow >= MAX_GUESSES) {
      activeRow = MAX_GUESSES; // lock
      return;
    }

    // Prepare next row (clear it to empty)
    renderTypedRow();
  }

  function onHiddenInput(e: Event) {
    if (solved) return;
    if (activeRow >= MAX_GUESSES) return;

    const value = (e.currentTarget as HTMLInputElement).value;

    // digits only, max length 4
    current = value.replace(/\D/g, "").slice(0, DIGITS);

    // keep hidden input in sync (so backspace/paste behaves predictably)
    if (hiddenInput) hiddenInput.value = current;

    renderTypedRow();

    // auto-submit when 4 digits typed
    if (current.length === DIGITS) {
      commitRowIfComplete();
      // clear hidden input after commit
      if (hiddenInput) hiddenInput.value = "";
    }
  }

  function onKeyDown(e: KeyboardEvent) {
    // allow backspace even when hidden input is empty but we still have current
    if (e.key === "Backspace") {
      // Let the browser do default backspace in the hidden input.
      // We just sync on input event.
      return;
    }
  }
</script>

<section class="page">
  <div class="container">
    <h1 class="title">Guess the four digit number for today!</h1>

    <!-- Hidden input: captures typing reliably (desktop + mobile) -->
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

    <!-- For debugging while building; remove later -->
    <!-- <p>answer: {answer}</p> -->
  </div>
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

  /* hidden input that still receives focus/keyboard */
  .hidden-capture {
    position: absolute;
    opacity: 0;
    pointer-events: none;
    width: 1px;
    height: 1px;
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

  /* the row currently being typed into */
  .active-row {
    border-color: #666;
  }

  /* typing (not submitted yet) */
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
</style>