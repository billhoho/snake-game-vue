<template>
    <div v-if="!gameState.isPlaying && !gameState.isGameOver" class="start-overlay">
      <div class="start-menu">
        <h1>SNAKE COMBAT</h1>
        
        <div class="info-grid">
          <div class="info-section">
            <h3>🎮 遊戲規則</h3>
            <ul>
              <li><span>🍎 成長：</span>吃掉 HP 比你低的敵人來增加 HP</li>
              <li><span>⚠️ 危險：</span>碰撞牆壁、自身或強敵將結束遊戲</li>
              <li><span>🛡️ 護盾：</span>收集藍色球可抵擋一次傷害</li>
              <li><span>👾 敵人：</span>每 7 秒隨機生成。紅色為弱敵，紫色為強敵</li>
              <li><span>👹 BOSS：</span>HP 達到 999 時將觸發最終 Boss 戰</li>
            </ul>
          </div>
          <div class="info-section">
            <h3>⌨️ 操作指南</h3>
            <ul>
              <li><span>方向鍵：</span>控制移動方向</li>
              <li><span>Shift：</span>加速衝刺</li>
              <li><span>Space / Enter：</span>開始遊戲 / 身體翻轉 (U-Turn)</li>
            </ul>
          </div>
        </div>
        
        <button class="start-btn" @click="startGame">START GAME</button>
      </div>
    </div>

    <div class="game-container">
    <div class="stats-container">
      <div class="stat">❤️ {{ playerHP }}</div>
      <div class="stat">⏱️ {{ formatTime(timer) }}</div>
      <div class="stat">🛡️ {{ shieldCount }}</div>
      <div class="stat" @click="togglePause" v-if="gameState.isPlaying" style="cursor: pointer; user-select: none; margin-left: auto;">
        {{ gameState.isPaused ? '▶️' : '⏸️' }}
      </div>
      <div class="stat" @click="toggleMute" style="cursor: pointer; user-select: none;">
        {{ isMuted ? '🔇' : '🔊' }}
      </div>
    </div>
    <div class="game-board-wrapper">
    <div 
      class="game-board" 
      :style="{ gridTemplateColumns: `repeat(${gridSize}, 1fr)` }"
    >
      <div 
        v-for="(cell, index) in allCells" 
        :key="index" 
        class="cell" 
        :class="{ 
           'snake-head': getSnakePart(cell) === 'head',
           'snake-body': getSnakePart(cell) === 'body',
           'snake-tail': getSnakePart(cell) === 'tail',
           'enemy-easy': getEnemy(cell) && getEnemy(cell).hp < playerHP,
           'enemy-hard': getEnemy(cell) && getEnemy(cell).hp >= playerHP,
           'shield': isShield(cell),
           'wall': isWall(cell),
           'boss-warning': getBossWarningClass(cell),
           'boss-attack-warning': getBossAttackClass(cell) === 'warning',
           'boss-attack-hit': getBossAttackClass(cell) === 'hit',
           ...getDirectionClass(cell)
         }"
 
        :style="getEnemy(cell) ? { animationDelay: getEnemy(cell).animationDelay + 's' } : {}"
      >
        <span v-if="getEnemy(cell)" class="enemy-hp">{{ getEnemy(cell).hp }}</span>
      </div>
    </div>
 
      <div v-if="gameState.isPaused" class="pause-overlay">
        <div class="pause-menu">
          <div class="pause-icon">⏸️</div>
          <h2>暫停中</h2>
          <div class="pause-buttons">
            <button @click="togglePause">繼續遊戲</button>
            <button @click="backToMenu" class="menu-btn">主選單</button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="gameState.isPlaying && !gameState.isPaused" class="virtual-controls">
      <div class="d-pad">
        <button class="v-btn up" @mousedown="moveDirection(0, -1)" @touchstart.prevent="moveDirection(0, -1)">▲</button>
        <div class="d-pad-mid">
          <button class="v-btn left" @mousedown="moveDirection(-1, 0)" @touchstart.prevent="moveDirection(-1, 0)">◀</button>
          <button class="v-btn right" @mousedown="moveDirection(1, 0)" @touchstart.prevent="moveDirection(1, 0)">▶</button>
        </div>
        <button class="v-btn down" @mousedown="moveDirection(0, 1)" @touchstart.prevent="moveDirection(0, 1)">▼</button>
      </div>
      <div class="action-buttons">
        <button 
          class="v-btn action-btn" 
          :class="{ 'active': isAccelerating }"
          @mousedown="setAcceleration(true)" 
          @mouseup="setAcceleration(false)"
          @mouseleave="setAcceleration(false)"
          @touchstart.prevent="setAcceleration(true)" 
          @touchend.prevent="setAcceleration(false)"
        >
          加速
        </button>
        <button class="v-btn action-btn" @mousedown="flipBody" @touchstart.prevent="flipBody">翻轉</button>
      </div>
    </div>

    <div class="controls">
    </div>
    <div v-if="gameState.isGameOver" class="game-over">
      <h2>遊戲結束！</h2>
      <p v-if="deathReason === 'wall'">你撞到牆壁了！</p>
      <p v-else-if="deathReason === 'self'">你咬到自己了！</p>
      <p v-else-if="deathReason === 'enemy'">被更強的敵人擊敗了！</p>
      <p>最終 HP: {{ playerHP }}</p>
      <p>最高紀錄: {{ highScore }}</p>
      <div class="game-over-buttons">
        <button @click="startGame">再玩一次</button>
        <button @click="backToMenu" class="menu-btn">主選單</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue';

