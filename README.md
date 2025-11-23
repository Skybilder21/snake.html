<!doctype html>
<html lang="de">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Snake — Sofort spielbar</title>
  <style>
    :root{
      --bg: #0f1724;
      --panel: #0b1220;
      --accent: #7ee787;
      --accent-2: #4bd1ff;
      --muted: #94a3b8;
      --danger: #ff6b6b;
      font-family: Inter, ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
    }
    *{box-sizing:border-box}
    html,body{height:100%}
    body{
      margin:0;
      display:flex;
      align-items:center;
      justify-content:center;
      gap:24px;
      background:linear-gradient(180deg,#071026 0%, var(--bg) 100%);
      color:#e6eef8;
      padding:24px;
    }

    .container{
      display:grid;
      grid-template-columns: 480px 320px;
      gap:20px;
      align-items:start;
      width:100%;
      max-width:940px;
    }

    .card{
      background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
      border-radius:12px;
      padding:18px;
      box-shadow: 0 6px 18px rgba(2,6,23,0.6);
      border: 1px solid rgba(255,255,255,0.03);
    }

    header h1{
      margin:0 0 6px 0;
      font-size:20px;
      letter-spacing:0.2px;
    }
    header p{margin:0;font-size:13px;color:var(--muted)}

    #gameWrap{
      display:flex;
      gap:12px;
      align-items:center;
      justify-content:center;
      flex-direction:column;
    }

    canvas{
      background: linear-gradient(180deg, rgba(11,17,28,0.7), rgba(8,12,20,0.9));
      border-radius:10px;
      box-shadow: inset 0 -20px 40px rgba(0,0,0,0.5);
      width: 420px;
      height: 420px;
      image-rendering: pixelated;
    }

    .controls{
      display:flex;
      gap:8px;
      margin-top:10px;
    }
    button{
      background:linear-gradient(180deg,var(--panel), rgba(255,255,255,0.02));
      border:1px solid rgba(255,255,255,0.04);
      color:inherit;
      padding:8px 12px;
      border-radius:8px;
      cursor:pointer;
      font-weight:600;
      letter-spacing:0.2px;
    }
    button.ghost{
      background:transparent;
      border:1px solid rgba(255,255,255,0.04);
      color:var(--muted);
      font-weight:600;
    }
    .stats{
      display:flex;
      flex-direction:column;
      gap:10px;
    }
    .statRow{
      display:flex;
      justify-content:space-between;
      gap:8px;
      align-items:center;
      font-size:14px;
    }
    .score{
      font-size:28px;
      font-weight:700;
      color:var(--accent);
    }
    .hint{
      font-size:13px;color:var(--muted);
      margin-top:6px;
    }

    .small{
      font-size:12px;color:var(--muted);
    }

    footer .credits{
      margin-top:12px;
      font-size:12px;color:var(--muted);
    }

    @media (max-width:900px){
      .container{grid-template-columns:1fr; max-width:520px}
      canvas{width:360px;height:360px;}
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="card" id="gameWrap">
      <header>
        <h1>Snake — Sofort spielbar</h1>
        <p>Benutze Pfeiltasten oder WASD. Leertaste = Start/Restart.</p>
      </header>

      <canvas id="game" width="420" height="420"></canvas>

      <div class="controls">
        <button id="startBtn">Start / Neustart</button>
        <button id="pauseBtn" class="ghost">Pause</button>
        <button id="speedBtn" class="ghost">Geschwindigkeit: Normal</button>
      </div>

      <div class="hint small">Tip: Du kannst während des Spiels die Größe des Browserfensters ändern — das Spiel bleibt pixelgenau.</div>
      <div class="credits small">Made for you — direkt im Browser.</div>
    </div>

    <div class="card stats">
      <div>
        <div class="statRow"><div>Score</div><div class="score" id="score">0</div></div>
        <div class="statRow"><div>Highscore (Session)</div><div id="high">0</div></div>
        <div class="statRow"><div>Level</div><div id="level">1</div></div>
        <div class="statRow"><div>Snake-Länge</div><div id="length">1</div></div>
      </div>

      <div>
        <h3 style="margin:10px 0 6px 0">Regeln</h3>
        <ul style="margin:0 0 6px 18px;padding:0;color:var(--muted)">
          <li>Iss das grüne Futter, um zu wachsen.</li>
          <li>Triff nicht die Wand oder den eigenen Körper.</li>
          <li>Leertaste: Start/Neustart, P: Pause</li>
        </ul>
      </div>

      <div>
        <h3 style="margin:10px 0 6px 0">Steuerung</h3>
        <div style="color:var(--muted)">Pfeile / WASD</div>
      </div>
    </div>
  </div>

  <script>
    // Configuration
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d', { alpha: false });
    const size = 21;               // grid size (cells per row/col)
    const cell = Math.floor(canvas.width / size); // pixels per cell -> ensures pixelated look
    const startSpeed = 120;        // ms per tick (lower = faster)
    const speedOptions = [160, 120, 90, 65]; // easy -> insane
    let speedIndex = 1;

    // DOM
    const scoreEl = document.getElementById('score');
    const highEl = document.getElementById('high');
    const levelEl = document.getElementById('level');
    const lengthEl = document.getElementById('length');
    const startBtn = document.getElementById('startBtn');
    const pauseBtn = document.getElementById('pauseBtn');
    const speedBtn = document.getElementById('speedBtn');

    let highscore = 0;

    // Game state
    let snake;
    let dir;
    let nextDir;
    let food;
    let running = false;
    let paused = false;
    let tickTimer = null;
    let score = 0;
    let level = 1;

    function resetGame() {
      snake = [{x: Math.floor(size/2), y: Math.floor(size/2)}]; // head
      dir = {x:1,y:0}; // right
      nextDir = {x:1,y:0};
      placeFood();
      score = 0;
      level = 1;
      running = false;
      paused = false;
      updateUI();
    }

    function placeFood(){
      while(true){
        const fx = Math.floor(Math.random()*size);
        const fy = Math.floor(Math.random()*size);
        // avoid placing on snake
        if(!snake.some(s => s.x===fx && s.y===fy)){
          food = {x:fx,y:fy};
          return;
        }
      }
    }

    function updateUI(){
      scoreEl.textContent = score;
      highEl.textContent = highscore;
      levelEl.textContent = level;
      lengthEl.textContent = snake.length;
      speedBtn.textContent = `Geschwindigkeit: ${["Leicht","Normal","Schnell","Wahnsinn"][speedIndex]}`;
      pauseBtn.textContent = paused ? 'Fortsetzen' : 'Pause';
    }

    function gameTick(){
      if(!running || paused) return;
      // Apply next direction (prevents reversing)
      if ((nextDir.x !== -dir.x || nextDir.y !== -dir.y) || snake.length === 1) {
        dir = nextDir;
      }

      const newHead = {x: snake[0].x + dir.x, y: snake[0].y + dir.y};

      // Collision with walls -> game over
      if(newHead.x < 0 || newHead.x >= size || newHead.y < 0 || newHead.y >= size){
        return gameOver();
      }

      // Collision with self
      if(snake.some(seg => seg.x === newHead.x && seg.y === newHead.y)){
        return gameOver();
      }

      // Move snake
      snake.unshift(newHead);

      // Eating food?
      if(newHead.x === food.x && newHead.y === food.y){
        score += 10;
        level = 1 + Math.floor(score / 50);
        placeFood();
      } else {
        snake.pop(); // remove tail
      }

      // Update highscore
      if(score > highscore) highscore = score;

      // redraw
      draw();
      updateUI();
    }

    function gameOver(){
      running = false;
      paused = false;
      clearInterval(tickTimer);
      tickTimer = null;
      // Flash effect
      flashGameOver().then(() => {
        // leave score visible; user can restart
      });
    }

    function flashGameOver(){
      return new Promise(res=>{
        const original = ctx.getImageData(0,0,canvas.width,canvas.height);
        // draw red overlay quickly
        ctx.fillStyle = 'rgba(255,107,107,0.12)';
        ctx.fillRect(0,0,canvas.width,canvas.height);
        setTimeout(()=>{
          ctx.putImageData(original,0,0);
          res();
        }, 250);
      });
    }

    function draw(){
      // background
      ctx.fillStyle = '#071b2b';
      ctx.fillRect(0,0,canvas.width,canvas.height);

      // grid (subtle)
      ctx.strokeStyle = 'rgba(255,255,255,0.02)';
      ctx.lineWidth = 1;
      for(let i=0;i<=size;i++){
        const p = i*cell + 0.5;
        ctx.beginPath();
        ctx.moveTo(p,0);
        ctx.lineTo(p,canvas.height);
        ctx.stroke();
        ctx.beginPath();
        ctx.moveTo(0,p);
        ctx.lineTo(canvas.width,p);
        ctx.stroke();
      }

      // food
      drawCell(food.x, food.y, true);

      // snake
      for(let i=0;i<snake.length;i++){
        const s = snake[i];
        const isHead = (i===0);
        drawSnakeCell(s.x, s.y, isHead, i);
      }
    }

    function drawCell(x,y, isFood=false){
      const px = x*cell;
      const py = y*cell;
      if(isFood){
        // small glowing square
        ctx.fillStyle = '#59d06a';
        roundRect(px+4,py+4,cell-8,cell-8,4);
        ctx.fill();
        // glow
        ctx.fillStyle = 'rgba(89,208,106,0.08)';
        ctx.fillRect(px+0,py+0,cell,cell);
      } else {
        ctx.fillStyle = '#1b2b3a';
        roundRect(px,py,cell,cell,3);
        ctx.fill();
      }
    }

    function drawSnakeCell(x,y,isHead,i){
      const px = x*cell;
      const py = y*cell;

      // head brighter gradient
      if(isHead){
        const g = ctx.createLinearGradient(px,py,px+cell,py+cell);
        g.addColorStop(0,'#8ef6a0');
        g.addColorStop(1,'#4bd1ff');
        ctx.fillStyle = g;
        roundRect(px+2,py+2,cell-4,cell-4,4);
        ctx.fill();
      } else {
        // body with subtle shading based on index
        const shade = Math.min(0.6, 0.24 + (i / snake.length) * 0.6);
        ctx.fillStyle = `rgba(74,195,123,${0.5 + shade*0.5})`;
        roundRect(px+2,py+2,cell-4,cell-4,3);
        ctx.fill();
      }
    }

    // helper: rounded rect path
    function roundRect(x, y, w, h, r) {
      const min = Math.min(w,h) / 2;
      if(r > min) r = min;
      ctx.beginPath();
      ctx.moveTo(x+r,y);
      ctx.arcTo(x+w,y,x+w,y+h,r);
      ctx.arcTo(x+w,y+h,x,y+h,r);
      ctx.arcTo(x,y+h,x,y,r);
      ctx.arcTo(x,y,x+w,y,r);
      ctx.closePath();
    }

    // Input
    window.addEventListener('keydown', (e) => {
      if(e.key === ' '){
        e.preventDefault();
        if(!running) startGame();
        else restartGame();
      }
      if(e.key.toLowerCase() === 'p'){
        togglePause();
      }
      const map = {
        'ArrowUp':'up','ArrowDown':'down','ArrowLeft':'left','ArrowRight':'right',
        'w':'up','s':'down','a':'left','d':'right'
      };
      const k = e.key;
      const dirName = map[k] || map[k.toLowerCase()];
      if(dirName) {
        e.preventDefault();
        switch(dirName){
          case 'up': nextDir = {x:0,y:-1}; break;
          case 'down': nextDir = {x:0,y:1}; break;
          case 'left': nextDir = {x:-1,y:0}; break;
          case 'right': nextDir = {x:1,y:0}; break;
        }
      }
    });

    // Buttons
    startBtn.addEventListener('click', () => {
      if(!running) startGame();
      else restartGame();
    });

    pauseBtn.addEventListener('click', togglePause);

    speedBtn.addEventListener('click', () => {
      speedIndex = (speedIndex + 1) % speedOptions.length;
      applySpeed();
      updateUI();
    });

    function applySpeed(){
      if(tickTimer) {
        clearInterval(tickTimer);
        tickTimer = setInterval(gameTick, speedOptions[speedIndex]);
      }
    }

    function startGame(){
      running = true;
      paused = false;
      if(tickTimer) clearInterval(tickTimer);
      tickTimer = setInterval(gameTick, speedOptions[speedIndex]);
      updateUI();
    }

    function restartGame(){
      resetGame();
      running = true;
      paused = false;
      if(tickTimer) clearInterval(tickTimer);
      tickTimer = setInterval(gameTick, speedOptions[speedIndex]);
      draw();
      updateUI();
    }

    function togglePause(){
      if(!running) return;
      paused = !paused;
      updateUI();
    }

    // initialize
    resetGame();
    draw();

    // optional: auto-scale canvas rendering to keep pixel look crisp on high-DPI displays
    function fixHiDPICanvas(){
      const dpr = Math.max(1, window.devicePixelRatio || 1);
      // keep physical canvas size but scale internal resolution
      const w = canvas.width;
      const h = canvas.height;
      canvas.width = w * dpr;
      canvas.height = h * dpr;
      canvas.style.width = w + 'px';
      canvas.style.height = h + 'px';
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    }
    fixHiDPICanvas();
    // re-draw after DPI fix
    draw();

    // touch controls (simple): up/down/left/right by tapping canvas quadrants
    canvas.addEventListener('pointerdown', (ev) => {
      const rect = canvas.getBoundingClientRect();
      const x = ev.clientX - rect.left;
      const y = ev.clientY - rect.top;
      const cx = rect.width/2;
      const cy = rect.height/2;
      const dx = x - cx;
      const dy = y - cy;
      if(Math.abs(dx) > Math.abs(dy)){
        nextDir = dx > 0 ? {x:1,y:0} : {x:-1,y:0};
      } else {
        nextDir = dy > 0 ? {x:0,y:1} : {x:0,y:-1};
      }
      // quick start if not running
      if(!running){
        startGame();
      }
    });

    // ensure UI updates every second (score changes by ticks)
    setInterval(updateUI, 300);

    // save highscore per session (LocalStorage would persist across sessions; keep session-only for now)
    // If you'd like persistence, uncomment:
    // if(localStorage.getItem('snake_high')) highscore = parseInt(localStorage.getItem('snake_high'));

    // End of script
  </script>
</body>
</html>
