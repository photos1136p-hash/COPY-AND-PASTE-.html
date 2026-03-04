<!DOCTYPE html>

<html lang="en">
<head>
<meta charset="UTF-8">
<title>DEADZONE — Survive the Night</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { background: #000; overflow: hidden; font-family: 'Courier New', monospace; }
  #canvas { display: block; width: 100vw; height: 100vh; }
  #hud {
    position: fixed; top: 0; left: 0; width: 100%; height: 100%;
    pointer-events: none; z-index: 10;
  }
  #crosshair {
    position: absolute; top: 50%; left: 50%; transform: translate(-50%,-50%);
    width: 20px; height: 20px;
  }
  #crosshair::before, #crosshair::after {
    content: ''; position: absolute; background: rgba(255,255,255,0.85);
  }
  #crosshair::before { width: 2px; height: 100%; left: 50%; transform: translateX(-50%); }
  #crosshair::after { height: 2px; width: 100%; top: 50%; transform: translateY(-50%); }
  #health-bar {
    position: absolute; bottom: 40px; left: 40px;
    width: 220px;
  }
  #health-label { color: #ff4444; font-size: 11px; letter-spacing: 3px; margin-bottom: 6px; text-shadow: 0 0 8px #ff4444; }
  #health-track { width: 100%; height: 6px; background: rgba(255,68,68,0.2); border: 1px solid rgba(255,68,68,0.4); }
  #health-fill { height: 100%; background: linear-gradient(90deg, #ff2222, #ff6644); transition: width 0.3s; box-shadow: 0 0 8px #ff4444; }
  #ammo {
    position: absolute; bottom: 40px; right: 40px; text-align: right;
  }
  #ammo-count { font-size: 48px; color: #fff; text-shadow: 0 0 20px rgba(255,255,255,0.3); line-height: 1; }
  #ammo-label { font-size: 10px; color: rgba(255,255,255,0.4); letter-spacing: 3px; }
  #wave-info {
    position: absolute; top: 30px; left: 50%; transform: translateX(-50%);
    text-align: center;
  }
  #wave-num { font-size: 13px; color: #ffaa00; letter-spacing: 4px; text-shadow: 0 0 12px #ffaa00; }
  #enemy-count { font-size: 11px; color: rgba(255,255,255,0.5); letter-spacing: 2px; margin-top: 4px; }
  #score-display {
    position: absolute; top: 30px; right: 40px;
    color: rgba(255,255,255,0.6); font-size: 12px; letter-spacing: 2px; text-align: right;
  }
  #score-val { font-size: 24px; color: #fff; }
  #kill-feed {
    position: absolute; top: 100px; right: 40px;
    text-align: right;
  }
  .kill-msg {
    color: #ff6644; font-size: 11px; letter-spacing: 1px;
    animation: fadeKill 2s forwards;
  }
  @keyframes fadeKill { 0%{opacity:1;transform:translateX(0)} 80%{opacity:1} 100%{opacity:0;transform:translateX(20px)} }
  #overlay {
    position: fixed; top: 0; left: 0; width: 100%; height: 100%;
    background: rgba(0,0,0,0.92); display: flex; flex-direction: column;
    align-items: center; justify-content: center; z-index: 100;
    color: #fff;
  }
  #overlay h1 {
    font-size: 72px; letter-spacing: 12px; color: #ff2222;
    text-shadow: 0 0 40px #ff2222, 0 0 80px rgba(255,34,34,0.3);
    margin-bottom: 10px;
  }
  #overlay .sub {
    font-size: 13px; letter-spacing: 6px; color: rgba(255,255,255,0.4);
    margin-bottom: 60px;
  }
  #start-btn {
    padding: 18px 60px; background: transparent;
    border: 1px solid rgba(255,34,34,0.6); color: #ff4444;
    font-family: 'Courier New', monospace; font-size: 14px; letter-spacing: 4px;
    cursor: pointer; transition: all 0.3s; pointer-events: all;
    text-transform: uppercase;
  }
  #start-btn:hover { background: rgba(255,34,34,0.1); border-color: #ff4444; box-shadow: 0 0 30px rgba(255,34,34,0.3); }
  #controls-hint {
    margin-top: 40px; color: rgba(255,255,255,0.25);
    font-size: 11px; letter-spacing: 2px; line-height: 2; text-align: center;
  }
  #damage-overlay {
    position: fixed; top: 0; left: 0; width: 100%; height: 100%;
    background: radial-gradient(ellipse at center, transparent 40%, rgba(255,0,0,0.5) 100%);
    pointer-events: none; z-index: 9; opacity: 0; transition: opacity 0.1s;
  }
  #reload-msg {
    position: absolute; bottom: 120px; left: 50%; transform: translateX(-50%);
    color: #ffaa00; font-size: 12px; letter-spacing: 4px;
    opacity: 0; transition: opacity 0.3s;
  }
  #gameover {
    display: none; position: fixed; top:0; left:0; width:100%; height:100%;
    background: rgba(0,0,0,0.9); z-index: 200;
    flex-direction: column; align-items: center; justify-content: center;
    color: #fff;
  }
  #gameover h2 { font-size: 60px; letter-spacing: 10px; color: #ff2222; text-shadow: 0 0 40px #ff2222; }
  #gameover .stats { margin: 30px 0; color: rgba(255,255,255,0.5); font-size: 13px; letter-spacing: 3px; line-height: 2.5; text-align: center; }
  #gameover .stats span { color: #fff; }
  #restart-btn {
    padding: 16px 50px; background: transparent;
    border: 1px solid rgba(255,34,34,0.6); color: #ff4444;
    font-family: 'Courier New', monospace; font-size: 13px; letter-spacing: 4px;
    cursor: pointer; transition: all 0.3s; pointer-events: all;
    margin-top: 10px;
  }
  #restart-btn:hover { background: rgba(255,34,34,0.1); box-shadow: 0 0 30px rgba(255,34,34,0.3); }
  #muzzle-flash {
    position: fixed; top: 0; left: 0; width: 100%; height: 100%;
    background: rgba(255,220,100,0.06); pointer-events: none; z-index: 8;
    opacity: 0;
  }
  #wave-announce {
    position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
    text-align: center; opacity: 0; pointer-events: none;
    transition: opacity 0.5s;
  }
  #wave-announce .wave-title { font-size: 50px; color: #ffaa00; letter-spacing: 8px; text-shadow: 0 0 30px #ffaa00; }
  #wave-announce .wave-sub { font-size: 12px; color: rgba(255,255,255,0.5); letter-spacing: 4px; margin-top: 10px; }