const gridSize = 20;
const initialDirection = { x: 0, y: -1 }; 
const BPM = 120; 
const MOVE_INTERVAL = 60000 / BPM;

const snake = ref([]);
const enemies = ref([]);
const shields = ref([]);
const direction = ref({ ...initialDirection });
const lastProcessedDirection = ref({ ...initialDirection });
const directionsHistory = ref([]);
const inputQueue = ref([]);
const playerHP = ref(10);
const shieldCount = ref(0);
const lastMagnitude = ref(1);
const isAccelerating = ref(false);
const timer = ref(0);
const spawnTimer = ref(0);
const highScore = ref(parseInt(localStorage.getItem('snake-high-score')) || 0);
const maze = ref([]);
const isMuted = ref(false);
const deathReason = ref('');

const bossState = ref('idle'); // 'idle', 'warning', 'active'
const bossSpawnPos = ref(null);
const bossPos = ref(null);
const bossAttackTimer = ref(0);

const gameState = ref({
  isPlaying: false,
  isGameOver: false,
  isPaused: false,
});

let gameTimeout = null;
let timerInterval = null;
let audioCtx = null;

const allCells = computed(() => {
  const cells = [];
  for (let y = 0; y < gridSize; y++) {
    for (let x = 0; x < gridSize; x++) {
      cells.push({ x, y });
    }
  }
  return cells;
});

const getSnakePart = (cell) => {
  const index = snake.value.findIndex(segment => segment.x === cell.x && segment.y === cell.y);
  if (index === -1) return null;
  if (index === 0) return 'head';
  if (index === snake.value.length - 1) return 'tail';
  return 'body';
};

const getDirectionClass = (cell) => {
  if (getSnakePart(cell) !== 'head') return {};
  
  // Visual direction is the next intended move in the queue, 
  // or the current movement direction if the queue is empty.
  const vDir = inputQueue.value.length > 0 
    ? inputQueue.value[0] 
    : direction.value;

  if (vDir.x === 0 && vDir.y === -1) return { 'dir-up': true };
  if (vDir.x === 0 && vDir.y === 1) return { 'dir-down': true };
  if (vDir.x === -1 && vDir.y === 0) return { 'dir-left': true };
  if (vDir.x === 1 && vDir.y === 0) return { 'dir-right': true };
  
  return {};
};

const getBossWarningClass = (cell) => {
  if (bossState.value !== 'warning' || !bossSpawnPos.value) return false;
  const dx = Math.abs(cell.x - bossSpawnPos.value.x);
  const dy = Math.abs(cell.y - bossSpawnPos.value.y);
  return dx <= 1 && dy <= 1;
};

const getBossAttackClass = (cell) => {
  if (bossState.value !== 'active' || !bossPos.value) return null;
  const dx = Math.abs(cell.x - bossPos.value.x);
  const dy = Math.abs(cell.y - bossPos.value.y);
  if (dx === 0 && dy === 0) {
    return bossAttackTimer.value <= 3 ? 'preparing' : null;
  }
  if (dx <= 1 && dy <= 1) {
    return bossAttackTimer.value === 4 ? 'warning' : (bossAttackTimer.value === 5 ? 'hit' : null);
  }
  return null;
};

const getEnemy = (cell) => {
  return enemies.value.find(e => e.x === cell.x && e.y === cell.y);
};

const isEnemy = (cell) => {
  return !!getEnemy(cell);
};

const isShield = (cell) => {
  return shields.value.some(s => s.x === cell.x && s.y === cell.y);
};

const isWall = (cell) => {
  return maze.value[cell.y] && maze.value[cell.y][cell.x] === 1;
};

const getRandomEmptyCell = () => {
  const availableCells = [];
  for (let y = 0; y < gridSize; y++) {
    for (let x = 0; x < gridSize; x++) {
      const isSafeZone = x <= 2 && y <= 2;
      const isOnSnake = snake.value.some(s => s.x === x && s.y === y);
      const isOnEnemy = enemies.value.some(e => e.x === x && e.y === y);
      const isOnShield = shields.value.some(s => s.x === x && s.y === y);
      
      if (maze.value[y][x] === 0 && !isOnSnake && !isSafeZone && !isOnEnemy && !isOnShield) {
        availableCells.push({ x, y });
      }
    }
  }
  
  if (availableCells.length === 0) return null;
  return availableCells[Math.floor(Math.random() * availableCells.length)];
};

