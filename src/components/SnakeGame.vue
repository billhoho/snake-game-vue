<template>
  <div class="game-container">
    <div class="stats-container">
      <div class="stat">Score: {{ score }}</div>
      <div class="stat">Time: {{ formatTime(timer) }}</div>
      <div class="stat">Best: {{ highScore }}</div>
    </div>
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
          'food': isFood(cell) 
        }"
      ></div>
    </div>
    <div class="controls">
      <button v-if="!gameState.isPlaying" @click="startGame">Start Game</button>
      <button v-else @click="resetGame">Reset</button>
    </div>
    <div v-if="gameState.isGameOver" class="game-over">
      <h2>Game Over!</h2>
      <p>Your Score: {{ score }}</p>
      <p>Best Score: {{ highScore }}</p>
      <button @click="startGame">Play Again</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue';

const gridSize = 20;
const initialSnake = [{ x: 10, y: 10 }, { x: 10, y: 11 }, { x: 10, y: 12 }];
const initialDirection = { x: 0, y: -1 }; // Moving up

const snake = ref([...initialSnake]);
const food = ref({ x: 5, y: 5 });
const direction = ref({ ...initialDirection });
const lastProcessedDirection = ref({ ...initialDirection });
const score = ref(0);
const timer = ref(0);
const highScore = ref(parseInt(localStorage.getItem('snake-high-score')) || 0);

const gameState = ref({
  isPlaying: false,
  isGameOver: false,
});

let gameInterval = null;
let timerInterval = null;

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

const isFood = (cell) => {
  return food.value.x === cell.x && food.value.y === cell.y;
};

const generateFood = () => {
  let newFood;
  while (true) {
    newFood = {
      x: Math.floor(Math.random() * gridSize),
      y: Math.floor(Math.random() * gridSize),
    };
    if (!snake.value.some(segment => segment.x === newFood.x && segment.y === newFood.y)) {
      break;
    }
  }
  food.value = newFood;
};

const moveSnake = () => {
  // Update the last processed direction to current direction
  lastProcessedDirection.value = { ...direction.value };

  const head = { ...snake.value[0] };
  head.x += direction.value.x;
  head.y += direction.value.y;

  // Wall collision
  if (head.x < 0 || head.x >= gridSize || head.y < 0 || head.y >= gridSize) {
    gameOver();
    return;
  }

  // Self collision
  if (snake.value.some(segment => segment.x === head.x && segment.y === head.y)) {
    gameOver();
    return;
  }

  snake.value.unshift(head);

  // Food collision
  if (head.x === food.value.x && head.y === food.value.y) {
    score.value++;
    generateFood();
  } else {
    snake.value.pop();
  }
};

const startGame = () => {
  resetGame();
  gameState.value.isPlaying = true;
  gameState.value.isGameOver = false;
  gameInterval = setInterval(moveSnake, 150);
  timerInterval = setInterval(() => {
    timer.value++;
  }, 1000);
};

const resetGame = () => {
  snake.value = [...initialSnake];
  direction.value = { ...initialDirection };
  lastProcessedDirection.value = { ...initialDirection };
  score.value = 0;
  timer.value = 0;
  generateFood();
};

const gameOver = () => {
  gameState.value.isPlaying = false;
  gameState.value.isGameOver = true;
  clearInterval(gameInterval);
  clearInterval(timerInterval);

  if (score.value > highScore.value) {
    highScore.value = score.value;
    localStorage.setItem('snake-high-score', highScore.value.toString());
  }
};

const formatTime = (seconds) => {
  const mins = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return `${mins}:${secs.toString().padStart(2, '0')}`;
};

const handleKeyDown = (e) => {
  switch (e.key) {
    case 'ArrowUp':
      if (lastProcessedDirection.value.y !== 1) direction.value = { x: 0, y: -1 };
      break;
    case 'ArrowDown':
      if (lastProcessedDirection.value.y !== -1) direction.value = { x: 0, y: 1 };
      break;
    case 'ArrowLeft':
      if (lastProcessedDirection.value.x !== 1) direction.value = { x: -1, y: 0 };
      break;
    case 'ArrowRight':
      if (lastProcessedDirection.value.x !== -1) direction.value = { x: 1, y: 0 };
      break;
  }
};

onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown);
  clearInterval(gameInterval);
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
  color: #333;
}

.stats-container {
  display: flex;
  gap: 2rem;
  margin-bottom: 1rem;
  font-size: 1.5rem;
  font-weight: bold;
}

.stat {
  display: flex;
  align-items: center;
}

.game-board {
  display: grid;
  background-color: #ddd;
  border: 5px solid #555;
  width: 400px;
  height: 400px;
}

.cell {
  width: 20px;
  height: 20px;
  border: 0.1px solid rgba(0,0,0,0.05);
  box-sizing: border-box;
}

.snake-head {
  background-color: #2e7d32;
  border-radius: 4px 4px 2px 2px;
  z-index: 2;
  box-shadow: inset 0 0 4px rgba(0,0,0,0.3);
}

.snake-body {
  background-color: #4caf50;
  border-radius: 2px;
}

.snake-tail {
  background-color: #81c784;
  border-radius: 2px 2px 4px 4px;
}

.food {
  background-color: #f44336;
  border-radius: 50%;
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