</style>
</head>
<body>
<canvas id="canvas"></canvas>

<div id="hud">
  <div id="crosshair"></div>
  <div id="health-bar">
    <div id="health-label">HEALTH</div>
    <div id="health-track"><div id="health-fill" style="width:100%"></div></div>
  </div>
  <div id="ammo">
    <div id="ammo-count">30</div>
    <div id="ammo-label">ROUNDS</div>
  </div>
  <div id="wave-info">
    <div id="wave-num">WAVE 1</div>
    <div id="enemy-count">ENEMIES: 5</div>
  </div>
  <div id="score-display">
    <div style="font-size:11px;letter-spacing:3px;margin-bottom:4px">SCORE</div>
    <div id="score-val">0</div>
  </div>
  <div id="kill-feed"></div>
  <div id="damage-overlay"></div>
  <div id="muzzle-flash"></div>
  <div id="reload-msg">RELOADING...</div>
  <div id="wave-announce">
    <div class="wave-title" id="wave-title-text">WAVE 1</div>
    <div class="wave-sub" id="wave-sub-text">SURVIVE</div>
  </div>
</div>

<div id="overlay">
  <h1>DEADZONE</h1>
  <div class="sub">SURVIVE THE NIGHT</div>
  <button id="start-btn" onclick="startGame()">ENTER ZONE</button>
  <div class="controls-hint" id="controls-hint">
    <div id="controls-hint" class="controls-hint">
      WASD — MOVE &nbsp;|&nbsp; MOUSE — AIM &nbsp;|&nbsp; CLICK — FIRE &nbsp;|&nbsp; R — RELOAD &nbsp;|&nbsp; SHIFT — SPRINT
    </div>
  </div>
</div>

<div id="gameover">
  <h2>YOU DIED</h2>
  <div class="stats">
    WAVE REACHED &nbsp;<span id="go-wave">1</span><br>
    KILLS &nbsp;<span id="go-kills">0</span><br>
    SCORE &nbsp;<span id="go-score">0</span>
  </div>
  <button id="restart-btn" onclick="restartGame()">TRY AGAIN</button>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