const generateEnemy = (forcedEasy = false) => {
  const pos = getRandomEmptyCell();
  if (!pos) return;
  const isEasy = forcedEasy || Math.random() < 0.5;
  let hp;
  if (isEasy) {
    hp = Math.max(1, Math.floor(Math.random() * playerHP.value));
  } else {
    hp = Math.floor(playerHP.value + Math.random() * (playerHP.value + 1));
  }
  enemies.value.push({ 
    ...pos, 
    hp, 
    animationDelay: Math.random() * 2 
  });
};

const spawnBoss = () => {
  if (!bossSpawnPos.value) return;
  bossState.value = 'active';
  bossPos.value = { ...bossSpawnPos.value };
  enemies.value.push({
    ...bossPos.value,
    hp: 998,
    animationDelay: Math.random() * 2
  });
};

const triggerBossFight = () => {
  enemies.value = [];
  
  const corners = [
    { x: 0, y: 0 }, { x: 0, y: gridSize - 1 },
    { x: gridSize - 1, y: 0 }, { x: gridSize - 1, y: gridSize - 1 }
  ];
  
  const head = snake.value[0];
  let furthestCorner = corners[0];
  let maxDist = -1;
  
  corners.forEach(c => {
    const dist = Math.abs(c.x - head.x) + Math.abs(c.y - head.y);
    if (dist > maxDist) {
      maxDist = dist;
      furthestCorner = c;
    }
  });
  
  // Search 3x3 area around furthest corner for empty cell
  let foundPos = null;
  for (let dy = -1; dy <= 1; dy++) {
    for (let dx = -1; dx <= 1; dx++) {
      const nx = furthestCorner.x + dx;
      const ny = furthestCorner.y + dy;
      if (nx >= 0 && nx < gridSize && ny >= 0 && ny < gridSize) {
        const isOnSnake = snake.value.some(s => s.x === nx && s.y === ny);
        if (maze.value[ny][nx] === 0 && !isOnSnake) {
          foundPos = { x: nx, y: ny };
          break;
        }
      }
    }
    if (foundPos) break;
  }
  
  if (!foundPos) foundPos = furthestCorner; // Fallback
  
  bossSpawnPos.value = foundPos;
  bossState.value = 'warning';
  
  setTimeout(() => {
    if (gameState.value.isPlaying && !gameState.value.isPaused) {
      spawnBoss();
    }
  }, 5000);
};

const spawnShield = () => {
  const pos = getRandomEmptyCell();
  if (pos) {
    shields.value.push(pos);
  }
};


