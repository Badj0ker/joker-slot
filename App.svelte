<script>
  import { onMount } from 'svelte';

  const testMode = true;

  const ROWS = 5;
  const COLS = 6;
  const SIZE = 80;
  const GAP = 10;
  const MIN_FOR_WIN = 8;

  let grid = [];
  let gridEl;
  let running = false;

  let currentWin = 0;
  let currentMultiplier = 1;
  let totalFreeSpinsWin = 0;

  let fallingCells = new Set();
  let clusterCells = new Set();
  let flashMultiplier = false;

  let showSmallWin = false;
  let smallWinAmount = 0;
  let smallWinMulti = 1;

  let freeSpinsLeft = 0;
  let isFreeSpins = false;
  let initialFreeSpins = 0;

  let showBonusStart = false;
  let bonusSpinsAwarded = 0;
  let showBonusEnd = false;
  let bonusTotalWin = 0;

  let spinCounter = 0;

  const LOW_SYMBOLS = [
    { id: 'L1', image: '/symbols/symbol1.png' },
    { id: 'L2', image: '/symbols/symbol2.png' },
    { id: 'L3', image: '/symbols/symbol3.png' },
    { id: 'L4', image: '/symbols/symbol4.png' },
    { id: 'L5', image: '/symbols/symbol5.png' }
  ];

  const PREMIUM_SYMBOLS = [
    { id: 'P1', image: '/symbols/elite1.png' },
    { id: 'P2', image: '/symbols/elite2.png' },
    { id: 'P3', image: '/symbols/elite3.png' },
    { id: 'P4', image: '/symbols/elite4.png' },
    { id: 'P5', image: '/symbols/elite5.png' }
  ];

  const MULTIS = [
    { id: '2X', isMultiplier: true, value: 2, image: '/symbols/multix2.png' },
    { id: '5X', isMultiplier: true, value: 5, image: '/symbols/multix5.png' },
    { id: '10X', isMultiplier: true, value: 10, image: '/symbols/multix10.png' },
    { id: '50X', isMultiplier: true, value: 50, image: '/symbols/multix50.png' }
  ];

  const SCATTER = {
    id: 'SC',
    image: '/symbols/scatter.png',
    isScatter: true
  };

  const PAYTABLE = {
    L1: 0.1,
    L2: 0.15,
    L3: 0.2,
    L4: 0.25,
    L5: 0.3,
    P1: 1.0,
    P2: 1.5,
    P3: 2.0,
    P4: 3.0,
    P5: 5.0
  };

  function wait(ms) {
    return new Promise(res => setTimeout(res, ms));
  }

  function randomCell() {
    const scatterChance = isFreeSpins ? 0.005 : 0.015;
    if (Math.random() < scatterChance) return structuredClone(SCATTER);
    if (Math.random() < 0.02) return structuredClone(MULTIS[Math.floor(Math.random() * MULTIS.length)]);
    if (Math.random() < 0.65) return structuredClone(LOW_SYMBOLS[Math.floor(Math.random() * LOW_SYMBOLS.length)]);
    return structuredClone(PREMIUM_SYMBOLS[Math.floor(Math.random() * PREMIUM_SYMBOLS.length)]);
  }

  function renderGrid() {
    gridEl.innerHTML = '';
    for (let r = 0; r < ROWS; r++) {
      for (let c = 0; c < COLS; c++) {
        const cell = grid[r][c];
        const div = document.createElement('div');
        div.style.cssText = `width:${SIZE}px;height:${SIZE}px;border-radius:16px;display:flex;align-items:center;justify-content:center;position:relative;background:transparent;z-index:10;`;

        if (cell?.image) {
          const img = document.createElement('img');
          img.src = cell.image;
          img.style.cssText = 'width:100%;height:100%;object-fit:contain;z-index:11;';
          if (cell.isMultiplier && flashMultiplier) img.classList.add('multi-flash');
          div.appendChild(img);
        }

        if (clusterCells.has(`${r}-${c}`)) div.classList.add('cluster-glow');

        if (fallingCells.has(`${r}-${c}`)) {
          div.style.transform = 'translateY(-300px)';
          div.style.opacity = '0';
          div.style.zIndex = '50';
          requestAnimationFrame(() => {
            div.style.transition = 'all 0.7s cubic-bezier(0.2, 0.8, 0.2, 1)';
            div.style.transform = 'translateY(0)';
            div.style.opacity = '1';
          });
        }

        gridEl.appendChild(div);
      }
    }
    fallingCells.clear();
    clusterCells.clear();
  }

  function countScatters() {
    let count = 0;
    grid.forEach(row => row.forEach(cell => { if (cell?.isScatter) count++; }));
    return count;
  }

  function getWinningIds() {
    const counts = {};
    grid.flat().forEach(c => {
      if (c && !c.isMultiplier && !c.isScatter) counts[c.id] = (counts[c.id] || 0) + 1;
    });
    return new Set(Object.keys(counts).filter(id => counts[id] >= MIN_FOR_WIN));
  }

  async function explodeWinners() {
    const winners = getWinningIds();
    if (winners.size === 0) return false;

    clusterCells.clear();
    let thisTumbleWin = 0;

    grid.forEach((row, r) => row.forEach((cell, c) => {
      if (cell && !cell.isMultiplier && !cell.isScatter && winners.has(cell.id)) {
        clusterCells.add(`${r}-${c}`);
        thisTumbleWin += PAYTABLE[cell.id] || 0.1;
      }
    }));

    renderGrid();
    await wait(600);

    grid.forEach((row, r) => row.forEach((cell, c) => {
      if (cell && !cell.isMultiplier && !cell.isScatter && winners.has(cell.id)) {
        grid[r][c] = null;
      }
    }));
    renderGrid();
    await wait(150);

    const multis = [];
    let collectedThisTumble = 0;

    grid.forEach((row, r) => row.forEach((cell, c) => {
      if (cell?.isMultiplier) {
        multis.push({ r, c, value: cell.value });
        collectedThisTumble += cell.value;
      }
    }));

    if (multis.length > 0) {
      flashMultiplier = true;
      renderGrid();

      multis.forEach(({ r, c }) => {
        const zap = document.createElement('div');
        zap.className = 'zap';
        zap.style.position = 'absolute';
        zap.style.left = `${c * (SIZE + GAP) + 20}px`;
        zap.style.top = `${r * (SIZE + GAP) + 20}px`;
        zap.style.width = `${SIZE}px`;
        zap.style.height = `${SIZE}px`;
        zap.style.zIndex = '60';
        gridEl.appendChild(zap);
        setTimeout(() => zap.remove(), 500);
      });

      await wait(700);
      flashMultiplier = false;

      multis.forEach(({ r, c }) => grid[r][c] = null);
    }

    if (thisTumbleWin > 0) {
      const winThisRound = thisTumbleWin * currentMultiplier;
      currentWin += winThisRound;
      if (isFreeSpins) totalFreeSpinsWin += winThisRound;

      smallWinAmount = thisTumbleWin;
      smallWinMulti = currentMultiplier;
      showSmallWin = true;
      setTimeout(() => showSmallWin = false, 2200);

      if (collectedThisTumble > 0) currentMultiplier += collectedThisTumble;
    } else {
      currentMultiplier += collectedThisTumble;
    }

    renderGrid();
    return true;
  }

  async function applyGravityAnimated() {
    let moved = false;

    for (let c = 0; c < COLS; c++) {
      for (let r = ROWS - 1; r >= 0; r--) {
        if (grid[r][c] === null) {
          for (let k = r - 1; k >= 0; k--) {
            if (grid[k][c]) {
              grid[r][c] = grid[k][c];
              grid[k][c] = null;
              fallingCells.add(`${r}-${c}`);
              moved = true;
              break;
            }
          }
        }
      }

      for (let r = 0; r < ROWS; r++) {
        if (grid[r][c] === null) {
          grid[r][c] = randomCell();
          fallingCells.add(`${r}-${c}`);
          moved = true;
        }
      }
    }

    if (moved) {
      renderGrid();
      await wait(800);
    }
  }

  async function spinIn() {
    grid = Array(ROWS).fill(null).map(() => Array(COLS).fill(null));
    renderGrid();

    for (let c = 0; c < COLS; c++) {
      for (let r = 0; r < ROWS; r++) {
        await wait(60);
        grid[r][c] = randomCell();
        fallingCells.add(`${r}-${c}`);
        renderGrid();
      }
    }
  }

  async function playSpin() {
    if (running) return;
    running = true;

    const wasFreeSpins = isFreeSpins;

    if (!isFreeSpins) {
      spinCounter++;
      if (spinCounter >= 3) spinCounter = 0;
    }

    currentWin = 0;
    if (!isFreeSpins) currentMultiplier = 1;
    showSmallWin = false;

    await spinIn();

    if (spinCounter === 0 && !isFreeSpins && testMode) {
      for (let r = 0; r < ROWS; r++) {
        for (let c = 0; c < COLS; c++) {
          if (grid[r][c]?.isScatter) grid[r][c] = randomCell();
        }
      }
      let placed = 0;
      while (placed < 4) {
        const r = Math.floor(Math.random() * ROWS);
        const c = Math.floor(Math.random() * COLS);
        if (!grid[r][c]?.isScatter) {
          grid[r][c] = structuredClone(SCATTER);
          placed++;
        }
      }
      renderGrid();
    }

    await wait(400);

    let hadWin = false;
    while (await explodeWinners()) {
      hadWin = true;
      await applyGravityAnimated();
      await wait(400);
    }

    const scatters = countScatters();

    if (scatters >= 4 && !isFreeSpins) {
      bonusSpinsAwarded = testMode ? 15 : (scatters === 4 ? 15 : scatters === 5 ? 20 : 25);
      freeSpinsLeft = bonusSpinsAwarded;
      initialFreeSpins = bonusSpinsAwarded;
      totalFreeSpinsWin = 0;
      isFreeSpins = true;
      showBonusStart = true;

      await new Promise(resolve => {
        const handler = () => {
          document.body.removeEventListener('click', handler);
          showBonusStart = false;
          resolve(null);
          setTimeout(playSpin, 300);
        };
        document.body.addEventListener('click', handler);
      });
    }

    if (isFreeSpins && wasFreeSpins && scatters >= 3) {
      freeSpinsLeft += 5;
    }

    if (isFreeSpins && wasFreeSpins) {
      freeSpinsLeft--;
      if (freeSpinsLeft > 0) {
        setTimeout(playSpin, 800);
      } else {
        isFreeSpins = false;
        bonusTotalWin = totalFreeSpinsWin;
        showBonusEnd = true;

        await new Promise(resolve => {
          const handler = () => {
            document.body.removeEventListener('click', handler);
            showBonusEnd = false;
            currentMultiplier = 1;
            resolve(null);
          };
          document.body.addEventListener('click', handler);
        });
      }
    }

    if (!hadWin && !isFreeSpins) currentMultiplier = 1;

    running = false;
  }

  onMount(() => {
    gridEl.style.cssText = `display:grid;grid-template-columns:repeat(${COLS},${SIZE}px);grid-template-rows:repeat(${ROWS},${SIZE}px);gap:${GAP}px;z-index:10;position:relative;`;

    const btn = document.createElement('button');
    btn.textContent = 'SPIN';
    Object.assign(btn.style, {
      position: 'fixed',
      bottom: '40px',
      left: '50%',
      transform: 'translateX(-50%)',
      padding: '18px 80px',
      fontSize: '28px',
      borderRadius: '50px',
      border: 'none',
      background: '#ff3366',
      color: 'white',
      cursor: 'pointer',
      fontWeight: 'bold',
      zIndex: '100'
    });
    btn.onclick = playSpin;
    document.body.appendChild(btn);
  });
