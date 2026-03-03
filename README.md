<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Drift Circuit Racer (Single HTML)</title>
  <style>
    html, body { margin:0; height:100%; overflow:hidden; background:#070a12; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; }
    canvas { display:block; }
    #hud { position:fixed; inset:0; pointer-events:none; color:rgba(255,255,255,0.92); text-shadow:0 2px 14px rgba(0,0,0,.7); }
    #panel {
      position:absolute; left:12px; top:12px;
      background: rgba(10,14,22,0.58);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 14px;
      padding: 10px 12px;
      width: min(420px, calc(100vw - 24px));
      backdrop-filter: blur(10px);
    }
    #title { font-weight:900; font-size:15px; margin-bottom:6px; letter-spacing: .2px; }
    #stats { font-size:13px; opacity:.95; line-height:1.35; }
    #hint {
      position:absolute; left:50%; top:55%;
      transform: translate(-50%,-50%);
      text-align:center;
      background: rgba(0,0,0,0.42);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 16px;
      padding: 10px 14px;
      backdrop-filter: blur(12px);
      max-width: 92vw;
      transition: opacity .18s ease;
    }
    #hint.hidden { opacity:0; }
    #barWrap {
      margin-top:8px;
      height:8px; border-radius:999px; overflow:hidden;
      background: rgba(255,255,255,0.10);
      border: 1px solid rgba(255,255,255,0.10);
    }
    #bar {
      height:100%; width:0%;
      background: linear-gradient(90deg, rgba(120,200,255,0.95), rgba(255,140,210,0.95));
      box-shadow: 0 0 18px rgba(120,200,255,0.30);
    }
    #minimap {
      position:absolute; right:12px; top:12px;
      width: 190px; height: 190px;
      background: rgba(10,14,22,0.58);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 14px;
      backdrop-filter: blur(10px);
      overflow:hidden;
    }
    #minimap canvas { width:100%; height:100%; display:block; }
  </style>
