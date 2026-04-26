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
const losses = ref(0);
const gameOver = ref(false);
const hasStarted = ref(false);
const lastDirectionBy = ref<"braut" | "braeutigam">("braut");
const lastDirectionIcon = ref<Direction>("right");

const snake = ref<Position[]>([]);
const food = ref<Position>({ x: 0, y: 0 });
const direction = ref<Direction>("right");
const pendingDirection = ref<Direction>("right");
const intervalId = ref<number | null>(null);

const canvasWidth = columns * tileSize;
const canvasHeight = rows * tileSize;
const appleViewBoxSize = 24;
const appleBodyPathData = "M12 7.4C9.3 4 4.5 4.8 4 10c-.5 5 3 9.7 8 9.7s8.5-4.7 8-9.7c-.5-5.2-5.3-6-8-2.6Z";
const appleStemPathData = "M12 7.3c-.1-2.1.7-3.8 2.2-5";
const appleLeafPathData = "M12.8 4.4c1.4-1.1 2.9-1.4 4.6-.8-1 1.5-2.6 2.1-4.6.8Z";
const colorBoardBg = "#1e293b";
const colorGrid = "#64748b";
const colorSnakeBody = "#16a34a";
const colorSnakeHighlight = "#86efac";
const colorFoodBody = "#ff4d6d";
const colorFoodStem = "#6b3f1d";
const colorFoodLeaf = "#34d399";
const colorOverlayBg = "rgba(15, 23, 42, 0.8)";
const colorOverlayText = "#ffffff";
const colorOverlayTextOutline = "#020617";

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
    losses.value += 1;
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
  ctx.strokeStyle = colorGrid;
  ctx.lineWidth = 1.2;

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

function drawFood(ctx: CanvasRenderingContext2D, position: Position): void {
  const baseX = position.x * tileSize;
  const baseY = position.y * tileSize;
  const appleSize = tileSize - 2;
  const offset = (tileSize - appleSize) / 2;
  const scale = appleSize / appleViewBoxSize;

  const appleBodyPath = new Path2D(appleBodyPathData);
  const appleStemPath = new Path2D(appleStemPathData);
  const appleLeafPath = new Path2D(appleLeafPathData);

  ctx.save();
  ctx.translate(baseX + offset, baseY + offset);
  ctx.scale(scale, scale);

  ctx.fillStyle = colorFoodBody;
  ctx.fill(appleBodyPath);

  ctx.strokeStyle = colorFoodStem;
  ctx.lineWidth = 1.6;
  ctx.lineCap = "round";
  ctx.stroke(appleStemPath);

  ctx.fillStyle = colorFoodLeaf;
  ctx.fill(appleLeafPath);

  ctx.restore();
}

function getDirectionVector(currentDirection: Direction): Position {
  if (currentDirection === "up") return { x: 0, y: -1 };
  if (currentDirection === "down") return { x: 0, y: 1 };
  if (currentDirection === "left") return { x: -1, y: 0 };
  return { x: 1, y: 0 };
}

function getPerpendicularVector(currentDirection: Direction): Position {
  if (currentDirection === "up") return { x: 1, y: 0 };
  if (currentDirection === "down") return { x: -1, y: 0 };
  if (currentDirection === "left") return { x: 0, y: -1 };
  return { x: 0, y: 1 };
}

function getSegmentCenter(position: Position): Position {
  return {
    x: position.x * tileSize + tileSize / 2,
    y: position.y * tileSize + tileSize / 2
  };
}

