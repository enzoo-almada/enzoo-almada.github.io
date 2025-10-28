<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1" />
<title>Coração "I love you" - Canvas</title>
<style>
  :root{
    --bg: #000;
    --text-color: #ea80b0;
  }

  html, body {
    height: 100%;
    margin: 0;
    background: var(--bg);
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
  }

  #container {
    position: relative;
    width: 100vw;
    height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  canvas {
    width: 100vw;
    height: 100vh;
    display: block;
    touch-action: manipulation;
  }

  .hint {
    position: absolute;
    bottom: 24px;
    left: 50%;
    transform: translateX(-50%);
    color: #fff8;
    font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
    font-size: 13px;
    pointer-events: none;
  }
</style>
</head>
<body>
  <div id="container">
    <canvas id="c"></canvas>
    <div class="hint">Toque ou clique para alternar pulsação 💖</div>
  </div>

<script>
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
let DPR = Math.max(1, window.devicePixelRatio || 1);

function resizeCanvas() {
  canvas.width = window.innerWidth * DPR;
  canvas.height = window.innerHeight * DPR;
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
}

window.addEventListener('resize', resizeCanvas);
resizeCanvas();

// --- Parâmetros ---
const TEXT = "I love you";
const COUNT = 160;
const SCALE = 9;
const baseFontSize = 14;
const color = '#ea80b0';
const glowColor = 'rgba(255,255,255,0.25)';

let items = [];
let time = 0;
let rotating = true;
let pulse = true;
let last = performance.now();

function heartPoint(t) {
  const x = 16 * Math.pow(Math.sin(t), 3);
  const y = 13 * Math.cos(t) - 5 * Math.cos(2*t) - 2 * Math.cos(3*t) - Math.cos(4*t);
  return { x, y: -y };
}

function buildItems() {
  items = [];
  for (let i = 0; i < COUNT; i++) {
    const t = (i / COUNT) * Math.PI * 2;
    const p = heartPoint(t);
    const jitter = (Math.random() - 0.5) * 0.6;
    const x = p.x + jitter;
    const y = p.y + jitter;
    const dt = 0.0001;
    const p2 = heartPoint(t + dt);
    const tangentAngle = Math.atan2(p2.y - p.y, p2.x - p.x);
    const dist = Math.hypot(p.x, p.y);
    items.push({
      x, y, angle: tangentAngle, dist,
      phase: Math.random() * Math.PI * 2,
      scale: 1 - (dist / 30) * 0.25 + (Math.random() - 0.5) * 0.06,
      alpha: 0.85 + (Math.random() * 0.15)
    });
  }
}

function mapToCanvas(p, width, height) {
  const cx = width / 2;
  const cy = height / 2;
  const scale = Math.min(width, height) / (SCALE * 10);
  return { x: cx + p.x * scale, y: cy + p.y * scale };
}

function draw(now) {
  const dt = (now - last) / 1000;
  last = now;
  time += dt;
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  const w = canvas.width / DPR;
  const h = canvas.height / DPR;
  const globalRot = rotating ? (time * 12 * Math.PI / 180) : 0;
  const pulseFactor = pulse ? (1 + 0.06 * Math.sin(time * 2.5)) : 1;
  const sorted = items.slice().sort((a, b) => a.dist - b.dist);
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';

  for (let i = 0; i < sorted.length; i++) {
    const it = sorted[i];
    const mapped = mapToCanvas({ x: it.x, y: it.y }, w, h);
    const wobble = Math.sin(time * 2 + it.phase) * (2 + it.dist * 0.04);
    const depthScale = 1 - (it.dist / 40) * 0.08;
    const finalScale = it.scale * depthScale * pulseFactor;
    const x = mapped.x + wobble;
    const y = mapped.y + Math.cos(time + it.phase) * 1.5;
    const fontPx = Math.max(8, baseFontSize * finalScale);
    ctx.font = `${fontPx}px monospace`;
    ctx.save();
    ctx.translate(w/2, h/2);
    ctx.rotate(globalRot * (0.8 + it.dist * 0.002));
    ctx.translate(-w/2, -h/2);
    ctx.shadowColor = glowColor;
    ctx.shadowBlur = 10 + (1 - it.dist / 40) * 8;
    ctx.fillStyle = color;
    ctx.globalAlpha = Math.min(1, it.alpha);
    ctx.translate(x, y);
    ctx.rotate(it.angle * 0.8 + Math.sin(time + it.phase) * 0.07);
    ctx.fillText(TEXT, 0, 0);
    ctx.shadowBlur = 2;
    ctx.globalAlpha = 0.6 * it.alpha;
    ctx.fillText(TEXT, 0, 0);
    ctx.restore();
  }

  requestAnimationFrame(draw);
}

function init() {
  buildItems();
  resizeCanvas();
  last = performance.now();
  requestAnimationFrame(draw);
}

canvas.addEventListener('click', () => pulse = !pulse);
canvas.addEventListener('touchstart', () => pulse = !pulse, { passive: true });
init();
</script>
</body>
</html>
