<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Star Collector — mysys2</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <header>
    <h1>Star Collector</h1>
    <p>Use arrow keys or WASD to move. Collect stars, avoid enemies. Press R to restart.</p>
  </header>
  <main>
    <canvas id="game" width="800" height="520"></canvas>
    <div id="hud">
      <span id="score">Score: 0</span>
      <span id="lives">Lives: 3</span>
      <button id="restart">Restart</button>
    </div>
  </main>
  <footer>
    <small>Built for: mysys2 — drop this folder in your web server or open <code>index.html</code> locally.</small>
  </footer>
  <script src="game.js"></script>
</body>
</html>
*{box-sizing:border-box;margin:0;padding:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,'Helvetica Neue',Arial}
body{display:flex;flex-direction:column;align-items:center;gap:12px;padding:18px;background:linear-gradient(180deg,#0f172a,#071024);color:#e6eef8}
header h1{font-size:28px}
canvas{background:#0b1220;border:4px solid #1f2937;border-radius:8px;display:block}
#hud{display:flex;gap:12px;align-items:center;margin-top:8px}
#hud span{background:rgba(255,255,255,0.04);padding:6px 10px;border-radius:6px}
button{padding:6px 10px;border-radius:6px;border:none;background:#2563eb;color:white;cursor:pointer}
button:hover{opacity:0.9}
footer{margin-top:8px;font-size:12px;color:#9aa9c9}
code{background:#071024;padding:2px 6px;border-radius:4px}
// Simple canvas game: Star Collector
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');

const W = canvas.width, H = canvas.height;
let keys = {};
let score = 0;
let lives = 3;
let gameOver = false;

document.addEventListener('keydown', e => { keys[e.key.toLowerCase()] = true; if (e.key === 'r' || e.key === 'R') restart(); });
document.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });
document.getElementById('restart').addEventListener('click', restart);

function rand(min,max){ return Math.random()*(max-min)+min; }

class Player {
  constructor(){
    this.size = 28;
    this.x = 60; this.y = H/2;
    this.speed = 4;
    this.color = '#60a5fa';
  }
  update(){
    if (keys['arrowup']||keys['w']) this.y -= this.speed;
    if (keys['arrowdown']||keys['s']) this.y += this.speed;
    if (keys['arrowleft']||keys['a']) this.x -= this.speed;
    if (keys['arrowright']||keys['d']) this.x += this.speed;
    // bounds
    this.x = Math.max(0, Math.min(W-this.size, this.x));
    this.y = Math.max(0, Math.min(H-this.size, this.y));
  }
  draw(){
    ctx.fillStyle = this.color;
    ctx.fillRect(this.x,this.y,this.size,this.size);
  }
  rect(){ return {x:this.x,y:this.y,w:this.size,h:this.size}; }
}

class Star {
  constructor(){
    this.size = 16;
    this.reset();
  }
  reset(){
    this.x = rand(200, W-40);
    this.y = rand(20, H-40);
    this.collected = false;
  }
  draw(){
    if (this.collected) return;
    ctx.fillStyle = '#fbbf24';
    // simple star: draw rotated square
    ctx.save();
    ctx.translate(this.x+this.size/2, this.y+this.size/2);
    ctx.rotate(Math.PI/4);
    ctx.fillRect(-this.size/2, -this.size/2, this.size, this.size);
    ctx.restore();
  }
  rect(){ return {x:this.x,y:this.y,w:this.size,h:this.size}; }
}

class Enemy {
  constructor(speed=1.8){
    this.size = 26;
    this.speed = speed;
    this.reset();
    this.color = '#fb7185';
  }
  reset(){
    this.x = W + rand(20, 200);
    this.y = rand(10, H-36);
    this.vx = - (this.speed + rand(0,1.6));
  }
  update(){
    this.x += this.vx;
    if (this.x < -this.size) this.reset();
  }
  draw(){
    ctx.fillStyle = this.color;
    ctx.beginPath();
    ctx.arc(this.x+this.size/2, this.y+this.size/2, this.size/2, 0, Math.PI*2);
    ctx.fill();
  }
  rect(){ return {x:this.x,y:this.y,w:this.size,h:this.size}; }
}

function collided(a,b){
  return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
}

// game objects
let player = new Player();
let stars = [];
let enemies = [];
let spawnTimer = 0;
let enemySpeed = 1.8;

function resetGame(){
  score = 0; lives = 3; gameOver = false;
  player = new Player();
  stars = [new Star()];
  enemies = [new Enemy(enemySpeed)];
  spawnTimer = 0;
  enemySpeed = 1.8;
  updateHUD();
}

function updateHUD(){
  document.getElementById('score').textContent = 'Score: ' + score;
  document.getElementById('lives').textContent = 'Lives: ' + lives;
}

function restart(){ resetGame(); }

function spawnEnemy(){
  enemies.push(new Enemy(enemySpeed));
}

function gameTick(){
  if (gameOver) return draw();
  // update
  player.update();
  stars.forEach(s=>{
    if (!s.collected && collided(player.rect(), s.rect())){
      s.collected = true;
      score += 10;
      updateHUD();
      // spawn a new star after short delay
      setTimeout(()=>{ const s2 = new Star(); stars.push(s2); }, 120);
      // increase difficulty slightly
      if (score % 30 === 0) { enemySpeed += 0.4; spawnEnemy(); }
    }
  });
  enemies.forEach(e=>{
    e.update();
    if (collided(player.rect(), e.rect())){
      e.reset();
      lives -= 1;
      updateHUD();
      if (lives <= 0){ gameOver = true; showGameOver(); }
    }
  });

  spawnTimer += 1;
  if (spawnTimer > 600){ spawnTimer = 0; spawnEnemy(); }

  draw();
  requestAnimationFrame(gameTick);
}

function showGameOver(){
  // tiny overlay
  ctx.fillStyle = 'rgba(2,6,23,0.6)';
  ctx.fillRect(0,0,W,H);
  ctx.fillStyle = 'white';
  ctx.font = '28px system-ui';
  ctx.textAlign = 'center';
  ctx.fillText('Game Over', W/2, H/2 - 10);
  ctx.font = '16px system-ui';
  ctx.fillText('Press R or click Restart to play again', W/2, H/2 + 20);
}

function draw(){
  ctx.clearRect(0,0,W,H);
  // background grid
  ctx.fillStyle = '#071124';
  ctx.fillRect(0,0,W,H);
  // subtle stars background
  for(let i=0;i<50;i++){
    const sx = (i*73)%W, sy = (i*97)%H;
    ctx.fillStyle = 'rgba(255,255,255,0.02)';
    ctx.fillRect(sx,sy,2,2);
  }
  // objects
  stars.forEach(s=>s.draw());
  enemies.forEach(e=>e.draw());
  player.draw();

  if (gameOver) showGameOver();
}

// start
resetGame();
requestAnimationFrame(gameTick);
Star Collector — mysys2
======================

A small browser game you can build/run from the folder `mysys2`.

How to run
----------
1. Unzip (if zipped) and open `index.html` in your browser.  
   - For best results run from a local web server (e.g. `python -m http.server`), but double-clicking works too.
2. Controls: Arrow keys or WASD to move. Press R or the Restart button to restart.
3. Goal: Collect stars (+10 points) and avoid enemies. You have 3 lives.

Files
-----
- index.html — main HTML page
- style.css  — simple styles
- game.js    — JavaScript game logic
- README.md  — this file

Notes
-----
This is a lightweight starter. Tell me if you want:
- More levels, sounds, images/sprites
- Mobile/touch controls
- High-score save/export
- A build for Electron, PWA, or embedding into an existing project
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
  <title>Star Collector — mysys2 (Full)</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <header>
    <h1>Star Collector</h1>
    <p>Collect stars, avoid enemies. Arrow keys / WASD / touch joystick. Press R to restart.</p>
  </header>
  <main>
    <canvas id="game" width="800" height="520" aria-label="Game canvas"></canvas>

    <div id="hud">
      <span id="score">Score: 0</span>
      <span id="lives">Lives: 3</span>
      <span id="highscore">High: 0</span>
      <button id="restart">Restart</button>
      <button id="show-hs">Show Highscores</button>
    </div>

    <div id="touch-controls" aria-hidden="true">
      <div id="joystick" role="application" aria-label="Joystick">
        <div id="stick"></div>
      </div>
      <button id="touch-restart" class="touch-btn" title="Restart">⟲</button>
    </div>

    <div id="hs-modal" class="modal hidden" role="dialog" aria-modal="true" aria-labelledby="hs-title">
      <div class="modal-content">
        <h2 id="hs-title">High Scores</h2>
        <ol id="hs-list"></ol>
        <button id="hs-close">Close</button>
      </div>
    </div>
  </main>

  <footer>
    <small>Open <code>index.html</code> in your browser. Electron scaffold included for packaging.</small>
  </footer>

  <script src="game.js"></script>
</body>
</html>
*{box-sizing:border-box;margin:0;padding:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,'Helvetica Neue',Arial}
body{display:flex;flex-direction:column;align-items:center;gap:12px;padding:18px;background:linear-gradient(180deg,#0f172a,#071024);color:#e6eef8;min-height:100vh}
header h1{font-size:28px}
canvas{background:#0b1220;border:4px solid #1f2937;border-radius:8px;display:block;touch-action:none}
#hud{display:flex;gap:12px;align-items:center;margin-top:8px}
#hud span{background:rgba(255,255,255,0.04);padding:6px 10px;border-radius:6px}
button{padding:6px 10px;border-radius:6px;border:none;background:#2563eb;color:white;cursor:pointer}
button:hover{opacity:0.95}
footer{margin-top:8px;font-size:12px;color:#9aa9c9}
code{background:#071024;padding:2px 6px;border-radius:4px}

/* touch controls */
#touch-controls{position:fixed;left:18px;bottom:18px;display:flex;gap:12px;align-items:center;pointer-events:auto}
#joystick{width:140px;height:140px;border-radius:999px;background:linear-gradient(180deg,rgba(255,255,255,0.03),rgba(255,255,255,0.01));position:relative;touch-action:none;border:2px solid rgba(255,255,255,0.03)}
#stick{width:64px;height:64px;border-radius:999px;background:rgba(255,255,255,0.06);position:absolute;left:38px;top:38px;display:flex;align-items:center;justify-content:center;transition:left 0.04s, top 0.04s}
.touch-btn{width:64px;height:64px;border-radius:999px;background:#2563eb;color:white;font-size:20px;border:none;box-shadow:0 6px 18px rgba(2,6,23,0.6)}
.hidden{display:none}

/* modal */
.modal{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;background:rgba(0,0,0,0.4)}
.modal-content{background:#071124;padding:18px;border-radius:10px;min-width:260px;border:1px solid rgba(255,255,255,0.04)}
.modal-content h2{margin-bottom:8px}
.modal-content ol{padding-left:18px;margin-bottom:12px}
/* Star Collector — single-file game logic
   Features:
   - Keyboard controls (Arrows / WASD)
   - Touch joystick + restart
   - Canvas-drawn sprites (no external image files)
   - Simple WebAudio-based sound effects
   - High score + top-10 saved in localStorage
   - Basic Electron scaffold compatibility
*/

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

let keys = {};
let score = 0;
let lives = 3;
let gameOver = false;
let enemySpeed = 1.8;
let spawnTimer = 0;

// touch joystick
const stick = document.getElementById('stick');
const joystick = document.getElementById('joystick');
let joyActive = false;
let joyPos = {x:0,y:0};

// audio
let audioCtx = null;
function ensureAudio(){
  if (audioCtx) return;
  try { audioCtx = new (window.AudioContext || window.webkitAudioContext)(); } catch(e) { audioCtx = null; }
}
function playBeep(freq=440, duration=0.08, type='sine'){
  ensureAudio();
  if (!audioCtx) return;
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = type;
  o.frequency.value = freq;
  g.gain.value = 0.08;
  o.connect(g); g.connect(audioCtx.destination);
  o.start();
  o.stop(audioCtx.currentTime + duration);
}
function playCollect(){ playBeep(880,0.06); setTimeout(()=>playBeep(1320,0.05),40); }
function playHit(){ playBeep(220,0.12,'square'); }
function playGameOver(){ playBeep(160,0.18); setTimeout(()=>playBeep(120,0.18),120); }

// helpers
function rand(min,max){ return Math.random()*(max-min)+min; }
function collided(a,b){ return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y; }

// draw helpers (canvas "sprites")
function drawPlayer(x,y,size){
  // glossy circle with border
  const cx = x + size/2, cy = y + size/2, r = size/2;
  // shadow
  ctx.beginPath();
  ctx.arc(cx+2, cy+3, r, 0, Math.PI*2);
  ctx.fillStyle = 'rgba(0,0,0,0.18)';
  ctx.fill();
  // body
  const grad = ctx.createLinearGradient(x, y, x, y+size);
  grad.addColorStop(0, '#93c5fd');
  grad.addColorStop(1, '#60a5fa');
  ctx.beginPath();
  ctx.arc(cx, cy, r-1, 0, Math.PI*2);
  ctx.fillStyle = grad;
  ctx.fill();
  // highlight
  ctx.beginPath();
  ctx.arc(cx - r*0.25, cy - r*0.35, r*0.25, 0, Math.PI*2);
  ctx.fillStyle = 'rgba(255,255,255,0.28)';
  ctx.fill();
}
function drawStar(x,y,size){
  // 5-point star polygon
  const cx = x + size/2, cy = y + size/2;
  const outer = size*0.45, inner = outer*0.45;
  const pts = [];
  for (let i=0;i<10;i++){
    const angle = Math.PI/2 + i*(Math.PI/5);
    const r = (i%2===0)?outer:inner;
    pts.push([cx + r*Math.cos(angle), cy - r*Math.sin(angle)]);
  }
  ctx.beginPath();
  ctx.moveTo(pts[0][0], pts[0][1]);
  for (let i=1;i<pts.length;i++) ctx.lineTo(pts[i][0], pts[i][1]);
  ctx.closePath();
  ctx.fillStyle = '#fbbf24';
  ctx.fill();
  // small stroke
  ctx.lineWidth = 1;
  ctx.strokeStyle = 'rgba(0,0,0,0.08)';
  ctx.stroke();
}
function drawEnemy(x,y,size){
  const cx = x + size/2, cy = y + size/2, r = size/2;
  // body
  ctx.beginPath();
  ctx.arc(cx, cy, r, 0, Math.PI*2);
  ctx.fillStyle = '#fb7185';
  ctx.fill();
  // eye
  ctx.beginPath();
  ctx.arc(cx - r*0.18, cy - r*0.18, r*0.18, 0, Math.PI*2);
  ctx.fillStyle = 'white';
  ctx.fill();
  ctx.beginPath();
  ctx.arc(cx - r*0.18, cy - r*0.18, r*0.08, 0, Math.PI*2);
  ctx.fillStyle = 'black';
  ctx.fill();
}

// Entities
class Player {
  constructor(){
    this.size = 48;
    this.x = 60; this.y = H/2 - this.size/2;
    this.speed = 3.8;
  }
  update(){
    if (keys['arrowup']||keys['w']) this.y -= this.speed;
    if (keys['arrowdown']||keys['s']) this.y += this.speed;
    if (keys['arrowleft']||keys['a']) this.x -= this.speed;
    if (keys['arrowright']||keys['d']) this.x += this.speed;
    // joystick
    if (Math.abs(joyPos.x) > 0.05 || Math.abs(joyPos.y) > 0.05){
      this.x += joyPos.x * this.speed * 1.6;
      this.y += joyPos.y * this.speed * 1.6;
    }
    this.x = Math.max(0, Math.min(W-this.size, this.x));
    this.y = Math.max(0, Math.min(H-this.size, this.y));
  }
  draw(){ drawPlayer(this.x, this.y, this.size); }
  rect(){ return {x:this.x,y:this.y,w:this.size,h:this.size}; }
}

class Star {
  constructor(){ this.size = 28; this.reset(); }
  reset(){ this.x = rand(200, W-60); this.y = rand(20, H-60); this.collected = false; }
  draw(){ if (!this.collected) drawStar(this.x, this.y, this.size); }
  rect(){ return {x:this.x,y:this.y,w:this.size,h:this.size}; }
}

class Enemy {
  constructor(speed=1.8){ this.size = 40; this.speed = speed; this.reset(); }
  reset(){ this.x = W + rand(20,200); this.y = rand(10, H-36); this.vx = - (this.speed + rand(0,1.6)); }
  update(){ this.x += this.vx; if (this.x < -this.size) this.reset(); }
  draw(){ drawEnemy(this.x, this.y, this.size); }
  rect(){ return {x:this.x,y:this.y,w:this.size,h:this.size}; }
}

// game state
let player = new Player();
let stars = [];
let enemies = [];

function resetGame(){
  score = 0; lives = 3; gameOver = false;
  player = new Player();
  stars = [new Star()];
  enemies = [new Enemy(enemySpeed)];
  spawnTimer = 0;
  enemySpeed = 1.8;
  updateHUD();
}

function updateHUD(){
  document.getElementById('score').textContent = 'Score: ' + score;
  document.getElementById('lives').textContent = 'Lives: ' + lives;
  document.getElementById('highscore').textContent = 'High: ' + (loadHighScore() || 0);
}
function restart(){ resetGame(); }

// highscores
function saveHighScore(val){ const prev = loadHighScore()||0; if (val>prev) localStorage.setItem('sc_high', String(val)); }
function loadHighScore(){ try{ return Number(localStorage.getItem('sc_high')) || 0; }catch(e){ return 0; } }
function pushHighScore(val){
  try{
    const key = 'sc_list';
    const list = JSON.parse(localStorage.getItem(key) || '[]');
    list.push({score: val, date: (new Date()).toISOString()});
    list.sort((a,b)=>b.score-a.score);
    localStorage.setItem(key, JSON.stringify(list.slice(0,10)));
  }catch(e){}
}
function showHS(){
  const list = JSON.parse(localStorage.getItem('sc_list') || '[]');
  const el = document.getElementById('hs-list'); el.innerHTML = '';
  if (list.length===0){ el.innerHTML = '<li>No scores yet</li>'; } else {
    list.forEach(item => {
      const d = new Date(item.date);
      const li = document.createElement('li');
      li.textContent = `${item.score} — ${d.toLocaleString()}`;
      el.appendChild(li);
    });
  }
  document.getElementById('hs-modal').classList.remove('hidden');
}

// main loop
function spawnEnemy(){ enemies.push(new Enemy(enemySpeed)); }

function gameTick(){
  if (gameOver) return draw();
  player.update();

  stars.forEach(s=>{
    if (!s.collected && collided(player.rect(), s.rect())){
      s.collected = true;
      score += 10;
      updateHUD();
      playCollect();
      setTimeout(()=>stars.push(new Star()), 120);
      if (score % 30 === 0){ enemySpeed += 0.4; spawnEnemy(); }
    }
  });

  enemies.forEach(e=>{
    e.update();
    if (collided(player.rect(), e.rect())){
      e.reset();
      lives -= 1;
      updateHUD();
      playHit();
      if (lives <= 0){ gameOver = true; handleGameOver(); }
    }
  });

  spawnTimer += 1;
  if (spawnTimer > 600){ spawnTimer = 0; spawnEnemy(); }

  draw();
  requestAnimationFrame(gameTick);
}

function handleGameOver(){
  playGameOver();
  saveHighScore(score);
  pushHighScore(score);
  updateHUD();
}

// drawing scene
function draw(){
  ctx.clearRect(0,0,W,H);
  // background
  ctx.fillStyle = '#071124';
  ctx.fillRect(0,0,W,H);
  for (let i=0;i<50;i++){
    const sx = (i*73)%W, sy = (i*97)%H;
    ctx.fillStyle = 'rgba(255,255,255,0.02)';
    ctx.fillRect(sx,sy,2,2);
  }
  // objects
  stars.forEach(s=>s.draw());
  enemies.forEach(e=>e.draw());
  player.draw();

  if (gameOver){
    ctx.fillStyle = 'rgba(2,6,23,0.6)';
    ctx.fillRect(0,0,W,H);
    ctx.fillStyle = 'white';
    ctx.font = '28px system-ui';
    ctx.textAlign = 'center';
    ctx.fillText('Game Over', W/2, H/2 - 10);
    ctx.font = '16px system-ui';
    ctx.fillText('Press R or touch ⟲ to play again', W/2, H/2 + 20);
  }
}

// input handlers
document.addEventListener('keydown', e => { keys[e.key.toLowerCase()] = true; if (e.key==='r' || e.key==='R') restart(); });
document.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });
document.getElementById('restart').addEventListener('click', restart);
document.getElementById('show-hs').addEventListener('click', showHS);
document.getElementById('hs-close').addEventListener('click', ()=>{ document.getElementById('hs-modal').classList.add('hidden'); });
document.getElementById('touch-restart').addEventListener('click', ()=>{ restart(); });

// joystick handlers (touch)
function getLocalPos(evt, el){
  const rect = el.getBoundingClientRect();
  const t = evt.touches ? evt.touches[0] : evt;
  return {x: t.clientX - rect.left, y: t.clientY - rect.top};
}
joystick.addEventListener('touchstart', e => { e.preventDefault(); joyActive = true; updateStick(getLocalPos(e, joystick)); });
joystick.addEventListener('touchmove', e => { e.preventDefault(); if (!joyActive) return; updateStick(getLocalPos(e, joystick)); });
joystick.addEventListener('touchend', e => { e.preventDefault(); joyActive = false; resetStick(); });
joystick.addEventListener('mousedown', e => { joyActive = true; updateStick(getLocalPos(e, joystick)); });
window.addEventListener('mousemove', e => { if (!joyActive) return; updateStick(getLocalPos(e, joystick)); });
window.addEventListener('mouseup', e => { if (!joyActive) return; joyActive = false; resetStick(); });

function updateStick(p){
  const rect = joystick.getBoundingClientRect();
  const cx = rect.width/2, cy = rect.height/2;
  const dx = p.x - cx, dy = p.y - cy;
  const max = rect.width/2 - 24;
  const dist = Math.min(Math.hypot(dx,dy), max);
  const angle = Math.atan2(dy,dx);
  const nx = cx + Math.cos(angle)*dist, ny = cy + Math.sin(angle)*dist;
  stick.style.left = (nx - stick.offsetWidth/2)+'px';
  stick.style.top = (ny - stick.offsetHeight/2)+'px';
  joyPos.x = (nx - cx)/max;
  joyPos.y = (ny - cy)/max;
}
function resetStick(){
  stick.style.left = (joystick.offsetWidth/2 - stick.offsetWidth/2) + 'px';
  stick.style.top = (joystick.offsetHeight/2 - stick.offsetHeight/2) + 'px';
  joyPos = {x:0,y:0};
}

// init
function init(){
  resetGame();
  updateHUD();
  requestAnimationFrame(gameTick);
  // resume audio on first user interaction on mobile
  document.addEventListener('click', function once(){ ensureAudio(); if (audioCtx && audioCtx.state === 'suspended') audioCtx.resume(); document.removeEventListener('click', once); });
}
init();
Star Collector — mysys2 (Full)
======================

A lightweight browser game built with a single HTML/CSS/JS project.

Features
- Keyboard controls (Arrow keys / WASD)
- Touch joystick + restart for mobile
- Canvas-drawn sprites (no external image files)
- WebAudio sound effects (beeps) — no audio files required
- High score (single best) and top-10 list saved to localStorage
- Basic Electron scaffold files included (package.json, main.js) for packaging

How to run (browser)
--------------------
1. Put the files in a folder `mysys2`.
2. Start a local HTTP server in the folder (recommended):
or open `index.html` directly in your browser (some browsers restrict certain features when loaded via file://).
3. Controls:
- Keyboard: Arrow keys or WASD. Press R to restart.
- Touch: Use the joystick at the lower-left and the circular button to restart.
4. Click "Show Highscores" in the HUD to view the top 10 saved scores.

Electron packaging (basic)
--------------------------
Files included:
- package.json
- main.js

To run in Electron:
1. `npm install`
2. `npx electron .`

Notes
-----
- If you want sprites, recorded sound effects, or music files instead of the WebAudio beeps, you can add them to the folder and I will update the code to use them.
- Want features like multiple levels, power-ups, or an online leaderboard? Tell me which and I'll implement them.
{
  "name": "star-collector-mysys2",
  "version": "1.0.0",
  "description": "Star Collector packaged with Electron (scaffold)",
  "main": "main.js",
  "scripts": {
    "start": "electron ."
  },
  "devDependencies": {
    "electron": "^26.0.0"
  }
}
const { app, BrowserWindow } = require('electron');
const path = require('path');

function createWindow(){
  const win = new BrowserWindow({
    width: 900, height: 640,
    webPreferences: { nodeIntegration: false, contextIsolation: true }
  });
  win.loadFile('index.html');
}

app.whenReady().then(createWindow);
app.on('window-all-closed', ()=>{ if (process.platform !== 'darwin') app.quit(); });