<script>
// ===== GAME STATE =====
let scene, camera, renderer, clock;
let player = { health: 100, maxHealth: 100, speed: 0.12, sprintSpeed: 0.22 };
let ammo = 30, maxAmmo = 30, reloading = false, reloadTime = 2000;
let score = 0, kills = 0, wave = 1, waveActive = false;
let enemies = [], bullets = [], enemyBullets = [];
let gameRunning = false, gameOver = false;
let keys = {}, mouseX = 0, mouseY = 0;
let yaw = 0, pitch = 0;
let groundMeshes = [], trees = [], rocks = [];
let nextWaveTimer = 0, waveEnemyCount = 0;
let bobTime = 0, lastShot = 0;
let flashTimeout;
let muzzleFlashMesh;

// DOM refs
const healthFill = document.getElementById('health-fill');
const ammoCount = document.getElementById('ammo-count');
const waveNum = document.getElementById('wave-num');
const enemyCountEl = document.getElementById('enemy-count');
const scoreVal = document.getElementById('score-val');
const killFeed = document.getElementById('kill-feed');
const damageOverlay = document.getElementById('damage-overlay');
const muzzleFlashEl = document.getElementById('muzzle-flash');
const reloadMsg = document.getElementById('reload-msg');
const waveAnnounce = document.getElementById('wave-announce');
const gameover = document.getElementById('gameover');
const overlay = document.getElementById('overlay');

function initThree() {
  scene = new THREE.Scene();
  scene.fog = new THREE.FogExp2(0x0a0e14, 0.045);
  scene.background = new THREE.Color(0x050810);

  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 300);
  camera.position.set(0, 1.7, 0);

  renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('canvas'), antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 0.6;
  renderer.outputEncoding = THREE.sRGBEncoding;

  clock = new THREE.Clock();
  buildWorld();
  buildGun();
  startWave(1);
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });
}

function buildWorld() {
  // Ground — tiled plane with normal-ish look
  const groundGeo = new THREE.PlaneGeometry(200, 200, 60, 60);
  // Subtle vertex displacement for terrain feel
  const pos = groundGeo.attributes.position;
  for (let i = 0; i < pos.count; i++) {
    const x = pos.getX(i), z = pos.getZ(i);
    const d = Math.sqrt(x*x+z*z);
    if (d > 5) {
      pos.setY(i, Math.sin(x*0.15)*Math.cos(z*0.12)*1.2 + Math.sin(x*0.4+z*0.3)*0.3);
    }
  }
  groundGeo.computeVertexNormals();
  const groundMat = new THREE.MeshLambertMaterial({ color: 0x1a2a15 });
  const ground = new THREE.Mesh(groundGeo, groundMat);
  ground.rotation.x = -Math.PI / 2;
  ground.receiveShadow = true;
  scene.add(ground);

  // Moon light
  const moon = new THREE.DirectionalLight(0x6688bb, 0.8);
  moon.position.set(30, 80, -40);
  moon.castShadow = true;
  moon.shadow.mapSize.width = 2048;
  moon.shadow.mapSize.height = 2048;
  moon.shadow.camera.near = 0.5;
  moon.shadow.camera.far = 200;
  moon.shadow.camera.left = -80;
  moon.shadow.camera.right = 80;
  moon.shadow.camera.top = 80;
  moon.shadow.camera.bottom = -80;
  scene.add(moon);

  // Ambient
  scene.add(new THREE.AmbientLight(0x111a22, 1.5));

  // Red accent point lights (campfire feel)
  const fireLight = new THREE.PointLight(0xff4400, 2, 20);
  fireLight.position.set(0, 1, 0);
  scene.add(fireLight);

  // Stars
  const starGeo = new THREE.BufferGeometry();
  const starVerts = [];
  for (let i = 0; i < 3000; i++) {
    starVerts.push((Math.random()-0.5)*500, Math.random()*200+30, (Math.random()-0.5)*500);
  }
  starGeo.setAttribute('position', new THREE.Float32BufferAttribute(starVerts, 3));
  const starMat = new THREE.PointsMaterial({ color: 0xffffff, size: 0.3 });
  scene.add(new THREE.Points(starGeo, starMat));

  // Trees
  for (let i = 0; i < 80; i++) {
    let angle = Math.random() * Math.PI * 2;
    let r = 15 + Math.random() * 55;
    let tx = Math.cos(angle) * r, tz = Math.sin(angle) * r;
    addTree(tx, tz);
  }

  // Rocks
  for (let i = 0; i < 30; i++) {
    let angle = Math.random() * Math.PI * 2;
    let r = 8 + Math.random() * 40;
    addRock(Math.cos(angle)*r, Math.sin(angle)*r);
  }

  // Perimeter wall (invisible boundary)
  for (let i = 0; i < 8; i++) {
    const a = (i / 8) * Math.PI * 2;
    const wallLight = new THREE.PointLight(0x002244, 3, 15);
    wallLight.position.set(Math.cos(a)*65, 3, Math.sin(a)*65);
    scene.add(wallLight);
  }
}

