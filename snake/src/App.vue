<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from "vue";

type Direction = "up" | "down" | "left" | "right"
type MusicState = "idle" | "playing" | "lose" | "win"
type GameSpeed = "slow" | "normal" | "fast"

interface Position {
  x: number;
  y: number;
}

interface SpeedOption {
  value: GameSpeed;
  label: string;
  tickMs: number;
}

const tileSize = 24;
const columns = 15;
const rows = 15;
const tickMs = 130;
const speedOptions: SpeedOption[] = [
  { value: "slow", label: "Langsam", tickMs: 220 },
  { value: "normal", label: "Normal", tickMs },
  { value: "fast", label: "Schnell", tickMs: 85 }
];

const canvasRef = ref<HTMLCanvasElement | null>(null);
const score = ref(0);
const losses = ref(0);
const gameOver = ref(false);
const hasWon = ref(false);
const hasStarted = ref(false);
const lastDirectionBy = ref<"braut" | "braeutigam">("braut");
const lastDirectionIcon = ref<Direction>("right");

const snake = ref<Position[]>([]);
const food = ref<Position>({ x: 0, y: 0 });
const direction = ref<Direction>("right");
const pendingDirection = ref<Direction>("right");
const intervalId = ref<number | null>(null);
const audioContextRef = ref<AudioContext | null>(null);
const masterGainRef = ref<GainNode | null>(null);
const musicIntervalId = ref<number | null>(null);
const musicStep = ref(0);
const musicState = ref<MusicState>("idle");
const boatSailColorBySegment = ref<string[]>([]);
const nextBoatColorIndex = ref(0);
const selectedSpeed = ref<GameSpeed>("normal");
const isSettingsOpen = ref(false);

const canvasWidth = columns * tileSize;
const canvasHeight = rows * tileSize;
const heartViewBoxSize = 24;
const heartPathData =
  "M12 21.35 10.55 20.03C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3 9.24 3 10.91 3.81 12 5.09 13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5 22 12.28 18.6 15.36 13.45 20.04L12 21.35Z";
const colorBoardBg = "#e2e8f0";
const colorGrid = "#94a3b8";
const colorBoatHull = "#8b5a2b";
const colorBoatHullShadow = "#6f451f";
const colorBoatDeck = "#d6b58a";
const colorBoatSail = "#ffffff";
const colorBoatOutline = "#2b1a0e";
const boatSailColors = ["#dc2626", "#2563eb", "#16a34a", "#f59e0b", "#7c3aed"];
const colorFoodBody = "#bf0b2e";
const colorFoodOutline = "#7c0b26";
const colorOverlayBg = "rgba(248, 250, 252, 0.84)";
const colorOverlayText = "#0f172a";
const colorOverlayTextOutline = "#cbd5e1";
const musicNotes = [261.63, 329.63, 392.0, 349.23, 329.63, 293.66, 329.63, 392.0];
const loseMusicNotes = [293.66, 261.63, 220.0, 196.0, 174.61, 196.0];
const winMusicNotes = [523.25, 659.25, 783.99, 1046.5, 783.99, 1318.51, 1046.5, 783.99];
const winScore = 20;
const currentTickMs = computed(() => {
  return speedOptions.find((option) => option.value === selectedSpeed.value)?.tickMs ?? tickMs;
});

function initAudio(): void {
  if (audioContextRef.value && masterGainRef.value) {
    return;
  }

  const context = new window.AudioContext();
  const masterGain = context.createGain();
  masterGain.gain.value = 0.12;
  masterGain.connect(context.destination);

  audioContextRef.value = context;
  masterGainRef.value = masterGain;
}

function ensureAudioRunning(): void {
  initAudio();

  const context = audioContextRef.value;
  if (context && context.state === "suspended") {
    void context.resume();
  }
}

function playTone(
  frequency: number,
  durationSeconds: number,
  type: OscillatorType,
  volume: number,
  delaySeconds = 0
): void {
  const context = audioContextRef.value;
  const masterGain = masterGainRef.value;
  if (!context || !masterGain) return;

  const now = context.currentTime + delaySeconds;
  const oscillator = context.createOscillator();
  const gain = context.createGain();

  oscillator.type = type;
  oscillator.frequency.setValueAtTime(frequency, now);

  gain.gain.setValueAtTime(0.0001, now);
  gain.gain.exponentialRampToValueAtTime(Math.max(0.0001, volume), now + 0.02);
  gain.gain.exponentialRampToValueAtTime(0.0001, now + durationSeconds);

  oscillator.connect(gain);
  gain.connect(masterGain);

  oscillator.start(now);
  oscillator.stop(now + durationSeconds + 0.03);
}

