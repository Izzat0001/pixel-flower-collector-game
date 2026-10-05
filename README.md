const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
ctx.imageSmoothingEnabled = false;

const scoreEl = document.getElementById('score');

const WORLD = {
  cols: 18,
  rows: 12,
  tileW: 42,
  tileH: 24,
};

const FLOWER_COLORS = ['#f9f7f2', '#ff4a4a', '#ff73b7', '#4d7cf8', '#9e62ff', '#63d9ff', '#f9d14d'];

const state = {
  flowers: [],
  score: 0,
  maxFlowers: 15,
  lastSpawn: 0,
  spawnEvery: 5000,
  pointer: { x: 0, y: 0 },
  selectedFlowerId: null,
};

const player = {
  x: 7,
  y: 7,
  targetX: 7,
  targetY: 7,
  targetFlowerId: null,
  speed: 2.8,
  step: 0,
};

function resetCanvasSize() {
  const scale = window.devicePixelRatio || 1;
  const w = window.innerWidth;
  const h = window.innerHeight;
  canvas.width = Math.floor(w * scale);
  canvas.height = Math.floor(h * scale);
  canvas.style.width = `${w}px`;
  canvas.style.height = `${h}px`;
  ctx.setTransform(scale, 0, 0, scale, 0, 0);
}

function isoToScreen(x, y, heightOffset = 0) {
  const centerX = canvas.width / (window.devicePixelRatio || 1) / 2;
  const centerY = canvas.height / (window.devicePixelRatio || 1) * 0.72;
  const sx = (x - y) * (WORLD.tileW / 2) + centerX;
  const sy = (x + y) * (WORLD.tileH / 2) + centerY - heightOffset;
  return { x: sx, y: sy };
}

function drawBackground() {
  const w = canvas.width / (window.devicePixelRatio || 1);
  const h = canvas.height / (window.devicePixelRatio || 1);

  const sky = ctx.createLinearGradient(0, 0, 0, h);
  sky.addColorStop(0, '#a6e6ff');
  sky.addColorStop(0.45, '#d6f3d3');
  sky.addColorStop(1, '#7fc76d');
  ctx.fillStyle = sky;
  ctx.fillRect(0, 0, w, h);

  ctx.fillStyle = 'rgba(90, 164, 99, 0.55)';
  ctx.beginPath();
  ctx.moveTo(0, h * 0.62);
  ctx.quadraticCurveTo(w * 0.2, h * 0.42, w * 0.4, h * 0.62);
  ctx.quadraticCurveTo(w * 0.64, h * 0.7, w * 0.82, h * 0.56);
  ctx.quadraticCurveTo(w * 0.9, h * 0.5, w, h * 0.62);
  ctx.lineTo(w, h);
  ctx.lineTo(0, h);
  ctx.closePath();
  ctx.fill();

  for (let y = 0; y < WORLD.rows; y++) {
    for (let x = 0; x < WORLD.cols; x++) {
      const px = x - y;
      const py = (x + y) * 0.5;
      const tx = (px * WORLD.tileW) + (w / 2) + 12;
      const ty = (py * WORLD.tileH) + (h * 0.7) + (x % 4) * 0.8;
      const shade = ['#5bbf53', '#79d063', '#60b94f', '#4b983d', '#d9d66a'];
      const color = shade[(x * 3 + y * 5) % shade.length];

      ctx.fillStyle = color;
      ctx.beginPath();
      ctx.moveTo(tx, ty);
      ctx.lineTo(tx + WORLD.tileW / 2, ty + WORLD.tileH / 2);
      ctx.lineTo(tx, ty + WORLD.tileH);
      ctx.lineTo(tx - WORLD.tileW / 2, ty + WORLD.tileH / 2);
      ctx.closePath();
      ctx.fill();

      ctx.strokeStyle = 'rgba(255,255,255,0.08)';
      ctx.stroke();
    }
  }
}

function flowerAtScreenPosition(flower) {
  const p = isoToScreen(flower.x, flower.y, 10);
  return {
    x: p.x,
    y: p.y,
    radius: 16,
  };
}

function spawnFlower() {
  const x = 2 + Math.random() * (WORLD.cols - 4);
  const y = 2 + Math.random() * (WORLD.rows - 4);

  const tooClose = state.flowers.some((flower) => {
    const dx = flower.x - x;
    const dy = flower.y - y;
    return Math.hypot(dx, dy) < 1.5;
  });

  if (tooClose) {
    return; 
  }

  const flower = {
    id: `${Date.now()}-${Math.random()}`,
    x,
    y,
    color: FLOWER_COLORS[Math.floor(Math.random() * FLOWER_COLORS.length)],
    rotation: Math.random() * Math.PI,
    scale: 0.9 + Math.random() * 0.8,
  };

  state.flowers.push(flower);
}

function ensureFlowers() {
  if (state.flowers.length < state.maxFlowers) {
    const need = state.maxFlowers - state.flowers.length;
    for (let i = 0; i < need; i++) {
      spawnFlower();
    }
  }
}

function updateScore() {
  scoreEl.textContent = String(state.score);
}