function addTree(x, z) {
  const h = 4 + Math.random() * 5;
  const trunkGeo = new THREE.CylinderGeometry(0.15, 0.3, h, 6);
  const trunkMat = new THREE.MeshLambertMaterial({ color: 0x2a1a0a });
  const trunk = new THREE.Mesh(trunkGeo, trunkMat);
  trunk.position.set(x, h/2, z);
  trunk.castShadow = true;
  scene.add(trunk);

  // Foliage layers
  for (let i = 0; i < 3; i++) {
    const r = 1.8 - i * 0.4;
    const fGeo = new THREE.ConeGeometry(r, 2.5, 7);
    const fMat = new THREE.MeshLambertMaterial({ color: new THREE.Color(0x0d2a0d).lerp(new THREE.Color(0x1a4a1a), Math.random()) });
    const f = new THREE.Mesh(fGeo, fMat);
    f.position.set(x, h - 0.5 + i * 1.8, z);
    f.rotation.y = Math.random() * Math.PI;
    f.castShadow = true;
    scene.add(f);
  }
  trees.push({ x, z, r: 0.4 });
}

function addRock(x, z) {
  const s = 0.5 + Math.random() * 1.5;
  const geo = new THREE.DodecahedronGeometry(s, 0);
  const mat = new THREE.MeshLambertMaterial({ color: 0x2a2a2a });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.position.set(x, s * 0.3, z);
  mesh.rotation.set(Math.random(), Math.random(), Math.random());
  mesh.castShadow = true;
  scene.add(mesh);
  rocks.push({ x, z, r: s * 0.9 });
}

// ===== GUN =====
let gunGroup;
function buildGun() {
  gunGroup = new THREE.Group();
  // Barrel
  const barrelGeo = new THREE.BoxGeometry(0.06, 0.06, 0.5);
  const metalMat = new THREE.MeshLambertMaterial({ color: 0x222222 });
  const barrel = new THREE.Mesh(barrelGeo, metalMat);
  barrel.position.z = -0.35;
  gunGroup.add(barrel);
  // Body
  const bodyGeo = new THREE.BoxGeometry(0.1, 0.12, 0.35);
  const body = new THREE.Mesh(bodyGeo, metalMat);
  body.position.set(0, -0.02, -0.12);
  gunGroup.add(body);
  // Grip
  const gripGeo = new THREE.BoxGeometry(0.08, 0.14, 0.08);
  const gripMat = new THREE.MeshLambertMaterial({ color: 0x111111 });
  const grip = new THREE.Mesh(gripGeo, gripMat);
  grip.position.set(0, -0.12, 0.02);
  grip.rotation.x = 0.2;
  gunGroup.add(grip);
  // Scope
  const scopeGeo = new THREE.CylinderGeometry(0.025, 0.025, 0.18, 8);
  const scope = new THREE.Mesh(scopeGeo, metalMat);
  scope.rotation.x = Math.PI/2;
  scope.position.set(0, 0.075, -0.2);
  gunGroup.add(scope);

  gunGroup.position.set(0.22, -0.22, -0.5);
  camera.add(gunGroup);
  scene.add(camera);

  // Muzzle flash
  const flashGeo = new THREE.SphereGeometry(0.08, 6, 6);
  const flashMat = new THREE.MeshBasicMaterial({ color: 0xffcc44, transparent: true, opacity: 0 });
  muzzleFlashMesh = new THREE.Mesh(flashGeo, flashMat);
  muzzleFlashMesh.position.set(0, 0, -0.62);
  gunGroup.add(muzzleFlashMesh);
}