const generateMaze = () => {
  const internalSize = 21; 
  const newMaze = Array.from({ length: internalSize }, () => Array(internalSize).fill(1));
  
  const carve = (x, y) => {
    newMaze[y][x] = 0;
    const directions = [
      { x: 0, y: -2 }, { x: 0, y: 2 },
      { x: -2, y: 0 }, { x: 2, y: 0 }
    ].sort(() => Math.random() - 0.5);
    
    for (const { x: dx, y: dy } of directions) {
      const nx = x + dx;
      const ny = y + dy;
      if (nx >= 0 && nx < internalSize && ny >= 0 && ny < internalSize && newMaze[ny][nx] === 1) {
        newMaze[y + dy / 2][x + dx / 2] = 0;
        carve(nx, ny);
      }
    }
  };

  carve(0, 0);
  
  // Crop to 20x20
  const slicedMaze = newMaze.slice(0, gridSize).map(row => row.slice(0, gridSize));

  // 1. Ensure starting point is open
  slicedMaze[1][1] = 0;

  // 2. Connectivity Fix (Flood Fill)
  const getConnected = (m) => {
    const visited = new Set();
    const queue = [[1, 1]];
    visited.add(`1,1`);
    
    let head = 0;
    while(head < queue.length) {
      const [cx, cy] = queue[head++];
      const neighbors = [
        {x: cx+1, y: cy}, {x: cx-1, y: cy}, {x: cx, y: cy+1}, {x: cx, y: cy-1}
      ];
      for (const {x, y} of neighbors) {
        if (x >= 0 && x < gridSize && y >= 0 && y < gridSize && m[y][x] === 0 && !visited.has(`${x},${y}`)) {
          visited.add(`${x},${y}`);
          queue.push([x, y]);
        }
      }
    }
    return visited;
  };

  let reachable = getConnected(slicedMaze);
  
  // Find all isolated path cells
  let isolated = [];
  for (let y = 0; y < gridSize; y++) {
    for (let x = 0; x < gridSize; x++) {
      if (slicedMaze[y][x] === 0 && !reachable.has(`${x},${y}`)) {
        isolated.push({x, y});
      }
    }
  }

  // Connect each isolated area to the reachable set
  for (const cell of isolated) {
    if (reachable.has(`${cell.x},${cell.y}`)) continue;
    
    // Find nearest reachable cell and carve path
    let bestDist = Infinity;
    let target = null;
    
    // To keep it efficient, we just look for the nearest open cell that IS reachable
    // In a 20x20 grid, a simple search is fine
    for (let y = 0; y < gridSize; y++) {
      for (let x = 0; x < gridSize; x++) {
        if (reachable.has(`${x},${y}`)) {
          const dist = Math.abs(x - cell.x) + Math.abs(y - cell.y);
          if (dist < bestDist) {
            bestDist = dist;
            target = {x, y};
          }
        }
      }
    }

    if (target) {
      // Carve L-shaped path
      let curX = cell.x;
      let curY = cell.y;
      while (curX !== target.x) {
        curX += (target.x > curX ? 1 : -1);
        slicedMaze[curY][curX] = 0;
      }
      while (curY !== target.y) {
        curY += (target.y > curY ? 1 : -1);
        slicedMaze[curY][curX] = 0;
      }
      // Refresh reachable set
      reachable = getConnected(slicedMaze);
    }
  }

  // 3. Braid Logic: Remove Dead Ends
  const removeDeadEnds = (m) => {
    let foundDeadEnd = true;
    while (foundDeadEnd) {
      foundDeadEnd = false;
      for (let y = 0; y < gridSize; y++) {
        for (let x = 0; x < gridSize; x++) {
          if (m[y][x] === 0) {
            let neighbors = 0;
            if (y > 0 && m[y-1][x] === 0) neighbors++;
            if (y < gridSize - 1 && m[y+1][x] === 0) neighbors++;
            if (x > 0 && m[y][x-1] === 0) neighbors++;
            if (x < gridSize - 1 && m[y][x+1] === 0) neighbors++;
            
            if (neighbors === 1) {
              const walls = [];
              if (y > 0 && m[y-1][x] === 1) walls.push({x: 0, y: -1});
              if (y < gridSize - 1 && m[y+1][x] === 1) walls.push({x: 0, y: 1});
              if (x > 0 && m[y][x-1] === 1) walls.push({x: -1, y: 0});
              if (x < gridSize - 1 && m[y][x+1] === 1) walls.push({x: 1, y: 0});
              
              if (walls.length > 0) {
                const choice = walls[Math.floor(Math.random() * walls.length)];
                m[y + choice.y][x + choice.x] = 0;
                foundDeadEnd = true;
              }
            }
          }
        }
      }
    }
  };
  
  removeDeadEnds(slicedMaze);
  
  // Final check for starting point exits
  const sNeighbors = [
    {x: 0, y: 1}, {x: 2, y: 1}, {x: 1, y: 0}, {x: 1, y: 2}
  ].filter(n => n.x >= 0 && n.x < gridSize && n.y >= 0 && n.y < gridSize && slicedMaze[n.y][n.x] === 0).length;
  
  if (sNeighbors < 2) {
    const sWalls = [
      {x: 0, y: 1}, {x: 2, y: 1}, {x: 1, y: 0}, {x: 1, y: 2}
    ].filter(n => n.x >= 0 && n.x < gridSize && n.y >= 0 && n.y < gridSize && slicedMaze[n.y][n.x] === 1);
    
    if (sWalls.length > 0) {
      const sChoice = sWalls[Math.floor(Math.random() * sWalls.length)];
      slicedMaze[sChoice.y][sChoice.x] = 0;
    }
  }

  maze.value = slicedMaze;
};

const spawnTimedEnemy = () => {
  const hasEasy = enemies.value.some(e => e.hp < playerHP.value);
  generateEnemy(!hasEasy);
};

const spawnInitialEnemies = (count) => {
  generateEnemy(true);
  for (let i = 1; i < count; i++) {
    generateEnemy();
  }
};

const playBeat = () => {
  if (isMuted.value || !audioCtx) return;
  
  const osc = audioCtx.createOscillator();
  const gain = audioCtx.createGain();
  
  osc.type = 'sine';
  osc.frequency.setValueAtTime(150, audioCtx.currentTime);
  osc.frequency.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.1);
  
  gain.gain.setValueAtTime(0.3, audioCtx.currentTime);
  gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.1);
  
  osc.connect(gain);
  gain.connect(audioCtx.destination);
  
  osc.start();
  osc.stop(audioCtx.currentTime + 0.1);
};

