# Dice Around

A browser-based local multiplayer racing game built with Svelte and Vite.

**One pawn. Multiple players. Race to your finish.**

## Gameplay

- Play with **2 or 3 players**.
- Customize each player's name from the settings drawer.
- Take turns rolling a six-sided die and move the shared pawn toward your finish.
- The player who reaches their finish exactly wins.
- Rolls that overshoot the finish are not applied and pass the turn to the next player.
- Start a new game at any time with the **New game** button.

### 2-player mode

- The red player races from the center toward the right-hand finish.
- The blue player races from the center toward the left-hand finish.
- The board has 15 spaces, including the shared start and both finish spaces.

### 3-player mode

- Red, blue, and green players share the same center start.
- Each player follows their own colored route outward toward a finish.
- The shared pawn moves backward along a route until it reaches the start, if needed, before continuing on the active player's route.

## Features

- Animated dice rolls and pawn movement
- Configurable 2-player or 3-player mode
- Optional 10-second turn timer
- Custom player names
- Automatic turn switching
- Responsive board layouts for desktop and mobile
- Toast feedback for rolls, turn changes, timeouts, and wins
- Reduced-motion support
- Keyboard-accessible settings drawer and controls

## Tech stack

- [Svelte 5](https://svelte.dev/)
- [Vite 6](https://vite.dev/)
- [Tailwind CSS 4](https://tailwindcss.com/)

## Getting started

### Prerequisites

- Node.js 18 or newer
- npm

### Installation

```bash
npm install
```

### Development

```bash
npm run dev
```

Vite prints the local development URL in the terminal, usually `http://localhost:5173`.

### Production build

```bash
npm run build
```

The production files are written to the `dist` directory.

### Preview the production build

```bash
npm run preview
```
