<script setup lang="ts">
import { onMounted, onUnmounted, ref } from "vue";

type Direction = "up" | "down" | "left" | "right"

interface Position {
  x: number;
  y: number;
}

const tileSize = 24;
const columns = 20;
const rows = 20;
const tickMs = 130;

const canvasRef = ref<HTMLCanvasElement | null>(null);
const score = ref(0);
const gameOver = ref(false);
const hasStarted = ref(false);

const snake = ref<Position[]>([]);
const food = ref<Position>({ x: 0, y: 0 });
const direction = ref<Direction>("right");
const pendingDirection = ref<Direction>("right");
const intervalId = ref<number | null>(null);

const canvasWidth = columns * tileSize;
const canvasHeight = rows * tileSize;

function randomInt(maxExclusive: number): number {
  return Math.floor(Math.random() * maxExclusive);
}

function isOpposite(next: Direction, current: Direction): boolean {
  return (
    (next === "up" && current === "down") ||
    (next === "down" && current === "up") ||
    (next === "left" && current === "right") ||
    (next === "right" && current === "left")
  );
}

function containsPosition(positions: Position[], position: Position): boolean {
  return positions.some((item) => item.x === position.x && item.y === position.y);
}

function spawnFood(): void {
  if (columns < 3 || rows < 3) {
    // Fallback fuer ungueltige Spielfeldgroessen ohne Innenbereich.
    food.value = { x: 0, y: 0 };
    return;
  }

  let nextFood: Position;

  do {
    nextFood = {
      x: randomInt(columns - 2) + 1,
      y: randomInt(rows - 2) + 1
    };
  } while (containsPosition(snake.value, nextFood));

  food.value = nextFood;
}

function resetGame(): void {
  const centerX = Math.floor(columns / 2);
  const centerY = Math.floor(rows / 2);

  snake.value = [
    { x: centerX - 1, y: centerY },
    { x: centerX, y: centerY },
    { x: centerX + 1, y: centerY }
  ];
  direction.value = "right";
  pendingDirection.value = "right";
  score.value = 0;
  gameOver.value = false;
  hasStarted.value = false;
  spawnFood();
  draw();
}

function nextHeadPosition(head: Position, currentDirection: Direction): Position {
  if (currentDirection === "up") return { x: head.x, y: head.y - 1 };
  if (currentDirection === "down") return { x: head.x, y: head.y + 1 };
  if (currentDirection === "left") return { x: head.x - 1, y: head.y };
  return { x: head.x + 1, y: head.y };
}

function stopLoop(): void {
  if (intervalId.value !== null) {
    window.clearInterval(intervalId.value);
    intervalId.value = null;
  }
}

function startLoop(): void {
  stopLoop();
  intervalId.value = window.setInterval(() => {
    tick();
  }, tickMs);
}

function tick(): void {
  if (gameOver.value || !hasStarted.value) return;

  if (!isOpposite(pendingDirection.value, direction.value)) {
    direction.value = pendingDirection.value;
  }

  const head = snake.value[snake.value.length - 1];
  if (!head) {
    gameOver.value = true;
    stopLoop();
    draw();
    return;
  }

  const nextHead = nextHeadPosition(head, direction.value);

  const hitWall =
    nextHead.x < 0 || nextHead.x >= columns || nextHead.y < 0 || nextHead.y >= rows;
  const hitSelf = containsPosition(snake.value, nextHead);

  if (hitWall || hitSelf) {
    gameOver.value = true;
    stopLoop();
    draw();
    return;
  }

  const nextSnake = [...snake.value, nextHead];
  const didEatFood = nextHead.x === food.value.x && nextHead.y === food.value.y;

  if (didEatFood) {
    score.value += 1;
    snake.value = nextSnake;
    spawnFood();
  } else {
    nextSnake.shift();
    snake.value = nextSnake;
  }

  draw();
}

function drawGrid(ctx: CanvasRenderingContext2D): void {
  ctx.strokeStyle = "#1f2937";
  ctx.lineWidth = 1;

  for (let x = 0; x <= columns; x += 1) {
    const px = x * tileSize + 0.5;
    ctx.beginPath();
    ctx.moveTo(px, 0);
    ctx.lineTo(px, canvasHeight);
    ctx.stroke();
  }

  for (let y = 0; y <= rows; y += 1) {
    const py = y * tileSize + 0.5;
    ctx.beginPath();
    ctx.moveTo(0, py);
    ctx.lineTo(canvasWidth, py);
    ctx.stroke();
  }
}

function syncCanvasResolution(canvas: HTMLCanvasElement, ctx: CanvasRenderingContext2D): void {
  const dpr = window.devicePixelRatio || 1;
  const displayWidth = Math.max(1, Math.round(canvas.clientWidth));
  const displayHeight = Math.max(1, Math.round(canvas.clientHeight));
  const pixelWidth = Math.max(1, Math.round(displayWidth * dpr));
  const pixelHeight = Math.max(1, Math.round(displayHeight * dpr));

  if (canvas.width !== pixelWidth || canvas.height !== pixelHeight) {
    canvas.width = pixelWidth;
    canvas.height = pixelHeight;
  }

  ctx.setTransform(canvas.width / canvasWidth, 0, 0, canvas.height / canvasHeight, 0, 0);
}