// ===== ENEMIES =====
function spawnEnemy() {
  const angle = Math.random() * Math.PI * 2;
  const r = 30 + Math.random() * 15;
  const x = Math.cos(angle) * r;
  const z = Math.sin(angle) * r;

  const group = new THREE.Group();
  // Body
  const bodyGeo = new THREE.BoxGeometry(0.7, 1.0, 0.4);
  const bodyMat = new THREE.MeshLambertMaterial({ color: 0x331100 });
  const body = new THREE.Mesh(bodyGeo, bodyMat);
  body.position.y = 1.0;
  body.castShadow = true;
  group.add(body);
  // Head
  const headGeo = new THREE.BoxGeometry(0.5, 0.5, 0.45);
  const headMat = new THREE.MeshLambertMaterial({ color: 0x662200 });
  const head = new THREE.Mesh(headGeo, headMat);
  head.position.y = 1.75;
  head.castShadow = true;
  group.add(head);
  // Eyes glow
  const eyeGeo = new THREE.SphereGeometry(0.07, 6, 6);
  const eyeMat = new THREE.MeshBasicMaterial({ color: 0xff2200 });
  [-0.12, 0.12].forEach(ex => {
    const eye = new THREE.Mesh(eyeGeo, eyeMat);
    eye.position.set(ex, 1.78, -0.22);
    group.add(eye);
  });
  // Legs
  [-0.18, 0.18].forEach(lx => {
    const legGeo = new THREE.BoxGeometry(0.22, 0.8, 0.22);
    const leg = new THREE.Mesh(legGeo, bodyMat);
    leg.position.set(lx, 0.4, 0);
    leg.castShadow = true;
    group.add(leg);
  });
  // Arms
  [-0.5, 0.5].forEach(ax => {
    const armGeo = new THREE.BoxGeometry(0.18, 0.75, 0.18);
    const arm = new THREE.Mesh(armGeo, bodyMat);
    arm.position.set(ax, 1.05, 0);
    arm.castShadow = true;
    group.add(arm);
  });

  // Eye point light
  const eyeLight = new THREE.PointLight(0xff2200, 1.5, 5);
  eyeLight.position.set(0, 1.78, -0.3);
  group.add(eyeLight);

  group.position.set(x, 0, z);
  scene.add(group);

  const hp = 40 + wave * 15;
  enemies.push({
    mesh: group,
    hp, maxHp: hp,
    x, z,
    speed: 0.025 + wave * 0.005 + Math.random() * 0.01,
    lastShot: 0,
    shotInterval: 2500 - wave * 150,
    walkTime: 0,
    alive: true
  });
}

function startWave(num) {
  wave = num;
  waveEnemyCount = 3 + num * 3;
  waveActive = true;
  waveNum.textContent = `WAVE ${num}`;
  updateEnemyCount();

  // Announce
  document.getElementById('wave-title-text').textContent = `WAVE ${num}`;
  document.getElementById('wave-sub-text').textContent = `${waveEnemyCount} ENEMIES INCOMING`;
  waveAnnounce.style.opacity = '1';
  setTimeout(() => { waveAnnounce.style.opacity = '0'; }, 2500);

  // Spawn enemies staggered
  let spawned = 0;
  const spawnNext = () => {
    if (spawned < waveEnemyCount && gameRunning) {
      spawnEnemy();
      spawned++;
      setTimeout(spawnNext, 600 - Math.min(wave*40, 400));
    }
  };
  spawnNext();
}

function updateEnemyCount() {
  enemyCountEl.textContent = `ENEMIES: ${enemies.filter(e=>e.alive).length}`;
}

