<!DOCTYPE html>

<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>APEX RUSH — 3D Racing</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { background: #000; overflow: hidden; font-family: 'Courier New', monospace; }
  #canvas { display: block; width: 100vw; height: 100vh; }

#hud {
position: fixed; top: 0; left: 0; width: 100%; height: 100%;
pointer-events: none; z-index: 10;
}

#speedometer {
position: absolute; bottom: 30px; right: 40px;
background: rgba(0,0,0,0.75);
border: 2px solid #ff4400;
border-radius: 12px;
padding: 14px 22px;
color: #ff4400;
text-align: center;
backdrop-filter: blur(6px);
}
#speedometer .speed-val {
font-size: 52px; font-weight: 900; line-height: 1;
color: #fff; text-shadow: 0 0 16px #ff4400;
}
#speedometer .speed-label { font-size: 12px; letter-spacing: 4px; color: #ff4400; }

#gear-box {
position: absolute; bottom: 30px; right: 200px;
background: rgba(0,0,0,0.75);
border: 2px solid #ff4400;
border-radius: 12px;
padding: 14px 22px;
color: #ff4400;
text-align: center;
backdrop-filter: blur(6px);
}
#gear-box .gear-val { font-size: 52px; font-weight: 900; color: #fff; text-shadow: 0 0 16px #ffaa00; }
#gear-box .gear-label { font-size: 12px; letter-spacing: 4px; color: #ffaa00; }

#lap-info {
position: absolute; top: 20px; left: 50%; transform: translateX(-50%);
background: rgba(0,0,0,0.75);
border: 2px solid #ff4400;
border-radius: 12px;
padding: 10px 30px;
color: #fff;
text-align: center;
backdrop-filter: blur(6px);
font-size: 14px;
letter-spacing: 2px;
}
#lap-info span { color: #ff4400; font-weight: bold; font-size: 18px; }

#controls-hint {
position: absolute; bottom: 30px; left: 40px;
background: rgba(0,0,0,0.6);
border: 1px solid #333;
border-radius: 10px;
padding: 12px 18px;
color: #888;
font-size: 12px;
line-height: 1.8;
backdrop-filter: blur(4px);
}
#controls-hint b { color: #ff4400; }

#title-screen {
position: fixed; inset: 0; background: #000;
display: flex; flex-direction: column;
align-items: center; justify-content: center;
z-index: 100;
background: radial-gradient(ellipse at center, #1a0000 0%, #000 70%);
}
#title-screen h1 {
font-size: 80px; font-weight: 900; letter-spacing: 10px;
color: #fff; text-shadow: 0 0 40px #ff4400, 0 0 80px #ff2200;
margin-bottom: 8px;
animation: pulse 2s infinite;
}
#title-screen .sub {
font-size: 14px; letter-spacing: 8px; color: #ff4400; margin-bottom: 60px;
}
#title-screen button {
background: #ff4400; color: #fff; border: none;
padding: 18px 60px; font-size: 20px; font-weight: bold;
letter-spacing: 4px; cursor: pointer; border-radius: 4px;
text-transform: uppercase;
box-shadow: 0 0 30px #ff4400;
transition: all 0.2s;
pointer-events: all;
}
#title-screen button:hover { background: #ff6600; transform: scale(1.05); }

#rpm-bar {
position: absolute; bottom: 110px; right: 40px;
width: 230px; height: 10px;
background: #111; border-radius: 5px;
border: 1px solid #333; overflow: hidden;
}
#rpm-fill {
height: 100%; width: 0%;
background: linear-gradient(90deg, #ff4400, #ffcc00);
border-radius: 5px;
transition: width 0.05s;
}
#rpm-label {
position: absolute; bottom: 124px; right: 40px;
color: #555; font-size: 10px; letter-spacing: 3px;
}

#mini-map {
position: absolute; top: 20px; right: 20px;
width: 130px; height: 130px;
background: rgba(0,0,0,0.7);
border: 2px solid #ff4400;
border-radius: 50%;
overflow: hidden;
backdrop-filter: blur(4px);
}
#mini-map canvas { width: 130px; height: 130px; }

#crash-flash {
position: fixed; inset: 0;
background: rgba(255,50,0,0.3);
pointer-events: none;
opacity: 0;
z-index: 50;
transition: opacity 0.1s;
}