function playEatSound(): void {
  ensureAudioRunning();
  playTone(740, 0.08, "triangle", 0.19);
  playTone(988, 0.1, "triangle", 0.17, 0.06);
}

function playTurnSound(): void {
  ensureAudioRunning();
  playTone(460, 0.05, "square", 0.12);
}

function playLoseSound(): void {
  ensureAudioRunning();
  playTone(330, 0.14, "sawtooth", 0.18);
  playTone(220, 0.18, "sawtooth", 0.17, 0.12);
  playTone(146.83, 0.24, "sawtooth", 0.15, 0.26);
}

function playWinSound(): void {
  ensureAudioRunning();
  playTone(523.25, 0.14, "triangle", 0.16);
  playTone(659.25, 0.14, "triangle", 0.16, 0.1);
  playTone(783.99, 0.16, "triangle", 0.18, 0.2);
  playTone(1046.5, 0.26, "triangle", 0.2, 0.32);
}

function startBackgroundMusic(nextState: Exclude<MusicState, "idle">): void {
  ensureAudioRunning();
  stopBackgroundMusic();
  musicState.value = nextState;
  musicStep.value = 0;

  const intervalMs = nextState === "playing" ? 360 : nextState === "lose" ? 520 : 320;

  musicIntervalId.value = window.setInterval(() => {
    const notes =
      musicState.value === "playing"
        ? musicNotes
        : musicState.value === "lose"
          ? loseMusicNotes
          : winMusicNotes;
    const note = notes[musicStep.value % notes.length];
    musicStep.value += 1;
    if (!note) return;

    if (musicState.value === "playing") {
      playTone(note, 0.28, "sine", 0.07);
      playTone(note / 2, 0.24, "triangle", 0.04, 0.02);
      return;
    }

    if (musicState.value === "lose") {
      playTone(note, 0.42, "sine", 0.075);
      playTone(note / 2, 0.46, "triangle", 0.042, 0.02);
      return;
    }

    playTone(note, 0.22, "triangle", 0.1);
    playTone(note * 1.5, 0.16, "square", 0.055, 0.04);
  }, intervalMs);
}

function stopBackgroundMusic(): void {
  if (musicIntervalId.value !== null) {
    window.clearInterval(musicIntervalId.value);
    musicIntervalId.value = null;
  }
}

function setMusicState(nextState: MusicState): void {
  if (musicState.value === nextState && (nextState === "idle" || musicIntervalId.value !== null)) {
    return;
  }

  if (nextState === "idle") {
    stopBackgroundMusic();
    musicState.value = "idle";
    return;
  }

  startBackgroundMusic(nextState);
}

function randomInt(maxExclusive: number): number {
  return Math.floor(Math.random() * maxExclusive);
}

function getNextBoatSailColor(): string {
  if (boatSailColors.length === 0) {
    return colorBoatSail;
  }

  const color = boatSailColors[nextBoatColorIndex.value % boatSailColors.length] ?? colorBoatSail;
  nextBoatColorIndex.value += 1;
  return color;
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
  nextBoatColorIndex.value = 0;
  boatSailColorBySegment.value = snake.value.map(() => getNextBoatSailColor());
  direction.value = "right";
  pendingDirection.value = "right";
  score.value = 0;
  gameOver.value = false;
  hasWon.value = false;
  hasStarted.value = false;
  setMusicState("idle");
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
  }, currentTickMs.value);
}

