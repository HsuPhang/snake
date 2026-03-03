<script setup>
import { ref, onMounted, onUnmounted, watch, computed } from 'vue';

// 游戏状态
const canvasRef = ref(null);
const score = ref(0);
const gameOver = ref(false);
const gameStarted = ref(true); // 游戏一开始就处于已开始状态
const isPaused = ref(true); // 游戏一开始就暂停
const direction = ref('right');
const nextDirection = ref('right');
const snake = ref([]);
const food = ref({});
const gridSize = 20;
const tileCount = 20;
let gameLoop = null;
const difficulty = ref('easy'); // 难度选择：easy, hard
const canvasSize = ref({ width: 400, height: 400 });

// 根据难度计算游戏速度
const gameSpeed = computed(() => {
  return difficulty.value === 'easy' ? 150 : 80;
});

// 初始化游戏
const initGame = () => {
  // 初始化蛇
  snake.value = [
    { x: 10, y: 10 },
    { x: 9, y: 10 },
    { x: 8, y: 10 }
  ];
  
  // 初始化食物
  spawnFood();
  
  // 重置分数和游戏状态
  score.value = 0;
  gameOver.value = false;
  direction.value = 'right';
  nextDirection.value = 'right';
};

// 生成食物
const spawnFood = () => {
  food.value = {
    x: Math.floor(Math.random() * tileCount),
    y: Math.floor(Math.random() * tileCount)
  };
  
  // 确保食物不会出现在蛇身上
  for (let segment of snake.value) {
    if (segment.x === food.value.x && segment.y === food.value.y) {
      spawnFood();
      return;
    }
  }
};

// 绘制游戏
const drawGame = () => {
  const canvas = canvasRef.value;
  if (!canvas) return;
  
  const ctx = canvas.getContext('2d');
  if (!ctx) return;
  
  // 绘制背景
  ctx.fillStyle = '#f5f5f5';
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  
  // 计算实际游戏区域大小
  const gameAreaSize = tileCount * gridSize;
  const offsetX = (canvas.width - gameAreaSize) / 2;
  const offsetY = (canvas.height - gameAreaSize) / 2;
  
  // 绘制游戏区域边框
  ctx.strokeStyle = '#4CAF50';
  ctx.lineWidth = 2;
  ctx.strokeRect(offsetX, offsetY, gameAreaSize, gameAreaSize);
  
  // 绘制网格
  ctx.strokeStyle = '#e0e0e0';
  ctx.lineWidth = 1;
  for (let i = 0; i <= tileCount; i++) {
    ctx.beginPath();
    ctx.moveTo(offsetX + i * gridSize, offsetY);
    ctx.lineTo(offsetX + i * gridSize, offsetY + gameAreaSize);
    ctx.stroke();
    ctx.beginPath();
    ctx.moveTo(offsetX, offsetY + i * gridSize);
    ctx.lineTo(offsetX + gameAreaSize, offsetY + i * gridSize);
    ctx.stroke();
  }
  
  // 绘制蛇
  if (snake.value && snake.value.length > 0) {
    ctx.fillStyle = '#4CAF50';
    for (let segment of snake.value) {
      ctx.fillRect(offsetX + segment.x * gridSize, offsetY + segment.y * gridSize, gridSize - 2, gridSize - 2);
    }
  }
  
  // 绘制食物
  if (food.value && food.value.x !== undefined && food.value.y !== undefined) {
    ctx.fillStyle = '#FF5722';
    ctx.fillRect(offsetX + food.value.x * gridSize, offsetY + food.value.y * gridSize, gridSize - 2, gridSize - 2);
  }
};

// 更新游戏
const updateGame = () => {
  if (gameOver.value) return;
  
  // 更新方向
  direction.value = nextDirection.value;
  
  // 移动蛇头
  const head = { ...snake.value[0] };
  
  switch (direction.value) {
    case 'up':
      head.y--;
      break;
    case 'down':
      head.y++;
      break;
    case 'left':
      head.x--;
      break;
    case 'right':
      head.x++;
      break;
  }
  
  // 检查边界碰撞
  if (head.x < 0 || head.x >= tileCount || head.y < 0 || head.y >= tileCount) {
    gameOver.value = true;
    if (gameLoop) clearInterval(gameLoop); // 清除游戏循环
    return;
  }
  
  // 检查自身碰撞
  for (let i = 1; i < snake.value.length; i++) {
    if (head.x === snake.value[i].x && head.y === snake.value[i].y) {
      gameOver.value = true;
      if (gameLoop) clearInterval(gameLoop); // 清除游戏循环
      return;
    }
  }
  
  // 检查食物碰撞
  if (head.x === food.value.x && head.y === food.value.y) {
    score.value++;
    spawnFood();
    // 蛇增长
    snake.value.unshift(head);
  } else {
    // 移动蛇
    snake.value.unshift(head);
    snake.value.pop();
  }
};