@keyframes pulse {
0%, 100% { text-shadow: 0 0 40px #ff4400, 0 0 80px #ff2200; }
50% { text-shadow: 0 0 60px #ff6600, 0 0 120px #ff4400; }
}

#nitro-bar-wrap {
position: absolute; bottom: 30px; left: 50%; transform: translateX(-50%);
text-align: center;
}
#nitro-label { color: #00ccff; font-size: 11px; letter-spacing: 4px; margin-bottom: 4px; }
#nitro-bar {
width: 180px; height: 8px;
background: #111; border-radius: 4px;
border: 1px solid #333; overflow: hidden;
}
#nitro-fill {
height: 100%; width: 100%;
background: linear-gradient(90deg, #00ccff, #0044ff);
border-radius: 4px;
transition: width 0.05s;
}
</style>

</head>
<body>

<div id="title-screen">
  <h1>APEX RUSH</h1>
  <div class="sub">3D CIRCUIT RACING</div>
  <button onclick="startGame()">START RACE</button>
  <div style="color:#444;font-size:12px;margin-top:30px;letter-spacing:2px;">
    W/S = THROTTLE/BRAKE &nbsp;|&nbsp; A/D = STEER &nbsp;|&nbsp; SHIFT = NITRO &nbsp;|&nbsp; R = RESET
  </div>
</div>

<canvas id="canvas"></canvas>

<div id="hud" style="display:none;">
  <div id="lap-info">LAP <span id="lap-num">1</span> / 3 &nbsp;&nbsp; TIME <span id="lap-time">0:00.000</span></div>
  <div id="mini-map"><canvas id="map-canvas" width="130" height="130"></canvas></div>
  <div id="speedometer">
    <div class="speed-val" id="speed-display">0</div>
    <div class="speed-label">KM/H</div>
  </div>
  <div id="gear-box">
    <div class="gear-val" id="gear-display">1</div>
    <div class="gear-label">GEAR</div>
  </div>
  <div id="rpm-label">RPM</div>
  <div id="rpm-bar"><div id="rpm-fill"></div></div>
  <div id="nitro-bar-wrap">
    <div id="nitro-label">⚡ NITRO</div>
    <div id="nitro-bar"><div id="nitro-fill"></div></div>
  </div>
  <div id="controls-hint">
    <b>W</b> Throttle &nbsp; <b>S</b> Brake<br>
    <b>A/D</b> Steer &nbsp; <b>SHIFT</b> Nitro<br>
    <b>R</b> Reset Car
  </div>
</div>
<div id="crash-flash"></div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

<script>
// ─────────────────────────────────────────────
//  SCENE SETUP
// ─────────────────────────────────────────────
const canvas = document.getElementById('canvas');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.setPixelRatio(window.devicePixelRatio);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x0a0010);
scene.fog = new THREE.FogExp2(0x0a0010, 0.008);

const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 800);

// ─────────────────────────────────────────────
//  LIGHTS
// ─────────────────────────────────────────────
scene.add(new THREE.AmbientLight(0x222244, 1.2));

const sun = new THREE.DirectionalLight(0xffeedd, 2.5);
sun.position.set(80, 120, 60);
sun.castShadow = true;
sun.shadow.mapSize.set(2048, 2048);
sun.shadow.camera.near = 1;
sun.shadow.camera.far = 500;
sun.shadow.camera.left = sun.shadow.camera.bottom = -150;
sun.shadow.camera.right = sun.shadow.camera.top = 150;
scene.add(sun);

// Track lights
function addTrackLight(x, y, z, color) {
  const l = new THREE.PointLight(color, 3, 60);
  l.position.set(x, y, z);
  scene.add(l);
  // Light pole
  const pole = new THREE.Mesh(
    new THREE.CylinderGeometry(0.15, 0.15, y, 6),
    new THREE.MeshStandardMaterial({ color: 0x444455, metalness: 0.8 })
  );
  pole.position.set(x, y / 2, z);
  scene.add(pole);
  const bulb = new THREE.Mesh(
    new THREE.SphereGeometry(0.5, 8, 8),
    new THREE.MeshStandardMaterial({ color: color, emissive: color, emissiveIntensity: 2 })
  );
  bulb.position.set(x, y, z);
  scene.add(bulb);
}

