<script>
  import { onMount } from 'svelte';

  // Game state - Load from localStorage or use defaults
  let balance = 200;
  let initialBalance = 200;
  let targetGoal = 100000;
  let isEditingTarget = false;
  let isEditingBalance = false;
  let targetInput = targetGoal;
  let balanceInput = balance;
  let lotSize = 0.01;
  let grid = [];
  let nextGrid = []; // Pre-arranged grid for next attempt
  let revealedBox = null;
  let gameActive = true;
  let isProcessing = false; // Prevents any clicks during processing
  let lastResult = null;
  let revealTimeout = null;

  // Settings
  let showSettings = false;
  let winMultiplier = 100; // $1 for 0.01 lot size
  let lossMultiplier = 500; // $10 for 0.01 lot size
  let prizePercentage = 83; // 90% prizes, 10% bombs
  let maxLeverage = 2000; // Maximum leverage allowed
  const LOT_VALUE = 100000; // 1 lot = $100,000

  // Settings inputs (for editing)
  let winMultiplierInput = winMultiplier;
  let lossMultiplierInput = lossMultiplier;
  let prizePercentageInput = prizePercentage;
  let maxLeverageInput = maxLeverage;
  let initialBalanceInput = initialBalance;

  // Calculate maximum allowed lot size based on balance and leverage
  function getMaxLotSize() {
    // Cap 1: Based on max loss not exceeding balance
    const maxLotFromBalance = balance / lossMultiplier;

    // Cap 2: Based on max leverage
    const positionValue = lotSize * LOT_VALUE;
    const maxPositionValue = balance * maxLeverage;
    const maxLotFromLeverage = maxPositionValue / LOT_VALUE;

    // Return the minimum of both caps
    return Math.min(maxLotFromBalance, maxLotFromLeverage);
  }

  // Calculate current leverage
  $: currentLeverage = balance > 0 ? (lotSize * LOT_VALUE / balance).toFixed(1) : 0;

  // Initialize grid
  function createGrid() {
    let boxes = [];
    const bombCount = Math.round((100 - prizePercentage) * 100 / 100);
    const prizeCount = 100 - bombCount;

    for (let i = 0; i < bombCount; i++) {
      boxes.push({ type: 'bomb', revealed: false });
    }
    for (let i = 0; i < prizeCount; i++) {
      boxes.push({ type: 'prize', revealed: false });
    }

    // Shuffle the array
    for (let i = boxes.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [boxes[i], boxes[j]] = [boxes[j], boxes[i]];
    }

    return boxes;
  }

  function initializeGrid() {
    grid = createGrid();
    nextGrid = createGrid(); // Pre-arrange next grid
    revealedBox = null;
    isProcessing = false;
  }

  // Handle box click
  function handleBoxClick(index) {
    const clickTime = Date.now();
    console.log(`Click on box ${index} at ${clickTime}, isProcessing: ${isProcessing}, revealedBox: ${revealedBox}`);

    if (!gameActive) {
      console.log('Game not active, ignoring click');
      return;
    }

    // If currently processing, immediately reset and process new click
    if (isProcessing) {
      console.log('Already processing, immediately resetting for new click...');

      // Clear the timeout
      if (revealTimeout) {
        clearTimeout(revealTimeout);
        revealTimeout = null;
      }

      // Reset the revealed box
      revealedBox = null;

      // Use the pre-arranged next grid
      grid = nextGrid;
      nextGrid = createGrid();

      console.log('Grid reset, processing new click on box', index);
    }

    // Start processing
    isProcessing = true;
    const box = grid[index];

    // Set the revealed box
    revealedBox = index;
    console.log(`Revealing box ${index}, type: ${box.type}`);

    // Calculate win/loss
    if (box.type === 'bomb') {
      balance = Math.max(0, balance - (lossMultiplier * lotSize));
      lastResult = 'loss';
    } else {
      balance = balance + (winMultiplier * lotSize);
      lastResult = 'win';
    }

    // Save to localStorage
    saveGameState();

    // Check if game should continue
    if (balance <= 0) {
      gameActive = false;
    }

    // Check if target reached
    if (balance >= targetGoal) {
      gameActive = false;
    }

    // Validate lot size after balance change
    validateLotSize();

    // After showing the result, flip back and reshuffle
    revealTimeout = setTimeout(() => {
      const resetTime = Date.now();
      console.log(`Timeout triggered at ${resetTime}, resetting grid...`);
      revealedBox = null;

      // Use pre-arranged grid
      if (gameActive) {
        grid = nextGrid;
        nextGrid = createGrid();
      }

      // Allow new clicks after grid is reset
      isProcessing = false;
      console.log('Grid reset complete, ready for new clicks');
    }, 1200); // Show result for 1.2 seconds
  }

  // Validate and adjust lot size if needed
  function validateLotSize() {
    const maxLot = getMaxLotSize();
    if (lotSize > maxLot) {
      lotSize = Math.max(0.01, Math.floor(maxLot * 100) / 100); // Round down to 2 decimals
    }
  }

  // Reset game
  function resetGame() {
    balance = initialBalance;
    gameActive = true;
    lotSize = 0.01;
    localStorage.removeItem('riskManagementBalance');
    initializeGrid();
    validateLotSize();
  }

  // Update lot size
  function updateLotSize(e) {
    const value = parseFloat(e.target.value);
    if (!isNaN(value) && value >= 0.01) {
      const maxLot = getMaxLotSize();
      lotSize = Math.min(value, maxLot);
    }
  }

  // Format currency
  function formatCurrency(amount) {
    return `$${amount.toFixed(2).replace(/\B(?=(\d{3})+(?!\d))/g, ",")}`;
  }

  // Target editing functions
  function startEditingTarget() {
    isEditingTarget = true;
    targetInput = targetGoal;
  }

  function saveTarget() {
    const value = parseFloat(targetInput);
    if (!isNaN(value) && value > 0) {
      targetGoal = value;
      localStorage.setItem('riskManagementTarget', targetGoal.toString());
    }
    isEditingTarget = false;
  }

  function cancelEditingTarget() {
    isEditingTarget = false;
    targetInput = targetGoal;
  }

  // Balance editing functions
  function startEditingBalance() {
    isEditingBalance = true;
    balanceInput = balance;
  }

  function saveBalance() {
    const value = parseFloat(balanceInput);
    if (!isNaN(value) && value >= 0) {
      balance = value;
      saveGameState();
      validateLotSize();
      if (balance <= 0) {
        gameActive = false;
      } else {
        gameActive = true;
      }
    }
    isEditingBalance = false;
  }

  function cancelEditingBalance() {
    isEditingBalance = false;
    balanceInput = balance;
  }

  // Settings functions
  function openSettings() {
    showSettings = true;
    winMultiplierInput = winMultiplier;
    lossMultiplierInput = lossMultiplier;
    prizePercentageInput = prizePercentage;
    maxLeverageInput = maxLeverage;
    initialBalanceInput = initialBalance;
  }

  function saveSettings() {
    const winVal = parseFloat(winMultiplierInput);
    const lossVal = parseFloat(lossMultiplierInput);
    const prizeVal = parseFloat(prizePercentageInput);
    const leverageVal = parseFloat(maxLeverageInput);
    const initialVal = parseFloat(initialBalanceInput);

    if (!isNaN(winVal) && winVal > 0) {
      winMultiplier = winVal;
      localStorage.setItem('riskManagementWinMultiplier', winMultiplier.toString());
    }

    if (!isNaN(lossVal) && lossVal > 0) {
      lossMultiplier = lossVal;
      localStorage.setItem('riskManagementLossMultiplier', lossMultiplier.toString());
    }

    if (!isNaN(prizeVal) && prizeVal > 0 && prizeVal <= 100) {
      prizePercentage = prizeVal;
      localStorage.setItem('riskManagementPrizePercentage', prizePercentage.toString());
      // Recreate grids with new percentage
      initializeGrid();
    }

    if (!isNaN(leverageVal) && leverageVal > 0) {
      maxLeverage = leverageVal;
      localStorage.setItem('riskManagementMaxLeverage', maxLeverage.toString());
    }

    if (!isNaN(initialVal) && initialVal > 0) {
      initialBalance = initialVal;
      localStorage.setItem('riskManagementInitialBalance', initialBalance.toString());
    }

    validateLotSize();
    showSettings = false;
  }

  function cancelSettings() {
    showSettings = false;
  }

  // Save game state to localStorage
  function saveGameState() {
    localStorage.setItem('riskManagementBalance', balance.toString());
  }

  // Load game state from localStorage
  function loadGameState() {
    const savedBalance = localStorage.getItem('riskManagementBalance');
    const savedTarget = localStorage.getItem('riskManagementTarget');
    const savedWinMultiplier = localStorage.getItem('riskManagementWinMultiplier');
    const savedLossMultiplier = localStorage.getItem('riskManagementLossMultiplier');
    const savedPrizePercentage = localStorage.getItem('riskManagementPrizePercentage');
    const savedMaxLeverage = localStorage.getItem('riskManagementMaxLeverage');
    const savedInitialBalance = localStorage.getItem('riskManagementInitialBalance');

    if (savedBalance !== null) {
      balance = parseFloat(savedBalance);
      if (balance <= 0) gameActive = false;
    }

    if (savedTarget !== null) {
      targetGoal = parseFloat(savedTarget);
      targetInput = targetGoal;
    }

    if (savedWinMultiplier !== null) {
      winMultiplier = parseFloat(savedWinMultiplier);
    }

    if (savedLossMultiplier !== null) {
      lossMultiplier = parseFloat(savedLossMultiplier);
    }

    if (savedPrizePercentage !== null) {
      prizePercentage = parseFloat(savedPrizePercentage);
    }

    if (savedMaxLeverage !== null) {
      maxLeverage = parseFloat(savedMaxLeverage);
    }

    if (savedInitialBalance !== null) {
      initialBalance = parseFloat(savedInitialBalance);
    }
  }

  // Calculate risk to reward ratio
  $: riskRewardRatio = (lossMultiplier / winMultiplier).toFixed(1);

  onMount(() => {
    loadGameState();
    initializeGrid();
    validateLotSize();
  });