function drawFlower(flower) {
  const p = isoToScreen(flower.x, flower.y, 10);
  const radius = 14 * flower.scale;

  ctx.fillStyle = 'rgba(0, 0, 0, 0.22)';
  ctx.beginPath();
  ctx.ellipse(p.x, p.y + 18, 18, 7, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.strokeStyle = '#2f7d30';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(p.x, p.y + 4);
  ctx.lineTo(p.x, p.y - 26);
  ctx.stroke();

  ctx.fillStyle = '#4b8b2d';
  ctx.beginPath();
  ctx.arc(p.x - 6, p.y - 18, 5, 0, Math.PI * 2);
  ctx.arc(p.x + 6, p.y - 16, 5, 0, Math.PI * 2);
  ctx.fill();

  const petalCount = 7;
  for (let i = 0; i < petalCount; i++) {
    const angle = (Math.PI * 2 * i) / petalCount + flower.rotation;
    const px = p.x + Math.cos(angle) * (radius * 0.9);
    const py = p.y - 26 + Math.sin(angle) * (radius * 0.9);

    ctx.fillStyle = flower.color;
    ctx.beginPath();
    ctx.ellipse(px, py, 7, 12, angle, 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.fillStyle = '#f9cf4d';
  ctx.beginPath();
  ctx.arc(p.x, p.y - 26, 6, 0, Math.PI * 2);
  ctx.fill();
}

function drawPlayer() {
  const p = isoToScreen(player.x, player.y, 9);

  ctx.fillStyle = 'rgba(0,0,0,0.2)';
  ctx.beginPath();
  ctx.ellipse(p.x, p.y + 18, 14, 6, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#d8a072';
  ctx.fillRect(p.x - 5, p.y - 10, 10, 8);

  ctx.fillStyle = '#3d2b1f';
  ctx.fillRect(p.x - 3, p.y - 18, 6, 8);

  ctx.fillStyle = '#5ca4ff';
  ctx.fillRect(p.x - 7, p.y - 22, 4, 4);
  ctx.fillRect(p.x + 3, p.y - 22, 4, 4);

  ctx.fillStyle = '#2d2d2d';
  ctx.fillRect(p.x - 8, p.y - 2, 4, 10);
  ctx.fillRect(p.x + 4, p.y - 2, 4, 10);

  ctx.fillStyle = '#f3c67a';
  ctx.fillRect(p.x - 6, p.y - 20, 12, 4);
}

function getFlowerById(id) {
  return state.flowers.find((flower) => flower.id === id) || null;
}

function collectFlower(flowerId) {
  const index = state.flowers.findIndex((flower) => flower.id === flowerId);
  if (index < 0) return;

  state.flowers.splice(index, 1);
  state.score += 1;
  updateScore();
  state.selectedFlowerId = null;
  player.targetFlowerId = null;
  player.targetX = player.x;
  player.targetY = player.y;
}

function updatePlayer(dt) {
  const targetFlower = player.targetFlowerId ? getFlowerById(player.targetFlowerId) : null;

  if (targetFlower) {
    const dx = targetFlower.x - player.x;
    const dy = targetFlower.y - player.y;
    const distance = Math.hypot(dx, dy);

    if (distance > 0.4) {
      const step = (player.speed * dt) / 16.67;
      const moveX = (dx / distance) * Math.min(step, distance);
      const moveY = (dy / distance) * Math.min(step, distance);
      player.x += moveX;
      player.y += moveY;
    } else {
      collectFlower(targetFlower.id);
    }
  }
}

function pointerToWorld(x, y) {
  const worldX = ((x - (canvas.width / (window.devicePixelRatio || 1)) / 2) / (WORLD.tileW / 2) + y / (WORLD.tileH / 2)) / 2;
  const worldY = ((y - (canvas.height / (window.devicePixelRatio || 1)) * 0.72) / (WORLD.tileH / 2) - worldX) * 0.5;
  return { x: worldX, y: worldY };
}

function handlePointerDown(event) {
  const rect = canvas.getBoundingClientRect();
  const px = event.clientX - rect.left;
  const py = event.clientY - rect.top;

  for (let i = state.flowers.length - 1; i >= 0; i--) {
    const flower = state.flowers[i];
    const pos = flowerAtScreenPosition(flower);
    const dx = px - pos.x;
    const dy = py - pos.y;

    if (Math.hypot(dx, dy) < 18) {
      player.targetFlowerId = flower.id;
      state.selectedFlowerId = flower.id;
      return;
    }
  }
}

function gameLoop(timestamp) {
  const dt = timestamp - (gameLoop.lastTime || timestamp);
  gameLoop.lastTime = timestamp;

  if (timestamp - state.lastSpawn > state.spawnEvery) {
    state.lastSpawn = timestamp;
    if (state.flowers.length < state.maxFlowers) {
      spawnFlower();
    }
  }

  updatePlayer(dt);
  drawBackground();

  for (const flower of state.flowers) {
    drawFlower(flower);
  }

  drawPlayer();

  requestAnimationFrame(gameLoop);
}

function init() {
  resetCanvasSize();
  ensureFlowers();
  updateScore();

  canvas.addEventListener('pointerdown', handlePointerDown);
  window.addEventListener('resize', resetCanvasSize);

  requestAnimationFrame(gameLoop);
}

init();

window.addEventListener('pointermove', (event) => {
  const rect = canvas.getBoundingClientRect();
  state.pointer.x = event.clientX - rect.left;
  state.pointer.y = event.clientY - rect.top;
});