// ─────────────────────────────────────────────
//  TRACK DEFINITION (smooth oval circuit)
// ─────────────────────────────────────────────
const TRACK_WIDTH = 18;
const trackPoints = [];
const segments = 80;
// Complex figure-8 style track
for (let i = 0; i < segments; i++) {
  const t = (i / segments) * Math.PI * 2;
  const r1 = 120, r2 = 70;
  let x, z;
  // Lemniscate-like but closed oval with chicane variation
  if (t < Math.PI) {
    x = r1 * Math.cos(t) - 30;
    z = r2 * Math.sin(t);
  } else {
    x = r1 * 0.6 * Math.cos(t) + 40;
    z = r2 * 1.3 * Math.sin(t);
  }
  // Height variation (banking)
  const y = Math.sin(t * 2) * 3;
  trackPoints.push(new THREE.Vector3(x, y, z));
}
trackPoints.push(trackPoints[0].clone());

const trackCurve = new THREE.CatmullRomCurve3(trackPoints, true, 'catmullrom', 0.5);
const CURVE_POINTS = 300;
const curvePoints = trackCurve.getPoints(CURVE_POINTS);
const curveTangents = [];
for (let i = 0; i < CURVE_POINTS; i++) {
  curveTangents.push(trackCurve.getTangent(i / CURVE_POINTS));
}

// Build track mesh
function buildTrack() {
  const pts = trackCurve.getPoints(400);
  const trackShape = new THREE.Shape();
  trackShape.moveTo(-TRACK_WIDTH / 2, 0);
  trackShape.lineTo(TRACK_WIDTH / 2, 0);

  const positions = [];
  const normals = [];
  const uvs = [];
  const indices = [];

  const N = pts.length;
  for (let i = 0; i < N; i++) {
    const curr = pts[i];
    const next = pts[(i + 1) % N];
    const tan = new THREE.Vector3().subVectors(next, curr).normalize();
    const up = new THREE.Vector3(0, 1, 0);
    const right = new THREE.Vector3().crossVectors(tan, up).normalize();

    const left = curr.clone().addScaledVector(right, -TRACK_WIDTH / 2);
    const right2 = curr.clone().addScaledVector(right, TRACK_WIDTH / 2);

    positions.push(left.x, left.y, left.z, right2.x, right2.y, right2.z);
    normals.push(0, 1, 0, 0, 1, 0);
    uvs.push(0, i / N, 1, i / N);

    if (i < N - 1) {
      const a = i * 2, b = i * 2 + 1, c = (i + 1) * 2, d = (i + 1) * 2 + 1;
      indices.push(a, b, c, b, d, c);
    }
  }
  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
  geo.setAttribute('normal', new THREE.Float32BufferAttribute(normals, 3));
  geo.setAttribute('uv', new THREE.Float32BufferAttribute(uvs, 2));
  geo.setIndex(indices);
  geo.computeVertexNormals();

  const mat = new THREE.MeshStandardMaterial({
    color: 0x1a1a2e,
    roughness: 0.8,
    metalness: 0.1,
  });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.receiveShadow = true;
  scene.add(mesh);

  // White lane lines
  for (let side = -1; side <= 1; side += 2) {
    const linePts = pts.map(p => {
      const next = pts[(pts.indexOf(p) + 1) % N] || pts[0];
      const tan = new THREE.Vector3().subVectors(next, p).normalize();
      const right = new THREE.Vector3().crossVectors(tan, new THREE.Vector3(0,1,0)).normalize();
      return p.clone().addScaledVector(right, side * (TRACK_WIDTH / 2 - 0.5)).add(new THREE.Vector3(0, 0.05, 0));
    });
    const lineGeo = new THREE.BufferGeometry().setFromPoints(linePts);
    const lineMat = new THREE.LineBasicMaterial({ color: 0xffffff, linewidth: 2 });
    scene.add(new THREE.Line(lineGeo, lineMat));
  }

  // Center dashes
  for (let i = 0; i < N; i += 8) {
    const p = pts[i].clone().add(new THREE.Vector3(0, 0.06, 0));
    const p2 = pts[Math.min(i + 3, N - 1)].clone().add(new THREE.Vector3(0, 0.06, 0));
    const dashGeo = new THREE.BufferGeometry().setFromPoints([p, p2]);
    const dashMat = new THREE.LineBasicMaterial({ color: 0xffff00, linewidth: 1 });
    scene.add(new THREE.Line(dashGeo, dashMat));
  }

  // Barriers
  for (let side = -1; side <= 1; side += 2) {
    for (let i = 0; i < N; i += 3) {
      const p = pts[i];
      const next = pts[(i + 1) % N];
      const tan = new THREE.Vector3().subVectors(next, p).normalize();
      const right = new THREE.Vector3().crossVectors(tan, new THREE.Vector3(0,1,0)).normalize();
      const pos = p.clone().addScaledVector(right, side * (TRACK_WIDTH / 2 + 1));
      const geo = new THREE.BoxGeometry(2, 1.5, 0.4);
      const mat = new THREE.MeshStandardMaterial({
        color: side === -1 ? 0xff0000 : 0xffffff,
        roughness: 0.5
      });
      const barrier = new THREE.Mesh(geo, mat);
      barrier.position.copy(pos).add(new THREE.Vector3(0, 0.75, 0));
      barrier.lookAt(pos.clone().add(tan));
      barrier.castShadow = true;
      scene.add(barrier);
    }
  }

  // Track lights
  for (let i = 0; i < N; i += 25) {
    const p = pts[i];
    const col = [0xff4400, 0x00aaff, 0x00ff88][i % 3];
    addTrackLight(p.x, 18, p.z, col);
  }
}

