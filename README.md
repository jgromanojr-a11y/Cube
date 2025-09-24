<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no" />
  <title>Espaço-Z</title>
  <style>
    /* Estilos CSS do jogo */
    body, html {
      margin: 0;
      padding: 0;
      overflow: hidden;
      background-color: #000;
      font-family: 'Arial', sans-serif;
      color: white;
      touch-action: manipulation; /* Impede zoom duplo toque */
    }

    #game-container {
      position: absolute;
      width: 100%;
      height: 100%;
    }

    canvas {
      display: block;
      width: 100%;
      height: 100%;
    }

    #ui {
      position: absolute;
      bottom: 0;
      width: 100%;
      display: flex;
      flex-direction: column;
      align-items: center;
      padding: 10px;
      box-sizing: border-box;
      user-select: none; /* Impede seleção de texto na UI */
    }

    #stats {
      display: flex;
      justify-content: space-between;
      width: 100%;
      max-width: 600px;
      padding: 5px 20px;
      font-size: 1em;
    }

    #controls {
      display: flex;
      justify-content: space-between;
      align-items: center;
      width: 100%;
      max-width: 600px;
      padding-top: 10px;
    }

    #joystick {
      width: 130px;
      height: 130px;
      background-color: rgba(255, 255, 255, 0.2);
      border-radius: 50%;
      display: flex;
      justify-content: center;
      align-items: center;
      position: relative;
      touch-action: none;
    }

    #stick {
      width: 60px;
      height: 60px;
      background-color: rgba(255, 255, 255, 0.5);
      border-radius: 50%;
      position: absolute;
    }

    #pauseBtn, #fireBtn {
      width: 80px;
      height: 80px;
      border-radius: 50%;
      font-size: 1.2em;
      font-weight: bold;
      color: white;
      border: none;
      cursor: pointer;
      touch-action: manipulation;
    }

    #pauseBtn {
      background-color: rgba(50, 50, 50, 0.7);
    }

    #fireBtn {
      background-color: rgba(255, 0, 0, 0.7);
    }

    .overlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background-color: rgba(0, 0, 0, 0.8);
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      text-align: center;
      opacity: 0;
      visibility: hidden;
      transition: opacity 0.5s ease;
      z-index: 10;
    }

    .overlay.show {
      opacity: 1;
      visibility: visible;
    }

    .overlay h1 {
      font-size: 3em;
      margin-bottom: 20px;
    }

    .overlay p {
      font-size: 1.5em;
    }

    .diff-buttons button, #restartBtn {
      font-size: 1.2em;
      padding: 15px 30px;
      margin: 10px;
      cursor: pointer;
      background-color: #333;
      color: white;
      border: 2px solid #555;
      border-radius: 5px;
      transition: background-color 0.3s;
    }

    .diff-buttons button:hover, #restartBtn:hover {
      background-color: #555;
    }

    .diff-buttons {
      display: flex;
      flex-direction: column;
      margin-top: 20px;
    }
  </style>