const updateBoss = () => {
  if (bossState.value !== 'active' || !bossPos.value) return;
  
  bossAttackTimer.value++;
  if (bossAttackTimer.value > 5) bossAttackTimer.value = 1;
  
  // Phase 1-3: Move towards player
  if (bossAttackTimer.value <= 3) {
    const head = snake.value[0];
    const dx = head.x - bossPos.value.x;
    const dy = head.y - bossPos.value.y;
    
    const possibleMoves = [];
    
    // Determine primary and secondary axis based on distance
    if (Math.abs(dx) > Math.abs(dy)) {
      possibleMoves.push({ x: dx > 0 ? 1 : -1, y: 0 });
      possibleMoves.push({ x: 0, y: dy > 0 ? 1 : -1 });
    } else if (Math.abs(dy) > Math.abs(dx)) {
      possibleMoves.push({ x: 0, y: dy > 0 ? 1 : -1 });
      possibleMoves.push({ x: dx > 0 ? 1 : -1, y: 0 });
    } else {
      possibleMoves.push({ x: dx > 0 ? 1 : -1, y: 0 });
      possibleMoves.push({ x: 0, y: dy > 0 ? 1 : -1 });
    }

    let moved = false;
    for (const move of possibleMoves) {
      const nx = bossPos.value.x + move.x;
      const ny = bossPos.value.y + move.y;
      if (nx >= 0 && nx < gridSize && ny >= 0 && ny < gridSize && maze.value[ny][nx] === 0 && !snake.value.some(s => s.x === nx && s.y === ny)) {
        bossPos.value = { x: nx, y: ny };
        const bossIdx = enemies.value.findIndex(e => e.hp === 998);
        if (bossIdx !== -1) {
          enemies.value[bossIdx].x = nx;
          enemies.value[bossIdx].y = ny;
        }
        moved = true;
        break;
      }
    }

    // If still stuck, try any random empty adjacent cell to avoid appearing frozen
    if (!moved) {
      const randomMoves = [
        {x: 1, y: 0}, {x: -1, y: 0}, {x: 0, y: 1}, {x: 0, y: -1}
      ].sort(() => Math.random() - 0.5);
      for (const move of randomMoves) {
        const nx = bossPos.value.x + move.x;
        const ny = bossPos.value.y + move.y;
        if (nx >= 0 && nx < gridSize && ny >= 0 && ny < gridSize && maze.value[ny][nx] === 0 && !snake.value.some(s => s.x === nx && s.y === ny)) {
          bossPos.value = { x: nx, y: ny };
          const bossIdx = enemies.value.findIndex(e => e.hp === 998);
          if (bossIdx !== -1) {
            enemies.value[bossIdx].x = nx;
            enemies.value[bossIdx].y = ny;
          }
          break;
        }
      }
    }
  }
  
  // Phase 5: Attack
  if (bossAttackTimer.value === 5) {
    const hit = snake.value.some(s => Math.abs(s.x - bossPos.value.x) <= 1 && Math.abs(s.y - bossPos.value.y) <= 1);
    if (hit) {
      if (shieldCount.value > 0) {
        shieldCount.value--;
      } else {
        deathReason.value = 'enemy';
        gameOver();
      }
    }
  }
};

const moveSnake = () => {
  playBeat();
  
  // 1. Process Input Queue: Only apply one direction change per beat
  if (inputQueue.value.length > 0) {
    direction.value = inputQueue.value.shift();
  }

  const currentDir = { ...direction.value };

  const head = { ...snake.value[0] };
  head.x += currentDir.x;
  head.y += currentDir.y;

  // 2. Out of bounds check
  if (head.x < 0 || head.x >= gridSize || head.y < 0 || head.y >= gridSize) {
    if (shieldCount.value > 0) {
      shieldCount.value--;
      return; // Block movement
    }
    deathReason.value = 'wall';
    gameOver();
    return;
  }

  // 3. Wall collision check
  if (maze.value[head.y][head.x] === 1) {
    if (shieldCount.value > 0) {
      shieldCount.value--;
      return; // Block movement
    }
    deathReason.value = 'wall';
    gameOver();
    return;
  }

  // 4. Self collision check
  if (snake.value.some(segment => segment.x === head.x && segment.y === head.y)) {
    if (shieldCount.value > 0) {
      shieldCount.value--;
      // Allow pass through
    } else {
      deathReason.value = 'self';
      gameOver();
      return;
    }
  }

  // 5. Shield collection check
  const shieldIndex = shields.value.findIndex(s => s.x === head.x && s.y === head.y);
  if (shieldIndex !== -1) {
    shieldCount.value++;
    shields.value.splice(shieldIndex, 1);
  }

  // 6. Enemy combat logic
  const enemyIndex = enemies.value.findIndex(e => e.x === head.x && e.y === head.y);
  if (enemyIndex !== -1) {
    const target = enemies.value[enemyIndex];
    if (playerHP.value > target.hp) {
      playerHP.value += target.hp;
      enemies.value.splice(enemyIndex, 1);
      if (target.hp === 998) {
        bossState.value = 'idle';
        bossPos.value = null;
        bossAttackTimer.value = 0;
      }
    } else {
      if (shieldCount.value > 0) {
        shieldCount.value--;
        enemies.value.splice(enemyIndex, 1); // Eat enemy but no HP gain
      } else {
        deathReason.value = 'enemy';
        gameOver();
        return;
      }
    }
    
    // Ensure there are always at least 3 enemies
    if (bossState.value === 'idle' && enemies.value.length < 3) {
      generateEnemy();
    }
  }
  
  snake.value.unshift(head);
  
  // Update direction history: push the direction that just moved the head
  directionsHistory.value.push(currentDir);
  if (directionsHistory.value.length >= snake.value.length) {
    directionsHistory.value.shift();
  }
  
  lastProcessedDirection.value = currentDir;

  // Boss Trigger Check
  if (playerHP.value >= 999 && bossState.value === 'idle') {
    triggerBossFight();
  }

  // Update Boss Logic
  updateBoss();
  
  // 7. Real-time Enemy Spawning Logic
  // Calculate the time elapsed since the last move
  const elapsed = isAccelerating.value ? MOVE_INTERVAL / 2 : MOVE_INTERVAL;
  spawnTimer.value += elapsed;
  if (spawnTimer.value >= 7000) {
    if (bossState.value === 'idle') {
      spawnTimedEnemy();
    }
    spawnTimer.value -= 7000; // Use subtraction to preserve overflow
  }
  
  // Magnitude check for shield spawning

  const currentMag = Math.floor(Math.log10(playerHP.value));
  if (currentMag > lastMagnitude.value) {
    spawnShield();
  }
  lastMagnitude.value = currentMag;

  const targetLength = Math.floor(Math.log10(playerHP.value)) + 1;
  if (snake.value.length > targetLength) {
    snake.value.pop();
    if (directionsHistory.value.length >= snake.value.length) {
        directionsHistory.value.shift();
    }
  }
};

