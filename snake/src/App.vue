<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from 'vue'

type Direction = 'up' | 'down' | 'left' | 'right'

interface Position {
  x: number
  y: number
}

const tileSize = 24
const columns = 20
const rows = 20
const tickMs = 130

const canvasRef = ref<HTMLCanvasElement | null>(null)
const score = ref(0)
const gameOver = ref(false)

const snake = ref<Position[]>([])
const food = ref<Position>({ x: 0, y: 0 })
const direction = ref<Direction>('right')
const pendingDirection = ref<Direction>('right')
const intervalId = ref<number | null>(null)

const canvasWidth = columns * tileSize
const canvasHeight = rows * tileSize
const boardSizeText = computed(() => `${columns}x${rows}`)

function randomInt(maxExclusive: number): number {
  return Math.floor(Math.random() * maxExclusive)
}

function isOpposite(next: Direction, current: Direction): boolean {
  return (
	(next === 'up' && current === 'down') ||
	(next === 'down' && current === 'up') ||
	(next === 'left' && current === 'right') ||
	(next === 'right' && current === 'left')
  )
}

function containsPosition(positions: Position[], position: Position): boolean {
  return positions.some((item) => item.x === position.x && item.y === position.y)
}

function spawnFood(): void {
  let nextFood: Position

  do {
	nextFood = {
	  x: randomInt(columns),
	  y: randomInt(rows),
	}
  } while (containsPosition(snake.value, nextFood))

  food.value = nextFood
}

function resetGame(): void {
  const centerX = Math.floor(columns / 2)
  const centerY = Math.floor(rows / 2)

  snake.value = [
	{ x: centerX - 1, y: centerY },
	{ x: centerX, y: centerY },
	{ x: centerX + 1, y: centerY },
  ]
  direction.value = 'right'
  pendingDirection.value = 'right'
  score.value = 0
  gameOver.value = false
  spawnFood()
  draw()
}

function nextHeadPosition(head: Position, currentDirection: Direction): Position {
  if (currentDirection === 'up') return { x: head.x, y: head.y - 1 }
  if (currentDirection === 'down') return { x: head.x, y: head.y + 1 }
  if (currentDirection === 'left') return { x: head.x - 1, y: head.y }
  return { x: head.x + 1, y: head.y }
}

function stopLoop(): void {
  if (intervalId.value !== null) {
	window.clearInterval(intervalId.value)
	intervalId.value = null
  }
}

function startLoop(): void {
  stopLoop()
  intervalId.value = window.setInterval(() => {
	tick()
  }, tickMs)
}

function tick(): void {
  if (gameOver.value) return

  if (!isOpposite(pendingDirection.value, direction.value)) {
	direction.value = pendingDirection.value
  }

  const head = snake.value[snake.value.length - 1]
  if (!head) {
    gameOver.value = true
    stopLoop()
    draw()
    return
  }

  const nextHead = nextHeadPosition(head, direction.value)

  const hitWall =
	nextHead.x < 0 || nextHead.x >= columns || nextHead.y < 0 || nextHead.y >= rows
  const hitSelf = containsPosition(snake.value, nextHead)

  if (hitWall || hitSelf) {
	gameOver.value = true
	stopLoop()
	draw()
	return
  }

  const nextSnake = [...snake.value, nextHead]
  const didEatFood = nextHead.x === food.value.x && nextHead.y === food.value.y

  if (didEatFood) {
	score.value += 1
	snake.value = nextSnake
	spawnFood()
  } else {
	nextSnake.shift()
	snake.value = nextSnake
  }

  draw()
}

function drawGrid(ctx: CanvasRenderingContext2D): void {
  ctx.strokeStyle = '#1f2937'
  ctx.lineWidth = 1

  for (let x = 0; x <= columns; x += 1) {
	const px = x * tileSize + 0.5
	ctx.beginPath()
	ctx.moveTo(px, 0)
	ctx.lineTo(px, canvasHeight)
	ctx.stroke()
  }

  for (let y = 0; y <= rows; y += 1) {
	const py = y * tileSize + 0.5
	ctx.beginPath()
	ctx.moveTo(0, py)
	ctx.lineTo(canvasWidth, py)
	ctx.stroke()
  }
}

function draw(): void {
  const canvas = canvasRef.value
  if (!canvas) return

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  ctx.clearRect(0, 0, canvasWidth, canvasHeight)
  ctx.fillStyle = '#0f172a'
  ctx.fillRect(0, 0, canvasWidth, canvasHeight)

  drawGrid(ctx)

  ctx.fillStyle = '#f43f5e'
  ctx.fillRect(food.value.x * tileSize, food.value.y * tileSize, tileSize, tileSize)

  ctx.fillStyle = '#22c55e'
  snake.value.forEach((part, index) => {
	const inset = index === snake.value.length - 1 ? 2 : 3
	ctx.fillRect(
	  part.x * tileSize + inset,
	  part.y * tileSize + inset,
	  tileSize - inset * 2,
	  tileSize - inset * 2,
	)
  })

  if (gameOver.value) {
	ctx.fillStyle = 'rgba(2, 6, 23, 0.72)'
	ctx.fillRect(0, 0, canvasWidth, canvasHeight)

	ctx.fillStyle = '#f8fafc'
	ctx.textAlign = 'center'
	ctx.textBaseline = 'middle'
	ctx.font = 'bold 34px Segoe UI'
	ctx.fillText('Game Over', canvasWidth / 2, canvasHeight / 2 - 20)
	ctx.font = '16px Segoe UI'
	ctx.fillText('Druecke Enter fuer Neustart', canvasWidth / 2, canvasHeight / 2 + 18)
  }
}

function onKeyDown(event: KeyboardEvent): void {
  switch (event.key) {
	case 'ArrowUp':
	  event.preventDefault()
	  pendingDirection.value = 'up'
	  break
	case 'ArrowDown':
	  event.preventDefault()
	  pendingDirection.value = 'down'
	  break
	case 'ArrowLeft':
	  event.preventDefault()
	  pendingDirection.value = 'left'
	  break
	case 'ArrowRight':
	  event.preventDefault()
	  pendingDirection.value = 'right'
	  break
	case 'Enter':
	  if (gameOver.value) {
		event.preventDefault()
		resetGame()
		startLoop()
	  }
	  break
  }
}

onMounted(() => {
  resetGame()
  startLoop()
  window.addEventListener('keydown', onKeyDown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', onKeyDown)
  stopLoop()
})
</script>

<template>
  <main class="app">
	<section class="hud">
	  <h1>Snake</h1>
	  <p>Score: {{ score }}</p>
	  <p>Board: {{ boardSizeText }}</p>
	  <p class="hint">Steuerung: Pfeiltasten, Neustart: Enter</p>
	</section>

	<canvas
	  ref="canvasRef"
	  class="board"
	  :width="canvasWidth"
	  :height="canvasHeight"
	  aria-label="Snake Spielfeld"
	  role="img"
	/>
  </main>
</template>

<style scoped>
.app {
  min-height: 100vh;
  display: grid;
  place-items: center;
  gap: 0.75rem;
  padding: 1.5rem;
  background: radial-gradient(circle at top, #111827, #020617 65%);
  color: #e2e8f0;
}

.hud {
  text-align: center;
}

.hud h1 {
  font-size: 2.1rem;
  line-height: 1.1;
  margin-bottom: 0.2rem;
}

.hint {
  color: #94a3b8;
}

.board {
  border: 2px solid #334155;
  border-radius: 8px;
  image-rendering: pixelated;
  box-shadow: 0 12px 32px rgba(0, 0, 0, 0.45);
}
</style>