</script>

<div class="game-container">
  <header class="game-header">
    <div class="title">
      <h1>The Game of <span class="risk-text">Risk</span></h1>
    </div>
    <div class="header-right">
      <div class="stats">
        <div class="target-goal">
          <span class="target-label">Target:</span>
          {#if isEditingTarget}
            <div class="target-edit">
              <input
                type="number"
                bind:value={targetInput}
                on:keydown={(e) => {
                  if (e.key === 'Enter') saveTarget();
                  if (e.key === 'Escape') cancelEditingTarget();
                }}
                class="target-input"
                min="100"
                step="100"
                autofocus
              />
              <button class="save-btn" on:click={saveTarget}>✓</button>
              <button class="cancel-btn" on:click={cancelEditingTarget}>✕</button>
            </div>
          {:else}
            <button
              class="target-amount"
              class:achieved={balance >= targetGoal}
              on:click={startEditingTarget}
            >
              {formatCurrency(targetGoal)}
            </button>
          {/if}
        </div>
        <div class="balance">
          <span class="balance-label">Balance:</span>
          {#if isEditingBalance}
            <div class="balance-edit">
              <input
                type="number"
                bind:value={balanceInput}
                on:keydown={(e) => {
                  if (e.key === 'Enter') saveBalance();
                  if (e.key === 'Escape') cancelEditingBalance();
                }}
                class="balance-input"
                min="0"
                step="10"
                autofocus
              />
              <button class="save-btn" on:click={saveBalance}>✓</button>
              <button class="cancel-btn" on:click={cancelEditingBalance}>✕</button>
            </div>
          {:else}
            <button
              class="balance-amount"
              class:danger={balance < 20}
              class:success={balance >= targetGoal}
              on:click={startEditingBalance}
            >
              {formatCurrency(balance)}
            </button>
          {/if}
        </div>
      </div>
      <button class="settings-btn" on:click={openSettings} title="Settings">
        ⚙️
      </button>
    </div>
  </header>

  <div class="controls">
    <div class="lot-size-control">
      <label for="lotSize">Lot Size:</label>
      <input
        id="lotSize"
        type="number"
        min="0.01"
        max={getMaxLotSize().toFixed(2)}
        step="0.01"
        bind:value={lotSize}
        on:change={updateLotSize}
        disabled={!gameActive}
      />
      <span class="lot-info">
        Win: {formatCurrency(winMultiplier * lotSize)} | Lose: {formatCurrency(lossMultiplier * lotSize)}
      </span>
    </div>
    <button class="reset-button" on:click={resetGame}>
      🔄 New Game
    </button>
  </div>

  {#if balance >= targetGoal}
    <div class="target-reached">
      <h2>🎉 Target Reached! 🎉</h2>
      <p>Congratulations! You've reached your goal of {formatCurrency(targetGoal)}!</p>
    </div>
  {/if}

  <div class="game-grid">
    {#each grid as box, index}
      <button
        class="game-box"
        class:flipping={revealedBox === index}
        class:bomb={revealedBox === index && box.type === 'bomb'}
        class:prize={revealedBox === index && box.type === 'prize'}
        disabled={!gameActive || isProcessing}
        on:click={() => handleBoxClick(index)}
      >
        <div class="box-inner">
          <div class="box-front">
            <span class="icon question-icon">❓</span>
          </div>
          <div class="box-back">
            {#if revealedBox === index}
              {#if box.type === 'bomb'}
                <span class="icon bomb-icon">💣</span>
              {:else}
                <span class="icon prize-icon">💰</span>
              {/if}
            {/if}
          </div>
        </div>
      </button>
    {/each}
  </div>

  {#if balance <= 0}
    <div class="game-over">
      <h2>Game Over! 😢</h2>
      <p>You've run out of money!</p>
    </div>
  {/if}
</div>

<!-- Settings Modal -->
{#if showSettings}
  <div class="modal-overlay" on:click={cancelSettings}>
    <div class="modal" on:click|stopPropagation>
      <h2>Settings</h2>

      <div class="settings-group">
        <label>
          Initial Balance:
          <input
            type="number"
            bind:value={initialBalanceInput}
            min="10"
            step="10"
          />
          <span class="helper-text">Starting balance for new games (default: $200)</span>
        </label>
      </div>

      <div class="settings-group">
        <label>
          Maximum Leverage:
          <input
            type="number"
            bind:value={maxLeverageInput}
            min="1"
            step="100"
          />
          <span class="helper-text">Maximum leverage allowed (default: 2000x)</span>
        </label>
      </div>

      <div class="settings-group">
        <label>
          Win Amount (per lot):
          <input
            type="number"
            bind:value={winMultiplierInput}
            min="1"
            step="10"
          />
          <span class="helper-text">Amount won per lot size (default: 100 = $1 for 0.01 lot)</span>
        </label>
      </div>

      <div class="settings-group">
        <label>
          Loss Amount (per lot):
          <input
            type="number"
            bind:value={lossMultiplierInput}
            min="1"
            step="10"
          />
          <span class="helper-text">Amount lost per lot size (default: 1000 = $10 for 0.01 lot)</span>
        </label>
      </div>

      <div class="settings-group">
        <label>
          Safe Boxes Percentage:
          <input
            type="number"
            bind:value={prizePercentageInput}
            min="1"
            max="99"
            step="1"
          />
          <span class="helper-text">Percentage of safe boxes (default: 90%)</span>
        </label>
      </div>

      <div class="settings-preview">
        <p>With these settings and 0.01 lot size:</p>
        <p>• Win: {formatCurrency(winMultiplierInput * 0.01)}</p>
        <p>• Lose: {formatCurrency(lossMultiplierInput * 0.01)}</p>
        <p>• Risk:Reward Ratio: 1:{(lossMultiplierInput / winMultiplierInput).toFixed(1)}</p>
        <p>• {prizePercentageInput} safe boxes, {100 - prizePercentageInput} bombs</p>
        <p>• 1 lot = {formatCurrency(LOT_VALUE)} position</p>
      </div>

      <div class="modal-buttons">
        <button class="save-settings-btn" on:click={saveSettings}>Save</button>
        <button class="cancel-settings-btn" on:click={cancelSettings}>Cancel</button>
      </div>
    </div>
  </div>
{/if}

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif;
    background-color: #1e1e2e; /* Catppuccin Mocha Base */
    color: #cdd6f4; /* Catppuccin Mocha Text */
    min-height: 100vh;
  }

  .game-container {
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
  }

  .game-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 30px;
    padding: 20px;
    background: #181825; /* Catppuccin Mocha Mantle */
    border-radius: 12px;
    border: 2px solid #313244; /* Catppuccin Mocha Surface0 */
    position: relative;
  }

  .title h1 {
    margin: 0;
    font-size: 1.8rem;
    color: #cdd6f4; /* Default text color */
  }

  .risk-text {
    background: linear-gradient(135deg, #8b0000, #dc143c, #ff0000, #b22222); /* Bloody gradient */
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
    font-weight: bold;
    text-shadow: 2px 2px 4px rgba(139, 0, 0, 0.3);
    animation: bloodPulse 3s ease-in-out infinite;
  }

  @keyframes bloodPulse {
    0%, 100% {
      filter: brightness(1) saturate(1);
    }
    50% {
      filter: brightness(1.2) saturate(1.5);
    }
  }

  .header-right {
    display: flex;
    align-items: center;
    gap: 30px;
  }

  .stats {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 10px;
  }

  .target-goal {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .target-label {
    font-size: 0.9rem;
    color: #a6adc8; /* Catppuccin Mocha Subtext0 */
  }

  .target-amount {
    font-size: 1.2rem;
    font-weight: 500;
    color: #f9e2af; /* Catppuccin Mocha Yellow */
    background: none;
    border: none;
    cursor: pointer;
    padding: 4px 8px;
    border-radius: 6px;
    transition: all 0.2s;
  }

  .target-amount:hover {
    background: rgba(249, 226, 175, 0.1);
  }

  .target-amount.achieved {
    color: #a6e3a1; /* Catppuccin Mocha Green */
  }

  .settings-btn {
    background: none;
    border: none;
    cursor: pointer;
    font-size: 1.2rem;
    padding: 6px;
    border-radius: 8px;
    transition: all 0.2s;
  }

  .settings-btn:hover {
    background: rgba(137, 180, 250, 0.1);
    transform: rotate(45deg);
  }

  .target-edit, .balance-edit {
    display: flex;
    gap: 5px;
    align-items: center;
  }

  .target-input, .balance-input {
    padding: 4px 8px;
    border-radius: 6px;
    border: 2px solid #45475a; /* Catppuccin Mocha Surface1 */
    background: #1e1e2e; /* Catppuccin Mocha Base */
    color: #cdd6f4; /* Catppuccin Mocha Text */
    font-size: 1rem;
    width: 120px;
  }

  .save-btn, .cancel-btn {
    padding: 4px 8px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 0.9rem;
  }

  .save-btn {
    background: #a6e3a1; /* Catppuccin Mocha Green */
    color: #1e1e2e;
  }

  .cancel-btn {
    background: #f38ba8; /* Catppuccin Mocha Red */
    color: #1e1e2e;
  }

  .balance {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 5px;
  }

  .balance-label {
    font-size: 0.9rem;
    color: #a6adc8; /* Catppuccin Mocha Subtext0 */
  }

  .balance-amount {
    font-size: 1.8rem;
    font-weight: bold;
    color: #a6e3a1; /* Catppuccin Mocha Green */
    background: none;
    border: none;
    cursor: pointer;
    padding: 4px 8px;
    border-radius: 6px;
    transition: all 0.3s ease;
  }

  .balance-amount:hover {
    background: rgba(166, 227, 161, 0.1);
  }

  .balance-amount.danger {
    color: #f38ba8; /* Catppuccin Mocha Red */
    animation: pulse 1s infinite;
  }

  .balance-amount.success {
    color: #a6e3a1; /* Catppuccin Mocha Green */
    animation: glow 2s ease-in-out infinite;
  }

  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.7; }
  }

  @keyframes glow {
    0%, 100% { text-shadow: 0 0 10px rgba(166, 227, 161, 0.5); }
    50% { text-shadow: 0 0 20px rgba(166, 227, 161, 0.8); }
  }

  .controls {
    display: flex;
    gap: 20px;
    align-items: center;
    margin-bottom: 30px;
    padding: 20px;
    background: #313244; /* Catppuccin Mocha Surface0 */
    border-radius: 12px;
    flex-wrap: wrap;
  }

  .lot-size-control {
    display: flex;
    align-items: center;
    gap: 10px;
    flex: 1;
    min-width: 200px;
  }

  .lot-size-control label {
    font-weight: 500;
    color: #bac2de; /* Catppuccin Mocha Subtext1 */
  }

  .lot-size-control input {
    padding: 8px 12px;
    border-radius: 8px;
    border: 2px solid #45475a; /* Catppuccin Mocha Surface1 */
    background: #1e1e2e; /* Catppuccin Mocha Base */
    color: #cdd6f4; /* Catppuccin Mocha Text */
    font-size: 1rem;
    width: 100px;
    transition: all 0.3s ease;
  }

  .lot-size-control input:focus {
    outline: none;
    border-color: #89b4fa; /* Catppuccin Mocha Blue */
    box-shadow: 0 0 0 3px rgba(137, 180, 250, 0.2);
  }

  .lot-size-control input:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .lot-info {
    font-size: 0.9rem;
    color: #7f849c; /* Catppuccin Mocha Overlay1 */
  }

  .reset-button {
    padding: 10px 20px;
    font-size: 1rem;
    font-weight: 500;
    border: none;
    border-radius: 8px;
    background: linear-gradient(135deg, #89b4fa, #89dceb); /* Blue to Sky */
    color: #1e1e2e; /* Catppuccin Mocha Base */
    cursor: pointer;
    transition: all 0.3s ease;
  }

  .reset-button:hover {
    transform: translateY(-2px);
    box-shadow: 0 5px 15px rgba(137, 180, 250, 0.4);
  }

  .target-reached {
    margin-bottom: 20px;
    padding: 20px;
    background: linear-gradient(135deg, rgba(166, 227, 161, 0.2), rgba(148, 226, 213, 0.2));
    border: 2px solid #a6e3a1; /* Catppuccin Mocha Green */
    border-radius: 12px;
    text-align: center;
  }

  .target-reached h2 {
    color: #a6e3a1; /* Catppuccin Mocha Green */
    margin-top: 0;
  }

  .game-grid {
    display: grid;
    grid-template-columns: repeat(10, 1fr);
    gap: 8px;
    padding: 20px;
    background: #181825; /* Catppuccin Mocha Mantle */
    border-radius: 12px;
    border: 2px solid #313244; /* Catppuccin Mocha Surface0 */
  }

  .game-box {
    aspect-ratio: 1;
    border: none;
    border-radius: 8px;
    background: transparent;
    cursor: pointer;
    font-size: 1.5rem;
    padding: 0;
    position: relative;
    perspective: 1000px;
  }

  .game-box:hover:not(:disabled) .box-inner {
    transform: scale(1.05);
  }

  .game-box:disabled {
    cursor: default;
  }

  .box-inner {
    position: relative;
    width: 100%;
    height: 100%;
    transition: transform 0.5s cubic-bezier(0.4, 0, 0.2, 1);
    transform-style: preserve-3d;
  }

  .game-box.flipping .box-inner {
    transform: rotateY(180deg);
  }

  .box-front, .box-back {
    position: absolute;
    width: 100%;
    height: 100%;
    backface-visibility: hidden;
    border-radius: 8px;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.3s ease;
  }

  .box-front {
    background: #313244; /* Catppuccin Mocha Surface0 */
  }

  .box-back {
    background: #313244; /* Default background */
    transform: rotateY(180deg);
  }

  .game-box.bomb .box-back {
    background: linear-gradient(135deg, #f38ba8, #eba0ac); /* Red to Maroon */
  }

  .game-box.prize .box-back {
    background: linear-gradient(135deg, #a6e3a1, #94e2d5); /* Green to Teal */
  }

  .icon {
    font-size: 1.2rem;
    display: block;
    line-height: 1;
  }

  .question-icon {
    filter: grayscale(0.3);
  }

  .bomb-icon, .prize-icon {
    font-size: 1.5rem;
  }

  .game-over {
    margin-top: 30px;
    padding: 20px;
    background: linear-gradient(135deg, rgba(243, 139, 168, 0.2), rgba(235, 160, 172, 0.2));
    border: 2px solid #f38ba8; /* Catppuccin Mocha Red */
    border-radius: 12px;
    text-align: center;
  }

  .game-over h2 {
    color: #f38ba8; /* Catppuccin Mocha Red */
    margin-top: 0;
  }

  /* Settings Modal */
  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(17, 17, 27, 0.8);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }

  .modal {
    background: #1e1e2e; /* Catppuccin Mocha Base */
    border: 2px solid #313244; /* Catppuccin Mocha Surface0 */
    border-radius: 12px;
    padding: 30px;
    max-width: 500px;
    width: 90%;
    max-height: 80vh;
    overflow-y: auto;
  }

  .modal h2 {
    margin-top: 0;
    color: #cba6f7; /* Catppuccin Mocha Mauve */
  }

  .settings-group {
    margin-bottom: 20px;
  }

  .settings-group label {
    display: flex;
    flex-direction: column;
    gap: 8px;
    color: #bac2de; /* Catppuccin Mocha Subtext1 */
  }

  .settings-group input {
    padding: 8px 12px;
    border-radius: 8px;
    border: 2px solid #45475a; /* Catppuccin Mocha Surface1 */
    background: #181825; /* Catppuccin Mocha Mantle */
    color: #cdd6f4; /* Catppuccin Mocha Text */
    font-size: 1rem;
  }

  .helper-text {
    font-size: 0.85rem;
    color: #7f849c; /* Catppuccin Mocha Overlay1 */
  }

  .settings-preview {
    background: #313244; /* Catppuccin Mocha Surface0 */
    border-radius: 8px;
    padding: 15px;
    margin: 20px 0;
  }

  .settings-preview p {
    margin: 5px 0;
    color: #a6adc8; /* Catppuccin Mocha Subtext0 */
  }

  .modal-buttons {
    display: flex;
    gap: 10px;
    justify-content: flex-end;
  }

  .save-settings-btn, .cancel-settings-btn {
    padding: 10px 20px;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    font-size: 1rem;
    font-weight: 500;
    transition: all 0.2s;
  }

  .save-settings-btn {
    background: linear-gradient(135deg, #a6e3a1, #94e2d5); /* Green to Teal */
    color: #1e1e2e;
  }

  .cancel-settings-btn {
    background: #45475a; /* Catppuccin Mocha Surface1 */
    color: #cdd6f4;
  }

  .save-settings-btn:hover, .cancel-settings-btn:hover {
    transform: translateY(-2px);
  }

  /* Responsive design for mobile */
  @media (max-width: 768px) {
    .game-container {
      padding: 10px;
    }

    .game-header {
      flex-direction: column;
      gap: 15px;
      text-align: center;
    }

    .header-right {
      flex-direction: column;
      gap: 15px;
      width: 100%;
    }

    .settings-btn {
      position: static;
      margin-top: 10px;
    }

    .title h1 {
      font-size: 1.5rem;
    }

    .stats {
      align-items: center;
      width: 100%;
    }

    .balance {
      align-items: center;
    }

    .controls {
      flex-direction: column;
    }

    .lot-size-control {
      min-width: 100%;
    }

    .game-grid {
      gap: 4px;
      padding: 10px;
    }

    .game-box {
      font-size: 1rem;
    }

    .icon {
      font-size: 0.9rem;
    }
  }

  @media (max-width: 480px) {
    .title h1 {
      font-size: 1.2rem;
    }

    .balance-amount {
      font-size: 1.5rem;
    }

    .target-amount {
      font-size: 1rem;
    }

    .game-grid {
      gap: 2px;
    }

    .icon {
      font-size: 0.8rem;
    }
  }
</style>