buildTrack();

// ─────────────────────────────────────────────
//  ENVIRONMENT
// ─────────────────────────────────────────────
// Ground plane
const groundGeo = new THREE.PlaneGeometry(800, 800, 20, 20);
const groundMat = new THREE.MeshStandardMaterial({ color: 0x0d1117, roughness: 1 });
const ground = new THREE.Mesh(groundGeo, groundMat);
ground.rotation.x = -Math.PI / 2;
ground.position.y = -0.5;
ground.receiveShadow = true;
scene.add(ground);

// Stars
const starGeo = new THREE.BufferGeometry();
const starVerts = [];
for (let i = 0; i < 3000; i++) {
  starVerts.push((Math.random() - 0.5) * 800, Math.random() * 200 + 20, (Math.random() - 0.5) * 800);
}
starGeo.setAttribute('position', new THREE.Float32BufferAttribute(starVerts, 3));
const stars = new THREE.Points(starGeo, new THREE.PointsMaterial({ color: 0xffffff, size: 0.4 }));
scene.add(stars);

// Grandstand
function addGrandstand(x, z, rot) {
  const group = new THREE.Group();
  for (let row = 0; row < 5; row++) {
    for (let col = 0; col < 20; col++) {
      const seat = new THREE.Mesh(
        new THREE.BoxGeometry(1.2, 0.5, 0.8),
        new THREE.MeshStandardMaterial({ color: [0xff0000,0x0000ff,0xffff00,0x00ff00][Math.floor(Math.random()*4)] })
      );
      seat.position.set(col * 1.3 - 13, row * 1.2, row * 0.8);
      group.add(seat);
    }
  }
  group.position.set(x, 0, z);
  group.rotation.y = rot;
  scene.add(group);
}
addGrandstand(-80, 0, Math.PI / 4);
addGrandstand(80, 0, -Math.PI / 4);

// Decorative cones
function addCone(x, z) {
  const cone = new THREE.Mesh(
    new THREE.ConeGeometry(0.4, 1.2, 8),
    new THREE.MeshStandardMaterial({ color: 0xff6600 })
  );
  cone.position.set(x, 0.6, z);
  scene.add(cone);
}
for (let i = 0; i < 30; i++) {
  const p = curvePoints[Math.floor(i * CURVE_POINTS / 30)];
  addCone(p.x + (Math.random() - 0.5) * TRACK_WIDTH * 0.6, p.z + (Math.random() - 0.5) * 3);
}

// ─────────────────────────────────────────────
//  CAR
// ─────────────────────────────────────────────
const carGroup = new THREE.Group();
scene.add(carGroup);