const startGame = () => {
  if (!audioCtx) {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  }
  
  resetGame();
  gameState.value.isPlaying = true;
  gameState.value.isGameOver = false;
  gameState.value.isPaused = false;
  
  const gameLoop = () => {
    if (!gameState.value.isPlaying || gameState.value.isPaused) return;
    moveSnake();
    const currentInterval = isAccelerating.value ? MOVE_INTERVAL / 2 : MOVE_INTERVAL;
    gameTimeout = setTimeout(gameLoop, currentInterval);
  };
  
  gameTimeout = setTimeout(gameLoop, MOVE_INTERVAL);
  
  timerInterval = setInterval(() => {
    timer.value++;
  }, 1000);
};

const resetGame = () => {
  generateMaze();
  playerHP.value = 10;
  shieldCount.value = 0;
  lastMagnitude.value = Math.floor(Math.log10(playerHP.value));
  snake.value = [{ x: 1, y: 1 }];
  directionsHistory.value = [];
  inputQueue.value = [];
  spawnTimer.value = 0;
  enemies.value = [];
  shields.value = [];
  bossState.value = 'idle';
  bossSpawnPos.value = null;
  bossPos.value = null;
  bossAttackTimer.value = 0;
  
  const possibleDirections = [
    { x: 1, y: 0 }, { x: -1, y: 0 }, { x: 0, y: 1 }, { x: 0, y: -1 },
  ];
  
  const validDir = possibleDirections.find(d => {
    const nx = 1 + d.x;
    const ny = 1 + d.y;
    return nx >= 0 && nx < gridSize && ny >= 0 && ny < gridSize && maze.value[ny][nx] === 0;
  }) || initialDirection;

  direction.value = { x: validDir.x, y: validDir.y };
  lastProcessedDirection.value = { x: validDir.x, y: validDir.y };
  timer.value = 0;
  
  spawnInitialEnemies(5);
};

const togglePause = () => {
  gameState.value.isPaused = !gameState.value.isPaused;
  
  if (gameState.value.isPaused) {
    clearTimeout(gameTimeout);
    clearInterval(timerInterval);
  } else {
    const gameLoop = () => {
      if (!gameState.value.isPlaying || gameState.value.isPaused) return;
      moveSnake();
      const currentInterval = isAccelerating.value ? MOVE_INTERVAL / 2 : MOVE_INTERVAL;
      gameTimeout = setTimeout(gameLoop, currentInterval);
    };
    gameTimeout = setTimeout(gameLoop, isAccelerating.value ? MOVE_INTERVAL / 2 : MOVE_INTERVAL);
    timerInterval = setInterval(() => {
      timer.value++;
    }, 1000);
  }
};

const gameOver = () => {
  gameState.value.isPlaying = false;
  gameState.value.isGameOver = true;
  clearTimeout(gameTimeout);
  clearInterval(timerInterval);

  if (playerHP.value > highScore.value) {
    highScore.value = playerHP.value;
    localStorage.setItem('snake-high-score', highScore.value.toString());
  }
};

const toggleMute = () => {
  isMuted.value = !isMuted.value;
};

const moveDirection = (x, y) => {
  if (!gameState.value.isPlaying || gameState.value.isPaused) return;
  const lastDir = inputQueue.value.length > 0 
    ? inputQueue.value[inputQueue.value.length - 1] 
    : direction.value;

  if (x === 0 && y === -1 && lastDir.y !== 1) inputQueue.value.push({ x: 0, y: -1 });
  else if (x === 0 && y === 1 && lastDir.y !== -1) inputQueue.value.push({ x: 0, y: 1 });
  else if (x === -1 && y === 0 && lastDir.x !== 1) inputQueue.value.push({ x: -1, y: 0 });
  else if (x === 1 && y === 0 && lastDir.x !== -1) inputQueue.value.push({ x: 1, y: 0 });
};

const setAcceleration = (state) => {
  if (!gameState.value.isPlaying || gameState.value.isPaused) return;
  isAccelerating.value = state;
};

const flipBody = () => {
  if (!gameState.value.isPlaying || gameState.value.isPaused) return;
  const firstMove = directionsHistory.value.length > 0 
    ? { x: -directionsHistory.value[0].x, y: -directionsHistory.value[0].y } 
    : { x: -direction.value.x, y: -direction.value.y };

  snake.value = [...snake.value].reverse();
  directionsHistory.value = [...directionsHistory.value]
    .reverse()
    .map(d => ({ x: -d.x, y: -d.y }));

  direction.value = firstMove;
  lastProcessedDirection.value = { ...firstMove };
  inputQueue.value = [];
};