</head>
<body>
  <div id="game-container">
    <canvas id="game"></canvas>
  </div>

  <div id="ui">
    <div id="stats">
      <div id="lives">Vidas: 5</div>
      <div id="score">Score: 0</div>
      <div id="diff">Dificuldade: Fácil</div>
    </div>
    <div id="controls">
      <div id="joystick">
        <div id="stick"></div>
      </div>
      <button id="pauseBtn">⏸</button>
      <button id="fireBtn">FOGO</button>
    </div>
  </div>

  <div id="startOverlay" class="overlay show">
    <h1>Espaço-Z</h1>
    <p>Escolha a Dificuldade</p>
    <div class="diff-buttons">
      <button class="diffBtn" data-diff="easy">Fácil</button>
      <button class="diffBtn" data-diff="medium">Médio</button>
      <button class="diffBtn" data-diff="hard">Difícil</button>
    </div>
  </div>

  <div id="gameOver" class="overlay">
    <h1>Game Over</h1>
    <p id="finalScore">Score: 0</p>
    <button id="restartBtn">Tente Novamente</button>
  </div>

  <script>
    (() => {
      // ===== Canvas (HiDPI) =====
      const canvas = document.getElementById('game');
      const ctx = canvas.getContext('2d');
      let W = 0, H = 0, DPR = 1;
      function resize() {
        DPR = Math.max(1, Math.floor(window.devicePixelRatio || 1));
        W = Math.floor(window.innerWidth * DPR);
        H = Math.floor(window.innerHeight * DPR);
        canvas.width = W;
        canvas.height = H;
        canvas.style.width = (W / DPR) + 'px';
        canvas.style.height = (H / DPR) + 'px';
      }
      window.addEventListener('resize', resize, { passive: true });
      resize();

      // ===== UI =====
      const scoreEl = document.getElementById('score');
      const livesEl = document.getElementById('lives');
      const diffEl = document.getElementById('diff');
      const pauseBtn = document.getElementById('pauseBtn');
      const startOverlay = document.getElementById('startOverlay');
      const gameOver = document.getElementById('gameOver');
      const finalScore = document.getElementById('finalScore');
      const restartBtn = document.getElementById('restartBtn');

      document.querySelectorAll('.diffBtn').forEach(b => {
        const startHandler = (e) => {
          e.preventDefault();
          startGame(b.dataset.diff);
          b.removeEventListener('click', clickHandler);
        };
        const clickHandler = () => startGame(b.dataset.diff);

        b.addEventListener('touchstart', startHandler, { passive: false });
        b.addEventListener('click', clickHandler);
      });

      restartBtn.addEventListener('click', () => {
        startOverlay.classList.add('show');
        gameOver.classList.remove('show');
        state.running = false;
      });

      pauseBtn.addEventListener('click', () => {
        if (!state.running) return;
        state.paused = !state.paused;
        pauseBtn.textContent = state.paused ? '▶️' : '⏸';
      });

      // ===== WebAudio (sons) =====
      let audioCtx = null;
      function unlockAudio() {
        if (!audioCtx) {
          audioCtx = new (window.AudioContext || window.webkitAudioContext)();
          const o = audioCtx.createOscillator();
          const g = audioCtx.createGain();
          g.gain.value = 0.0001;
          o.connect(g).connect(audioCtx.destination);
          o.start();
          o.stop(audioCtx.currentTime + 0.01);
        }
      }
      function beep(type = 'square', f = 600, dur = 0.08, vol = 0.2, slide = -300) {
        if (!audioCtx) return;
        const o = audioCtx.createOscillator(), g = audioCtx.createGain();
        o.type = type;
        o.frequency.value = f;
        const t = audioCtx.currentTime;
        g.gain.setValueAtTime(0, t);
        g.gain.linearRampToValueAtTime(vol, t + 0.01);
        g.gain.exponentialRampToValueAtTime(0.0001, t + dur);
        o.connect(g).connect(audioCtx.destination);
        if (slide !== 0) {
          o.frequency.setValueAtTime(f, t);
          o.frequency.exponentialRampToValueAtTime(Math.max(60, f + slide), t + dur);
        }
        o.start(t);
        o.stop(t + dur + 0.02);
      }
      const SFX = {
        shot: () => beep('square', 720, 0.07, 0.20, -400),
        boom: () => {
          beep('triangle', 220, 0.20, 0.25, -120);
          setTimeout(() => beep('triangle', 120, 0.16, 0.18, -30), 60);
        }
      };

      // ===== Controles: Joystick + Fogo =====
      const joyBase = document.getElementById('joystick');
      const joyStick = document.getElementById('stick');
      const fireBtn = document.getElementById('fireBtn');

      let joy = { active: false, dx: 0, dy: 0, radius: 65 };

      function moveStick(x, y) {
        const r = joy.radius;
        const dx = x - r;
        const dy = y - r;
        const dist = Math.sqrt(dx * dx + dy * dy);

        if (dist > r) {
          const angle = Math.atan2(dy, dx);
          joyStick.style.left = `${r + r * Math.cos(angle)}px`;
          joyStick.style.top = `${r + r * Math.sin(angle)}px`;
        } else {
          joyStick.style.left = `${x}px`;
          joyStick.style.top = `${y}px`;
        }
        joy.dx = dx / r;
        joy.dy = dy / r;
      }

      function joyPosFromEvent(e) {
        const t = e.touches ? e.touches[0] : e;
        const r = joyBase.getBoundingClientRect();
        return { x: (t.clientX - r.left), y: (t.clientY - r.top) };
      }
      function joyStart(e) {
        unlockAudio();
        joy.active = true;
        const p = joyPosFromEvent(e);
        moveStick(p.x, p.y);
        e.preventDefault();
      }
      function joyMove(e) {
        if (!joy.active) return;
        const p = joyPosFromEvent(e);
        moveStick(p.x, p.y);
        e.preventDefault();
      }
      function joyEnd() {
        joy.active = false;
        joy.dx = joy.dy = 0;
        joyStick.style.left = '50%';
        joyStick.style.top = '50%';
      }
      joyBase.addEventListener('touchstart', joyStart, { passive: false });
      joyBase.addEventListener('touchmove', joyMove, { passive: false });
      joyBase.addEventListener('touchend', joyEnd, { passive: false });
      joyBase.addEventListener('mousedown', joyStart);
      window.addEventListener('mousemove', joyMove);
      window.addEventListener('mouseup', joyEnd);

      let firing = false;
      fireBtn.addEventListener('touchstart', () => { unlockAudio(); firing = true; }, { passive: true });
      fireBtn.addEventListener('touchend', () => { firing = false; }, { passive: true });
      fireBtn.addEventListener('mousedown', () => { unlockAudio(); firing = true; });
      fireBtn.addEventListener('mouseup', () => { firing = false; });

      // ===== Configurações =====
      const DIFFS = {
        easy: { label: 'Fácil', spawn: 900, speed: 0.16, max: 5 },
        medium: { label: 'Médio', spawn: 650, speed: 0.22, max: 7 },
        hard: { label: 'Difícil', spawn: 420, speed: 0.30, max: 9 },
      };
      const ENEMY_TYPES = [
        { name: 'small', w: 24, h: 22, color: '#ff6363', score: 10, speedMul: 1.0 },
        { name: 'medium', w: 32, h: 28, color: '#ffb84a', score: 20, speedMul: 0.9 },
        { name: 'large', w: 42, h: 36, color: '#7cff84', score: 50, speedMul: 0.75 },
      ];

      // ===== Estado do jogo =====
      const state = {
        running: false,
        paused: false,
        score: 0,
        lives: 5,
        diff: 'easy',
        spawnTimer: 0,
        lastShot: 0,
        time: 0
      };

      const player = { x: W / 2, y: H * 0.85, w: 100 * DPR, h: 75 * DPR, speed: 0.6 };
      const bullets = [];
      const enemies = [];
      const particles = [];

      function reset(d) {
        state.running = true;
        state.paused = false;
        state.score = 0;
        state.lives = 5;
        state.diff = d;
        state.spawnTimer = 0;
        state.lastShot = 0;
        state.time = 0;
        bullets.length = 0;
        enemies.length = 0;
        particles.length = 0;
        player.x = W / 2;
        player.y = H * 0.85;
        diffEl.textContent = 'Dificuldade: ' + DIFFS[d].label;
        livesEl.textContent = 'Vidas: ' + state.lives;
        scoreEl.textContent = 'Score: ' + state.score;
        pauseBtn.textContent = '⏸';
      }

      // ===== Helpers =====
      function rand(a, b) {
        return a + Math.random() * (b - a);
      }
      function clamp(v, a, b) {
        return Math.max(a, Math.min(b, v));
      }
      function overlap(a, b) {
        return Math.abs(a.x - b.x) * 2 < (a.w + b.w) && Math.abs(a.y - b.y) * 2 < (a.h + b.h);
      }

      // ===== Spawns =====
      function spawnEnemy() {
        if (enemies.length >= DIFFS[state.diff].max) return;
        const t = ENEMY_TYPES[Math.floor(Math.random() * ENEMY_TYPES.length)];
        const e = {
          type: t.name,
          score: t.score,
          color: t.color,
          w: t.w * DPR,
          h: t.h * DPR,
          x: rand(30 * DPR, W - 30 * DPR),
          y: -40 * DPR,
          vy: (DIFFS[state.diff].speed * t.speedMul) * DPR,
          vx: (Math.random() < 0.5 ? -1 : 1) * rand(0.05, 0.12) * DPR
        };
        enemies.push(e);
      }

      // ===== Efeitos =====
      function spawnExplosion(x, y, n = 16, color = '#ffa800') {
        for (let i = 0; i < n; i++) {
          particles.push({
            x,
            y,
            r: rand(1, 3) * DPR,
            life: rand(220, 480),
            vx: Math.cos(i / n * Math.PI * 2) * rand(0.08, 0.35) * DPR,
            vy: Math.sin(i / n * Math.PI * 2) * rand(0.08, 0.35) * DPR,
            color
          });
        }
      }

      // ===== Fluxo =====
      function startGame(d) {
        unlockAudio();
        startOverlay.classList.remove('show');
        gameOver.classList.remove('show');
        reset(d);
        last = performance.now();
      }

      function gameOverNow() {
        state.running = false;
        finalScore.textContent = 'Score: ' + state.score;
        gameOver.classList.add('show');
      }

      // ===== Loop =====
      let last = performance.now();
      function loop(now) {
        const dt = now - last;
        last = now;
        if (!state.running || state.paused) {
          draw();
          requestAnimationFrame(loop);
          return;
        }

        state.time += dt;
        update(dt);
        draw();
        requestAnimationFrame(loop);
      }

      function update(dt) {
        const dx = joy.dx;
        const dy = joy.dy;
        const speed = player.speed * dt * DPR * 0.8;
        player.x += dx * speed * 3.2;
        player.y += dy * speed * 3.2;

        player.x = clamp(player.x, player.w / 2 + 10 * DPR, W - player.w / 2 - 10 * DPR);
        player.y = clamp(player.y, player.h / 2 + 10 * DPR, H - player.h / 2 - 10 * DPR);

        if (firing && state.time - state.lastShot > 160) {
          bullets.push({ x: player.x, y: player.y - player.h / 2, w: 6 * DPR, h: 14 * DPR, vy: -0.9 * DPR });
          state.lastShot = state.time;
          SFX.shot();
        }
        for (let i = bullets.length - 1; i >= 0; i--) {
          const b = bullets[i];
          b.y += b.vy * dt;
          if (b.y < -30) bullets.splice(i, 1);
        }

        state.spawnTimer += dt;
        const spawnEvery = DIFFS[state.diff].spawn;
        if (state.spawnTimer > spawnEvery) {
          state.spawnTimer = 0;
          spawnEnemy();
        }

        for (let i = enemies.length - 1; i >= 0; i--) {
          const e = enemies[i];
          e.y += e.vy * dt;
          e.x += e.vx * dt;
          if (e.x < e.w / 2 || e.x > W - e.w / 2) e.vx *= -1;

          for (let j = bullets.length - 1; j >= 0; j--) {
            const b = bullets[j];
            if (overlap({ x: b.x, y: b.y, w: b.w, h: b.h }, e)) {
              bullets.splice(j, 1);
              enemies.splice(i, 1);
              spawnExplosion(e.x, e.y, 18, e.color);
              SFX.boom();
              state.score += e.score;
              scoreEl.textContent = 'Score: ' + state.score;
              break;
            }
          }
        }

        for (let i = enemies.length - 1; i >= 0; i--) {
          const e = enemies[i];
          if (overlap(e, player)) {
            enemies.splice(i, 1);
            spawnExplosion(player.x, player.y, 24, '#66ffff');
            SFX.boom();
            state.lives -= 1;
            livesEl.textContent = 'Vidas: ' + state.lives;
            if (state.lives <= 0) {
              gameOverNow();
              return;
            }
          }

          if (e.y - e.h / 2 > H + 40) enemies.splice(i, 1);
        }

        for (let i = particles.length - 1; i >= 0; i--) {
          const p = particles[i];
          p.x += p.vx * dt;
          p.y += p.vy * dt;
          p.life -= dt;
          p.vy += 0.00015 * dt;
          if (p.life <= 0) particles.splice(i, 1);
        }
      }

      function draw() {
        ctx.clearRect(0, 0, W, H);

        // Desenha planetas no fundo
        const numPlanets = 5;
        for (let i = 0; i < numPlanets; i++) {
          const planetX = rand(0, W);
          const planetY = rand(0, H * 0.6);
          const planetRadius = rand(20 * DPR, 60 * DPR);
          const planetColor = `rgba(${rand(50, 150)}, ${rand(50, 150)}, ${rand(100, 200)}, 0.7)`;
          ctx.fillStyle = planetColor;
          ctx.beginPath();
          ctx.arc(planetX, planetY, planetRadius, 0, Math.PI * 2);
          ctx.fill();
        }

        // Desenha estrelas
        for (let i = 0; i < 70; i++) {
          const sx = (i * 97 + (state.time * 0.05)) % W;
          const sy = (i * 131 + (state.time * 0.12)) % H;
          ctx.globalAlpha = 0.35 + ((i % 3) / 10);
          ctx.fillStyle = '#8ff';
          ctx.fillRect(sx, sy, 2 * DPR, 2 * DPR);
        }
        ctx.globalAlpha = 1;

        // Desenha nave do jogador (nave do anexo)
        drawShip(player.x, player.y, player.w, player.h, '#66ffff');

        ctx.fillStyle = '#9ff';
        bullets.forEach(b => {
          roundedRect(b.x - b.w / 2, b.y - b.h / 2, b.w, b.h, 2 * DPR);
          ctx.fill();
        });
        enemies.forEach(e => drawAlien(e.x, e.y, e.w, e.h, e.color));
        particles.forEach(p => {
          ctx.globalAlpha = Math.max(0, p.life / 480);
          ctx.fillStyle = p.color;
          ctx.beginPath();
          ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
          ctx.fill();
          ctx.globalAlpha = 1;
        });
      }

      function roundedRect(x, y, w, h, r) {
        ctx.beginPath();
        ctx.moveTo(x + r, y);
        ctx.arcTo(x + w, y, x + w, y + h, r);
        ctx.arcTo(x + w, y + h, x, y + h, r);
        ctx.arcTo(x, y + h, x, y, r);
        ctx.arcTo(x, y, x + w, y, r);
        ctx.closePath();
      }

      // Função de desenho da nave baseado na imagem (substitui a original)
      function drawShip(x, y, w, h, color) {
        ctx.save();
        ctx.translate(x, y);

        const scale = w / 100; // escala base para ajustar tamanho do desenho

        // Corpo principal (a nave azul arredondada)
        ctx.fillStyle = '#5bc0de';
        ctx.strokeStyle = '#0c3d62';
        ctx.lineWidth = 3 * DPR;

        // Oval principal da nave
        ctx.beginPath();
        ctx.ellipse(0, 0, 50 * scale, 35 * scale, 0, 0, Math.PI * 2);
        ctx.fill();
        ctx.stroke();

        // Faixa amarela da nave
        ctx.fillStyle = '#f1c40f';
        ctx.beginPath();
        ctx.ellipse(0, 8 * scale, 60 * scale, 15 * scale, 0, 0, Math.PI * 2);
        ctx.fill();
        ctx.stroke();

        // Perninhas da nave
        ctx.fillStyle = '#f1c40f';
        ctx.strokeStyle = '#0c3d62';
        ctx.lineWidth = 2 * DPR;

        // Perninha esquerda
        ctx.beginPath();
        ctx.ellipse(-30 * scale, 45 * scale, 12 * scale, 12 * scale, 0, 0, Math.PI * 2);
        ctx.fill();
        ctx.stroke();

        // Perninha direita
        ctx.beginPath();
        ctx.ellipse(30 * scale, 45 * scale, 12 * scale, 12 * scale, 0, 0, Math.PI * 2);
        ctx.fill();
        ctx.stroke();

        // Cinto da nave
        ctx.fillStyle = '#f1c40f';
        ctx.strokeStyle = '#d35400';
        ctx.lineWidth = 3 * DPR;
        ctx.beginPath();
        ctx.rect(-20 * scale, -5 * scale, 40 * scale, 20 * scale);
        ctx.fill();
        ctx.stroke();

        // Fivela do cinto
        ctx.fillStyle = '#d35400';
        ctx.strokeStyle = '#e67e22';
        ctx.lineWidth = 2 * DPR;
        ctx.beginPath();
        ctx.rect(-8 * scale, 0, 16 * scale, 12 * scale);
        ctx.fill();
        ctx.stroke();

        // Olhos da nave
        ctx.fillStyle = '#fdebd0';
        ctx.strokeStyle = '#1b2631';
        ctx.lineWidth = 3 * DPR;

        // Olho esquerdo
        ctx.beginPath();
        ctx.arc(-20 * scale, -15 * scale, 15 * scale, 0, Math.PI * 2);
        ctx.fill();
        ctx.stroke();

        ctx.fillStyle = '#1b2631';
        ctx.beginPath();
        ctx.arc(-20 * scale, -15 * scale, 10 * scale, 0, Math.PI * 2);
        ctx.fill();

        // Olho direito
        ctx.fillStyle = '#fdebd0';
        ctx.beginPath();
        ctx.arc(20 * scale, -15 * scale, 15 * scale, 0, Math.PI * 2);
        ctx.fill();
        ctx.stroke();

        ctx.fillStyle = '#1b2631';
        ctx.beginPath();
        ctx.arc(20 * scale, -15 * scale, 10 * scale, 0, Math.PI * 2);
        ctx.fill();

        // Bico sorriso
        ctx.strokeStyle = '#1b2631';
        ctx.lineWidth = 2 * DPR;
        ctx.beginPath();
        ctx.arc(0, 0, 12 * scale, 0, Math.PI, false);
        ctx.stroke();

        // Cubo de Rubik no topo
        const cubeSize = 50 * scale;
        ctx.lineWidth = 2 * DPR;

        const startX = -cubeSize / 2;
        const startY = -cubeSize * 1.5;
        
        // Cubo principal
        ctx.fillStyle = '#ea4335'; // vermelho base
        ctx.strokeStyle = '#222';
        ctx.beginPath();
        ctx.moveTo(startX, startY);
        ctx.lineTo(startX + cubeSize, startY);
        ctx.lineTo(startX + cubeSize, startY + cubeSize);
        ctx.lineTo(startX, startY + cubeSize);
        ctx.closePath();
        ctx.fill();
        ctx.stroke();

        // Linhas internas do cubo
        const small = cubeSize / 3;
        ctx.strokeStyle = '#222';
        ctx.beginPath();
        // linhas horizontais
        ctx.moveTo(startX, startY + small);
        ctx.lineTo(startX + cubeSize, startY + small);
        ctx.moveTo(startX, startY + 2 * small);
        ctx.lineTo(startX + cubeSize, startY + 2 * small);
        // linhas verticais
        ctx.moveTo(startX + small, startY);
        ctx.lineTo(startX + small, startY + cubeSize);
        ctx.moveTo(startX + 2 * small, startY);
        ctx.lineTo(startX + 2 * small, startY + cubeSize);
        ctx.stroke();

        // Peças coloridas do cubo (simplificado)
        const colors = ['#fbbc05', '#34a853', '#4285f4', '#ea4335']; // amarelo, verde, azul, vermelho
        for (let i = 0; i < 3; i++) {
          for (let j = 0; j < 3; j++) {
            ctx.fillStyle = colors[(i + j) % colors.length];
            ctx.fillRect(startX + j * small + 2, startY + i * small + 2, small - 4, small - 4);
          }
        }

        ctx.restore();
      }

      function drawAlien(x, y, w, h, color) {
        ctx.save();
        ctx.translate(x, y);

        // Corpo principal do alien
        ctx.fillStyle = color;
        ctx.beginPath();
        ctx.ellipse(0, 0, w / 2, h / 3, 0, 0, Math.PI * 2);
        ctx.fill();

        // Olhos
        ctx.fillStyle = '#fff';
        ctx.beginPath();
        ctx.arc(-w / 4, -h / 6, w / 8, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.arc(w / 4, -h / 6, w / 8, 0, Math.PI * 2);
        ctx.fill();

        // Detalhes nos olhos
        ctx.fillStyle = '#000';
        ctx.beginPath();
        ctx.arc(-w / 4, -h / 6, w / 16, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.arc(w / 4, -h / 6, w / 16, 0, Math.PI * 2);
        ctx.fill();

        // "Antenas" ou detalhes extras
        ctx.strokeStyle = color;
        ctx.lineWidth = 2 * DPR;
        ctx.beginPath();
        ctx.moveTo(-w / 3, -h / 2);
        ctx.lineTo(-w / 4, -h / 3);
        ctx.moveTo(w / 3, -h / 2);
        ctx.lineTo(w / 4, -h / 3);
        ctx.stroke();

        ctx.restore();
      }

      requestAnimationFrame(loop);
    })();
  </script>
</body>
</html>

