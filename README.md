<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Jump Runner — Simple Jump-Only Game</title>
  <style>
    html,body { height:100%; margin:0; background: linear-gradient(#cfefff,#b0e0ff); font-family: system-ui, sans-serif; }
    #wrap { display:flex; align-items:center; justify-content:center; height:100vh; }
    canvas { width: 100%; max-width: 1280px; height: auto; background: transparent; border-radius:10px; box-shadow:0 8px 30px rgba(0,0,0,0.18); }
    .hud { position:fixed; left:12px; top:12px; background:rgba(255,255,255,0.9); padding:8px 10px; border-radius:8px; }
    .hint { position:fixed; right:12px; top:12px; background:rgba(255,255,255,0.9); padding:8px 10px; border-radius:8px; text-align:right; }
    @media (orientation:portrait) {
      body::before { content:"Rotate device to landscape for best experience"; position:fixed; inset:0; display:flex; align-items:center; justify-content:center; background:rgba(0,0,0,0.6); color:white; font-size:18px; }
    }
  </style>
</head>
<body>
  <div id="wrap"><canvas id="game"></canvas></div>
  <div class="hud">Score: <span id="score">0</span></div>
  <div class="hint">Space / ↑ / Tap — Jump</div>

<script>
(() => {
  // --- Configuration ---
  const GAME_W = 1000;
  const GAME_H = 500;
  const GROUND_Y = GAME_H - 80;
  const GRAVITY = 2000; // px/s^2
  const JUMP_V = -780;  // initial jump velocity
  const OBSTACLE_MIN_GAP = 600; // min distance between obstacles
  const OBSTACLE_MAX_GAP = 1200;
  const WORLD_SPEED = 360; // base leftward speed

  // --- Canvas & resize ---
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  function resize() {
    // keep logical resolution fixed; scale CSS size to fit viewport while retaining aspect
    canvas.width = GAME_W; canvas.height = GAME_H;
    const vw = window.innerWidth - 24;
    const vh = window.innerHeight - 24;
    const aspect = GAME_W / GAME_H;
    let w = vw, h = Math.round(w / aspect);
    if (h > vh) { h = vh; w = Math.round(h * aspect); }
    canvas.style.width = w + 'px';
    canvas.style.height = h + 'px';
  }
  window.addEventListener('resize', resize);
  resize();

  // --- Game state ---
  let lastTime = performance.now();
  let score = 0;
  document.getElementById('score').textContent = '0';
  let gameOver = false;

  // player is a simple box
  const player = {
    x: 160,
    y: GROUND_Y - 60, // center y
    w: 48,
    h: 60,
    vy: 0,
    grounded: true,
    color: '#FF6B6B'
  };

  // obstacles array: { x, w, h }
  const obstacles = [];
  let nextObstacleX = GAME_W + 200;

  // spawn first a few obstacles
  function spawnObstacleAt(x) {
    const h = 30 + Math.random()*60;
    const w = 30 + Math.random()*50;
    obstacles.push({ x: x, w: w, h: h, y: GROUND_Y - h });
  }
  spawnObstacleAt(nextObstacleX);
  nextObstacleX += OBSTACLE_MIN_GAP + Math.random()*(OBSTACLE_MAX_GAP-OBSTACLE_MIN_GAP);

  // --- Input: single action (jump) ---
  const keys = {};
  window.addEventListener('keydown', e => {
    keys[e.key.toLowerCase()] = true;
    if (e.key === ' ' || e.key === 'ArrowUp') e.preventDefault();
  });
  window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

  // touch: tap anywhere to jump
  canvas.addEventListener('touchstart', (ev) => {
    ev.preventDefault();
    doJump();
  }, {passive:false});
  canvas.addEventListener('mousedown', () => { // click to jump or restart
    if (gameOver) restart(); else doJump();
  });

  function doJump() {
    if (player.grounded) {
      player.vy = JUMP_V;
      player.grounded = false;
    }
  }

  // --- Collision helper ---
  function rectOverlap(a, b) {
    return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + (b.h||0) && a.y + a.h > b.y;
  }

  // --- Reset / Restart ---
  function restart() {
    obstacles.length = 0;
    player.y = GROUND_Y - player.h;
    player.vy = 0;
    player.grounded = true;
    score = 0;
    nextObstacleX = GAME_W + 200;
    spawnObstacleAt(nextObstacleX);
    nextObstacleX += OBSTACLE_MIN_GAP + Math.random()*(OBSTACLE_MAX_GAP-OBSTACLE_MIN_GAP);
    gameOver = false;
    document.getElementById('score').textContent = '0';
  }

  // --- Update & Render loop ---
  function update(dt) {
    if (gameOver) return;

    // Check keyboard jump
    if ((keys[' '] || keys['arrowup'] || keys['w']) && player.grounded) doJump();

    // Player physics
    player.vy += GRAVITY * dt;
    player.y += player.vy * dt;
    if (player.y + player.h >= GROUND_Y) {
      player.y = GROUND_Y - player.h;
      player.vy = 0;
      player.grounded = true;
    } else {
      player.grounded = false;
    }

    // Move obstacles left
    const speed = WORLD_SPEED + Math.floor(score / 10); // slightly speed up with score
    for (let i = obstacles.length - 1; i >= 0; i--) {
      obstacles[i].x -= speed * dt;
      // collision
      const pr = { x: player.x, y: player.y, w: player.w, h: player.h };
      const or = { x: obstacles[i].x, y: obstacles[i].y, w: obstacles[i].w, h: obstacles[i].h };
      if (rectOverlap(pr, or)) {
        gameOver = true;
      }
      // remove off-screen
      if (obstacles[i].x + obstacles[i].w < -50) obstacles.splice(i,1);
    }

    // spawn new obstacles when needed
    if (nextObstacleX - (obstacles.length ? 0 : 0) < GAME_W*2 && obstacles.length < 6) {
      if (obstacles.length === 0 || (obstacles[obstacles.length-1].x < GAME_W - 200)) {
        spawnObstacleAt(nextObstacleX);
        nextObstacleX += OBSTACLE_MIN_GAP + Math.random()*(OBSTACLE_MAX_GAP-OBSTACLE_MIN_GAP);
      }
    }

    // Also if there are few obstacles, keep scheduling
    if (obstacles.length < 3 && nextObstacleX < GAME_W*3) {
      spawnObstacleAt(nextObstacleX);
      nextObstacleX += OBSTACLE_MIN_GAP + Math.random()*(OBSTACLE_MAX_GAP-OBSTACLE_MIN_GAP);
    }

    // Score increases with time survived
    score += dt * 20;
    document.getElementById('score').textContent = Math.floor(score);
  }

  function draw() {
    // clear
    ctx.clearRect(0,0,GAME_W,GAME_H);

    // sky
    ctx.fillStyle = '#bfefff';
    ctx.fillRect(0,0,GAME_W,GAME_H);

    // sun
    ctx.beginPath(); ctx.arc(GAME_W - 110, 80, 44, 0, Math.PI*2); ctx.fillStyle = '#fff18a'; ctx.fill();

    // ground
    ctx.fillStyle = '#6c4a2f';
    ctx.fillRect(0, GROUND_Y, GAME_W, GAME_H - GROUND_Y);

    // decorative line
    ctx.fillStyle = '#3f2a16';
    for (let i = 0; i < GAME_W; i += 50) {
      ctx.fillRect(i+8, GROUND_Y+8, 34, 8);
    }

    // obstacles
    ctx.fillStyle = '#2b2b2b';
    obstacles.forEach(o => {
      ctx.fillRect(o.x, o.y, o.w, o.h);
      // hazard stripe
      ctx.fillStyle = '#ffd24d';
      ctx.fillRect(o.x + 6, o.y + 6, Math.min(18, o.w - 12), Math.min(10, o.h - 12));
      ctx.fillStyle = '#2b2b2b';
    });

    // player (simple rounded)
    ctx.save();
    ctx.translate(player.x + player.w/2, player.y + player.h/2);
    // shadow
    ctx.beginPath(); ctx.ellipse(0, player.h/2 + 12, player.w, 10, 0, 0, Math.PI*2); ctx.fillStyle = 'rgba(0,0,0,0.18)'; ctx.fill();
    ctx.fillStyle = player.color;
    roundRect(ctx, -player.w/2, -player.h/2, player.w, player.h, 8, true, false);
    // eye
    ctx.fillStyle = '#222';
    ctx.beginPath(); ctx.arc(8, -12, 5, 0, Math.PI*2); ctx.fill();
    ctx.restore();

    // if game over, overlay
    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.45)';
      ctx.fillRect(0,0,GAME_W,GAME_H);
      ctx.fillStyle = '#fff';
      ctx.font = '36px system-ui';
      ctx.textAlign = 'center';
      ctx.fillText('Game Over', GAME_W/2, GAME_H/2 - 10);
      ctx.font = '18px system-ui';
      ctx.fillText('Click / Tap to restart', GAME_W/2, GAME_H/2 + 26);
    }
  }

  // helper rounded rect
  function roundRect(ctx, x, y, w, h, r, fill, stroke) {
    if (typeof r === 'undefined') r = 5;
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.arcTo(x + w, y, x + w, y + h, r);
    ctx.arcTo(x + w, y + h, x, y + h, r);
    ctx.arcTo(x, y + h, x, y, r);
    ctx.arcTo(x, y, x + w, y, r);
    ctx.closePath();
    if (fill) ctx.fill();
    if (stroke) ctx.stroke();
  }

  // main loop
  function loop(now) {
    const dt = Math.min(0.033, (now - lastTime) / 1000);
    lastTime = now;
    update(dt);
    draw();
    requestAnimationFrame(loop);
  }

  // start
  lastTime = performance.now();
  requestAnimationFrame(loop);

  // restart on click when over
  canvas.addEventListener('click', () => { if (gameOver) restart(); });

  // keyboard: restart on Enter when over
  window.addEventListener('keydown', (e) => { if (gameOver && e.key === 'Enter') restart(); });

  // expose restart to console for debugging
  window.simpleRunnerRestart = restart;
})();
</script>
</body>
</html>