function drawSnake(ctx: CanvasRenderingContext2D): void {
  if (snake.value.length === 0) {
    return;
  }

  const segmentCenters = snake.value.map(getSegmentCenter);
  const firstSegment = segmentCenters[0];
  const head = segmentCenters[segmentCenters.length - 1];

  if (!firstSegment || !head) {
    return;
  }

  const bodyWidth = tileSize - 8;
  const bodyRadius = bodyWidth / 2;

  ctx.save();

  if (segmentCenters.length === 1) {
    ctx.fillStyle = colorSnakeBody;
    ctx.beginPath();
    ctx.arc(firstSegment.x, firstSegment.y, bodyRadius, 0, Math.PI * 2);
    ctx.fill();
  } else {
    ctx.strokeStyle = colorSnakeBody;
    ctx.lineWidth = bodyWidth;
    ctx.lineCap = "round";
    ctx.lineJoin = "round";
    ctx.beginPath();
    ctx.moveTo(firstSegment.x, firstSegment.y);

    for (let index = 1; index < segmentCenters.length; index += 1) {
      const segment = segmentCenters[index];
      if (!segment) continue;
      ctx.lineTo(segment.x, segment.y);
    }

    ctx.stroke();

    ctx.strokeStyle = colorSnakeHighlight;
    ctx.lineWidth = bodyWidth * 0.45;
    ctx.beginPath();
    ctx.moveTo(firstSegment.x, firstSegment.y);

    for (let index = 1; index < segmentCenters.length; index += 1) {
      const segment = segmentCenters[index];
      if (!segment) continue;
      ctx.lineTo(segment.x, segment.y);
    }

    ctx.stroke();
  }

  const headDirection = getDirectionVector(direction.value);
  const perpendicular = getPerpendicularVector(direction.value);
  const headRadius = bodyRadius + 2.5;
  const snoutRadius = headRadius * 0.88;
  const snoutCenter = {
    x: head.x + headDirection.x * headRadius * 0.5,
    y: head.y + headDirection.y * headRadius * 0.5
  };

  ctx.fillStyle = colorSnakeBody;
  ctx.beginPath();
  ctx.arc(head.x, head.y, headRadius, 0, Math.PI * 2);
  ctx.arc(snoutCenter.x, snoutCenter.y, snoutRadius, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = colorSnakeHighlight;
  ctx.beginPath();
  ctx.arc(
    head.x - headDirection.x * 1.2,
    head.y - headDirection.y * 1.2,
    headRadius * 0.72,
    0,
    Math.PI * 2
  );
  ctx.fill();

  const eyeOffsetForward = headRadius * 0.28;
  const eyeOffsetSide = headRadius * 0.42;
  const eyeRadius = Math.max(1.8, tileSize * 0.1);
  const pupilRadius = eyeRadius * 0.48;
  const pupilForwardOffset = eyeRadius * 0.32;

  const leftEye = {
    x: head.x + headDirection.x * eyeOffsetForward + perpendicular.x * eyeOffsetSide,
    y: head.y + headDirection.y * eyeOffsetForward + perpendicular.y * eyeOffsetSide
  };
  const rightEye = {
    x: head.x + headDirection.x * eyeOffsetForward - perpendicular.x * eyeOffsetSide,
    y: head.y + headDirection.y * eyeOffsetForward - perpendicular.y * eyeOffsetSide
  };

  ctx.fillStyle = "#f8fafc";
  ctx.beginPath();
  ctx.arc(leftEye.x, leftEye.y, eyeRadius, 0, Math.PI * 2);
  ctx.arc(rightEye.x, rightEye.y, eyeRadius, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#0f172a";
  ctx.beginPath();
  ctx.arc(
    leftEye.x + headDirection.x * pupilForwardOffset,
    leftEye.y + headDirection.y * pupilForwardOffset,
    pupilRadius,
    0,
    Math.PI * 2
  );
  ctx.arc(
    rightEye.x + headDirection.x * pupilForwardOffset,
    rightEye.y + headDirection.y * pupilForwardOffset,
    pupilRadius,
    0,
    Math.PI * 2
  );
  ctx.fill();

  ctx.restore();
}

function getFittedFontSize(
  ctx: CanvasRenderingContext2D,
  text: string,
  maxWidth: number,
  fontWeight: string,
  maxSize: number,
  minSize: number
): number {
  for (let size = maxSize; size >= minSize; size -= 1) {
    ctx.font = `${fontWeight} ${size}px Segoe UI`;
    if (ctx.measureText(text).width <= maxWidth) {
      return size;
    }
  }

  return minSize;
}

function drawCenteredTextLinesWithOutline(
  ctx: CanvasRenderingContext2D,
  lines: string[],
  centerX: number,
  startY: number,
  lineHeight: number,
  outlineWidth: number
): void {
  lines.forEach((line, index) => {
    const y = startY + index * lineHeight;
    ctx.strokeText(line, centerX, y);
    ctx.fillText(line, centerX, y);
  });

  ctx.lineWidth = outlineWidth;
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
  ctx.fillStyle = colorBoardBg;
  ctx.fillRect(0, 0, canvasWidth, canvasHeight);

  drawGrid(ctx);

  drawFood(ctx, food.value);

  drawSnake(ctx);

  if (gameOver.value || !hasStarted.value) {
    ctx.fillStyle = colorOverlayBg;
    ctx.fillRect(0, 0, canvasWidth, canvasHeight);

    ctx.fillStyle = colorOverlayText;
    ctx.strokeStyle = colorOverlayTextOutline;
    ctx.textAlign = "center";
    ctx.textBaseline = "top";
    if (gameOver.value) {
      const titleLines = ["GAME", "OVER"];
      const hintLines = ["Drücke eine Pfeiltaste", "zum Starten"];
      const titleMaxWidth = canvasWidth * 0.9;
      const hintMaxWidth = canvasWidth * 0.84;
      const titleFontSize = Math.min(
        ...titleLines.map((line) => getFittedFontSize(ctx, line, titleMaxWidth, "800", 116, 52))
      );
      const hintFontSize = Math.min(
        ...hintLines.map((line) => getFittedFontSize(ctx, line, hintMaxWidth, "600", 28, 18))
      );
      const titleLineHeight = titleFontSize * 0.88;
      const hintLineHeight = hintFontSize * 1.05;
      const blockHeight = titleLineHeight * titleLines.length + 22 + hintLineHeight * hintLines.length;
      const blockStartY = (canvasHeight - blockHeight) / 2;

      ctx.font = `800 ${titleFontSize}px Segoe UI`;
      ctx.lineWidth = 6;
      drawCenteredTextLinesWithOutline(ctx, titleLines, canvasWidth / 2, blockStartY, titleLineHeight, 6);

      ctx.font = `600 ${hintFontSize}px Segoe UI`;
      ctx.lineWidth = 3;
      drawCenteredTextLinesWithOutline(
        ctx,
        hintLines,
        canvasWidth / 2,
        blockStartY + titleLineHeight * titleLines.length + 22,
        hintLineHeight,
        3
      );
    } else {
      ctx.font = "bold 30px Segoe UI";

      const hintLines = ["Drücke eine Pfeiltaste", "zum Starten"];
      const hintMaxWidth = canvasWidth * 0.82;
      const hintFontSize = Math.min(
        ...hintLines.map((line) => getFittedFontSize(ctx, line, hintMaxWidth, "500", 24, 16))
      );
      const hintLineHeight = hintFontSize * 1.05;

      ctx.font = `500 ${hintFontSize}px Segoe UI`;
      ctx.lineWidth = 3;
      drawCenteredTextLinesWithOutline(ctx, hintLines, canvasWidth / 2, canvasHeight / 2 + 8, hintLineHeight, 3);
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

  lastDirectionIcon.value = nextDirection;
  lastDirectionBy.value = nextDirection === "up" || nextDirection === "down" ? "braeutigam" : "braut";

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
      <div class="hud-item">
        <p class="score-label">Score</p>

        <div class="hud-icon" aria-hidden="true">
          <svg class="label-icon" viewBox="0 0 24 24">
            <g transform="translate(0 1)">
              <path
                d="M12 7.4C9.3 4 4.5 4.8 4 10c-.5 5 3 9.7 8 9.7s8.5-4.7 8-9.7c-.5-5.2-5.3-6-8-2.6Z"
                fill="#ff4d6d"
              />
              <path
                d="M12 7.3c-.1-2.1.7-3.8 2.2-5"
                fill="none"
                stroke="#6b3f1d"
                stroke-width="1.6"
                stroke-linecap="round"
              />
              <path d="M12.8 4.4c1.4-1.1 2.9-1.4 4.6-.8-1 1.5-2.6 2.1-4.6.8Z" fill="#34d399" />
            </g>
          </svg>
        </div>

        <p class="score-value">{{ score }}</p>
      </div>

      <div class="hud-item">
        <p class="score-label">Verloren</p>

        <div class="hud-icon" aria-hidden="true">
          <svg class="label-icon" viewBox="0 0 24 24">
            <path
              d="M12 5.2c-3.4 0-5.9 2.4-5.9 5.5 0 2.1 1.1 3.9 2.9 4.9V18c0 .9.7 1.6 1.6 1.6h2.8c.9 0 1.6-.7 1.6-1.6v-2.4c1.8-1 2.9-2.8 2.9-4.9 0-3.1-2.5-5.5-5.9-5.5Z"
              fill="#f8fafc"
            />
            <ellipse cx="9.7" cy="10.4" rx="1.4" ry="1.7" fill="#0f172a" />
            <ellipse cx="14.3" cy="10.4" rx="1.4" ry="1.7" fill="#0f172a" />
            <path d="M12 12.1 10.9 14h2.2L12 12.1Z" fill="#0f172a" />
            <rect x="9.4" y="15.2" width="5.2" height="2" rx="0.6" fill="#e2e8f0" />
            <path
              d="M11.1 15.4v1.6M12.9 15.4v1.6"
              fill="none"
              stroke="#94a3b8"
              stroke-width="0.7"
              stroke-linecap="round"
            />
          </svg>
        </div>

        <p class="score-value">{{ losses }}</p>
      </div>

      <div class="last-direction">
        <p class="last-direction-label">Letzte Richtung von</p>

        <div class="last-direction-content">
          <div class="last-direction-arrow" aria-label="Letzte Richtung">
            <svg class="direction-icon" :class="`dir-${lastDirectionIcon}`" viewBox="0 0 24 24" aria-hidden="true">
              <path
                d="M12 3 4.8 10.2h4.6v10.8h5.2V10.2h4.6L12 3Z"
                fill="#f8fafc"
                stroke="#0f172a"
                stroke-width="1.3"
                stroke-linejoin="round"
              />
            </svg>
          </div>

          <svg v-if="lastDirectionBy === 'braut'" class="pixel-portrait" viewBox="0 0 60 60" aria-hidden="true">
            <rect x="14" y="4" width="32" height="4" fill="#f8fafc" />
            <rect x="10" y="8" width="40" height="4" fill="#f8fafc" />
            <rect x="8" y="12" width="44" height="20" fill="#eef3f8" />
            <rect x="6" y="20" width="6" height="22" fill="#e3ebf3" />
            <rect x="48" y="20" width="6" height="22" fill="#e3ebf3" />

            <rect x="16" y="10" width="28" height="5" fill="#9a6537" />
            <rect x="12" y="15" width="36" height="6" fill="#8a582f" />
            <rect x="10" y="21" width="8" height="14" fill="#7a4d2a" />
            <rect x="42" y="21" width="8" height="14" fill="#7a4d2a" />
            <rect x="18" y="20" width="24" height="18" fill="#f8ecdf" />
            <rect x="20" y="23" width="20" height="11" fill="#fcf2e8" />
            <rect x="24" y="35" width="12" height="3" fill="#e7d0bc" />
            <rect x="23" y="38" width="14" height="2" fill="#dcc2ad" />

            <rect x="23" y="25" width="4" height="2" fill="#7a4d2a" />
            <rect x="33" y="25" width="4" height="2" fill="#7a4d2a" />
            <rect x="24" y="27" width="3" height="3" fill="#0f172a" />
            <rect x="33" y="27" width="3" height="3" fill="#0f172a" />
            <rect x="28" y="31" width="4" height="2" fill="#f4a9a9" />
            <rect x="27" y="33" width="6" height="1" fill="#e68d8d" />

            <rect x="14" y="40" width="32" height="5" fill="#ffffff" />
            <rect x="10" y="45" width="40" height="6" fill="#f8fbff" />
            <rect x="8" y="51" width="44" height="5" fill="#edf4fb" />
            <rect x="22" y="41" width="16" height="4" fill="#f8fafc" />
            <rect x="26" y="45" width="8" height="4" fill="#dbe7f3" />
          </svg>

          <svg v-else class="pixel-portrait" viewBox="0 0 60 60" aria-hidden="true">
            <rect x="16" y="5" width="28" height="4" fill="#3a2517" />
            <rect x="12" y="9" width="36" height="5" fill="#2b1c12" />
            <rect x="10" y="14" width="40" height="5" fill="#24170f" />
            <rect x="10" y="19" width="8" height="11" fill="#24170f" />
            <rect x="42" y="19" width="8" height="11" fill="#24170f" />

            <rect x="18" y="18" width="24" height="18" fill="#f8ecdf" />
            <rect x="20" y="21" width="20" height="11" fill="#fcf2e8" />
            <rect x="23" y="33" width="14" height="3" fill="#e6cfbc" />
            <rect x="25" y="36" width="10" height="2" fill="#d9beaa" />

            <rect x="23" y="24" width="3" height="3" fill="#111827" />
            <rect x="34" y="24" width="3" height="3" fill="#111827" />
            <rect x="28" y="29" width="4" height="2" fill="#efaaaa" />
            <rect x="27" y="31" width="6" height="1" fill="#e08e8e" />

            <rect x="12" y="40" width="36" height="5" fill="#111111" />
            <rect x="9" y="45" width="42" height="6" fill="#0b0b0b" />
            <rect x="8" y="51" width="44" height="5" fill="#171717" />
            <rect x="24" y="40" width="12" height="5" fill="#f8fafc" />
            <rect x="26" y="45" width="8" height="11" fill="#f8fafc" />
            <rect x="28" y="45" width="4" height="11" fill="#111827" />
          </svg>

          <p class="last-direction-name">{{ lastDirectionBy === "braut" ? "Catherine" : "Stefan" }}</p>
        </div>
      </div>
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
  background: radial-gradient(circle at top, #334155, #0f172a 72%);
  color: #f8fafc;
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
  --counter-size: clamp(4rem, 8vw, 7rem);
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  width: 33%;
  flex: 0 0 33%;
  max-width: 33%;
  padding: 1.1rem 1.2rem;
  border-radius: 14px;
}

.hud-item {
  display: grid;
  grid-template-columns: max-content 1fr;
  grid-template-areas:
    ". label"
    "icon value";
  align-items: center;
  column-gap: 1rem;
  gap: 0.45rem;
}

.hud-item + .hud-item {
  margin-top: 64px;
}

.last-direction {
  margin-top: 128px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 0.8rem;
}

.last-direction-label {
  margin: 0;
  font-size: 1.2rem;
  text-transform: uppercase;
  letter-spacing: 0.14em;
  color: #dbeafe;
}

.last-direction-content {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.last-direction-arrow {
  width: 78px;
  height: 78px;
  border-radius: 12px;
  background: rgba(15, 23, 42, 0.75);
  border: 2px solid #93c5fd;
  display: grid;
  place-items: center;
  box-shadow: 0 0 0 2px rgba(186, 230, 253, 0.2);
}

.direction-icon {
  width: 54px;
  height: 54px;
  transform-origin: center;
}

.direction-icon.dir-up {
  transform: rotate(0deg);
}

.direction-icon.dir-right {
  transform: rotate(90deg);
}

.direction-icon.dir-down {
  transform: rotate(180deg);
}

.direction-icon.dir-left {
  transform: rotate(270deg);
}

.pixel-portrait {
  width: 172px;
  height: 216px;
  image-rendering: pixelated;
  shape-rendering: crispEdges;
}

.last-direction-name {
  margin: 0;
  font-size: 3rem;
  font-weight: 700;
  color: #f8fafc;
}

.score-label {
  grid-area: label;
  margin: 0;
  font-size: 1rem;
  text-transform: uppercase;
  letter-spacing: 0.18em;
  color: #dbeafe;
}

.hud-icon {
  grid-area: icon;
  display: flex;
  align-items: center;
  justify-content: center;
  flex: 0 0 auto;
  height: var(--counter-size);
}

.label-icon {
  display: block;
  width: auto;
  height: 100%;
}

.score-value {
  grid-area: value;
  margin: 0;
  font-size: var(--counter-size);
  line-height: 0.95;
  font-weight: 800;
  color: #f8fafc;
}

.board {
  width: auto;
  height: 100%;
  max-width: 100%;
  max-height: 100%;
  border: 3px solid #93c5fd;
  border-radius: 8px;
  box-shadow: 0 14px 34px rgba(0, 0, 0, 0.45), 0 0 0 2px rgba(186, 230, 253, 0.22);
}
</style>