function draw(): void {
  const canvas = canvasRef.value;
  if (!canvas) return;

  const ctx = canvas.getContext("2d");
  if (!ctx) return;

  syncCanvasResolution(canvas, ctx);

  ctx.clearRect(0, 0, canvasWidth, canvasHeight);
  ctx.fillStyle = "#0f172a";
  ctx.fillRect(0, 0, canvasWidth, canvasHeight);

  drawGrid(ctx);

  ctx.fillStyle = "#f43f5e";
  ctx.fillRect(food.value.x * tileSize, food.value.y * tileSize, tileSize, tileSize);

  ctx.fillStyle = "#22c55e";
  snake.value.forEach((part, index) => {
    const inset = index === snake.value.length - 1 ? 2 : 3;
    ctx.fillRect(
      part.x * tileSize + inset,
      part.y * tileSize + inset,
      tileSize - inset * 2,
      tileSize - inset * 2
    );
  });

  if (gameOver.value || !hasStarted.value) {
    ctx.fillStyle = "rgba(2, 6, 23, 0.72)";
    ctx.fillRect(0, 0, canvasWidth, canvasHeight);

    ctx.fillStyle = "#f8fafc";
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    if (gameOver.value) {
      ctx.font = "bold 34px Segoe UI";
      ctx.fillText("Game Over", canvasWidth / 2, canvasHeight / 2 - 20);
      ctx.font = "16px Segoe UI";
      ctx.fillText("Drücke eine Pfeiltaste für Neustart", canvasWidth / 2, canvasHeight / 2 + 18);
    } else {
      ctx.font = "bold 30px Segoe UI";
      ctx.fillText("Snake", canvasWidth / 2, canvasHeight / 2 - 20);
      ctx.font = "16px Segoe UI";
      ctx.fillText("Drücke eine Pfeiltaste zum Start", canvasWidth / 2, canvasHeight / 2 + 18);
    }
  }
}

function onResize(): void {
  draw();
}

function onKeyDown(event: KeyboardEvent): void {
  let nextDirection: Direction | null = null;

  switch (event.key) {
    case "ArrowUp":
      nextDirection = "up";
      break;
    case "ArrowDown":
      nextDirection = "down";
      break;
    case "ArrowLeft":
      nextDirection = "left";
      break;
    case "ArrowRight":
      nextDirection = "right";
      break;
  }

  if (!nextDirection) {
    return;
  }

  event.preventDefault();

  if (!hasStarted.value || gameOver.value) {
    if (gameOver.value) {
      resetGame();
    }

    hasStarted.value = true;

    if (!isOpposite(nextDirection, direction.value)) {
      direction.value = nextDirection;
    }
    pendingDirection.value = nextDirection;
    startLoop();
    draw();
    return;
  }

  pendingDirection.value = nextDirection;
}

onMounted(() => {
  resetGame();
  window.addEventListener("keydown", onKeyDown);
  window.addEventListener("resize", onResize);
});

onUnmounted(() => {
  window.removeEventListener("keydown", onKeyDown);
  window.removeEventListener("resize", onResize);
  stopLoop();
});
</script>

<template>
  <main class="app">
    <div class="board-area">
      <canvas
        ref="canvasRef"
        class="board"
        :width="canvasWidth"
        :height="canvasHeight"
        aria-label="Snake Spielfeld"
        role="img"
      />
    </div>

    <section class="hud">
      <p class="score-label">Score</p>
      <p class="score-value">{{ score }}</p>
    </section>
  </main>
</template>

<style scoped>
.app {
  height: 100dvh;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 3.5rem;
  padding: 1.5rem;
  box-sizing: border-box;
  overflow: hidden;
  background: radial-gradient(circle at top, #111827, #020617 65%);
  color: #e2e8f0;
}

.board-area {
  flex: 1 1 auto;
  min-width: 0;
  height: calc(100dvh - 3rem);
  display: flex;
  align-items: center;
  justify-content: center;
}

.hud {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 0.3rem;
  width: 33%;
  flex: 0 0 33%;
  max-width: 33%;
}

.score-label {
  margin: 0;
  font-size: 1rem;
  text-transform: uppercase;
  letter-spacing: 0.18em;
  color: #94a3b8;
}

.score-value {
  margin: 0;
  font-size: clamp(4rem, 8vw, 7rem);
  line-height: 0.95;
  font-weight: 800;
  color: #f8fafc;
}

.board {
  width: auto;
  height: 100%;
  max-width: 100%;
  max-height: 100%;
  border: 2px solid #334155;
  border-radius: 8px;
  box-shadow: 0 12px 32px rgba(0, 0, 0, 0.45);
}
</style>