const backToMenu = () => {
  gameState.value.isPlaying = false;
  gameState.value.isGameOver = false;
  gameState.value.isPaused = false;
  clearTimeout(gameTimeout);
  clearInterval(timerInterval);
};

const formatTime = (seconds) => {
  const mins = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return `${mins}:${secs.toString().padStart(2, '0')}`;
};

const handleKeyDown = (e) => {
  if (e.key === 'Enter' || e.key === ' ') {
    if (gameState.value.isGameOver || !gameState.value.isPlaying) {
      startGame();
      return;
    }
    flipBody();
    if (e.key === ' ') e.preventDefault(); // Prevent page scroll
    return;
  }
 
  if (!gameState.value.isPlaying || gameState.value.isPaused) return;
  if (e.key === 'Shift') {
    setAcceleration(true);
  }
 
  switch (e.key) {
    case 'ArrowUp':
      e.preventDefault();
      moveDirection(0, -1);
      break;
    case 'ArrowDown':
      e.preventDefault();
      moveDirection(0, 1);
      break;
    case 'ArrowLeft':
      e.preventDefault();
      moveDirection(-1, 0);
      break;
    case 'ArrowRight':
      e.preventDefault();
      moveDirection(1, 0);
      break;
  }
};
 
 
const handleKeyUp = (e) => {
  if (e.key === 'Shift') {
    setAcceleration(false);
  }
};

onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
  window.addEventListener('keyup', handleKeyUp);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown);
  window.removeEventListener('keyup', handleKeyUp);
  clearTimeout(gameTimeout);
  clearInterval(timerInterval);
});
</script>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-family: 'Arial', sans-serif;
  margin-top: 2rem;
  color: #fff;
}

.stats-container {
  display: flex;
  gap: 2rem;
  margin-bottom: 1rem;
  font-size: 1.5rem;
  font-weight: bold;
  color: #fff;
  text-shadow: 0 2px 4px rgba(0,0,0,0.5);
}

.stat {
  display: flex;
  align-items: center;
}

.game-board-wrapper {
  position: relative;
  width: 410px;
  height: 410px;
}

.game-board {
  display: grid;
  background-color: #ddd;
  border: 5px solid #555;
  width: 400px;
  height: 400px;
}

.pause-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.4);
  backdrop-filter: grayscale(1) blur(2px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
  border: 5px solid #555;
  box-sizing: border-box;
}

.pause-menu {
  background-color: rgba(255, 255, 255, 0.9);
  padding: 2rem;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 10px 30px rgba(0,0,0,0.5);
  color: #333;
}

.pause-icon {
  font-size: 3rem;
  color: #333;
  margin-bottom: 0.5rem;
}

.pause-menu h2 {
  margin-bottom: 1.5rem;
  font-size: 2rem;
}

.pause-buttons {
  display: flex;
  gap: 1rem;
}

.game-over-buttons {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin-top: 1.5rem;
}

.menu-btn {
  background-color: #757575 !important;
}

.menu-btn:hover {
  background-color: #616161 !important;
}

.virtual-controls {
  display: none;
}

@media (max-width: 768px) {
  .virtual-controls {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 3rem;
    margin-top: 2rem;
    user-select: none;
  }
}

.d-pad {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
}

.d-pad-mid {
  display: flex;
  gap: 15px;
}

.v-btn {
  width: 80px;
  height: 80px;
  border-radius: 15px;
  border: none;
  background-color: rgba(255, 255, 255, 0.2);
  color: white;
  font-size: 2rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.1s ease;
  box-shadow: 0 4px 0 rgba(0,0,0,0.3);
}

.v-btn:active {
  transform: translateY(3px);
  box-shadow: 0 1px 0 rgba(0,0,0,0.3);
  background-color: rgba(255, 255, 255, 0.4);
}