</script>

<div class="top-logo">
  <img src="/symbols/topjoker.png" alt="Top Joker">
</div>

<div class="slot-container">
  <div bind:this={gridEl}></div>
</div>

<div class="side-panel">
  <div class="balance">BALANCE<br>10000$</div>
  <div class="bet">BET<br>10$</div>
  <div class="multi-box">
    MULTIPLIER
    <div class="multi-value">
      {#if currentMultiplier > 1}
        {currentMultiplier}×
      {:else}
        &nbsp;
      {/if}
    </div>
  </div>
</div>

{#if isFreeSpins}
  <div class="freespin-panel">
    FREE SPINS
    <div class="freespin-value">
      {freeSpinsLeft}/{initialFreeSpins}
    </div>
  </div>
{/if}

{#if showSmallWin}
  <div class="small-win">
    <div class="win-title">WIN</div>
    <div class="win-combo">{smallWinAmount.toFixed(2)} × {smallWinMulti}</div>
    <div class="win-total">{(smallWinAmount * smallWinMulti).toFixed(2)}$</div>
  </div>
{/if}

{#if showBonusStart}
  <div class="bonus-popup">
    <div class="bonus-title">FREE SPINS!</div>
    <div class="bonus-text">{bonusSpinsAwarded} FREE SPINS AWARDED!</div>
    <div class="bonus-click">Click anywhere to continue</div>
  </div>
{/if}

{#if showBonusEnd}
  <div class="bonus-popup">
    <div class="bonus-title">FREE SPINS COMPLETE!</div>
    <div class="bonus-text">Congratulations!</div>
    <div class="bonus-total">You won {bonusTotalWin.toFixed(2)}$</div>
    <div class="bonus-click">Click anywhere to continue</div>
  </div>
{/if}

<style>
  :global(body) {
    margin: 0;
    background: url('/symbols/bground.jpg') no-repeat center center fixed;
    background-size: cover;
    overflow: hidden;
    font-family: Arial Black, sans-serif;
  }

  .slot-container {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    padding: 50px;
    border-radius: 30px;
    z-index: 5;
    box-shadow: 
      0 0 20px #121312,
      0 0 40px #990e43,
      0 0 60px #3b8065,
      0 0 80px #00ffff,
      inset 0 0 25px rgba(0, 255, 157, 0.1);
    animation: neonBorderPulse 4s ease-in-out infinite;
  }

  @keyframes neonBorderPulse {
    0%, 100% { box-shadow: 0 0 20px #20312b, 0 0 40px #310522, 0 0 60px #45685a, 0 0 80px #00ffff, inset 0 0 25px rgba(0, 255, 157, 0.1); }
    50% { box-shadow: 0 0 30px #250106, 0 0 60px #362747, 0 0 90px #00ffff, 0 0 120px #313330, inset 0 0 80px rgba(57, 255, 20, 0.5); }
  }

  .side-panel {
    position: fixed;
    left: calc(50% - 265px - 350px);
    top: 50%;
    transform: translateY(-50%);
    width: 220px;
    z-index: 100;
  }

  .freespin-panel {
    position: fixed;
    right: calc(50% - 265px - 350px);
    top: 50%;
    transform: translateY(-50%);
    width: 220px;
    z-index: 100;
    background: rgba(0,0,0,0.65);
    border: 3px solid #ff3366;
    border-radius: 12px;
    padding: 14px 10px;
    font-size: 18px;
    font-weight: bold;
    text-align: center;
    color: white;
  }

  .freespin-value {
    font-size: 36px;
    color: #ff3366;
    text-shadow: 0 0 10px #ff3366;
    margin-top: 8px;
  }

  .balance, .bet, .multi-box {
    background: rgba(0,0,0,0.65);
    border: 3px solid;
    border-radius: 12px;
    padding: 14px 10px;
    margin: 12px 0;
    font-size: 18px;
    font-weight: bold;
    text-align: center;
    color: white;
  }

  .balance { border-color: #00ff9d; }
  .bet { border-color: #00d4ff; }
  .multi-box { border-color: #ffd700; min-height: 80px; display: flex; flex-direction: column; justify-content: center; }

  .multi-value {
    font-size: 36px;
    color: #ffd700;
    text-shadow: 0 0 10px gold;
    margin-top: 8px;
    min-height: 40px;
    line-height: 40px;
  }

  :global(.cluster-glow) {
    animation: clusterPulse 0.8s ease-in-out infinite alternate;
    box-shadow: 0 0 30px gold !important;
    transform: scale(1.05);
  }

  @keyframes clusterPulse {
    from { box-shadow: 0 0 20px white; transform: scale(1.05); }
    to { box-shadow: 0 0 60px gold; transform: scale(1.15); }
  }

  :global(.multi-flash) { animation: multiFlash 0.8s ease-out; }

  @keyframes multiFlash {
    0% { transform: scale(1); }
    50% { transform: scale(1.4); filter: brightness(4) drop-shadow(0 0 40px gold); }
    100% { transform: scale(1); }
  }

  :global(.zap) {
    background: repeating-linear-gradient(90deg, transparent, white 10%, transparent 20%);
    animation: zapAnim 0.5s linear;
    mix-blend-mode: screen;
    pointer-events: none;
  }

  @keyframes zapAnim {
    0% { transform: translateY(-100px); opacity: 0; }
    50% { opacity: 1; }
    100% { transform: translateY(100px); opacity: 0; }
  }

  .small-win {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: rgba(20,0,40,0.9);
    border: 5px solid #ffd700;
    border-radius: 16px;
    padding: 20px 50px;
    color: white;
    text-align: center;
    z-index: 9999;
    pointer-events: none;
    box-shadow: 0 0 40px rgba(255,215,0,0.6);
    animation: winPop 2.2s forwards;
  }

  .win-title { font-size: 52px; font-weight: 900; color: #ffd700; text-shadow: 0 0 25px gold; margin-bottom: 6px; }
  .win-combo { font-size: 38px; font-weight: bold; color: #00ffff; margin: 8px 0; }
  .win-total { font-size: 60px; font-weight: 900; color: #00ff9d; text-shadow: 0 0 20px #00ff9d; }

  @keyframes winPop {
    0% { opacity: 0; transform: translate(-50%, -50%) scale(0.6); }
    15% { opacity: 1; transform: translate(-50%, -50%) scale(1.08); }
    85% { opacity: 1; transform: translate(-50%, -50%) scale(1); }
    100% { opacity: 0; transform: translate(-50%, -50%) scale(0.95); }
  }

  .top-logo {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -205%);
    z-index: 200;
    pointer-events: none;
  }

  .top-logo img {
    width: 250px;
    height: auto;
    filter: drop-shadow(0 0 20px #00ff9d);
  }

  .bonus-popup {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: rgba(20, 0, 40, 0.95);
    border: 6px solid #ffd700;
    border-radius: 20px;
    padding: 40px 80px;
    text-align: center;
    color: white;
    z-index: 9999;
    box-shadow: 0 0 80px rgba(255, 215, 0, 0.8);
    animation: bonusPop 0.8s ease-out;
    cursor: pointer;
  }

  .bonus-title { font-size: 80px; font-weight: 900; color: #ffd700; text-shadow: 0 0 30px gold; margin-bottom: 20px; }
  .bonus-text { font-size: 50px; color: #00ff9d; margin: 20px 0; }
  .bonus-total { font-size: 70px; font-weight: 900; color: #00ff9d; text-shadow: 0 0 30px #00ff9d; margin: 20px 0; }
  .bonus-click { font-size: 30px; color: #ffffff; margin-top: 30px; opacity: 0.8; }

  @keyframes bonusPop {
    0% { opacity: 0; transform: translate(-50%, -50%) scale(0.5); }
    100% { opacity: 1; transform: translate(-50%, -50%) scale(1); }
  }
</style>