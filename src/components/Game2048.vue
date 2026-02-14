<template>
  <div class="game-container">
    <div class="header">
      <div class="score-container">
        <div class="score-label">СЧЕТ</div>
        <div class="score-value">{{ score }}</div>
      </div>
      <button class="restart-btn" @click="resetGame">Заново</button>
    </div>

    <div 
      class="board" 
      ref="boardRef"
      tabindex="0"
      @keydown.prevent="handleKey"
      @touchstart="handleTouchStart"
      @touchend="handleTouchEnd"
    >
      <div class="grid-background">
        <div v-for="n in 16" :key="n" class="grid-cell"></div>
      </div>

      <div class="tile-container">
        <div
          v-for="tile in tiles"
          :key="tile.id"
          class="tile"
          :class="['val-' + tile.value, { 'new': tile.isNew, 'merged': tile.isMerged }]"
          :style="getTileStyle(tile)"
        >
          {{ tile.value }}
        </div>
      </div>
      
      <div v-if="gameOver" class="game-over">
        <p>ИГРА ОКОНЧЕНА</p>
        <button @click="resetGame">Попробовать еще раз</button>
      </div>
    </div>
    
    <div class="instructions">
      Используй <strong>стрелки</strong> или <strong>свайпы</strong>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const GRID_SIZE = 4
const CELL_GAP = 10
const tiles = ref([])
const score = ref(0)
const gameOver = ref(false)
const boardRef = ref(null)

let startX = 0
let startY = 0

// Инициализация
const resetGame = () => {
  tiles.value = []
  score.value = 0
  gameOver.value = false
  addRandomTile()
  addRandomTile()
  // Фокус для клавиатуры
  if (boardRef.value) boardRef.value.focus()
}

const getTileStyle = (tile) => {
  // Для адаптивности используем проценты, так как размер поля может меняться
  // Позиция: (tile.col * 25)%
  return {
    left: `calc(${tile.col * 25}% + ${CELL_GAP/2}px)`,
    top: `calc(${tile.row * 25}% + ${CELL_GAP/2}px)`,
    width: `calc(25% - ${CELL_GAP}px)`,
    height: `calc(25% - ${CELL_GAP}px)`
  }
}

const addRandomTile = () => {
  const emptyCells = []
  for (let r = 0; r < GRID_SIZE; r++) {
    for (let c = 0; c < GRID_SIZE; c++) {
      if (!tiles.value.find(t => t.row === r && t.col === c)) {
        emptyCells.push({ r, c })
      }
    }
  }
  if (emptyCells.length === 0) return

  const { r, c } = emptyCells[Math.floor(Math.random() * emptyCells.length)]
  tiles.value.push({
    id: Date.now() + Math.random(),
    value: Math.random() < 0.9 ? 2 : 4,
    row: r,
    col: c,
    isNew: true
  })
}

// Логика движения
const move = (direction) => {
  if (gameOver.value) return

  // Очистка флагов
  tiles.value.forEach(t => { t.isNew = false; t.isMerged = false })

  let moved = false
  const vectors = {
    ArrowUp: { x: 0, y: -1 },
    ArrowDown: { x: 0, y: 1 },
    ArrowLeft: { x: -1, y: 0 },
    ArrowRight: { x: 1, y: 0 }
  }
  const vector = vectors[direction]
  if (!vector) return

  // Порядок обхода зависит от направления
  let traversals = { x: [], y: [] }
  for (let i = 0; i < GRID_SIZE; i++) {
    traversals.x.push(i)
    traversals.y.push(i)
  }
  if (vector.x === 1) traversals.x.reverse()
  if (vector.y === 1) traversals.y.reverse()

  const cellContent = (r, c) => tiles.value.find(t => t.row === r && t.col === c && !t.mergedToRemove)

  traversals.y.forEach(r => {
    traversals.x.forEach(c => {
      const tile = cellContent(r, c)
      if (tile) {
        let positions = findFarthestPosition(r, c, vector)
        let next = cellContent(positions.next.r, positions.next.c)

        if (next && next.value === tile.value && !next.isMerged) {
          // Merge
          const merged = {
            id: Date.now() + Math.random(),
            value: tile.value * 2,
            row: next.row,
            col: next.col,
            isMerged: true
          }
          // Помечаем старые для удаления
          tile.mergedToRemove = true 
          next.mergedToRemove = true // Важно не использовать этот next снова в этом ходу
          
          // Двигаем визуально старый тайл в точку слияния
          tile.row = next.row
          tile.col = next.col

          // Добавляем новый (но пока не рендерим, чтобы анимация прошла? 
          // Упростим: реактивность Vue сразу обновит DOM. 
          // Для красивой анимации нужно хранить старые. Сейчас делаем функционально.)
          
          tiles.value = tiles.value.filter(t => t !== tile && t !== next)
          tiles.value.push(merged)
          
          score.value += merged.value
          moved = true
        } else {
          // Move
          if (tile.row !== positions.farthest.r || tile.col !== positions.farthest.c) {
            tile.row = positions.farthest.r
            tile.col = positions.farthest.c
            moved = true
          }
        }
      }
    })
  })

  if (moved) {
    addRandomTile()
    if (!movesAvailable()) {
      gameOver.value = true
    }
  }
}

const findFarthestPosition = (r, c, vector) => {
  let prev
  do {
    prev = { r, c }
    r += vector.y
    c += vector.x
  } while (
    r >= 0 && r < GRID_SIZE &&
    c >= 0 && c < GRID_SIZE &&
    !tiles.value.find(t => t.row === r && t.col === c && !t.mergedToRemove)
  )
  return { farthest: prev, next: { r, c } }
}