</head>
<body>
  <div id="hud">
    <div id="panel">
      <div id="title">Drift Circuit Racer</div>
      <div id="stats">Loading…</div>
      <div id="barWrap"><div id="bar"></div></div>
      <div style="margin-top:8px; font-size:12px; opacity:0.75;">
        <b>W</b> throttle • <b>S</b> brake/reverse • <b>A/D</b> steer • <b>Space</b> handbrake drift • <b>R</b> reset
      </div>
    </div>

    <div id="minimap"><canvas id="map"></canvas></div>

    <div id="hint">
      <div style="font-weight:900; margin-bottom:6px;">Click to start</div>
      Finish <b>3 laps</b>. Use <b>Space</b> to break rear traction and drift corners.
    </div>
  </div>

  <script type="module">
    import * as THREE from "https://unpkg.com/three@0.160.0/build/three.module.js";

    // =========================
    // Small utilities
    // =========================
    const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
    const lerp = (a, b, t) => a + (b - a) * t;
    const smooth = (t) => t * t * (3 - 2 * t);
    const wrapPI = (a) => { while (a > Math.PI) a -= Math.PI*2; while (a < -Math.PI) a += Math.PI*2; return a; };

    // deterministic-ish random (so the track always looks the same)
    function rand01(seed) { const x = Math.sin(seed * 999.123) * 10000; return x - Math.floor(x); }

    // =========================
    // Three.js setup
    // =========================
    const scene = new THREE.Scene();
    scene.fog = new THREE.FogExp2(0x070a12, 0.015);

    const camera = new THREE.PerspectiveCamera(68, innerWidth / innerHeight, 0.1, 3500);
    const renderer = new THREE.WebGLRenderer({ antialias:true, powerPreference:"high-performance" });
    renderer.setSize(innerWidth, innerHeight);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    document.body.appendChild(renderer.domElement);

    // HUD
    const statsEl = document.getElementById("stats");
    const barEl = document.getElementById("bar");
    const hintEl = document.getElementById("hint");

    // Minimap
    const mapCanvas = document.getElementById("map");
    const mapCtx = mapCanvas.getContext("2d");
    function resizeMap() {
      const dpr = Math.min(devicePixelRatio, 2);
      const box = mapCanvas.getBoundingClientRect();
      mapCanvas.width = Math.floor(box.width * dpr);
      mapCanvas.height = Math.floor(box.height * dpr);
      mapCtx.setTransform(dpr, 0, 0, dpr, 0, 0);
    }

    // Sky gradient
    const sky = new THREE.Mesh(
      new THREE.SphereGeometry(1400, 32, 16),
      new THREE.ShaderMaterial({
        side: THREE.BackSide,
        uniforms: {
          topColor: { value: new THREE.Color(0x121a3a) },
          bottomColor: { value: new THREE.Color(0x04050a) },
          offset: { value: 16.0 },
          exponent: { value: 0.9 }
        },
        vertexShader: `
          varying vec3 vWorldPosition;
          void main(){
            vec4 wp = modelMatrix * vec4(position, 1.0);
            vWorldPosition = wp.xyz;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }`,
        fragmentShader: `
          uniform vec3 topColor;
          uniform vec3 bottomColor;
          uniform float offset;
          uniform float exponent;
          varying vec3 vWorldPosition;
          void main(){
            float h = normalize(vWorldPosition + vec3(0.0, offset, 0.0)).y;
            float t = pow(max(h, 0.0), exponent);
            gl_FragColor = vec4(mix(bottomColor, topColor, t), 1.0);
          }`
      })
    );
    scene.add(sky);

    // Lights
    scene.add(new THREE.HemisphereLight(0x9fd3ff, 0x0a0c14, 0.55));

    const sun = new THREE.DirectionalLight(0xffffff, 1.15);
    sun.position.set(80, 95, 40);
    sun.castShadow = true;
    sun.shadow.mapSize.set(2048, 2048);
    sun.shadow.camera.near = 1;
    sun.shadow.camera.far = 420;
    sun.shadow.camera.left = -140;
    sun.shadow.camera.right = 140;
    sun.shadow.camera.top = 140;
    sun.shadow.camera.bottom = -140;
    sun.shadow.bias = -0.00006;
    scene.add(sun);

    const neonA = new THREE.PointLight(0x66ccff, 1.0, 70, 2);
    neonA.position.set(-60, 8, -20);
    scene.add(neonA);

    const neonB = new THREE.PointLight(0xff77cc, 1.0, 70, 2);
    neonB.position.set(65, 8, 35);
    scene.add(neonB);

    // Ground
    const ground = new THREE.Mesh(
      new THREE.PlaneGeometry(900, 900),
      new THREE.MeshStandardMaterial({ color: 0x0b0f1a, roughness: 1.0, metalness: 0.0 })
    );
    ground.rotation.x = -Math.PI / 2;
    ground.receiveShadow = true;
    scene.add(ground);

    // =========================
    // Track: CatmullRom centerline → ribbon mesh, plus distance queries
    // =========================
    const TRACK_WIDTH = 14.5;

    // Hand-shaped but smoothed “proper” circuit (varied corners)
    const ctrl = [
      new THREE.Vector3(  0,0, -90),
      new THREE.Vector3( 55,0, -70),
      new THREE.Vector3( 95,0, -15),
      new THREE.Vector3( 70,0,  50),
      new THREE.Vector3( 15,0,  85),
      new THREE.Vector3(-55,0,  70),
      new THREE.Vector3(-95,0,  10),
      new THREE.Vector3(-70,0, -55),
    ];

    // Add small deterministic “designer” offsets so it feels less symmetric.
    for (let i = 0; i < ctrl.length; i++) {
      const n = (rand01(i+3) - 0.5) * 12;
      ctrl[i].x += n * 0.9;
      ctrl[i].z += n * 1.0;
    }

    const centerCurve = new THREE.CatmullRomCurve3(ctrl, true, "centripetal", 0.5);

    // Pre-sample for: mesh, minimap, nearest distance queries
    const SAMPLES = 1400;
    const samples = new Array(SAMPLES);
    const tangents = new Array(SAMPLES);
    for (let i = 0; i < SAMPLES; i++) {
      const t = i / SAMPLES;
      samples[i] = centerCurve.getPointAt(t);
      const tan = centerCurve.getTangentAt(t);
      tangents[i] = new THREE.Vector3(tan.x, 0, tan.z).normalize();
    }

    // Track ribbon geometry
    const pos = [];
    const uv = [];
    const idx = [];
    for (let i = 0; i <= SAMPLES; i++) {
      const j = i % SAMPLES;
      const p = samples[j];
      const t = tangents[j];
      const right = new THREE.Vector3(-t.z, 0, t.x);
      const leftP = p.clone().addScaledVector(right, -TRACK_WIDTH/2);
      const rightP = p.clone().addScaledVector(right,  TRACK_WIDTH/2);

      pos.push(leftP.x, 0.03, leftP.z, rightP.x, 0.03, rightP.z);
      uv.push(0, i * 0.06, 1, i * 0.06);
    }
    for (let i = 0; i < SAMPLES; i++) {
      const i0 = i*2, i1 = i*2+1, i2 = i*2+2, i3 = i*2+3;
      idx.push(i0, i2, i1, i2, i3, i1);
    }
    const trackGeo = new THREE.BufferGeometry();
    trackGeo.setAttribute("position", new THREE.Float32BufferAttribute(pos, 3));
    trackGeo.setAttribute("uv", new THREE.Float32BufferAttribute(uv, 2));
    trackGeo.setIndex(idx);
    trackGeo.computeVertexNormals();

    const trackMat = new THREE.MeshStandardMaterial({
      color: 0x141a25,
      roughness: 0.93,
      metalness: 0.05,
      emissive: 0x05070c,
      emissiveIntensity: 0.28
    });

    const track = new THREE.Mesh(trackGeo, trackMat);
    track.receiveShadow = true;
    scene.add(track);

    // Curbs (inner + outer) for visual cues
    function buildCurb(sideSign, colorHex) {
      const pts = [];
      for (let i = 0; i < 900; i++) {
        const t = i / 900;
        const p = centerCurve.getPointAt(t);
        const tan = centerCurve.getTangentAt(t);
        const tt = new THREE.Vector3(tan.x, 0, tan.z).normalize();
        const right = new THREE.Vector3(-tt.z, 0, tt.x);
        pts.push(p.clone().addScaledVector(right, sideSign * (TRACK_WIDTH/2 + 0.9)).setY(0.06));
      }
      const curve = new THREE.CatmullRomCurve3(pts, true, "centripetal");
      const geo = new THREE.TubeGeometry(curve, 1400, 0.26, 10, true);
      const mat = new THREE.MeshStandardMaterial({
        color: colorHex,
        roughness: 0.45,
        metalness: 0.35,
        emissive: colorHex,
        emissiveIntensity: 0.08
      });
      const m = new THREE.Mesh(geo, mat);
      m.castShadow = true;
      m.receiveShadow = true;
      scene.add(m);
    }
    buildCurb(+1, 0xff77cc);
    buildCurb(-1, 0x66ccff);

    // Start line + gate based on t=0
    const startT = 0.0;
    const startP = centerCurve.getPointAt(startT);
    const startTan = centerCurve.getTangentAt(startT);
    const startFwd = new THREE.Vector3(startTan.x, 0, startTan.z).normalize();

    const startLine = new THREE.Mesh(
      new THREE.PlaneGeometry(TRACK_WIDTH, 2.2),
      new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0.8, metalness: 0.1, emissive: 0x222222, emissiveIntensity: 0.35 })
    );
    startLine.rotation.x = -Math.PI / 2;
    startLine.position.copy(startP).addScaledVector(startFwd, 2.0);
    startLine.position.y = 0.035;
    scene.add(startLine);

    const gate = new THREE.Group();
    const postMat = new THREE.MeshStandardMaterial({ color: 0x0e1220, roughness: 0.45, metalness: 0.65, emissive: 0x0b0f1a, emissiveIntensity: 0.45 });
    const postGeo = new THREE.BoxGeometry(0.85, 5.8, 0.85);
    const beamGeo = new THREE.BoxGeometry(TRACK_WIDTH + 5, 0.7, 1.0);
    const postL = new THREE.Mesh(postGeo, postMat);
    const postR = new THREE.Mesh(postGeo, postMat);
    const beam  = new THREE.Mesh(beamGeo, postMat);
    postL.position.set(-TRACK_WIDTH/2 - 1.2, 2.9, 0);
    postR.position.set( TRACK_WIDTH/2 + 1.2, 2.9, 0);
    beam.position.set(0, 5.4, 0);
    gate.add(postL, postR, beam);
    gate.position.copy(startLine.position).addScaledVector(startFwd, -0.2);
    gate.lookAt(gate.position.clone().add(startFwd));
    scene.add(gate);

    const gateLight = new THREE.PointLight(0x66ccff, 1.6, 26, 2);
    gateLight.position.copy(gate.position).add(new THREE.Vector3(0, 5.2, 0));
    scene.add(gateLight);

    // Distance-to-track query (fast nearest sample)
    function nearestSampleIndex(x, z) {
      // quick guess by angle-ish is not reliable for irregular track; brute over small window around last index
      // we keep last idx and search locally for speed.
      return 0;
    }
    const nearestCache = { idx: 0 };
    function trackQuery(x, z) {
      // search around last idx (local) + fallback wider if needed
      let bestI = nearestCache.idx;
      let bestD2 = Infinity;

      const px = x, pz = z;
      const N = SAMPLES;

      const searchWindow = 80;
      for (let k = -searchWindow; k <= searchWindow; k++) {
        const i = (nearestCache.idx + k + N) % N;
        const p = samples[i];
        const dx = px - p.x, dz = pz - p.z;
        const d2 = dx*dx + dz*dz;
        if (d2 < bestD2) { bestD2 = d2; bestI = i; }
      }

      // if still far (teleport / reset), do a wider scan once
      if (bestD2 > 900) {
        bestD2 = Infinity;
        for (let i = 0; i < N; i += 2) {
          const p = samples[i];
          const dx = px - p.x, dz = pz - p.z;
          const d2 = dx*dx + dz*dz;
          if (d2 < bestD2) { bestD2 = d2; bestI = i; }
        }
      }

      nearestCache.idx = bestI;
      const dist = Math.sqrt(bestD2);
      const onTrack = dist <= TRACK_WIDTH * 0.5;
      const progress = bestI / N; // 0..1 around loop
      const tangent = tangents[bestI];
      return { dist, onTrack, progress, tangent };
    }

    // =========================
    // Car model (clean + “actually works”)
    // A lightweight bicycle model + lateral tire friction & handbrake rear grip reduction.
    // =========================
    const car = new THREE.Group();

    const bodyMat = new THREE.MeshStandardMaterial({
      color: 0x2aa8ff, roughness: 0.22, metalness: 0.78,
      emissive: 0x071018, emissiveIntensity: 0.25
    });
    const glassMat = new THREE.MeshStandardMaterial({
      color: 0x0b1a2a, roughness: 0.05, metalness: 0.15,
      transparent:true, opacity:0.55, emissive: 0x0b2a44, emissiveIntensity: 0.6
    });
    const trimMat = new THREE.MeshStandardMaterial({ color: 0x0a0c12, roughness: 0.55, metalness: 0.35 });

    const body = new THREE.Mesh(new THREE.BoxGeometry(2.1, 0.55, 4.2), bodyMat);
    body.position.y = 0.55; body.castShadow = true; car.add(body);
    const hood = new THREE.Mesh(new THREE.BoxGeometry(2.0, 0.35, 1.6), bodyMat);
    hood.position.set(0, 0.72, 1.05); hood.castShadow = true; car.add(hood);
    const roof = new THREE.Mesh(new THREE.BoxGeometry(1.7, 0.35, 1.4), bodyMat);
    roof.position.set(0, 1.05, -0.2); roof.castShadow = true; car.add(roof);
    const glass = new THREE.Mesh(new THREE.BoxGeometry(1.55, 0.40, 1.25), glassMat);
    glass.position.set(0, 1.02, -0.25); car.add(glass);

    const splitter = new THREE.Mesh(new THREE.BoxGeometry(2.25, 0.10, 0.55), trimMat);
    splitter.position.set(0, 0.32, 2.0); splitter.castShadow = true; car.add(splitter);
    const spoiler = new THREE.Mesh(new THREE.BoxGeometry(1.7, 0.12, 0.35), trimMat);
    spoiler.position.set(0, 1.15, -2.0); spoiler.castShadow = true; car.add(spoiler);

    const wheelGeo = new THREE.CylinderGeometry(0.42, 0.42, 0.28, 18);
    const wheelMat = new THREE.MeshStandardMaterial({ color: 0x0b0d12, roughness: 0.75, metalness: 0.25 });
    const rimMat = new THREE.MeshStandardMaterial({ color: 0x9fb3c8, roughness: 0.35, metalness: 0.75, emissive: 0x0a0a0a, emissiveIntensity: 0.25 });

    function makeWheel() {
      const w = new THREE.Group();
      const tire = new THREE.Mesh(wheelGeo, wheelMat); tire.rotation.z = Math.PI/2; tire.castShadow = true;
      const rim  = new THREE.Mesh(new THREE.CylinderGeometry(0.24, 0.24, 0.285, 12), rimMat); rim.rotation.z = Math.PI/2; rim.castShadow = true;
      w.add(tire, rim);
      return w;
    }
    const wFL = makeWheel(), wFR = makeWheel(), wRL = makeWheel(), wRR = makeWheel();
    wFL.position.set(-1.05, 0.40,  1.35);
    wFR.position.set( 1.05, 0.40,  1.35);
    wRL.position.set(-1.05, 0.40, -1.35);
    wRR.position.set( 1.05, 0.40, -1.35);
    car.add(wFL, wFR, wRL, wRR);

    const headMat = new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0x66ccff, emissiveIntensity: 2.0, roughness: 0.2, metalness: 0.1 });
    const tailMat = new THREE.MeshStandardMaterial({ color: 0xff88cc, emissive: 0xff44aa, emissiveIntensity: 2.2, roughness: 0.2, metalness: 0.1 });
    const headL = new THREE.Mesh(new THREE.BoxGeometry(0.35, 0.18, 0.12), headMat);
    const headR = headL.clone();
    headL.position.set(-0.65, 0.62, 2.15);
    headR.position.set( 0.65, 0.62, 2.15);
    car.add(headL, headR);

    const tailL = new THREE.Mesh(new THREE.BoxGeometry(0.35, 0.16, 0.12), tailMat);
    const tailR = tailL.clone();
    tailL.position.set(-0.65, 0.62, -2.15);
    tailR.position.set( 0.65, 0.62, -2.15);
    car.add(tailL, tailR);

    const under = new THREE.PointLight(0x66ccff, 0.9, 8, 2);
    under.position.set(0, 0.25, 0);
    car.add(under);

    car.castShadow = true;
    scene.add(car);

    // =========================
    // Car state + physics params
    // =========================
    const carState = {
      x: 0, z: 0,
      yaw: 0,                 // radians
      vx: 0, vz: 0,            // world velocity
      steer: 0,                // -1..1
      laps: 0, lapsToWin: 3,
      lapStartMs: performance.now(),
      bestLap: Infinity,
      driftScore: 0,
      totalScore: 0,
      lastGateSide: 0,
      startedLapCounting: false,
      finished: false
    };

    // Vehicle parameters (arcade but coherent)
    const V = {
      mass: 1200,
      wheelbase: 2.55,
      cgToFront: 1.25,
      cgToRear: 1.30,
      maxSteerRad: 0.55,          // ~31 degrees
      engineForce: 9000,          // N-ish
      brakeForce: 11000,
      reverseForce: 4500,
      rollingRes: 30,             // N per m/s
      aeroDrag: 0.42,             // N per (m/s)^2
      // tire lateral “stiffness” (bigger = more grip; we clamp forces)
      cornerStiffFront: 9.0,
      cornerStiffRear: 10.5
    };

    // Start pose
    function resetCar() {
      const p = startP.clone().addScaledVector(startFwd, -9);
      carState.x = p.x; carState.z = p.z;
      carState.yaw = Math.atan2(startFwd.x, startFwd.z);
      carState.vx = 0; carState.vz = 0;
      carState.steer = 0;
      carState.laps = 0;
      carState.lapStartMs = performance.now();
      carState.bestLap = Infinity;
      carState.driftScore = 0;
      carState.totalScore = 0;
      carState.finished = false;
      carState.startedLapCounting = false;
      carState.lastGateSide = 0;
      nearestCache.idx = 0;
      applyCarTransform();
      hintEl.classList.add("hidden");
    }

    function applyCarTransform() {
      car.position.set(carState.x, 0, carState.z);
      car.rotation.y = carState.yaw;
    }

    resetCar();

    // Gate plane for lap counting
    const gatePoint = startLine.position.clone();
    const gateNormal = startFwd.clone().normalize();
    function gateSide(x, z) {
      return (x - gatePoint.x) * gateNormal.x + (z - gatePoint.z) * gateNormal.z;
    }

    // =========================
    // Input
    // =========================
    const keys = new Set();
    addEventListener("keydown", (e) => keys.add(e.code));
    addEventListener("keyup", (e) => keys.delete(e.code));

    let started = false;
    addEventListener("click", () => { started = true; hintEl.classList.add("hidden"); });

    // =========================
    // Physics update (bicycle model + drifting)
    // =========================
    function updateCar(dt) {
      if (keys.has("KeyR")) { resetCar(); return; }
      if (carState.finished) return;

      const throttle = (keys.has("KeyW") ? 1 : 0);
      const brake = (keys.has("KeyS") ? 1 : 0);
      const steerInput = (keys.has("KeyD") ? 1 : 0) - (keys.has("KeyA") ? 1 : 0);
      const handbrake = keys.has("Space");

      // steering smoothing
      const steerTarget = steerInput;
      const steerRate = 10.0;
      carState.steer = lerp(carState.steer, steerTarget, 1 - Math.pow(0.0007, dt * steerRate));
      carState.steer = clamp(carState.steer, -1, 1);

      const steerAngle = carState.steer * V.maxSteerRad;

      // Convert velocity to vehicle local space
      const sin = Math.sin(carState.yaw), cos = Math.cos(carState.yaw);
      const vLocalX =  cos * carState.vx - sin * carState.vz; // forward
      const vLocalZ =  sin * carState.vx + cos * carState.vz; // right (lateral)

      const speed = Math.hypot(vLocalX, vLocalZ);

      // Track / offroad affects grip + power
      const tq = trackQuery(carState.x, carState.z);
      const onTrack = tq.onTrack;
      const offroad = !onTrack;

      // Engine / brake longitudinal force (local forward axis)
      let Fx = 0;

      if (throttle > 0) Fx += V.engineForce * (offroad ? 0.65 : 1.0);
      if (brake > 0) {
        // If moving forward, brake; else reverse
        if (vLocalX > 1.5) Fx -= V.brakeForce;
        else Fx -= V.reverseForce;
      }

      // Resistances
      Fx -= V.rollingRes * vLocalX;
      Fx -= V.aeroDrag * vLocalX * Math.abs(vLocalX);

      // Lateral tire forces (front & rear), using slip angle approximation:
      // alpha_f = atan((v_lat + a*r)/|v_long|) - delta
      // alpha_r = atan((v_lat - b*r)/|v_long|)
      // We'll solve yaw rate r from kinematics: r ≈ v_long * tan(delta) / wheelbase (bicycle)
      const vLong = vLocalX;
      const vLat  = vLocalZ;

      const absVLong = Math.max(2.0, Math.abs(vLong)); // avoid div by 0
      const yawRate = (vLong / V.wheelbase) * Math.tan(steerAngle);

      const alphaF = Math.atan2(vLat + V.cgToFront * yawRate, absVLong) - steerAngle;
      const alphaR = Math.atan2(vLat - V.cgToRear  * yawRate, absVLong);

      // Grip scaling: handbrake reduces REAR grip strongly (drift), offroad reduces both
      const gripRoad = offroad ? 0.55 : 1.0;
      const gripRearHB = handbrake ? 0.35 : 1.0;

      const Cf = V.cornerStiffFront * gripRoad;
      const Cr = V.cornerStiffRear * gripRoad * gripRearHB;

      // Lateral forces (local): Fy = -C * alpha, then clamp to friction-ish limit
      // A simple friction cap based on speed & surface
      const baseLatCap = (offroad ? 5200 : 7600);
      const capF = baseLatCap * (0.90 + 0.30 * clamp(speed/30, 0, 1));
      const capR = baseLatCap * (0.90 + 0.30 * clamp(speed/30, 0, 1));

      let FyF = clamp(-Cf * alphaF * 1000, -capF, capF);
      let FyR = clamp(-Cr * alphaR * 1000, -capR, capR);

      // If handbrake: add extra yaw looseness at speed (arcade feel)
      if (handbrake && speed > 10) {
        FyR *= 0.85;
      }

      // Combine forces in local frame:
      // Front lateral is rotated by steer angle
      const sinD = Math.sin(steerAngle), cosD = Math.cos(steerAngle);

      const FxF = 0;
      const FxR = Fx;

      const FxFrontLocal = FxF * cosD - FyF * sinD;
      const FyFrontLocal = FxF * sinD + FyF * cosD;

      const FxTotal = FxFrontLocal + FxR;
      const FyTotal = FyFrontLocal + FyR;

      // Accelerations (local)
      const ax = FxTotal / V.mass;
      const az = FyTotal / V.mass;

      // Integrate local velocities
      let newVLong = vLong + ax * dt;
      let newVLat  = vLat  + az * dt;

      // Extra lateral damping (stability), less on handbrake for drift
      const latDamp = (handbrake ? 0.86 : 0.72) * (offroad ? 0.9 : 1.0);
      newVLat *= Math.pow(latDamp, dt * 60);

      // Convert back to world velocity
      const worldVx =  cos * newVLong + sin * newVLat;
      const worldVz = -sin * newVLong + cos * newVLat;
      carState.vx = worldVx;
      carState.vz = worldVz;

      // Integrate yaw using yawRate (and allow some extra yaw from rear slip)
      // Add small slip-induced yaw: when rear grip is low, rear lateral can “rotate” the car.
      const slipYawBoost = (handbrake ? 0.65 : 0.25) * clamp(speed/25, 0, 1);
      const extraYaw = (-FyR / capR) * slipYawBoost;

      carState.yaw = wrapPI(carState.yaw + (yawRate + extraYaw) * dt);

      // Integrate position
      carState.x += carState.vx * dt;
      carState.z += carState.vz * dt;

      // Soft boundary: keep near map
      carState.x = clamp(carState.x, -380, 380);
      carState.z = clamp(carState.z, -380, 380);

      // Visual wheel spin/steer
      const wheelSpin = speed * dt / 0.42;
      wRL.rotation.x += wheelSpin;
      wRR.rotation.x += wheelSpin;
      wFL.rotation.x += wheelSpin;
      wFR.rotation.x += wheelSpin;
      const wheelSteer = steerAngle * 0.9;
      wFL.rotation.y = wheelSteer;
      wFR.rotation.y = wheelSteer;

      // Body roll/pitch
      const roll = clamp(newVLat * 0.04, -0.20, 0.20);
      const pitch = clamp(-newVLong * 0.012, -0.12, 0.12);
      body.rotation.z = lerp(body.rotation.z, roll, 1 - Math.pow(0.001, dt));
      body.rotation.x = lerp(body.rotation.x, pitch, 1 - Math.pow(0.001, dt));

      applyCarTransform();

      // Drift scoring based on slip angle between heading and velocity
      const vHeading = new THREE.Vector3(Math.sin(carState.yaw), 0, Math.cos(carState.yaw));
      const vWorld = new THREE.Vector3(carState.vx, 0, carState.vz);
      const vLen = vWorld.length();

      let slipDeg = 0;
      if (vLen > 2) {
        vWorld.normalize();
        const dot = clamp(vWorld.dot(vHeading), -1, 1);
        slipDeg = Math.acos(dot) * 180 / Math.PI;
      }

      const drifting = onTrack && vLen > 10 && slipDeg > 10;
      if (drifting) {
        const bonus = (slipDeg - 8) * (0.5 + clamp(vLen/35, 0, 1)) * (handbrake ? 1.2 : 1.0);
        carState.driftScore += bonus * dt * 10;
        carState.totalScore += bonus * dt * 6;
      } else {
        carState.driftScore *= Math.pow(0.55, dt * 6);
      }

      // Lap counting via gate plane crossing (negative -> positive) + near start region
      const side = gateSide(carState.x, carState.z);
      if (!carState.startedLapCounting) {
        carState.lastGateSide = side;
        if (vLen > 6) carState.startedLapCounting = true;
      } else {
        const crossed = (carState.lastGateSide < 0 && side >= 0);
        carState.lastGateSide = side;

        if (crossed) {
          const distStart = Math.hypot(carState.x - startLine.position.x, carState.z - startLine.position.z);
          const forwardVel = (carState.vx * gateNormal.x + carState.vz * gateNormal.z);
          if (distStart < 18 && forwardVel > 6) {
            const now = performance.now();
            const lapTime = (now - carState.lapStartMs) / 1000;
            carState.lapStartMs = now;

            if (carState.laps > 0) carState.bestLap = Math.min(carState.bestLap, lapTime);
            carState.laps++;

            if (carState.laps >= carState.lapsToWin) {
              carState.finished = true;
              hintEl.innerHTML =
                `<div style="font-weight:900; font-size:20px; margin-bottom:6px;">Finished 🏁</div>
                 Total drift score: <b>${Math.round(carState.totalScore)}</b><br>
                 ${isFinite(carState.bestLap) ? `Best lap: <b>${carState.bestLap.toFixed(1)}s</b><br>` : ``}
                 Press <b>R</b> to restart.`;
              hintEl.classList.remove("hidden");
            }
          }
        }
      }

      // HUD
      const kmh = Math.max(0, Math.round(vLen * 3.6));
      const lapTimeNow = (performance.now() - carState.lapStartMs) / 1000;
      const pct = Math.round((carState.laps / carState.lapsToWin) * 100);
      barEl.style.width = `${clamp(pct, 0, 100)}%`;

      statsEl.innerHTML =
        `Speed: <b>${kmh}</b> km/h ${offroad ? `• <b style="color:#ffb3c8">OFFROAD</b>` : ``}<br>` +
        `Slip: <b>${Math.round(slipDeg)}</b>° ${drifting ? `• <b style="color:#9ff">DRIFT!</b>` : ``}<br>` +
        `Lap: <b>${Math.min(carState.laps, carState.lapsToWin)}</b> / ${carState.lapsToWin} • Lap time: <b>${lapTimeNow.toFixed(1)}s</b>` +
        (isFinite(carState.bestLap) ? ` • Best: <b>${carState.bestLap.toFixed(1)}s</b>` : ``) +
        `<br>Drift score: <b>${Math.round(carState.totalScore)}</b>`;
    }

    // =========================
    // Camera follow (smooth chase)
    // =========================
    const camPos = new THREE.Vector3();
    const camTarget = new THREE.Vector3();

    function updateCamera(dt) {
      const forward = new THREE.Vector3(Math.sin(carState.yaw), 0, Math.cos(carState.yaw));
      const up = new THREE.Vector3(0, 1, 0);

      const vLen = Math.hypot(carState.vx, carState.vz);
      const back = 9.0 + Math.min(vLen * 0.12, 4.8);
      const height = 4.0 + Math.min(vLen * 0.05, 1.6);

      camTarget.set(carState.x, 0.9, carState.z).addScaledVector(forward, 2.3);

      camPos.set(carState.x, 0, carState.z)
            .addScaledVector(forward, -back)
            .addScaledVector(up, height);

      camera.position.lerp(camPos, 1 - Math.pow(0.0005, dt));
      camera.lookAt(camTarget);
    }

    // =========================
    // Scenery (lightweight, better “map vibe”)
    // =========================
    // Neon pylons + some rocks far from track so it looks like a place.
    function addPylon(x, z, h, color, emissive) {
      const m = new THREE.Mesh(
        new THREE.CylinderGeometry(0.35, 0.6, h, 10),
        new THREE.MeshStandardMaterial({
          color, roughness: 0.55, metalness: 0.55, emissive, emissiveIntensity: 0.32
        })
      );
      m.position.set(x, h/2, z);
      m.castShadow = true; m.receiveShadow = true;
      scene.add(m);

      const l = new THREE.PointLight(emissive, 0.65, 18, 2);
      l.position.set(x, h*0.8, z);
      scene.add(l);
    }

    function addRock(x, z, s) {
      const geo = new THREE.IcosahedronGeometry(1.5*s, 1);
      const p = geo.attributes.position;
      for (let i = 0; i < p.count; i++) {
        const j = (rand01(i + s*19) - 0.5) * 0.45;
        p.setXYZ(i, p.getX(i)*(1+j), p.getY(i)*(1+j*0.8), p.getZ(i)*(1+j));
      }
      geo.computeVertexNormals();
      const mat = new THREE.MeshStandardMaterial({ color: 0x283244, roughness: 0.9, metalness: 0.05 });
      const m = new THREE.Mesh(geo, mat);
      m.position.set(x, 0.6*s, z);
      m.rotation.y = rand01(x*3.1 + z*7.7) * Math.PI * 2;
      m.castShadow = true; m.receiveShadow = true;
      scene.add(m);
    }

    // Place pylons around the circuit
    for (let i = 0; i < 120; i++) {
      const t = i / 120;
      const p = centerCurve.getPointAt(t);
      const tan = centerCurve.getTangentAt(t);
      const f = new THREE.Vector3(tan.x, 0, tan.z).normalize();
      const right = new THREE.Vector3(-f.z, 0, f.x);
      const side = (i % 2 === 0) ? 1 : -1;
      const offset = TRACK_WIDTH/2 + 12 + rand01(i+9)*22;
      const pp = p.clone().addScaledVector(right, side * offset);
      const h = 4 + rand01(i+33)*12;
      const col = 0x0c1020;
      const em = (i % 3 === 0) ? 0xff77cc : 0x66ccff;
      if (rand01(i+1) > 0.35) addPylon(pp.x, pp.z, h, col, em);
    }

    // A few rocks in the outer areas
    for (let i = 0; i < 70; i++) {
      const a = (i / 70) * Math.PI * 2;
      const r = 170 + rand01(i+100)*230;
      const x = Math.cos(a) * r + (rand01(i+3)-0.5)*40;
      const z = Math.sin(a) * r + (rand01(i+7)-0.5)*40;
      addRock(x, z, 0.7 + rand01(i+200)*1.8);
    }

    // =========================
    // Minimap drawing
    // =========================
    function drawMinimap() {
      const w = mapCanvas.getBoundingClientRect().width;
      const h = mapCanvas.getBoundingClientRect().height;

      mapCtx.clearRect(0,0,w,h);
      mapCtx.fillStyle = "rgba(7,10,18,0)";
      mapCtx.fillRect(0,0,w,h);

      // Determine bounds for mapping
      // Use track sample min/max
      let minX=Infinity, maxX=-Infinity, minZ=Infinity, maxZ=-Infinity;
      for (let i = 0; i < SAMPLES; i += 6) {
        const p = samples[i];
        minX = Math.min(minX, p.x); maxX = Math.max(maxX, p.x);
        minZ = Math.min(minZ, p.z); maxZ = Math.max(maxZ, p.z);
      }
      const pad = 30;
      minX -= pad; maxX += pad; minZ -= pad; maxZ += pad;

      const sx = w / (maxX - minX);
      const sz = h / (maxZ - minZ);
      const s = Math.min(sx, sz);

      const ox = (w - (maxX - minX) * s) * 0.5 - minX * s;
      const oz = (h - (maxZ - minZ) * s) * 0.5 - minZ * s;

      const tx = (x) => x * s + ox;
      const tz = (z) => z * s + oz;

      // Track stroke (thick)
      mapCtx.lineCap = "round";
      mapCtx.lineJoin = "round";

      mapCtx.beginPath();
      for (let i = 0; i < SAMPLES; i += 4) {
        const p = samples[i];
        const X = tx(p.x), Y = tz(p.z);
        if (i === 0) mapCtx.moveTo(X, Y);
        else mapCtx.lineTo(X, Y);
      }
      mapCtx.closePath();

      // Outer glow
      mapCtx.strokeStyle = "rgba(120,200,255,0.16)";
      mapCtx.lineWidth = 12;
      mapCtx.stroke();

      // Main line
      mapCtx.strokeStyle = "rgba(255,255,255,0.65)";
      mapCtx.lineWidth = 4;
      mapCtx.stroke();

      // Start line marker
      mapCtx.fillStyle = "rgba(255,255,255,0.9)";
      mapCtx.beginPath();
      mapCtx.arc(tx(startLine.position.x), tz(startLine.position.z), 3.2, 0, Math.PI*2);
      mapCtx.fill();

      // Car dot + heading
      const cx = tx(carState.x), cy = tz(carState.z);
      mapCtx.fillStyle = "rgba(255,140,210,0.95)";
      mapCtx.beginPath();
      mapCtx.arc(cx, cy, 4.2, 0, Math.PI*2);
      mapCtx.fill();

      const hx = cx + Math.sin(carState.yaw) * 10;
      const hy = cy + Math.cos(carState.yaw) * 10;
      mapCtx.strokeStyle = "rgba(255,140,210,0.9)";
      mapCtx.lineWidth = 2;
      mapCtx.beginPath();
      mapCtx.moveTo(cx, cy);
      mapCtx.lineTo(hx, hy);
      mapCtx.stroke();
    }

    // =========================
    // Main loop
    // =========================
    const clock = new THREE.Clock();
    function animate() {
      requestAnimationFrame(animate);
      const dt = Math.min(clock.getDelta(), 0.033);

      if (started) {
        updateCar(dt);
        updateCamera(dt);
      }

      drawMinimap();
      renderer.render(scene, camera);
    }
    animate();

    // =========================
    // Resize
    // =========================
    function resize() {
      camera.aspect = innerWidth / innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(innerWidth, innerHeight);
      renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
      resizeMap();
    }
    addEventListener("resize", resize);
    resize();

    statsEl.textContent = "Click to start. W throttle, S brake/reverse, A/D steer, Space handbrake drift, R reset.";
  </script>
</body>
</html>