// 游戏主循环
const gameLoopFunction = () => {
  if (!isPaused.value && !gameOver.value) {
    updateGame();
  }
  drawGame();
};

// 处理键盘输入
const handleKeydown = (e) => {
  switch (e.key) {
    case 'ArrowUp':
      if (direction.value !== 'down') nextDirection.value = 'up';
      break;
    case 'ArrowDown':
      if (direction.value !== 'up') nextDirection.value = 'down';
      break;
    case 'ArrowLeft':
      if (direction.value !== 'right') nextDirection.value = 'left';
      break;
    case 'ArrowRight':
      if (direction.value !== 'left') nextDirection.value = 'right';
      break;
  }
};

// 开始游戏
const startGame = () => {
  initGame();
  gameStarted.value = true;
  gameOver.value = false;
  isPaused.value = true; // 游戏一开始就暂停
  if (gameLoop) clearInterval(gameLoop);
  gameLoop = setInterval(gameLoopFunction, gameSpeed.value);
};

// 重置游戏到开始界面
const resetGame = () => {
  gameOver.value = false;
  gameStarted.value = false;
  if (gameLoop) clearInterval(gameLoop);
  initGame(); // 重新初始化游戏数据
  drawGame(); // 重新绘制棋盘和蛇
};

// 适配屏幕大小
const resizeCanvas = () => {
  const container = document.querySelector('.game-container');
  if (container) {
    const containerWidth = container.clientWidth;
    const containerHeight = container.clientHeight;
    const maxSize = Math.min(containerWidth - 10, containerHeight - 80, 500);
    canvasSize.value = { width: maxSize, height: maxSize };
  }
};

// 监听难度变化
watch(difficulty, () => {
  if (gameLoop) {
    clearInterval(gameLoop);
    gameLoop = setInterval(gameLoopFunction, gameSpeed.value);
  }
});

// 生命周期钩子
const handleResize = () => {
  resizeCanvas();
  drawGame();
};

onMounted(() => {
  window.addEventListener('keydown', handleKeydown);
  window.addEventListener('resize', handleResize);
  startGame(); // 游戏一开始就开始并暂停
  resizeCanvas(); // 调整画布大小
  drawGame(); // 绘制棋盘和蛇
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown);
  window.removeEventListener('resize', handleResize);
  if (gameLoop) clearInterval(gameLoop);
});
</script>

<template>
  <div class="game-container">
    <h1>贪吃蛇游戏</h1>
    
    <div class="game-wrapper">
      <div class="canvas-container">
        <canvas 
          ref="canvasRef" 
          :width="canvasSize.width" 
          :height="canvasSize.height"
          class="game-canvas"
        ></canvas>
      </div>
      
      <div class="game-controls">
        <div class="control-section">
          <h3>游戏控制</h3>
          <div v-if="!gameStarted && !gameOver" class="start-controls">
            <div class="difficulty-selector">
              <span>选择难度:</span>
              <div class="difficulty-buttons">
                <button 
                  :class="{ active: difficulty === 'easy' }"
                  @click="difficulty = 'easy'"
                >
                  简单
                </button>
                <button 
                  :class="{ active: difficulty === 'hard' }"
                  @click="difficulty = 'hard'"
                >
                  进阶
                </button>
              </div>
            </div>
            <button @click="startGame" class="start-btn">开始游戏</button>
          </div>
          <div v-else-if="gameStarted && !gameOver" class="game-info">
            <div class="score">分数: {{ score }}</div>
            <div class="difficulty-selector">
              <span>难度:</span>
              <div class="difficulty-buttons">
                <button 
                  :class="{ active: difficulty === 'easy' }"
                  @click="difficulty = 'easy'"
                >
                  简单
                </button>
                <button 
                  :class="{ active: difficulty === 'hard' }"
                  @click="difficulty = 'hard'"
                >
                  进阶
                </button>
              </div>
            </div>
            <button @click="isPaused = !isPaused" class="pause-btn">
              {{ isPaused ? '开始游戏' : '暂停游戏' }}
            </button>
            <button @click="startGame" class="restart-btn">重新开始</button>
          </div>
          <div v-else-if="gameOver" class="game-over-controls">
            <div class="final-score">最终分数: {{ score }}</div>
            <div class="game-over-buttons">
              <button @click="startGame">再玩一次</button>
              <button @click="resetGame">重新选择</button>
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <!-- 游戏结束提示 -->
    <div v-if="gameOver" class="game-over-overlay">
      <h2>游戏结束!</h2>
      <button @click="gameOver = false" class="close-btn">关闭</button>
    </div>
  </div>