// Body
const bodyGeo = new THREE.BoxGeometry(2.2, 0.6, 4.8);
const bodyMat = new THREE.MeshStandardMaterial({ color: 0xff2200, metalness: 0.6, roughness: 0.3 });
const body = new THREE.Mesh(bodyGeo, bodyMat);
body.position.y = 0.55;
body.castShadow = true;
carGroup.add(body);

// Cabin
const cabinGeo = new THREE.BoxGeometry(1.6, 0.55, 2.2);
const cabinMat = new THREE.MeshStandardMaterial({ color: 0x111122, metalness: 0.3, roughness: 0.2, transparent: true, opacity: 0.85 });
const cabin = new THREE.Mesh(cabinGeo, cabinMat);
cabin.position.set(0, 1.1, 0.1);
cabin.castShadow = true;
carGroup.add(cabin);

// Spoiler
const spoilerGeo = new THREE.BoxGeometry(2.4, 0.1, 0.5);
const spoilerMat = new THREE.MeshStandardMaterial({ color: 0x111111, metalness: 0.8 });
const spoiler = new THREE.Mesh(spoilerGeo, spoilerMat);
spoiler.position.set(0, 1.05, -2.1);
carGroup.add(spoiler);
// Spoiler supports
for (let s = -1; s <= 1; s += 2) {
  const sup = new THREE.Mesh(new THREE.BoxGeometry(0.1, 0.5, 0.1), spoilerMat);
  sup.position.set(s * 0.9, 0.8, -2.1);
  carGroup.add(sup);
}

// Headlights
for (let s = -1; s <= 1; s += 2) {
  const hl = new THREE.Mesh(
    new THREE.SphereGeometry(0.18, 8, 8),
    new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0xffffcc, emissiveIntensity: 3 })
  );
  hl.position.set(s * 0.7, 0.55, 2.35);
  carGroup.add(hl);
  const light = new THREE.SpotLight(0xffffff, 3, 30, Math.PI / 6);
  light.position.copy(hl.position);
  light.target.position.set(s * 0.7, 0, 10);
  carGroup.add(light);
  carGroup.add(light.target);
}

// Tail lights
for (let s = -1; s <= 1; s += 2) {
  const tl = new THREE.Mesh(
    new THREE.SphereGeometry(0.15, 8, 8),
    new THREE.MeshStandardMaterial({ color: 0xff0000, emissive: 0xff0000, emissiveIntensity: 2 })
  );
  tl.position.set(s * 0.7, 0.55, -2.35);
  carGroup.add(tl);
}

// Wheels
const wheelGeo = new THREE.CylinderGeometry(0.45, 0.45, 0.4, 16);
const wheelMat = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 0.9 });
const rimMat = new THREE.MeshStandardMaterial({ color: 0xcccccc, metalness: 0.9, roughness: 0.1 });
const wheels = [];
const wheelPositions = [
  [-1.15, 0.45, 1.5], [1.15, 0.45, 1.5],
  [-1.15, 0.45, -1.5], [1.15, 0.45, -1.5]
];
for (const [x, y, z] of wheelPositions) {
  const wg = new THREE.Group();
  const w = new THREE.Mesh(wheelGeo, wheelMat);
  w.rotation.z = Math.PI / 2;
  wg.add(w);
  // Rim
  const rim = new THREE.Mesh(new THREE.CylinderGeometry(0.28, 0.28, 0.42, 8), rimMat);
  rim.rotation.z = Math.PI / 2;
  wg.add(rim);
  wg.position.set(x, y, z);
  carGroup.add(wg);
  wheels.push(wg);
}

// Exhaust particles
const exhaustParticles = [];
const exhaustGeo = new THREE.SphereGeometry(0.08, 4, 4);
const exhaustMat = new THREE.MeshBasicMaterial({ color: 0xff6600, transparent: true });
for (let i = 0; i < 20; i++) {
  const p = new THREE.Mesh(exhaustGeo, exhaustMat.clone());
  p.visible = false;
  scene.add(p);
  exhaustParticles.push({ mesh: p, life: 0, vel: new THREE.Vector3() });
}

// ─────────────────────────────────────────────
//  PHYSICS STATE
// ─────────────────────────────────────────────
const physics = {
  pos: new THREE.Vector3(curvePoints[0].x, curvePoints[0].y + 0.5, curvePoints[0].z),
  vel: new THREE.Vector3(),
  heading: 0,
  angVel: 0,
  speed: 0,
  gear: 1,
  rpm: 800,
  nitro: 100,
  onGround: true,
};