.action-buttons {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.action-btn {
  width: 140px;
  height: 80px;
  border-radius: 30px;
  font-weight: bold;
  font-size: 1.5rem;
  background-color: rgba(33, 150, 243, 0.4);
}

.action-btn.active {
  background-color: #2196f3;
  transform: translateY(3px);
  box-shadow: 0 1px 0 rgba(0,0,0,0.3);
}



.cell {
  width: 20px;
  height: 20px;
  border: 0.1px solid rgba(0,0,0,0.05);
  box-sizing: border-box;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.boss-warning {
  background-color: rgba(255, 0, 0, 0.5);
  animation: boss-flash 0.5s ease-in-out infinite;
  z-index: 1;
}

.boss-preparing {
  animation: boss-pulse 0.5s ease-in-out infinite;
  z-index: 2;
}

.boss-attack-warning {
  background-color: rgba(200, 200, 200, 0.6);
  z-index: 1;
}

.boss-attack-hit {
  background-color: rgba(255, 0, 0, 0.8);
  z-index: 1;
}

@keyframes boss-flash {
  0%, 100% { background-color: rgba(255, 0, 0, 0.8); }
  50% { background-color: transparent; }
}

@keyframes boss-pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.snake-head {
  background-color: #2e7d32;
  border-radius: 4px;
  z-index: 2;
  box-shadow: inset 0 0 4px rgba(0,0,0,0.3);
  position: relative;
}

/* Eyes */
.snake-head::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 12px;
  height: 8px;
  background-image: 
    radial-gradient(circle at 30% 50%, #000 15%, #fff 16%, #fff 35%, transparent 36%),
    radial-gradient(circle at 70% 50%, #000 15%, #fff 16%, #fff 35%, transparent 36%);
  z-index: 3;
  transition: transform 0.2s ease;
  transform: translate(-50%, -50%);
}

/* Mouth */
.snake-head::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 40%;
  height: 2px;
  background-color: #1b5e20;
  z-index: 3;
  transition: all 0.2s ease;
  transform: translate(-50%, -50%);
}

/* Direction Indicators */
.dir-up::after {
  transform: translate(-50%, -50%);
}
.dir-up::before {
  top: 10%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.dir-down::after {
  transform: translate(-50%, -50%);
}
.dir-down::before {
  top: 90%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.dir-left::after {
  transform: translate(-50%, -50%) rotate(90deg);
}
.dir-left::before {
  top: 50%;
  left: 10%;
  transform: translate(-50%, -50%) rotate(90deg);
}

.dir-right::after {
  transform: translate(-50%, -50%) rotate(90deg);
}
.dir-right::before {
  top: 50%;
  left: 90%;
  transform: translate(-50%, -50%) rotate(90deg);
}

.snake-body {



  background-color: #4caf50;
  border-radius: 2px;
}

.snake-tail {
  background-color: #81c784;
  border-radius: 2px 2px 4px 4px;
}

.enemy-easy {
  background-color: #f44336;
  border-radius: 50%;
  z-index: 1;
  animation: float 2s ease-in-out infinite, pulse 1.5s ease-in-out infinite;
}

.enemy-hard {
  background-color: #9c27b0;
  border-radius: 50%;
  z-index: 1;
  animation: float 2s ease-in-out infinite, pulse 1.5s ease-in-out infinite;
}

@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-2px); }
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.shield {
  background-color: #00d2ff;
  border-radius: 50%;
  z-index: 1;
  box-shadow: 0 0 8px #00d2ff;
}

.enemy-hp {
  color: white;
  font-size: 10px;
  font-weight: bold;
  pointer-events: none;
  z-index: 5;
  text-shadow: 1px 1px 1px rgba(0,0,0,0.5);
}

.wall {
  background-color: #555;
  box-shadow: inset 0 0 5px rgba(0,0,0,0.5);
}

.controls {
  margin-top: 1.5rem;
}

button {
  padding: 0.5rem 1.5rem;
  font-size: 1.2rem;
  cursor: pointer;
  background-color: #2196f3;
  color: white;
  border: none;
  border-radius: 4px;
  transition: background-color 0.2s;
}

button:hover {
  background-color: #1976d2;
}

.game-over {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: rgba(255, 255, 255, 0.9);
  padding: 2rem;
  border-radius: 10px;
  text-align: center;
  box-shadow: 0 0 20px rgba(0,0,0,0.2);
  z-index: 10;
  color: #333;
}

.start-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background-color: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}

.start-menu {
  background-color: rgba(255, 255, 255, 0.9);
  padding: 3rem;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  max-width: 800px;
  width: 90%;
  color: #333;
}

.start-menu h1 {
  font-size: 3rem;
  margin: 0;
  color: #1b5e20;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.1);
}

.subtitle {
  font-size: 1.2rem;
  color: #666;
  margin-bottom: 2rem;
}

.info-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 3rem;
  text-align: left;
  margin-bottom: 2.5rem;
}

.info-section h3 {
  font-size: 1.5rem;
  margin-bottom: 1rem;
  border-bottom: 2px solid #2196f3;
  padding-bottom: 0.5rem;
  display: inline-block;
}

.info-section ul {
  list-style: none;
  padding: 0;
}

.info-section li {
  margin-bottom: 0.8rem;
  font-size: 1.1rem;
  line-height: 1.4;
}

.info-section li span {
  font-weight: bold;
  color: #444;
}

.start-btn {
  padding: 1rem 3rem;
  font-size: 1.5rem;
  font-weight: bold;
  cursor: pointer;
  background-color: #2196f3;
  color: white;
  border: none;
  border-radius: 50px;
  transition: all 0.3s ease;
  width: auto;
  min-width: 250px;
}

.start-btn:hover {
  background-color: #1976d2;
  transform: scale(1.1);
  box-shadow: 0 5px 15px rgba(33, 150, 243, 0.4);
}


.game-over h2 {
  color: #f44336;
  margin-bottom: 1rem;
}

.game-over p {
  font-size: 1.2rem;
  margin: 0.5rem 0;
}
</style>
