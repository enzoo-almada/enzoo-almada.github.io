# enzoo-almada.github.io
<!DOCTYPE html>
<html lang="pt-BR">

<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>Coração "I love you" - Canvas</title>
    <style>
        :root {
            --bg: #000;
            --text-color: #ea80b0;
        }

        html,
        body {
            height: 100%;
            margin: 0;
            background: var(--bg);
            -webkit-font-smoothing: antialiased;
            -moz-osx-font-smoothing: grayscale;
            display: flex;
            align-items: center;
            justify-content: center;
            overflow: hidden;
        }

        #container {
            width: 100%;
            height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            position: relative;
        }

        canvas {
            max-width: 100%;
            max-height: 100%;
            display: block;
        }

        /* Legenda (opcional) */
        .hint {
            position: absolute;
            bottom: 24px;
            left: 50%;
            transform: translateX(-50%);
            color: #fff4;
            font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
            font-size: 13px;
            pointer-events: none;
        }
    </style>
</head>

<body>
    <div id="container">
        <canvas id="c"></canvas>
        <div class="hint">Clique/tocar para alternar pulsação</div>
    </div>

    <script>
        /*
          Canvas heart made of repeated text "I love you".
          - Precompute positions using parametric heart curve.
          - Animate rotation + small horizontal oscillation + pulsation toggle.
          - Use devicePixelRatio for crisp rendering.
        */

        const canvas = document.getElementById('c');
        const ctx = canvas.getContext('2d');

        let DPR = Math.max(1, window.devicePixelRatio || 1);

        function resizeCanvas() {
            const rect = canvas.getBoundingClientRect();
            canvas.width = Math.round(rect.width * DPR);
            canvas.height = Math.round(rect.height * DPR);
            canvas.style.width = rect.width + 'px';
            canvas.style.height = rect.height + 'px';
            ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
        }
        function fitFullScreen() {
            canvas.style.width = window.innerWidth + 'px';
            canvas.style.height = window.innerHeight + 'px';
            resizeCanvas();
        }

        fitFullScreen();
        window.addEventListener('resize', () => {
            DPR = Math.max(1, window.devicePixelRatio || 1);
            fitFullScreen();
        });

        // --- Parameters ---
        const TEXT = "I love you";
        const COUNT = 160;          // number of text items (lower = faster)
        const SCALE = 9;            // scale to enlarge heart shape
        const CENTER = { x: 0, y: 0 };
        const baseFontSize = 14;    // px (will be multiplied by DPR scaling in canvas render)
        const color = '#ea80b0';
        const glowColor = 'rgba(255,255,255,0.25)';

        let items = []; // will hold {x,y,angle,dist,phase,scale,alpha}
        let time = 0;
        let rotating = true;
        let pulse = true;
        let last = performance.now();

        // Parametric heart function (classic)
        function heartPoint(t) {
            // t in [0, 2π]
            const x = 16 * Math.pow(Math.sin(t), 3);
            const y = 13 * Math.cos(t) - 5 * Math.cos(2 * t) - 2 * Math.cos(3 * t) - Math.cos(4 * t);
            return { x, y: -y }; // invert y to have heart pointing up
        }

        // Precompute positions (uniformly sample t)
        function buildItems() {
            items = [];
            for (let i = 0; i < COUNT; i++) {
                // sample t from 0..2π
                const t = (i / COUNT) * Math.PI * 2;
                const p = heartPoint(t);
                // Add a tiny random radius jitter so texts don't overlap exactly
                const jitter = (Math.random() - 0.5) * 0.6;
                const x = p.x + jitter;
                const y = p.y + jitter;

                // Determine angle to rotate the text slightly to tangent direction
                // numeric derivative to compute approximate tangent
                const dt = 0.0001;
                const p2 = heartPoint(t + dt);
                const tangentAngle = Math.atan2(p2.y - p.y, p2.x - p.x);

                // distance from center for depth effect
                const dist = Math.hypot(p.x, p.y);

                items.push({
                    x, y,
                    angle: tangentAngle,
                    dist,
                    // each item has its own animation phase to slightly oscillate
                    phase: Math.random() * Math.PI * 2,
                    scale: 1 - (dist / 30) * 0.25 + (Math.random() - 0.5) * 0.06,
                    alpha: 0.85 + (Math.random() * 0.15)
                });
            }
        }

        // map heart coordinates to canvas pixel coords
        function mapToCanvas(p, width, height) {
            // center and scale
            const cx = width / 2;
            const cy = height / 2;
            // We scale heart to fit canvas
            const scale = Math.min(width, height) / (SCALE * 10); // empirical
            return {
                x: cx + p.x * scale,
                y: cy + p.y * scale
            };
        }

        // draw frame
        function draw(now) {
            const dt = (now - last) / 1000;
            last = now;
            time += dt;

            // clear
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // subtle background glow (optional)
            // ctx.fillStyle = 'rgba(0,0,0,0.0)';
            // ctx.fillRect(0,0,canvas.width, canvas.height);

            const w = canvas.width / DPR;
            const h = canvas.height / DPR;

            // global rotation slowly
            const globalRot = rotating ? (time * 12 * Math.PI / 180) : 0; // radians

            // pulse factor [0.95 .. 1.05]
            const pulseFactor = pulse ? (1 + 0.06 * Math.sin(time * 2.5)) : 1;

            // draw items sorted by dist so further items render first (optional depth)
            const sorted = items.slice().sort((a, b) => a.dist - b.dist);

            // draw each text
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            for (let i = 0; i < sorted.length; i++) {
                const it = sorted[i];

                // base mapped pos
                const mapped = mapToCanvas({ x: it.x, y: it.y }, w, h);

                // small horizontal wobble per item
                const wobble = Math.sin(time * 2 + it.phase) * (2 + it.dist * 0.04);

                // depth affect z-scale and alpha
                const depthScale = 1 - (it.dist / 40) * 0.08;
                const finalScale = it.scale * depthScale * pulseFactor;

                const x = mapped.x + wobble;
                const y = mapped.y + Math.cos(time + it.phase) * 1.5;

                // font size adjusted by DPR and finalScale
                const fontPx = Math.max(8, baseFontSize * finalScale);
                ctx.font = `${fontPx}px monospace`;

                // glow (soft white)
                ctx.save();

                // apply global rotation around center to get 3D-rotating look
                ctx.translate(w / 2, h / 2);
                ctx.rotate(globalRot * (0.8 + it.dist * 0.002)); // slight diff by distance
                ctx.translate(-w / 2, -h / 2);

                // outer glow
                ctx.shadowColor = glowColor;
                ctx.shadowBlur = 10 + (1 - it.dist / 40) * 8;
                ctx.fillStyle = color;
                ctx.globalAlpha = Math.min(1, it.alpha);

                // rotate each text slightly along tangent
                ctx.translate(x, y);
                ctx.rotate(it.angle * 0.8 + Math.sin(time + it.phase) * 0.07);
                ctx.fillText(TEXT, 0, 0);

                // subtle brighter center (duplicate draw with less blur)
                ctx.shadowBlur = 2;
                ctx.globalAlpha = 0.6 * it.alpha;
                ctx.fillText(TEXT, 0, 0);

                ctx.restore();
            }

            // loop
            requestAnimationFrame(draw);
        }

        // initialize
        function init() {
            buildItems();
            fitFullScreen();
            last = performance.now();
            requestAnimationFrame(draw);
        }

        // toggle pulse on click/tap
        canvas.addEventListener('click', () => pulse = !pulse);
        canvas.addEventListener('touchstart', () => pulse = !pulse, { passive: true });

        // start
        init();

    </script>
</body>

</html>
