<script>
  import { onMount, tick } from 'svelte';

  const TOTAL_SPACES = 15;
  const START_SPACE = 7;
  const LEFT_FINISH = 0;
  const RIGHT_FINISH = TOTAL_SPACES - 1;
  const FAN_START = 0;
  const FAN_BLOCK_COUNT = 6;
  const FAN_FINISH = FAN_BLOCK_COUNT + 1;
  const FAN_START_POSITION = { x: 90, y: 240 };
  const FAN_SPACE_X = [250, 335, 420, 505, 590, 675, 760];
  const DICE_FACES = ['⚀', '⚁', '⚂', '⚃', '⚄', '⚅'];
  const fanTracks = [120, 240, 360].map((laneY, playerIndex) => {
    const spaces = Array.from({ length: FAN_FINISH }, (_, index) => {
      return {
        index: index + 1,
        x: FAN_SPACE_X[index],
        y: laneY,
      };
    });
    const elbow = playerIndex === 0
      ? { x: 190, y: laneY }
      : playerIndex === 2
        ? { x: 190, y: laneY }
        : FAN_START_POSITION;
    return {
      playerIndex,
      color: ['#dc4f4b', '#3976d0', '#31845b'][playerIndex],
      laneY,
      spaces,
      elbow,
      finish: spaces[spaces.length - 1],
    };
  });
  const spaces = Array.from({ length: TOTAL_SPACES }, (_, index) => ({
    index,
    label: index === START_SPACE
      ? 'START'
      : index === LEFT_FINISH
        ? 'P2 WIN'
        : index === RIGHT_FINISH
          ? 'P1 WIN'
          : String.fromCharCode(65 + Math.abs(index - START_SPACE) - 1),
  }));

  let position = $state(START_SPACE);
  let fanPawnPosition = $state(FAN_START_POSITION);
  let fanPawnLane = $state(null);
  let fanPawnProgress = $state(FAN_START);
  let player = $state(1);
  let playerOneName = $state('Player 1');
  let playerTwoName = $state('Player 2');
  let playerThreeName = $state('Player 3');
  let playerCount = $state(2);
  let timerEnabled = $state(true);
  let settingsOpen = $state(false);
  let settingsDrawer = $state(null);
  let settingsButton = $state(null);
  let gameOver = $state(false);
  let moving = $state(false);
  let rolling = $state(false);
  let rolledValue = $state(null);
  let turnSeconds = $state(10);
  let pawnLeft = $state(0);
  let pawnPositioned = $state(false);
  let gameId = 0;
  let nextToastId = 0;
  let turnTimer;
  let toasts = $state([]);
  let boardElement = $state(null);
  let boardScrollElement = $state(null);

  $effect(() => {
    document.body.style.overflow = settingsOpen ? 'hidden' : '';
    return () => {
      document.body.style.overflow = '';
    };
  });

  const playerNames = $derived([playerOneName.trim() || 'Player 1', playerTwoName.trim() || 'Player 2', playerThreeName.trim() || 'Player 3']);
  const activePlayers = $derived(Array.from({ length: playerCount }, (_, index) => index + 1));
  const activePlayerName = $derived(playerNames[player - 1]);
  const activeFanTrack = $derived(fanTracks[player - 1]);
  const isWinner = $derived(playerCount === 3
    ? fanPawnLane === player - 1 && fanPawnProgress === FAN_FINISH
    : (player === 1 && position === RIGHT_FINISH) || (player === 2 && position === LEFT_FINISH));
  const statusTitle = $derived(gameOver ? `${activePlayerName} wins!` : `${activePlayerName}'s turn`);
  const statusDetail = $derived(gameOver ? 'Start a new game to play again' : `${['Red', 'Blue', 'Green'][player - 1]} to roll`);

  function updatePawnPosition() {
    const cell = boardElement?.querySelector(`[data-index="${position}"]`);
    if (cell) pawnLeft = cell.offsetLeft + cell.offsetWidth / 2;
  }

  function centerPosition(behavior = 'smooth') {
    const cell = boardElement?.querySelector(`[data-index="${position}"]`);
    if (!cell || !boardScrollElement) return;

    const center = cell.offsetLeft + cell.offsetWidth / 2;
    const left = Math.max(0, center - boardScrollElement.clientWidth / 2);
    boardScrollElement.scrollTo({ left, behavior });
  }

  function showToast(message) {
    const id = nextToastId++;
    toasts = [...toasts, { id, message }];
    setTimeout(() => {
      toasts = toasts.filter((item) => item.id !== id);
    }, 2500);
  }

  function wait(milliseconds) {
    return new Promise((resolve) => setTimeout(resolve, milliseconds));
  }

  function stopTurnTimer() {
    clearInterval(turnTimer);
    turnTimer = undefined;
  }

  function startTurnTimer() {
    stopTurnTimer();
    if (!timerEnabled) return;
    turnSeconds = 10;
    turnTimer = setInterval(() => {
      if (gameOver) return;
      if (turnSeconds <= 1) {
        advancePlayer();
        turnSeconds = 10;
        showToast(`Time's up. ${playerNames[player - 1]}'s turn`);
      } else {
        turnSeconds -= 1;
      }
    }, 1000);
  }

  function advancePlayer() {
    player = player >= playerCount ? 1 : player + 1;
  }

  function setPlayerCount(count) {
    if (playerCount === count) return;
    playerCount = count;
    startNewGame();
  }

  function setTimerEnabled(enabled) {
    timerEnabled = enabled;
    startNewGame();
  }

  async function rollDice() {
    if (gameOver || moving) return;

    stopTurnTimer();
    moving = true;
    rolling = true;
    const currentGame = gameId;
    const roll = 1 + Math.floor(Math.random() * 6);
    await wait(550);
    if (currentGame !== gameId) return;

    rolling = false;
    rolledValue = roll;
    showToast(`${activePlayerName} rolled a ${roll}`);

    const requiredScore = playerCount === 2
      ? player === 1 ? RIGHT_FINISH - position : position - LEFT_FINISH
      : fanPawnLane === player - 1
        ? FAN_FINISH - fanPawnProgress
        : fanPawnProgress + FAN_FINISH;
    const overshotFinish = roll > requiredScore;

    if (overshotFinish) {
      showToast(`${activePlayerName} needed ${requiredScore} to reach finish. Turn passed.`);
    } else if (playerCount === 2) {
      const direction = player === 1 ? 1 : -1;
      for (let step = 0; step < roll && position > LEFT_FINISH && position < RIGHT_FINISH; step += 1) {
        position += direction;
        await tick();
        updatePawnPosition();
        centerPosition();
        await wait(270);
        if (currentGame !== gameId) return;
      }
    } else {
      let headingOutward = fanPawnLane === null || fanPawnLane === player - 1;
      for (let step = 0; step < roll; step += 1) {
        if (!headingOutward && fanPawnLane !== null && fanPawnProgress > FAN_START) {
          fanPawnProgress -= 1;
          if (fanPawnProgress === FAN_START) {
            fanPawnLane = null;
            fanPawnPosition = FAN_START_POSITION;
            headingOutward = true;
          } else {
            fanPawnPosition = fanTracks[fanPawnLane].spaces[fanPawnProgress - 1];
          }
        } else {
          fanPawnLane = player - 1;
          fanPawnProgress += 1;
          fanPawnPosition = activeFanTrack.spaces[fanPawnProgress - 1];
        }
        await tick();
        await wait(270);
        if (currentGame !== gameId) return;
        if (fanPawnLane === player - 1 && fanPawnProgress === FAN_FINISH) break;
      }
    }

    if (isWinner) {
      gameOver = true;
      stopTurnTimer();
      showToast(`${activePlayerName} wins the race!`);
    } else {
      advancePlayer();
      startTurnTimer();
      showToast(`${activePlayerName}'s turn`);
    }
    moving = false;
  }

  async function startNewGame() {
    gameId += 1;
    stopTurnTimer();
    position = START_SPACE;
    fanPawnPosition = FAN_START_POSITION;
    fanPawnLane = null;
    fanPawnProgress = FAN_START;
    player = 1;
    gameOver = false;
    moving = false;
    rolling = false;
    rolledValue = null;
    toasts = [];
    await tick();
    if (playerCount === 3) {
      boardScrollElement?.scrollTo({ left: 0, behavior: 'auto' });
    } else {
      updatePawnPosition();
      centerPosition();
    }
    if (timerEnabled) startTurnTimer();
    showToast('New game started');
  }

  function closeSettings() {
    settingsOpen = false;
    tick().then(() => settingsButton?.focus());
  }

  async function openSettings() {
    settingsOpen = true;
    await tick();
    settingsDrawer?.querySelector('button')?.focus();
  }

  onMount(() => {
    updatePawnPosition();
    centerPosition('auto');
    requestAnimationFrame(() => {
      pawnPositioned = true;
    });
    startTurnTimer();
    const handleResize = () => {
      updatePawnPosition();
      centerPosition('auto');
    };
    const handleKeydown = (event) => {
      if (event.key === 'Escape' && settingsOpen) closeSettings();
    };
    window.addEventListener('resize', handleResize);
    window.addEventListener('keydown', handleKeydown);
    return () => {
      stopTurnTimer();
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('keydown', handleKeydown);
    };
  });