function tick(): void {
  if (gameOver.value || hasWon.value || !hasStarted.value) return;

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
    playLoseSound();
    setMusicState("lose");
    losses.value += 1;
    gameOver.value = true;
    stopLoop();
    draw();
    return;
  }

  const nextSnake = [...snake.value, nextHead];
  const didEatFood = nextHead.x === food.value.x && nextHead.y === food.value.y;

  if (didEatFood) {
    playEatSound();
    score.value += 1;
    snake.value = nextSnake;
    boatSailColorBySegment.value = [...boatSailColorBySegment.value, getNextBoatSailColor()];

    if (score.value >= winScore) {
      hasWon.value = true;
      stopLoop();
      playWinSound();
      setMusicState("win");
      draw();
      return;
    }

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
  const appleSize = (tileSize - 2) * 1.35;
  const offset = (tileSize - appleSize) / 2;
  const scale = appleSize / heartViewBoxSize;

  const heartPath = new Path2D(heartPathData);

  ctx.save();
  ctx.translate(baseX + offset, baseY + offset);
  ctx.scale(scale, scale);

  ctx.fillStyle = colorFoodBody;
  ctx.fill(heartPath);

  ctx.strokeStyle = colorFoodOutline;
  ctx.lineWidth = 1.5;
  ctx.lineJoin = "round";
  ctx.stroke(heartPath);

  ctx.restore();
}

function getDirectionVector(currentDirection: Direction): Position {
  if (currentDirection === "up") return { x: 0, y: -1 };
  if (currentDirection === "down") return { x: 0, y: 1 };
  if (currentDirection === "left") return { x: -1, y: 0 };
  return { x: 1, y: 0 };
}

function getSegmentCenter(position: Position): Position {
  return {
    x: position.x * tileSize + tileSize / 2,
    y: position.y * tileSize + tileSize / 2
  };
}

function getSegmentDirection(index: number, segments: Position[]): Position {
  const current = segments[index];
  if (!current) {
    return getDirectionVector(direction.value);
  }

  const next = segments[index + 1];
  if (next) {
    return {
      x: Math.sign(next.x - current.x),
      y: Math.sign(next.y - current.y)
    };
  }

  const previous = segments[index - 1];
  if (previous) {
    return {
      x: Math.sign(current.x - previous.x),
      y: Math.sign(current.y - previous.y)
    };
  }

  return getDirectionVector(direction.value);
}