const GEARS = [0, 0.015, 0.028, 0.042, 0.058, 0.072, 0.085];
const GEAR_LIMITS = [0, 60, 90, 130, 170, 210, 260];
const MAX_STEER = 0.045;
const FRICTION = 0.97;
const BRAKE_POWER = 0.94;
const GRAVITY = -0.015;
const BOUNCE_DAMPEN = 0.3;

let lapStart = Date.now();
let lapCount = 1;
let lastCheckpoint = 0;
let checkpointCooldown = 0;

// ─────────────────────────────────────────────
//  INPUT
// ─────────────────────────────────────────────
const keys = {};
document.addEventListener('keydown', e => { keys[e.code] = true; });
document.addEventListener('keyup', e => { keys[e.code] = false; });

// ─────────────────────────────────────────────
//  MINI MAP
// ─────────────────────────────────────────────
const mapCanvas = document.getElementById('map-canvas');
const mapCtx = mapCanvas.getContext('2d');
const MAP_SIZE = 130;
const MAP_SCALE = 0.45;

function drawMiniMap() {
  mapCtx.clearRect(0, 0, MAP_SIZE, MAP_SIZE);
  mapCtx.fillStyle = 'rgba(0,0,0,0.6)';
  mapCtx.fillRect(0, 0, MAP_SIZE, MAP_SIZE);

  // Track outline
  mapCtx.beginPath();
  mapCtx.strokeStyle = '#333';
  mapCtx.lineWidth = 6;
  for (let i = 0; i < curvePoints.length; i++) {
    const p = curvePoints[i];
    const mx = p.x * MAP_SCALE + MAP_SIZE / 2;
    const mz = p.z * MAP_SCALE + MAP_SIZE / 2;
    if (i === 0) mapCtx.moveTo(mx, mz);
    else mapCtx.lineTo(mx, mz);
  }
  mapCtx.closePath();
  mapCtx.stroke();

  mapCtx.beginPath();
  mapCtx.strokeStyle = '#ff4400';
  mapCtx.lineWidth = 2;
  for (let i = 0; i < curvePoints.length; i++) {
    const p = curvePoints[i];
    const mx = p.x * MAP_SCALE + MAP_SIZE / 2;
    const mz = p.z * MAP_SCALE + MAP_SIZE / 2;
    if (i === 0) mapCtx.moveTo(mx, mz);
    else mapCtx.lineTo(mx, mz);
  }
  mapCtx.closePath();
  mapCtx.stroke();

  // Car dot
  const cx = physics.pos.x * MAP_SCALE + MAP_SIZE / 2;
  const cz = physics.pos.z * MAP_SCALE + MAP_SIZE / 2;
  mapCtx.beginPath();
  mapCtx.arc(cx, cz, 4, 0, Math.PI * 2);
  mapCtx.fillStyle = '#fff';
  mapCtx.fill();
}

// ─────────────────────────────────────────────
//  FIND CLOSEST TRACK POINT
// ─────────────────────────────────────────────
function getClosestTrackPoint() {
  let minDist = Infinity, minIdx = 0;
  for (let i = 0; i < curvePoints.length; i++) {
    const d = physics.pos.distanceTo(curvePoints[i]);
    if (d < minDist) { minDist = d; minIdx = i; }
  }
  return { idx: minIdx, dist: minDist, point: curvePoints[minIdx] };
}

// ─────────────────────────────────────────────
//  CRASH FLASH
// ─────────────────────────────────────────────
function triggerCrash() {
  const flash = document.getElementById('crash-flash');
  flash.style.opacity = '1';
  setTimeout(() => flash.style.opacity = '0', 150);
  physics.vel.multiplyScalar(-0.3);
  physics.angVel *= -0.5;
  physics.speed *= 0.2;
}