</template>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  background-color: #ffffff;
  color: #333333;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  padding: 20px;
}

h1 {
  font-size: 2.5rem;
  margin-bottom: 1.5rem;
  color: #4CAF50;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.1);
}

.game-wrapper {
  display: flex;
  gap: 30px;
  align-items: flex-start;
  max-width: 1200px;
  width: 100%;
}

.canvas-container {
  position: relative;
  flex-shrink: 0;
}

.game-canvas {
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  background-color: #f5f5f5;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.pseudo-loading {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(245, 245, 245, 0.9);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  z-index: 5;
}

.loading-animation {
  display: flex;
  gap: 10px;
  margin-bottom: 15px;
}

.loading-circle {
  width: 20px;
  height: 20px;
  border: 3px solid #e0e0e0;
  border-top: 3px solid #4CAF50;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

.loading-circle:nth-child(2) {
  animation-delay: 0.2s;
}

.loading-circle:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.pseudo-loading p {
  color: #666666;
  font-size: 1rem;
  margin: 0;
}

.game-controls {
  flex: 1;
  min-width: 250px;
  max-width: 300px;
}

.control-section {
  background-color: #f9f9f9;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.control-section h3 {
  font-size: 1.2rem;
  margin-bottom: 1.5rem;
  color: #333333;
  text-align: center;
}

.start-controls, .game-info, .game-over-controls {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.score, .final-score {
  font-weight: bold;
  color: #2196F3;
  font-size: 1.1rem;
  text-align: center;
}

.difficulty-selector {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.difficulty-selector span {
  font-weight: 500;
  color: #333333;
}

.difficulty-buttons {
  display: flex;
  gap: 10px;
}

.difficulty-selector button {
  flex: 1;
  padding: 0.4rem 0.8rem;
  font-size: 0.9rem;
  background-color: #f0f0f0;
  color: #333333;
  border: 1px solid #e0e0e0;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.difficulty-selector button.active {
  background-color: #4CAF50;
  color: white;
  border-color: #4CAF50;
}

.difficulty-selector button:hover {
  background-color: #e0e0e0;
}

.difficulty-selector button.active:hover {
  background-color: #45a049;
}

.restart-btn, .start-btn, .pause-btn {
  padding: 0.8rem 1.5rem;
  font-size: 1.1rem;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
  width: 100%;
  margin-top: 10px;
}

.restart-btn:hover, .start-btn:hover, .pause-btn:hover {
  background-color: #45a049;
}

.pause-btn {
  background-color: #2196F3;
}

.pause-btn:hover {
  background-color: #1976D2;
}

.game-over-buttons {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.game-over-buttons button {
  padding: 0.8rem 1.5rem;
  font-size: 1.1rem;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
  width: 100%;
}

.game-over-buttons button:hover {
  background-color: #45a049;
}

.game-over-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: rgba(255, 255, 255, 0.95);
  padding: 2rem;
  border-radius: 12px;
  text-align: center;
  border: 2px solid #FF5722;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
  min-width: 200px;
}

.game-over-overlay h2 {
  color: #FF5722;
  margin: 0 0 1.5rem 0;
  font-size: 1.8rem;
}

.close-btn {
  padding: 0.5rem 1rem;
  font-size: 1rem;
  background-color: #f44336;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.close-btn:hover {
  background-color: #d32f2f;
}

/* 响应式设计 */
@media (max-width: 768px) {
  h1 {
    font-size: 2rem;
  }
  
  .game-wrapper {
    flex-direction: column;
    align-items: center;
  }
  
  .game-controls {
    max-width: 100%;
    width: 100%;
  }
  
  .game-over-overlay {
    padding: 1.5rem;
    min-width: 150px;
  }
  
  .game-over-overlay h2 {
    font-size: 1.5rem;
  }
}
</style>