# Number Guess Site

A simple daily 4-digit number guessing game built with **Svelte + TypeScript + Vite**.

## How it works

- Each day has one deterministic 4-digit answer.
- The daily answer is generated from the current **UK date** (`Europe/London`) so all players share the same daily number.
- You get **4 guesses** to solve it.
- After each guess:
  - **Green** = correct digit in the correct position
  - **Yellow** = digit exists but in a different position
  - **Gray** = digit is not in the answer

## Daily lock and local storage

This game stores progress in your browser only.

- `numberGuess_snapshot_v1` stores in-progress board state for today.
- `numberGuess_lastPlayedUK` locks replay once the daily game is finished.

If you already completed today’s game, the UI shows a lock message and asks you to come back tomorrow.

## Privacy

- No account is required.
- No server-side profile is used by the game logic.
- Data is stored locally in your browser for gameplay persistence.

Privacy page:
- https://www.damper.dev/privacy/number-guess

## Development

Install dependencies:

```bash
npm install
```

Run locally:

```bash
npm run dev
```

Build for production:

```bash
npm run build
```

Preview production build:

```bash
npm run preview
```