// ─────────────────────────────────────────────
//  PHYSICS UPDATE
// ─────────────────────────────────────────────
function updatePhysics(dt) {
  if (keys['KeyR']) resetCar();

  const throttle = keys['KeyW'] || keys['ArrowUp'];
  const brake    = keys['KeyS'] || keys['ArrowDown'];
  const left     = keys['KeyA'] || keys['ArrowLeft'];
  const right    = keys['KeyD'] || keys['ArrowRight'];
  const nitro    = (keys['ShiftLeft'] || keys['ShiftRight']) && physics.nitro > 0;

  // Steering
  const steerFactor = Math.max(0.2, 1 - physics.speed / 300);
  if (left)  physics.angVel += 0.003 * steerFactor;
  if (right) physics.angVel -= 0.003 * steerFactor;
  physics.angVel *= 0.82;
  physics.heading += physics.angVel * (physics.speed > 1 ? 1 : 0);

  // Direction vector
  const dir = new THREE.Vector3(Math.sin(physics.heading), 0, Math.cos(physics.heading));

  // Auto gear shift
  for (let g = GEARS.length - 1; g >= 1; g--) {
    if (physics.speed >= GEAR_LIMITS[g - 1]) { physics.gear = g; break; }
  }

  // Throttle
  let accel = 0;
  if (throttle) accel = GEARS[physics.gear];
  if (nitro)   { accel *= 2.2; physics.nitro = Math.max(0, physics.nitro - 0.5); }
  if (brake)   physics.vel.multiplyScalar(BRAKE_POWER);

  physics.vel.addScaledVector(dir, accel);

  // Friction
  if (!throttle) physics.vel.multiplyScalar(FRICTION);

  // Lateral damping (simulates grip)
  const lateral = new THREE.Vector3(-Math.cos(physics.heading), 0, Math.sin(physics.heading));
  const lateralSpeed = physics.vel.dot(lateral);
  physics.vel.addScaledVector(lateral, -lateralSpeed * 0.7);

  // Speed
  physics.speed = physics.vel.length() * 200;
  physics.speed = Math.min(physics.speed, 260);

  // RPM simulation
  const targetRpm = 800 + (physics.speed / GEAR_LIMITS[physics.gear]) * 7000;
  physics.rpm += (targetRpm - physics.rpm) * 0.1;
  physics.rpm = Math.max(800, Math.min(physics.rpm, 8000));

  // Nitro recharge
  if (!nitro) physics.nitro = Math.min(100, physics.nitro + 0.08);

  // Move
  physics.pos.add(physics.vel);

  // Gravity / ground
  const track = getClosestTrackPoint();
  const groundY = track.point.y + 0.5;
  if (physics.pos.y > groundY) {
    physics.vel.y += GRAVITY;
  } else {
    physics.pos.y = groundY;
    physics.vel.y = 0;
    physics.onGround = true;
  }

  // Barrier collision
  if (track.dist > TRACK_WIDTH / 2 + 1) {
    triggerCrash();
    // Push back onto track
    const toTrack = new THREE.Vector3().subVectors(track.point, physics.pos).normalize();
    physics.pos.addScaledVector(toTrack, track.dist - TRACK_WIDTH / 2);
  }

  // Lap detection (checkpoint at idx=0)
  const currIdx = track.idx;
  if (checkpointCooldown > 0) checkpointCooldown--;
  if (currIdx < 10 && lastCheckpoint > CURVE_POINTS - 10 && checkpointCooldown === 0) {
    lapCount++;
    lapStart = Date.now();
    checkpointCooldown = 120;
    if (lapCount > 3) lapCount = 1;
    document.getElementById('lap-num').textContent = lapCount;
  }
  lastCheckpoint = currIdx;
}