// ===== SHOOTING =====
function shoot() {
  if (!gameRunning || reloading || ammo <= 0) return;
  const now = Date.now();
  if (now - lastShot < 120) return;
  lastShot = now;

  ammo--;
  ammoCount.textContent = ammo;
  if (ammo === 0) startReload();

  // Muzzle flash
  muzzleFlashMesh.material.opacity = 0.9;
  muzzleFlashEl.style.opacity = '1';
  clearTimeout(flashTimeout);
  flashTimeout = setTimeout(() => {
    muzzleFlashMesh.material.opacity = 0;
    muzzleFlashEl.style.opacity = '0';
  }, 60);

  // Gun kick
  gunGroup.position.z += 0.05;
  setTimeout(() => { gunGroup.position.z -= 0.05; }, 80);

  // Raycast
  const dir = new THREE.Vector3();
  camera.getWorldDirection(dir);

  // Slight spread
  dir.x += (Math.random()-0.5)*0.02;
  dir.y += (Math.random()-0.5)*0.02;
  dir.normalize();

  // Check enemy hits
  const ray = new THREE.Raycaster(camera.position.clone(), dir, 0, 150);
  let hit = false;
  let closest = null, closestDist = Infinity;

  enemies.forEach(e => {
    if (!e.alive) return;
    const meshes = [];
    e.mesh.traverse(c => { if (c.isMesh) meshes.push(c); });
    const intersects = ray.intersectObjects(meshes);
    if (intersects.length > 0 && intersects[0].distance < closestDist) {
      closestDist = intersects[0].distance;
      closest = { enemy: e, point: intersects[0].point };
    }
  });

  if (closest) {
    hitEnemy(closest.enemy, 25 + Math.random() * 10);
    spawnHitParticles(closest.point, 0xff4400);
    hit = true;
  }

  // Bullet tracer
  spawnBulletTracer(camera.position.clone(), dir);
}

function hitEnemy(enemy, dmg) {
  enemy.hp -= dmg;
  enemy.mesh.children.forEach(c => {
    if (c.isMesh) {
      c.material = c.material.clone();
      c.material.emissive = new THREE.Color(0.5, 0.1, 0.0);
      setTimeout(() => { if(c.material) c.material.emissive.set(0,0,0); }, 100);
    }
  });
  if (enemy.hp <= 0) killEnemy(enemy);
}

function killEnemy(enemy) {
  enemy.alive = false;
  scene.remove(enemy.mesh);
  kills++;
  score += 100 + wave * 50;
  scoreVal.textContent = score;
  addKillMsg();
  updateEnemyCount();

  spawnHitParticles(enemy.mesh.position.clone().add(new THREE.Vector3(0,1,0)), 0x881100);

  const aliveEnemies = enemies.filter(e => e.alive);
  if (aliveEnemies.length === 0 && waveActive) {
    waveActive = false;
    ammo = maxAmmo;
    ammoCount.textContent = ammo;
    setTimeout(() => startWave(wave + 1), 3000);
  }
}

function addKillMsg() {
  const msgs = ['ELIMINATED', 'HOSTILE DOWN', 'NEUTRALIZED', 'TARGET DOWN', 'CLEAN SHOT'];
  const div = document.createElement('div');
  div.className = 'kill-msg';
  div.textContent = msgs[Math.floor(Math.random()*msgs.length)];
  killFeed.appendChild(div);
  setTimeout(() => div.remove(), 2000);
}

function startReload() {
  if (reloading || ammo === maxAmmo) return;
  reloading = true;
  reloadMsg.style.opacity = '1';
  gunGroup.rotation.z = 0.3;
  setTimeout(() => {
    reloading = false;
    ammo = maxAmmo;
    ammoCount.textContent = ammo;
    reloadMsg.style.opacity = '0';
    gunGroup.rotation.z = 0;
  }, reloadTime);
}

// ===== PARTICLES =====
let particles = [];
function spawnHitParticles(pos, color) {
  const c = new THREE.Color(color);
  for (let i = 0; i < 8; i++) {
    const geo = new THREE.SphereGeometry(0.04, 4, 4);
    const mat = new THREE.MeshBasicMaterial({ color: c });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.copy(pos);
    scene.add(mesh);
    const vel = new THREE.Vector3((Math.random()-0.5)*0.3, Math.random()*0.3, (Math.random()-0.5)*0.3);
    particles.push({ mesh, vel, life: 0.5 });
  }
}

function spawnBulletTracer(origin, dir) {
  const geo = new THREE.CylinderGeometry(0.01, 0.01, 2, 4);
  const mat = new THREE.MeshBasicMaterial({ color: 0xffee88, transparent: true, opacity: 0.6 });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.position.copy(origin).addScaledVector(dir, 1);
  mesh.lookAt(origin.clone().addScaledVector(dir, -1));
  mesh.rotation.x += Math.PI/2;
  scene.add(mesh);
  particles.push({ mesh, vel: dir.clone().multiplyScalar(0.8), life: 0.12, tracer: true });
}