</script>

<main class="game-shell mx-auto w-[min(1200px,calc(100%-32px))] sm:w-[min(1200px,calc(100%-48px))]" class:three-player-layout={playerCount === 3}>
  <header class="game-header">
    <div class="game-brand">
      <h1>Dice Around</h1>
      <p>One pawn. Multiple players. Race to your finish.</p>
    </div>
    <div class="header-actions">
      <div class="player-tabs" aria-label="Players">
        {#each activePlayers as playerNumber (playerNumber)}
          <div class="player-tab" class:active={player === playerNumber} class:tab-red={playerNumber === 1} class:tab-blue={playerNumber === 2} class:tab-green={playerNumber === 3} aria-current={player === playerNumber ? 'true' : undefined}>
            <span class="player-dot"></span><span>{playerNames[playerNumber - 1]}</span>
          </div>
        {/each}
      </div>
      <button bind:this={settingsButton} class="settings-button" type="button" aria-label="Open game settings" aria-expanded={settingsOpen} onclick={openSettings}>
        <svg viewBox="0 0 24 24" aria-hidden="true"><path d="M12 8.5a3.5 3.5 0 1 0 0 7 3.5 3.5 0 0 0 0-7Z"/><path d="m19.4 15 .1.1 1.4 1.1-1.4 2.4-1.7-.7a8.3 8.3 0 0 1-1.5.9l-.3 1.8h-2.8l-.3-1.8a8.3 8.3 0 0 1-1.5-.9l-1.7.7-1.4-2.4 1.4-1.1a7 7 0 0 1 0-1.8l-1.4-1.1 1.4-2.4 1.7.7a8.3 8.3 0 0 1 1.5-.9l.3-1.8h2.8l.3 1.8a8.3 8.3 0 0 1 1.5.9l1.7-.7 1.4 2.4-1.4 1.1a7 7 0 0 1 0 1.7Z"/></svg>
      </button>
    </div>
  </header>

  {#if settingsOpen}
    <button class="drawer-scrim" type="button" aria-label="Close settings" onclick={closeSettings}></button>
    <div bind:this={settingsDrawer} class="settings-drawer" role="dialog" aria-modal="true" aria-labelledby="settings-title">
      <div class="drawer-header">
        <div><p>GAME CONFIGURATION</p><h2 id="settings-title">Settings</h2></div>
        <button class="drawer-close" type="button" aria-label="Close settings" onclick={closeSettings}>×</button>
      </div>

      <div class="drawer-section">
        <span class="drawer-label">Players</span>
        <div class="mode-switch" role="group" aria-label="Number of players">
          <button type="button" class:chosen={playerCount === 2} aria-pressed={playerCount === 2} onclick={() => setPlayerCount(2)}>2 players</button>
          <button type="button" class:chosen={playerCount === 3} aria-pressed={playerCount === 3} onclick={() => setPlayerCount(3)}>3 players</button>
        </div>
      </div>

      <div class="drawer-section timer-setting">
        <span class="timer-setting-copy"><strong>Turn timer</strong><small>10 seconds per turn</small></span>
        <input class="timer-toggle" type="checkbox" checked={timerEnabled} onchange={(event) => setTimerEnabled(event.currentTarget.checked)} aria-label="Enable 10-second turn timer" />
      </div>

      <div class="drawer-section name-settings-drawer">
        <span class="drawer-label">Player names</span>
        <label class="name-field"><span>Red player</span><input bind:value={playerOneName} maxlength="20" autocomplete="off" aria-label="Red player name" /></label>
        <label class="name-field"><span>Blue player</span><input bind:value={playerTwoName} maxlength="20" autocomplete="off" aria-label="Blue player name" /></label>
        {#if playerCount === 3}
          <label class="name-field"><span>Green player</span><input bind:value={playerThreeName} maxlength="20" autocomplete="off" aria-label="Green player name" /></label>
        {/if}
      </div>

      <p class="settings-note">Changing the player count or timer starts a new game.</p>
    </div>
  {/if}

  <section class="game-area" class:three-player-panel={playerCount === 3} aria-label="Dice Around game board">
    <div class="board-scroll" bind:this={boardScrollElement}>
      {#if playerCount === 2}
        <div class="board" bind:this={boardElement}>
          {#each spaces as space (space.index)}
            <div data-index={space.index} class="cell" class:start={space.index === START_SPACE} class:finish-left={space.index === LEFT_FINISH} class:finish-right={space.index === RIGHT_FINISH}>
              <span class="cell-label">{space.label}</span>
            </div>
          {/each}
          <div
            class="pawn"
            class:blue={player === 2}
            class:positioned={pawnPositioned}
            style:left={`${pawnLeft}px`}
            role="img"
            aria-label={`${activePlayerName} pawn, space ${position + 1}`}
          >
            <svg viewBox="0 0 48 68" width="100%" height="100%" aria-hidden="true">
              <circle cx="24" cy="13" r="10" fill="currentColor" />
              <path d="M17 25h14l-2 6c1 9 5 15 9 21H10c4-6 8-12 9-21z" fill="currentColor" />
              <path d="M10 52h28l3 5v5H7v-5z" fill="currentColor" />
              <path d="M15 29h18" fill="none" stroke="white" stroke-opacity=".3" stroke-width="2" stroke-linecap="round" />
              <path d="M12 57h24" fill="none" stroke="white" stroke-opacity=".25" stroke-width="2" stroke-linecap="round" />
            </svg>
          </div>
        </div>
      {:else}
        <div class="fan-board" aria-label="Three-player board with a shared start and three routes">
          <svg class="fan-art" viewBox="0 0 800 480" role="img" aria-label="Three race routes with a shared start and horizontal lanes">
            {#each fanTracks as track (track.playerIndex)}
              <polyline
                points={`${FAN_START_POSITION.x},${FAN_START_POSITION.y} ${track.elbow.x},${track.elbow.y} ${track.spaces[0].x},${track.spaces[0].y} ${track.finish.x},${track.finish.y}`}
                fill="none"
                stroke={track.color}
                stroke-opacity=".18"
                stroke-width="20"
                stroke-linejoin="round"
                stroke-linecap="round"
              />
              <polyline
                points={`${FAN_START_POSITION.x},${FAN_START_POSITION.y} ${track.elbow.x},${track.elbow.y} ${track.spaces[0].x},${track.spaces[0].y} ${track.finish.x},${track.finish.y}`}
                fill="none"
                stroke={track.color}
                stroke-opacity=".72"
                stroke-width="2"
                stroke-linejoin="round"
                stroke-linecap="round"
              />
              {#each track.spaces as fanSpace (fanSpace.index)}
                <circle cx={fanSpace.x} cy={fanSpace.y} r="15" fill={fanSpace.index === FAN_FINISH ? track.color : '#ffffff'} stroke={track.color} stroke-width="2" />
                <text x={fanSpace.x} y={fanSpace.y + 3} text-anchor="middle" class="fan-space-label">{fanSpace.index === FAN_FINISH ? 'FIN' : String.fromCharCode(64 + fanSpace.index)}</text>
              {/each}
              <text x={track.spaces[0].x} y={track.laneY - 28} text-anchor="start" fill={track.color} class="fan-player-label">{playerNames[track.playerIndex]}</text>
            {/each}
            <circle cx={FAN_START_POSITION.x} cy={FAN_START_POSITION.y} r="28" fill="#ffffff" stroke="#8ba18e" stroke-width="3" />
            <text x={FAN_START_POSITION.x} y={FAN_START_POSITION.y + 4} text-anchor="middle" class="fan-start-label">START</text>
          </svg>
          <div
            class="fan-pawn"
            class:at-start={fanPawnLane === null}
            style:left={`${(fanPawnPosition.x / 800) * 100}%`}
            style:top={`${(fanPawnPosition.y / 480) * 100}%`}
            style:--pawn-color={activeFanTrack.color}
            role="img"
            aria-label={`Shared pawn, ${activePlayerName}'s turn, ${fanPawnLane === null ? 'shared start' : `${playerNames[fanPawnLane]} lane, space ${fanPawnProgress}`}`}
          >
            <svg viewBox="0 0 48 68" aria-hidden="true">
              <circle cx="24" cy="13" r="10" fill="currentColor" />
              <path d="M17 25h14l-2 6c1 9 5 15 9 21H10c4-6 8-12 9-21z" fill="currentColor" />
              <path d="M10 52h28l3 5v5H7v-5z" fill="currentColor" />
            </svg>
          </div>
        </div>
      {/if}
    </div>

  </section>

  <div class="controls-dock">
    <div class="controls-inner">
      <button class="roll-button" type="button" onclick={rollDice} disabled={gameOver || moving}>
        <span aria-hidden="true">⚄</span> Roll dice
      </button>
      <div class="dice" class:rolling role="img" aria-label={rolling ? 'Rolling dice' : rolledValue ? `Dice rolled ${rolledValue}` : 'Dice not rolled'}>
        {rolledValue ? DICE_FACES[rolledValue - 1] : '⚄'}
      </div>
      <div class="control-status" role="status" aria-live="polite">
        {statusTitle}<small>{statusDetail}</small>
      </div>
      {#if timerEnabled}
        <div class="turn-timer dock-turn-timer" class:urgent={turnSeconds <= 3} role="timer" aria-label={`${activePlayerName}, ${turnSeconds} seconds remaining`}>
          <span>{activePlayerName}</span>
          <strong>{turnSeconds}<small>s</small></strong>
          <div class="timer-track" aria-hidden="true"><div class="timer-fill" style:width={`${turnSeconds * 10}%`}></div></div>
        </div>
      {/if}
      <button class="reset-button" type="button" onclick={startNewGame}>New game</button>
    </div>
  </div>
</main>

<div class="toast-container" aria-live="polite" aria-atomic="false">
  {#each toasts as toast (toast.id)}
    <div class="toast">{toast.message}</div>
  {/each}
</div>