function drawBoatSegment(
  ctx: CanvasRenderingContext2D,
  center: Position,
  segmentDirection: Position,
  sailColor: string
): void {
  const directionX = segmentDirection.x === 0 && segmentDirection.y === 0 ? 0 : segmentDirection.x;
  const directionY = segmentDirection.x === 0 && segmentDirection.y === 0 ? -1 : segmentDirection.y;
  const angle = Math.atan2(directionY, directionX) + Math.PI / 2;

  const scale = 1.05;
  const hullLength = tileSize * 0.72;
  const hullWidth = tileSize * 0.42;

  ctx.save();
  ctx.translate(center.x, center.y);
  ctx.rotate(angle);
  ctx.scale(scale, scale);

  ctx.fillStyle = "rgba(15, 23, 42, 0.14)";
  ctx.beginPath();
  ctx.ellipse(0, hullLength * 0.48, hullWidth * 0.5, hullWidth * 0.2, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = colorBoatHull;
  ctx.strokeStyle = colorBoatOutline;
  ctx.lineWidth = 1.1;
  ctx.lineJoin = "round";
  ctx.beginPath();
  ctx.moveTo(0, -hullLength * 0.56);
  ctx.quadraticCurveTo(hullWidth * 0.58, -hullLength * 0.1, hullWidth * 0.42, hullLength * 0.5);
  ctx.lineTo(-hullWidth * 0.42, hullLength * 0.5);
  ctx.quadraticCurveTo(-hullWidth * 0.58, -hullLength * 0.1, 0, -hullLength * 0.56);
  ctx.closePath();
  ctx.fill();
  ctx.stroke();

  ctx.fillStyle = colorBoatHullShadow;
  ctx.beginPath();
  ctx.moveTo(0, -hullLength * 0.42);
  ctx.quadraticCurveTo(hullWidth * 0.34, -hullLength * 0.06, hullWidth * 0.26, hullLength * 0.34);
  ctx.lineTo(-hullWidth * 0.26, hullLength * 0.34);
  ctx.quadraticCurveTo(-hullWidth * 0.34, -hullLength * 0.06, 0, -hullLength * 0.42);
  ctx.closePath();
  ctx.fill();

  ctx.fillStyle = colorBoatDeck;
  ctx.beginPath();
  ctx.roundRect(-hullWidth * 0.18, hullLength * 0.02, hullWidth * 0.36, hullLength * 0.22, hullWidth * 0.08);
  ctx.fill();

  ctx.strokeStyle = colorBoatOutline;
  ctx.lineWidth = 0.9;
  ctx.beginPath();
  ctx.moveTo(0, hullLength * 0.16);
  ctx.lineTo(0, -hullLength * 0.28);
  ctx.stroke();

  ctx.fillStyle = sailColor;
  ctx.beginPath();
  ctx.moveTo(0, -hullLength * 0.26);
  ctx.lineTo(hullWidth * 0.44, -hullLength * 0.02);
  ctx.lineTo(0, -hullLength * 0.02);
  ctx.closePath();
  ctx.fill();
  ctx.stroke();

  ctx.restore();
}

function drawSnake(ctx: CanvasRenderingContext2D): void {
  if (snake.value.length === 0) {
    return;
  }

  const segmentCenters = snake.value.map(getSegmentCenter);
  segmentCenters.forEach((segmentCenter, index) => {
    const segmentDirection = getSegmentDirection(index, snake.value);
    const sailColor = boatSailColorBySegment.value[index] ?? colorBoatSail;
    drawBoatSegment(ctx, segmentCenter, segmentDirection, sailColor);
  });
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

  if (gameOver.value || hasWon.value || !hasStarted.value) {
    ctx.fillStyle = colorOverlayBg;
    ctx.fillRect(0, 0, canvasWidth, canvasHeight);

    ctx.fillStyle = colorOverlayText;
    ctx.strokeStyle = colorOverlayTextOutline;
    ctx.textAlign = "center";
    ctx.textBaseline = "top";
    if (hasWon.value) {
      const titleLines = ["SUPER", "TEAMWORK"];
      const codeLines = ["Der zweite Teil", "vom Code ist XYZ"];
      const titleMaxWidth = canvasWidth * 0.9;
      const codeMaxWidth = canvasWidth * 0.9;
      const titleFontSize = Math.min(
        ...titleLines.map((line) => getFittedFontSize(ctx, line, titleMaxWidth, "800", 116, 52))
      );
      const codeFontSize = Math.min(
        ...codeLines.map((line) => getFittedFontSize(ctx, line, codeMaxWidth, "800", 76, 34))
      );
      const titleLineHeight = titleFontSize * 0.88;
      const codeLineHeight = codeFontSize * 0.96;
      const spacerHeight = 22;
      const blockHeight = titleLineHeight * titleLines.length + spacerHeight + codeLineHeight * codeLines.length;
      const blockStartY = (canvasHeight - blockHeight) / 2;

      ctx.font = `800 ${titleFontSize}px Segoe UI`;
      ctx.lineWidth = 6;
      drawCenteredTextLinesWithOutline(ctx, titleLines, canvasWidth / 2, blockStartY, titleLineHeight, 6);

      ctx.font = `800 ${codeFontSize}px Segoe UI`;
      ctx.lineWidth = 6;
      drawCenteredTextLinesWithOutline(
        ctx,
        codeLines,
        canvasWidth / 2,
        blockStartY + titleLineHeight * titleLines.length + spacerHeight,
        codeLineHeight,
        6
      );
    } else if (gameOver.value) {
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
      const hintMaxWidth = canvasWidth * 0.84;
      const hintFontSize = Math.min(
        ...hintLines.map((line) => getFittedFontSize(ctx, line, hintMaxWidth, "600", 28, 18))
      );
      const hintLineHeight = hintFontSize * 1.05;
      const blockHeight = hintLineHeight * hintLines.length;
      const blockStartY = (canvasHeight - blockHeight) / 2;

      ctx.font = `600 ${hintFontSize}px Segoe UI`;
      ctx.lineWidth = 3;
      drawCenteredTextLinesWithOutline(ctx, hintLines, canvasWidth / 2, blockStartY, hintLineHeight, 3);
    }
  }
}

function onResize(): void {
  draw();
}

function onKeyDown(event: KeyboardEvent): void {
  if (event.key === "Escape" && isSettingsOpen.value) {
    closeSettingsDialog();
    event.preventDefault();
    return;
  }

  if (isSettingsOpen.value) {
    return;
  }

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

  if (hasWon.value) {
    event.preventDefault();
    return;
  }

  ensureAudioRunning();

  if (nextDirection !== pendingDirection.value) {
    playTurnSound();
  }

  lastDirectionIcon.value = nextDirection;
  lastDirectionBy.value = nextDirection === "up" || nextDirection === "down" ? "braeutigam" : "braut";

  event.preventDefault();

  if (!hasStarted.value || gameOver.value) {
    if (gameOver.value) {
      resetGame();
    }

    hasStarted.value = true;
    setMusicState("playing");

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

function toggleSettingsDialog(): void {
  isSettingsOpen.value = !isSettingsOpen.value;
}

function closeSettingsDialog(): void {
  isSettingsOpen.value = false;
}

function setGameSpeed(nextSpeed: GameSpeed): void {
  if (selectedSpeed.value === nextSpeed) {
    return;
  }

  selectedSpeed.value = nextSpeed;

  // Laufende Runde direkt mit neuer Tick-Rate fortsetzen.
  if (hasStarted.value && !gameOver.value && !hasWon.value) {
    startLoop();
  }
}

onMounted(() => {
  resetGame();
  window.addEventListener("keydown", onKeyDown);
  window.addEventListener("resize", onResize);
});

onUnmounted(() => {
  setMusicState("idle");

  if (audioContextRef.value) {
    void audioContextRef.value.close();
    audioContextRef.value = null;
    masterGainRef.value = null;
  }

  window.removeEventListener("keydown", onKeyDown);
  window.removeEventListener("resize", onResize);
  stopLoop();
});
</script>

<template>
  <main class="app">
    <button
      class="settings-trigger"
      type="button"
      aria-label="Einstellungen oeffnen"
      title="Einstellungen"
      @click="toggleSettingsDialog"
    >
      <svg viewBox="0 0 24 24" aria-hidden="true">
        <path
          d="M10.6 2h2.8l.5 2.1c.6.2 1.2.5 1.8.9l2-.8 2 2-1 2c.4.6.7 1.2.9 1.8l2.1.5v2.8l-2.1.5c-.2.6-.5 1.2-.9 1.8l1 2-2 2-2-.8c-.6.4-1.2.7-1.8.9l-.5 2.1h-2.8l-.5-2.1c-.6-.2-1.2-.5-1.8-.9l-2 .8-2-2 1-2c.4-.6.7-1.2.9-1.8L2 13.4v-2.8l2.1-.5c.2-.6.5-1.2.9-1.8l-1-2 2-2 2 .8c.6-.4 1.2-.7 1.8-.9L10.6 2Zm1.4 6a4 4 0 1 0 0 8 4 4 0 0 0 0-8Z"
          fill="currentColor"
        />
      </svg>
    </button>

    <div v-if="isSettingsOpen" class="settings-overlay" @click="closeSettingsDialog">
      <section
        class="settings-dialog"
        role="dialog"
        aria-modal="true"
        aria-labelledby="settings-title"
        @click.stop
      >
        <div class="settings-header">
          <h2 id="settings-title">Einstellungen</h2>
          <button class="settings-close" type="button" aria-label="Dialog schliessen" @click="closeSettingsDialog">
            ✕
          </button>
        </div>

        <p class="settings-subtitle">Geschwindigkeit</p>

        <div class="speed-options">
          <label
            v-for="option in speedOptions"
            :key="option.value"
            class="speed-option"
            :class="{ active: selectedSpeed === option.value }"
          >
            <input
              type="radio"
              name="game-speed"
              :value="option.value"
              :checked="selectedSpeed === option.value"
              @change="setGameSpeed(option.value)"
            >
            <span>{{ option.label }}</span>
          </label>
        </div>
      </section>
    </div>

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
          <svg class="label-icon score-heart-icon" viewBox="0 0 24 24">
            <g transform="translate(12 12) scale(0.8) translate(-12 -12)">
              <path
                d="M12 21.35 10.55 20.03C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3 9.24 3 10.91 3.81 12 5.09 13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5 22 12.28 18.6 15.36 13.45 20.04L12 21.35Z"
                fill="#bf0b2e"
                stroke="#7c0b26"
                stroke-width="1.2"
                stroke-linejoin="round"
              />
            </g>
          </svg>
        </div>

        <div class="score-value-row">
          <p class="score-value">{{ score }}</p>
          <p class="score-target">&nbsp;von {{ winScore }}</p>
        </div>
      </div>

      <div class="hud-item">
        <p class="score-label">Verloren</p>

        <div class="hud-icon" aria-hidden="true">
          <svg class="label-icon loss-plate-icon" viewBox="0 0 24 24">
            <circle cx="12" cy="12" r="8" fill="#ffffff" stroke="#0f172a" stroke-width="1.2" />
            <circle cx="12" cy="12" r="5.2" fill="#f8fafc" stroke="#cbd5e1" stroke-width="0.9" />
            <path
              d="M12 4 10.8 7.4 12.7 10.7 11.1 13.7 13 16.9 11.8 20"
              fill="none"
              stroke="#0f172a"
              stroke-width="0.95"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
            <path
              d="M12.7 10.8 15.9 9.4 14.1 12.4 17.2 13.7 19.9 12.1"
              fill="none"
              stroke="#0f172a"
              stroke-width="0.95"
              stroke-linecap="round"
              stroke-linejoin="round"
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
  background: radial-gradient(circle at top, #f8fafc, #dbeafe 72%);
  color: #0f172a;
  position: relative;
}

.settings-trigger {
  position: absolute;
  top: 1rem;
  right: 1rem;
  width: 52px;
  height: 52px;
  display: grid;
  place-items: center;
  border-radius: 12px;
  border: 2px solid #0f172a;
  background: rgba(248, 250, 252, 0.88);
  color: #0f172a;
  cursor: pointer;
  z-index: 25;
}

.settings-trigger:hover {
  background: #f8fafc;
}

.settings-trigger svg {
  width: 27px;
  height: 27px;
}

.settings-overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.36);
  display: grid;
  place-items: center;
  z-index: 40;
}

.settings-dialog {
  width: min(360px, calc(100vw - 2rem));
  background: #f8fafc;
  border: 2px solid #0f172a;
  border-radius: 14px;
  box-shadow: 0 18px 36px rgba(15, 23, 42, 0.22);
  padding: 1rem 1rem 1.1rem;
}

.settings-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.settings-header h2 {
  margin: 0;
  font-size: 2rem;
}

.settings-close {
  border: 1px solid #64748b;
  border-radius: 8px;
  background: #ffffff;
  color: #0f172a;
  width: 32px;
  height: 32px;
  cursor: pointer;
}

.settings-subtitle {
  margin: 1rem 0 0.5rem;
  font-size: 0.95rem;
  font-weight: 700;
  color: #334155;
  text-transform: uppercase;
  letter-spacing: 0.06em;
}

.speed-options {
  display: grid;
  gap: 0.55rem;
}

.speed-option {
  display: flex;
  align-items: center;
  gap: 0.65rem;
  border: 2px solid #cbd5e1;
  border-radius: 10px;
  padding: 0.6rem 0.7rem;
  background: #ffffff;
  cursor: pointer;
}

.speed-option.active {
  border-color: #2563eb;
  background: #eff6ff;
}

.speed-option input {
  accent-color: #2563eb;
}

.speed-option span {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
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
  --counter-size: 9rem;
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
    "label ."
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

.last-direction-label,
.score-label {
  margin: 0;
  font-size: 1.3rem;
  text-transform: uppercase;
  letter-spacing: 0.14em;
  color: #334155;
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
  border: 2px solid #0f172a;
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
  color: #0f172a;
}

.score-label {
  grid-area: label;
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

.direction-icon {
  filter: drop-shadow(1px 0 0 #0f172a) drop-shadow(-1px 0 0 #0f172a) drop-shadow(0 1px 0 #0f172a) drop-shadow(0 -1px 0 #0f172a);
}

.pixel-portrait {
  filter: drop-shadow(4px 0 0 #22273e) drop-shadow(-4px 0 0 #22273e) drop-shadow(0 4px 0 #22273e) drop-shadow(0 -4px 0 #22273e);
}

.score-heart-icon {
  filter: none;
}

.score-value {
  grid-area: value;
  margin: 0;
  font-size: var(--counter-size);
  line-height: 0.95;
  font-weight: 800;
  color: #0f172a;
}

.score-value-row {
  grid-area: value;
  display: flex;
  align-items: flex-end;
  gap: 0.45rem;
}

.score-target {
  margin: 0 0 0.75rem;
  font-size: calc(var(--counter-size) * 0.28);
  font-weight: 700;
  line-height: 1;
  color: #334155;
  text-transform: uppercase;
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