// ===== PLAYER MOVEMENT =====
function movePlayer(dt) {
  const sprint = keys['ShiftLeft'] || keys['ShiftRight'];
  const speed = sprint ? player.sprintSpeed : player.speed;

  const dir = new THREE.Vector3();
  const fwd = new THREE.Vector3(-Math.sin(yaw), 0, -Math.cos(yaw));
  const right = new THREE.Vector3(Math.cos(yaw), 0, -Math.sin(yaw));

  if (keys['KeyW'] || keys['ArrowUp']) dir.add(fwd);
  if (keys['KeyS'] || keys['ArrowDown']) dir.sub(fwd);
  if (keys['KeyA'] || keys['ArrowLeft']) dir.sub(right);
  if (keys['KeyD'] || keys['ArrowRight']) dir.add(right);
  if (dir.length() > 0) dir.normalize();

  let nx = camera.position.x + dir.x * speed;
  let nz = camera.position.z + dir.z * speed;

  // Boundary
  const maxR = 60;
  if (Math.sqrt(nx*nx+nz*nz) < maxR) {
    // Simple obstacle check (trees/rocks)
    let blocked = false;
    trees.forEach(t => { if (Math.sqrt((nx-t.x)**2+(nz-t.z)**2) < t.r+0.4) blocked = true; });
    rocks.forEach(r => { if (Math.sqrt((nx-r.x)**2+(nz-r.z)**2) < r.r+0.3) blocked = true; });
    if (!blocked) {
      camera.position.x = nx;
      camera.position.z = nz;
    }
  }

  // Head bob
  if (dir.length() > 0) {
    bobTime += dt * (sprint ? 12 : 8);
    camera.position.y = 1.7 + Math.sin(bobTime) * 0.04;
  } else {
    camera.position.y = 1.7 + Math.sin(bobTime) * 0.01;
  }
}

function updateEnemies(dt) {
  const now = Date.now();
  enemies.forEach(e => {
    if (!e.alive) return;
    const px = camera.position.x, pz = camera.position.z;
    const dx = px - e.x, dz = pz - e.z;
    const dist = Math.sqrt(dx*dx+dz*dz);

    // Move toward player
    if (dist > 2) {
      e.x += (dx/dist)*e.speed;
      e.z += (dz/dist)*e.speed;
      e.mesh.position.set(e.x, 0, e.z);
    }

    // Walk animation
    e.walkTime += dt * 5;
    e.mesh.rotation.y = Math.atan2(dx, dz);
    e.mesh.children.forEach((c,i) => {
      if (i < 2 && c.isMesh) { // legs
        // c.rotation.x = Math.sin(e.walkTime + i*Math.PI) * 0.4;
      }
    });

    // Look at player
    e.mesh.lookAt(new THREE.Vector3(px, 0, pz));

    // Attack
    if (dist < 2.5) {
      if (now - e.lastShot > 800) {
        e.lastShot = now;
        damagePlayer(8 + wave * 2);
      }
    } else if (dist < 35 && now - e.lastShot > e.shotInterval) {
      e.lastShot = now;
      fireEnemyBullet(e);
    }
  });
}

function fireEnemyBullet(enemy) {
  const px = camera.position.x, pz = camera.position.z;
  const dx = px - enemy.x, dz = pz - enemy.z;
  const dist = Math.sqrt(dx*dx+dz*dz);
  const dir = new THREE.Vector3(dx/dist + (Math.random()-0.5)*0.15, 0, dz/dist + (Math.random()-0.5)*0.15).normalize();

  const geo = new THREE.SphereGeometry(0.06, 6, 6);
  const mat = new THREE.MeshBasicMaterial({ color: 0xff4400 });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.position.set(enemy.x, 1.4, enemy.z);
  scene.add(mesh);
  enemyBullets.push({ mesh, dir, speed: 0.35, x: enemy.x, y: 1.4, z: enemy.z, life: 4 });
}