// ─────────────────────────────────────────────
//  CAR VISUAL UPDATE
// ─────────────────────────────────────────────
function updateCarVisuals() {
  carGroup.position.copy(physics.pos);
  carGroup.rotation.y = physics.heading;

  // Banking on curves
  carGroup.rotation.z = -physics.angVel * 8;

  // Wheel spin
  const spinRate = physics.speed * 0.002;
  for (const w of wheels) w.rotation.x -= spinRate;

  // Front wheel steer
  if (physics.angVel > 0.001) {
    wheels[0].rotation.y = 0.3;
    wheels[1].rotation.y = 0.3;
  } else if (physics.angVel < -0.001) {
    wheels[0].rotation.y = -0.3;
    wheels[1].rotation.y = -0.3;
  } else {
    wheels[0].rotation.y *= 0.8;
    wheels[1].rotation.y *= 0.8;
  }

  // Exhaust particles
  if (physics.speed > 5) {
    const exhaustPos = carGroup.localToWorld(new THREE.Vector3(0, 0.4, -2.4));
    const freeParticle = exhaustParticles.find(p => p.life <= 0);
    if (freeParticle) {
      freeParticle.mesh.position.copy(exhaustPos);
      freeParticle.mesh.visible = true;
      const dir = new THREE.Vector3(
        (Math.random() - 0.5) * 0.05,
        Math.random() * 0.03,
        -Math.random() * 0.1
      ).applyEuler(carGroup.rotation);
      freeParticle.vel.copy(dir);
      freeParticle.life = 30;
      freeParticle.mesh.material.opacity = 0.8;
      freeParticle.mesh.scale.setScalar(1);
    }
  }
  for (const p of exhaustParticles) {
    if (p.life > 0) {
      p.mesh.position.add(p.vel);
      p.life--;
      p.mesh.material.opacity = p.life / 30 * 0.8;
      p.mesh.scale.setScalar(1 + (30 - p.life) / 30 * 3);
      if (p.life <= 0) p.mesh.visible = false;
    }
  }
}

// ─────────────────────────────────────────────
//  CAMERA
// ─────────────────────────────────────────────
const camOffset = new THREE.Vector3(0, 5, -14);
const camTarget = new THREE.Vector3();
let camPos = new THREE.Vector3();

function updateCamera() {
  const offset = camOffset.clone().applyEuler(new THREE.Euler(0, physics.heading, 0));
  const desired = physics.pos.clone().add(offset);
  camPos.lerp(desired, 0.08);
  camera.position.copy(camPos);
  camTarget.lerp(physics.pos.clone().add(new THREE.Vector3(0, 1.5, 0)), 0.12);
  camera.lookAt(camTarget);
}

// ─────────────────────────────────────────────
//  HUD UPDATE
// ─────────────────────────────────────────────
function updateHUD() {
  document.getElementById('speed-display').textContent = Math.round(physics.speed);
  document.getElementById('gear-display').textContent = physics.gear;
  document.getElementById('rpm-fill').style.width = (physics.rpm / 8000 * 100) + '%';
  document.getElementById('nitro-fill').style.width = physics.nitro + '%';

  const elapsed = (Date.now() - lapStart) / 1000;
  const mins = Math.floor(elapsed / 60);
  const secs = (elapsed % 60).toFixed(3).padStart(6, '0');
  document.getElementById('lap-time').textContent = `${mins}:${secs}`;
}

// ─────────────────────────────────────────────
//  RESET CAR
// ─────────────────────────────────────────────
function resetCar() {
  const p = curvePoints[0];
  physics.pos.set(p.x, p.y + 0.5, p.z);
  physics.vel.set(0, 0, 0);
  physics.heading = 0;
  physics.angVel = 0;
  physics.speed = 0;
  physics.gear = 1;
  physics.rpm = 800;
}

// ─────────────────────────────────────────────
//  MAIN LOOP
// ─────────────────────────────────────────────
let running = false;
let frameId;

function gameLoop() {
  if (!running) return;
  frameId = requestAnimationFrame(gameLoop);

  updatePhysics(1 / 60);
  updateCarVisuals();
  updateCamera();
  updateHUD();
  drawMiniMap();

  // Animate stars
  stars.rotation.y += 0.0001;

  renderer.render(scene, camera);
}

function startGame() {
  document.getElementById('title-screen').style.display = 'none';
  document.getElementById('hud').style.display = 'block';
  resetCar();
  running = true;
  lapStart = Date.now();
  gameLoop();
}

// ─────────────────────────────────────────────
//  RESIZE
// ─────────────────────────────────────────────
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// Initial camera position
camera.position.set(curvePoints[0].x, curvePoints[0].y + 8, curvePoints[0].z - 20);
camera.lookAt(curvePoints[0].x, curvePoints[0].y, curvePoints[0].z);
camPos.copy(camera.position);

// Render title background
(function titleLoop() {
  if (running) return;
  requestAnimationFrame(titleLoop);
  stars.rotation.y += 0.0002;
  renderer.render(scene, camera);
})();
</script>

</body>
</html>