const movesAvailable = () => {
  return tiles.value.length < 16 || 
         tiles.value.some(t => {
           // Проверка соседей
           const dirs = [{x:1,y:0}, {x:-1,y:0}, {x:0,y:1}, {x:0,y:-1}]
           return dirs.some(d => {
             const neighbor = tiles.value.find(n => n.row === t.row + d.y && n.col === t.col + d.x)
             return neighbor && neighbor.value === t.value
           })
         })
}

// --- Input Handling ---

const handleKey = (e) => {
  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'].includes(e.key)) {
    move(e.key)
  }
}

const handleTouchStart = (e) => {
  if (e.touches.length > 1) return // игнорируем мультитач
  startX = e.touches[0].clientX
  startY = e.touches[0].clientY
  // e.preventDefault() // Блокируем скролл? Лучше через CSS touch-action
}

const handleTouchEnd = (e) => {
  if (!startX || !startY) return
  
  const endX = e.changedTouches[0].clientX
  const endY = e.changedTouches[0].clientY
  
  const dx = endX - startX
  const dy = endY - startY
  
  const absDx = Math.abs(dx)
  const absDy = Math.abs(dy)
  
  if (Math.max(absDx, absDy) > 30) { // порог реакции
    if (absDx > absDy) {
      move(dx > 0 ? 'ArrowRight' : 'ArrowLeft')
    } else {
      move(dy > 0 ? 'ArrowDown' : 'ArrowUp')
    }
  }
  
  startX = 0
  startY = 0
}

onMounted(() => {
  resetGame()
})
</script>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  max-width: 500px;
  margin: 0 auto;
  user-select: none;
  touch-action: none; /* Важно для мобилок: отключает скролл браузера внутри игры */
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  padding: 0 10px 20px;
}

.score-container {
  background: #bbada0;
  padding: 5px 15px;
  border-radius: 6px;
  color: white;
  text-align: center;
}
.score-label { font-size: 12px; font-weight: bold; color: #eee4da; text-transform: uppercase; }
.score-value { font-size: 24px; font-weight: bold; }

.restart-btn {
  background: #8f7a66;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 6px;
  font-weight: bold;
  cursor: pointer;
  font-size: 16px;
}

.board {
  position: relative;
  background: #bbada0;
  border-radius: 8px;
  width: 100%;
  aspect-ratio: 1 / 1; /* Квадрат */
  padding: 10px;
  box-sizing: border-box;
  outline: none;
}

.grid-background {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-template-rows: repeat(4, 1fr);
  gap: 10px;
  width: 100%;
  height: 100%;
}

.grid-cell {
  background: rgba(238, 228, 218, 0.35);
  border-radius: 4px;
  width: 100%;
  height: 100%;
}

.tile-container {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  padding: 10px; /* Совпадает с padding .board */
  pointer-events: none;
}

.tile {
  position: absolute;
  background: #eee4da;
  color: #776e65;
  font-weight: bold;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 32px;
  transition: all 0.15s ease-in-out;
  z-index: 10;
}

/* Цвета плиток */
.val-2 { background: #eee4da; }
.val-4 { background: #ede0c8; }
.val-8 { background: #f2b179; color: #f9f6f2; }
.val-16 { background: #f59563; color: #f9f6f2; }
.val-32 { background: #f67c5f; color: #f9f6f2; }
.val-64 { background: #f65e3b; color: #f9f6f2; }
.val-128 { background: #edcf72; color: #f9f6f2; font-size: 28px; box-shadow: 0 0 30px 10px rgba(243, 215, 116, 0.23815); }
.val-256 { background: #edcc61; color: #f9f6f2; font-size: 28px; box-shadow: 0 0 30px 10px rgba(243, 215, 116, 0.31746); }
.val-512 { background: #edc850; color: #f9f6f2; font-size: 28px; box-shadow: 0 0 30px 10px rgba(243, 215, 116, 0.39683); }
.val-1024 { background: #edc53f; color: #f9f6f2; font-size: 24px; }
.val-2048 { background: #edc22e; color: #f9f6f2; font-size: 24px; box-shadow: 0 0 30px 10px rgba(243, 215, 116, 0.55556); }

.new { animation: appear 0.2s ease; }
.merged { animation: pop 0.2s ease; }

@keyframes appear {
  0% { opacity: 0; transform: scale(0); }
  100% { opacity: 1; transform: scale(1); }
}
@keyframes pop {
  0% { transform: scale(0.8); }
  50% { transform: scale(1.1); }
  100% { transform: scale(1); }
}

.game-over {
  position: absolute;
  inset: 0;
  background: rgba(238, 228, 218, 0.73);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 100;
  font-size: 30px;
  font-weight: bold;
  color: #776e65;
  border-radius: 8px;
}
.game-over button {
  margin-top: 20px;
  padding: 10px 20px;
  background: #8f7a66;
  color: #f9f6f2;
  border: none;
  cursor: pointer;
  font-size: 18px;
  border-radius: 6px;
}

.instructions {
  margin-top: 20px;
  color: #776e65;
  font-size: 14px;
}

/* Адаптация под маленькие экраны */
@media (max-width: 420px) {
  .header { padding: 0 5px 15px; }
  .tile { font-size: 24px; }
  .val-128, .val-256, .val-512 { font-size: 20px; }
  .val-1024, .val-2048 { font-size: 18px; }
}
</style>