function updateEnemyBullets(dt) {
  enemyBullets = enemyBullets.filter(b => {
    b.x += b.dir.x * b.speed;
    b.z += b.dir.z * b.speed;
    b.mesh.position.set(b.x, b.y, b.z);
    b.life -= dt;

    // Hit player
    const px = camera.position.x, pz = camera.position.z;
    if (Math.sqrt((b.x-px)**2+(b.z-pz)**2) < 1.0) {
      damagePlayer(10 + wave * 3);
      scene.remove(b.mesh);
      return false;
    }
    if (b.life <= 0) { scene.remove(b.mesh); return false; }
    return true;
  });
}

function damagePlayer(dmg) {
  player.health = Math.max(0, player.health - dmg);
  healthFill.style.width = (player.health / player.maxHealth * 100) + '%';
  damageOverlay.style.opacity = '0.8';
  setTimeout(() => damageOverlay.style.opacity = '0', 200);
  if (player.health <= 0) triggerGameOver();
}

function updateParticles(dt) {
  particles = particles.filter(p => {
    p.life -= dt;
    p.mesh.position.add(p.vel.clone().multiplyScalar(dt * 60));
    if (!p.tracer) { p.vel.y -= 0.01; }
    p.mesh.material.opacity = Math.max(0, p.life / 0.5);
    if (p.life <= 0) { scene.remove(p.mesh); return false; }
    return true;
  });
}

function triggerGameOver() {
  gameRunning = false;
  gameOver = true;
  document.getElementById('go-wave').textContent = wave;
  document.getElementById('go-kills').textContent = kills;
  document.getElementById('go-score').textContent = score;
  gameover.style.display = 'flex';
  document.exitPointerLock();
}

// ===== GAME LOOP =====
function gameLoop() {
  if (!gameRunning) return;
  requestAnimationFrame(gameLoop);

  const dt = Math.min(clock.getDelta(), 0.05);

  movePlayer(dt);
  updateEnemies(dt);
  updateEnemyBullets(dt);
  updateParticles(dt);

  // Camera rotation
  camera.rotation.order = 'YXZ';
  camera.rotation.y = yaw;
  camera.rotation.x = pitch;

  // Gun sway
  gunGroup.position.x = 0.22 + Math.sin(bobTime*0.7)*0.005;
  gunGroup.position.y = -0.22 + Math.cos(bobTime)*0.005;

  renderer.render(scene, camera);
}

// ===== CONTROLS =====
document.addEventListener('keydown', e => {
  keys[e.code] = true;
  if (e.code === 'KeyR' && !reloading && ammo < maxAmmo && gameRunning) startReload();
});
document.addEventListener('keyup', e => { keys[e.code] = false; });

document.addEventListener('mousemove', e => {
  if (!gameRunning) return;
  const s = 0.002;
  yaw -= e.movementX * s;
  pitch -= e.movementY * s;
  pitch = Math.max(-0.6, Math.min(0.6, pitch));
});

document.addEventListener('mousedown', e => {
  if (!gameRunning) return;
  if (e.button === 0) shoot();
});

document.addEventListener('pointerlockchange', () => {
  if (!document.pointerLockElement && gameRunning && !gameOver) {
    // paused
  }
});

function requestLock() {
  document.getElementById('canvas').requestPointerLock();
}

function startGame() {
  overlay.style.display = 'none';
  initThree();
  gameRunning = true;
  requestLock();
  gameLoop();
}

function restartGame() {
  // Reset state
  enemies.forEach(e => { if(e.mesh) scene.remove(e.mesh); });
  enemyBullets.forEach(b => { if(b.mesh) scene.remove(b.mesh); });
  particles.forEach(p => { if(p.mesh) scene.remove(p.mesh); });
  enemies = []; enemyBullets = []; particles = []; trees = []; rocks = [];
  player.health = 100;
  ammo = 30;
  score = 0; kills = 0; wave = 1;
  gameOver = false;
  reloading = false;
  bobTime = 0; yaw = 0; pitch = 0;

  healthFill.style.width = '100%';
  ammoCount.textContent = maxAmmo;
  scoreVal.textContent = '0';
  gameover.style.display = 'none';

  // Clear scene
  while(scene.children.length > 0) scene.remove(scene.children[0]);
  camera.position.set(0, 1.7, 0);
  camera.add(gunGroup);
  scene.add(camera);
  buildWorld();
  gameRunning = true;
  requestLock();
  startWave(1);
}

document.getElementById('canvas').addEventListener('click', () => {
  if (gameRunning) requestLock();
});
</script>

</body>
</html>