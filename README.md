<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Drift Track Racer (Single HTML)</title>
  <style>
    html, body { margin:0; height:100%; overflow:hidden; background:#070a12; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; }
    canvas { display:block; }
    #hud {
      position:fixed; inset:0; pointer-events:none; color:rgba(255,255,255,0.92);
      text-shadow: 0 2px 14px rgba(0,0,0,0.7);
    }
    #panel {
      position:absolute; left:12px; top:12px;
      background: rgba(10,14,22,0.55);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 14px;
      padding: 10px 12px;
      width: min(380px, calc(100vw - 24px));
      backdrop-filter: blur(10px);
    }
    #title { font-weight:800; font-size:15px; margin-bottom:6px; }
    #stats { font-size:13px; opacity:0.95; line-height:1.35; }
    #hint {
      position:absolute; left:50%; top:55%;
      transform: translate(-50%,-50%);
      text-align:center;
      background: rgba(0,0,0,0.40);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 16px;
      padding: 10px 14px;
      backdrop-filter: blur(12px);
      max-width: 92vw;
      transition: opacity .2s ease;
    }
    #hint.hidden { opacity: 0; }
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
  </style>
</head>
<body>
  <div id="hud">
    <div id="panel">
      <div id="title">Drift Track Racer</div>
      <div id="stats">Loading…</div>
      <div id="barWrap"><div id="bar"></div></div>
      <div style="margin-top:8px; font-size:12px; opacity:0.75;">
        Controls: <b>W</b> accelerate, <b>S</b> brake/reverse, <b>A/D</b> steer, <b>Space</b> handbrake drift, <b>R</b> reset
      </div>
    </div>
    <div id="hint">
      <div style="font-weight:800; margin-bottom:6px;">Click to start</div>
      Stay on the track, drift the corners, and finish <b>3 laps</b>.
    </div>
  </div>

  <script type="module">
    import * as THREE from "https://unpkg.com/three@0.160.0/build/three.module.js";

    // ---------- Renderer / Scene ----------
    const scene = new THREE.Scene();
    scene.fog = new THREE.FogExp2(0x070a12, 0.016);

    const camera = new THREE.PerspectiveCamera(70, innerWidth / innerHeight, 0.1, 3000);
    camera.position.set(0, 6, 12);

    const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: "high-performance" });
    renderer.setSize(innerWidth, innerHeight);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    document.body.appendChild(renderer.domElement);

    // ---------- HUD ----------
    const statsEl = document.getElementById("stats");
    const barEl = document.getElementById("bar");
    const hintEl = document.getElementById("hint");

    // ---------- Lighting / Sky ----------
    const skyGeo = new THREE.SphereGeometry(1200, 32, 16);
    const skyMat = new THREE.ShaderMaterial({
      side: THREE.BackSide,
      uniforms: {
        topColor: { value: new THREE.Color(0x121a3a) },
        bottomColor: { value: new THREE.Color(0x04050a) },
        offset: { value: 14.0 },
        exponent: { value: 0.9 }
      },
      vertexShader: `
        varying vec3 vWorldPosition;
        void main() {
          vec4 worldPosition = modelMatrix * vec4(position, 1.0);
          vWorldPosition = worldPosition.xyz;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }`,
      fragmentShader: `
        uniform vec3 topColor;
        uniform vec3 bottomColor;
        uniform float offset;
        uniform float exponent;
        varying vec3 vWorldPosition;
        void main() {
          float h = normalize(vWorldPosition + vec3(0.0, offset, 0.0)).y;
          float t = pow(max(h, 0.0), exponent);
          gl_FragColor = vec4(mix(bottomColor, topColor, t), 1.0);
        }`
    });
    scene.add(new THREE.Mesh(skyGeo, skyMat));

    const hemi = new THREE.HemisphereLight(0x9fd3ff, 0x0a0c14, 0.50);
    scene.add(hemi);

    const sun = new THREE.DirectionalLight(0xffffff, 1.15);
    sun.position.set(60, 80, 30);
    sun.castShadow = true;
    sun.shadow.mapSize.set(2048, 2048);
    sun.shadow.camera.near = 1;
    sun.shadow.camera.far = 260;
    sun.shadow.camera.left = -90;
    sun.shadow.camera.right = 90;
    sun.shadow.camera.top = 90;
    sun.shadow.camera.bottom = -90;
    sun.shadow.bias = -0.00006;
    scene.add(sun);

    const neonA = new THREE.PointLight(0x66ccff, 1.0, 55, 2);
    neonA.position.set(-35, 6, -20);
    scene.add(neonA);

    const neonB = new THREE.PointLight(0xff77cc, 1.0, 55, 2);
    neonB.position.set(35, 6, 25);
    scene.add(neonB);

    // ---------- Ground ----------
    const ground = new THREE.Mesh(
      new THREE.PlaneGeometry(600, 600),
      new THREE.MeshStandardMaterial({ color: 0x0b0f1a, roughness: 1.0, metalness: 0.0 })
    );
    ground.rotation.x = -Math.PI / 2;
    ground.receiveShadow = true;
    scene.add(ground);

    // ---------- Track (ring-like circuit) ----------
    // Track centerline is a circle-ish shape (slightly warped).
    const TRACK_R = 52;
    const TRACK_W = 14;
    const INNER_R = TRACK_R - TRACK_W * 0.5;
    const OUTER_R = TRACK_R + TRACK_W * 0.5;

    function warpAngle(a) {
      // add a bit of variation so it isn't a perfect circle
      return a + Math.sin(a * 2.0) * 0.14 + Math.sin(a * 5.0) * 0.06;
    }
    function centerAtAngle(a) {
      const aa = warpAngle(a);
      const r = TRACK_R + Math.sin(a * 3.0) * 3.0 + Math.cos(a * 1.0) * 2.0;
      return new THREE.Vector3(Math.cos(aa) * r, 0, Math.sin(aa) * r);
    }

    // Build ribbon geometry for the asphalt
    const segments = 700;
    const positions = [];
    const uvs = [];
    const indices = [];
    for (let i = 0; i <= segments; i++) {
      const t = i / segments;
      const a = t * Math.PI * 2;
      const p = centerAtAngle(a);
      const p2 = centerAtAngle(a + 0.01);
      const tangent = p2.clone().sub(p).normalize();
      const normal = new THREE.Vector3(-tangent.z, 0, tangent.x); // right vector on XZ

      const left = p.clone().addScaledVector(normal, -TRACK_W / 2);
      const right = p.clone().addScaledVector(normal,  TRACK_W / 2);

      positions.push(left.x, 0.02, left.z);
      positions.push(right.x, 0.02, right.z);

      // UVs for a simple stripe feel
      uvs.push(0, t * 50);
      uvs.push(1, t * 50);
    }

    for (let i = 0; i < segments; i++) {
      const i0 = i * 2;
      const i1 = i * 2 + 1;
      const i2 = i * 2 + 2;
      const i3 = i * 2 + 3;
      // two triangles
      indices.push(i0, i2, i1);
      indices.push(i2, i3, i1);
    }

    const trackGeo = new THREE.BufferGeometry();
    trackGeo.setAttribute("position", new THREE.Float32BufferAttribute(positions, 3));
    trackGeo.setAttribute("uv", new THREE.Float32BufferAttribute(uvs, 2));
    trackGeo.setIndex(indices);
    trackGeo.computeVertexNormals();

    // Asphalt shader-ish look via mat params (fast)
    const trackMat = new THREE.MeshStandardMaterial({
      color: 0x151a24,
      roughness: 0.92,
      metalness: 0.05,
      emissive: 0x05070c,
      emissiveIntensity: 0.25
    });

    const track = new THREE.Mesh(trackGeo, trackMat);
    track.receiveShadow = true;
    scene.add(track);

    // Track edges (barriers)
    function makeBarrier(offsetSign) {
      const pts = [];
      for (let i = 0; i <= 400; i++) {
        const t = i / 400;
        const a = t * Math.PI * 2;
        const p = centerAtAngle(a);
        const p2 = centerAtAngle(a + 0.01);
        const tangent = p2.clone().sub(p).normalize();
        const normal = new THREE.Vector3(-tangent.z, 0, tangent.x);
        const edge = p.clone().addScaledVector(normal, offsetSign * (TRACK_W/2 + 0.8));
        pts.push(new THREE.Vector3(edge.x, 0.05, edge.z));
      }
      const curve = new THREE.CatmullRomCurve3(pts, true, "centripetal");
      const tube = new THREE.TubeGeometry(curve, 900, 0.25, 10, true);
      const mat = new THREE.MeshStandardMaterial({
        color: offsetSign > 0 ? 0x2c3650 : 0x2a2f40,
        roughness: 0.45,
        metalness: 0.55,
        emissive: offsetSign > 0 ? 0x112033 : 0x210b1d,
        emissiveIntensity: 0.25
      });
      const m = new THREE.Mesh(tube, mat);
      m.castShadow = true;
      m.receiveShadow = true;
      scene.add(m);
    }
    makeBarrier(+1);
    makeBarrier(-1);

    // Start/finish gate + line
    const startA = 0; // angle reference
    const startPos = centerAtAngle(startA);
    const startDir = centerAtAngle(startA + 0.01).sub(startPos).normalize();
    const startRight = new THREE.Vector3(-startDir.z, 0, startDir.x);

    const lineGeo = new THREE.PlaneGeometry(TRACK_W, 2.0);
    const lineMat = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0.8, metalness: 0.1, emissive: 0x1a1a1a, emissiveIntensity: 0.4 });
    const startLine = new THREE.Mesh(lineGeo, lineMat);
    startLine.rotation.x = -Math.PI / 2;
    startLine.position.copy(startPos).addScaledVector(startDir, 1.5);
    startLine.position.y = 0.03;
    scene.add(startLine);

    const gate = new THREE.Group();
    const postMat = new THREE.MeshStandardMaterial({ color: 0x0e1220, roughness: 0.5, metalness: 0.6, emissive: 0x0b0f1a, emissiveIntensity: 0.4 });
    const postGeo = new THREE.BoxGeometry(0.8, 5.5, 0.8);
    const beamGeo = new THREE.BoxGeometry(TRACK_W + 4, 0.7, 0.9);
    const postL = new THREE.Mesh(postGeo, postMat);
    const postR = new THREE.Mesh(postGeo, postMat);
    const beam = new THREE.Mesh(beamGeo, postMat);
    postL.position.set(-TRACK_W/2 - 1.2, 2.75, 0);
    postR.position.set( TRACK_W/2 + 1.2, 2.75, 0);
    beam.position.set(0, 5.2, 0);
    gate.add(postL, postR, beam);
    gate.position.copy(startLine.position).addScaledVector(startDir, -0.2);
    gate.position.y = 0;
    gate.lookAt(gate.position.clone().add(startDir));
    gate.castShadow = true;
    scene.add(gate);

    const gateLight = new THREE.PointLight(0x66ccff, 1.5, 22, 2);
    gateLight.position.copy(gate.position).add(new THREE.Vector3(0, 5.0, 0));
    scene.add(gateLight);

    // ---------- Car (procedural “cool car”) ----------
    const car = new THREE.Group();

    const bodyMat = new THREE.MeshStandardMaterial({
      color: 0x2aa8ff,
      roughness: 0.25,
      metalness: 0.75,
      emissive: 0x071018,
      emissiveIntensity: 0.25
    });
    const glassMat = new THREE.MeshStandardMaterial({
      color: 0x0b1a2a,
      roughness: 0.05,
      metalness: 0.15,
      transparent: true,
      opacity: 0.55,
      emissive: 0x0b2a44,
      emissiveIntensity: 0.6
    });
    const trimMat = new THREE.MeshStandardMaterial({
      color: 0x0a0c12,
      roughness: 0.55,
      metalness: 0.35
    });

    // Body
    const body = new THREE.Mesh(new THREE.BoxGeometry(2.1, 0.55, 4.2), bodyMat);
    body.position.y = 0.55;
    body.castShadow = true;
    car.add(body);

    // Hood + roof contour
    const hood = new THREE.Mesh(new THREE.BoxGeometry(2.0, 0.35, 1.6), bodyMat);
    hood.position.set(0, 0.72, 1.05);
    hood.castShadow = true;
    car.add(hood);

    const roof = new THREE.Mesh(new THREE.BoxGeometry(1.7, 0.35, 1.4), bodyMat);
    roof.position.set(0, 1.05, -0.2);
    roof.castShadow = true;
    car.add(roof);

    const glass = new THREE.Mesh(new THREE.BoxGeometry(1.55, 0.40, 1.25), glassMat);
    glass.position.set(0, 1.02, -0.25);
    car.add(glass);

    // Splitter + spoiler
    const splitter = new THREE.Mesh(new THREE.BoxGeometry(2.25, 0.10, 0.55), trimMat);
    splitter.position.set(0, 0.32, 2.0);
    splitter.castShadow = true;
    car.add(splitter);

    const spoiler = new THREE.Mesh(new THREE.BoxGeometry(1.7, 0.12, 0.35), trimMat);
    spoiler.position.set(0, 1.15, -2.0);
    spoiler.castShadow = true;
    car.add(spoiler);

    // Wheels
    const wheelGeo = new THREE.CylinderGeometry(0.42, 0.42, 0.28, 18);
    const wheelMat = new THREE.MeshStandardMaterial({ color: 0x0b0d12, roughness: 0.75, metalness: 0.25 });
    const rimMat = new THREE.MeshStandardMaterial({ color: 0x9fb3c8, roughness: 0.35, metalness: 0.75, emissive: 0x0a0a0a, emissiveIntensity: 0.25 });

    function makeWheel() {
      const w = new THREE.Group();
      const tire = new THREE.Mesh(wheelGeo, wheelMat);
      tire.rotation.z = Math.PI / 2;
      tire.castShadow = true;
      const rim = new THREE.Mesh(new THREE.CylinderGeometry(0.24, 0.24, 0.285, 12), rimMat);
      rim.rotation.z = Math.PI / 2;
      rim.castShadow = true;
      w.add(tire, rim);
      return w;
    }

    const wFL = makeWheel(), wFR = makeWheel(), wRL = makeWheel(), wRR = makeWheel();
    wFL.position.set(-1.05, 0.40,  1.35);
    wFR.position.set( 1.05, 0.40,  1.35);
    wRL.position.set(-1.05, 0.40, -1.35);
    wRR.position.set( 1.05, 0.40, -1.35);
    car.add(wFL, wFR, wRL, wRR);

    // Head/taillights
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

    // Underglow
    const under = new THREE.PointLight(0x66ccff, 0.9, 8, 2);
    under.position.set(0, 0.25, 0);
    car.add(under);

    scene.add(car);

    // ---------- Input ----------
    const keys = new Set();
    addEventListener("keydown", (e) => keys.add(e.code));
    addEventListener("keyup", (e) => keys.delete(e.code));

    // Optional click-to-start to “feel” like a game start
    let started = false;
    addEventListener("click", () => { started = true; hintEl.classList.add("hidden"); });

    // ---------- Car physics (arcade drift) ----------
    // We simulate a chassis velocity and blend towards forward direction.
    const carState = {
      pos: new THREE.Vector3(),
      vel: new THREE.Vector3(),
      heading: 0,              // radians (yaw)
      steer: 0,                // -1..1
      speed: 0,
      lastOnTrack: true,
      lap: 0,
      lapsToWin: 3,
      crossedGate: false,
      startTime: performance.now(),
      bestLap: Infinity,
      currentLapStart: performance.now(),
      driftScore: 0,
      totalScore: 0
    };

    // Place car on the start line facing along the track direction
    carState.pos.copy(startLine.position).addScaledVector(startDir, -6);
    carState.pos.y = 0.0;
    carState.heading = Math.atan2(startDir.x, startDir.z); // note: yaw for XZ
    car.position.set(carState.pos.x, 0, carState.pos.z);
    car.rotation.y = carState.heading;

    function resetCar() {
      carState.vel.set(0, 0, 0);
      carState.pos.copy(startLine.position).addScaledVector(startDir, -6);
      carState.heading = Math.atan2(startDir.x, startDir.z);
      carState.steer = 0;
      carState.crossedGate = false;
      carState.driftScore = 0;
      car.position.set(carState.pos.x, 0, carState.pos.z);
      car.rotation.y = carState.heading;
    }

    // Helpers
    function wrapAngle(a) {
      while (a > Math.PI) a -= Math.PI * 2;
      while (a < -Math.PI) a += Math.PI * 2;
      return a;
    }

    // Determine if point is within track (approx by comparing distance to warped centerline sample)
    // Faster: use radial approximation on warped radius; it's “good enough” for gameplay.
    function trackDistanceMetrics(x, z) {
      const ang = Math.atan2(z, x);
      const a = ang < 0 ? ang + Math.PI * 2 : ang;
      const p = centerAtAngle(a);
      const dx = x - p.x;
      const dz = z - p.z;
      const distToCenter = Math.hypot(dx, dz);
      const halfW = TRACK_W * 0.5;
      return { distToCenter, onTrack: distToCenter <= halfW };
    }

    // Start/finish gate crossing detection (plane test)
    const gatePlanePoint = startLine.position.clone();
    const gatePlaneNormal = startDir.clone(); // crossing along forward direction

    function signedToGatePlane(p) {
      return (p.x - gatePlanePoint.x) * gatePlaneNormal.x + (p.z - gatePlanePoint.z) * gatePlaneNormal.z;
    }

    // ---------- Camera follow ----------
    const camTarget = new THREE.Vector3();
    const camPos = new THREE.Vector3();

    function updateCamera(dt) {
      const forward = new THREE.Vector3(Math.sin(carState.heading), 0, Math.cos(carState.heading));
      const up = new THREE.Vector3(0, 1, 0);

      camTarget.set(car.position.x, 0.8, car.position.z).addScaledVector(forward, 2.2);
      // Chase cam behind and above; slightly widens during speed
      const speed = carState.vel.length();
      const back = 8.5 + Math.min(speed * 0.12, 4.0);
      const height = 3.9 + Math.min(speed * 0.05, 1.4);
      camPos.set(car.position.x, 0, car.position.z)
            .addScaledVector(forward, -back)
            .addScaledVector(up, height);

      camera.position.lerp(camPos, 1 - Math.pow(0.0005, dt));
      camera.lookAt(camTarget);
    }

    // ---------- Drift + movement ----------
    const clock = new THREE.Clock();

    function update(dt) {
      // reset
      if (keys.has("KeyR")) resetCar();

      const throttle = (keys.has("KeyW") ? 1 : 0) - (keys.has("KeyS") ? 1 : 0);
      const steerInput = (keys.has("KeyD") ? 1 : 0) - (keys.has("KeyA") ? 1 : 0);
      const handbrake = keys.has("Space");

      // Smooth steering
      const steerRate = 7.0;
      carState.steer += (steerInput - carState.steer) * (1 - Math.pow(0.0008, dt * steerRate));
      carState.steer = THREE.MathUtils.clamp(carState.steer, -1, 1);

      const forward = new THREE.Vector3(Math.sin(carState.heading), 0, Math.cos(carState.heading));
      const right = new THREE.Vector3(forward.z, 0, -forward.x);

      // Track check affects grip
      const { distToCenter, onTrack } = trackDistanceMetrics(carState.pos.x, carState.pos.z);

      // Engine + braking (arcade)
      const maxSpeed = 44;          // ~160 km/h feel
      const accel = onTrack ? 28 : 16;
      const brake = onTrack ? 34 : 24;
      const drag = onTrack ? 2.1 : 3.8;

      // Add forward acceleration
      const currentForwardSpeed = carState.vel.dot(forward);
      let desiredForward = currentForwardSpeed + throttle * accel * dt;

      // Braking stronger when opposite direction or holding S
      if (throttle < 0 && currentForwardSpeed > 2) desiredForward = currentForwardSpeed + throttle * brake * dt;

      // Clamp speed
      desiredForward = THREE.MathUtils.clamp(desiredForward, -12, maxSpeed);

      // Lateral velocity (side slip)
      const lateral = carState.vel.dot(right);

      // Drift mechanics:
      // - baseline grip pulls velocity towards forward direction
      // - handbrake reduces lateral grip a LOT, and reduces forward grip a bit
      // - at speed, steering creates yaw and also “kicks” some lateral velocity
      const speed = carState.vel.length();
      const speed01 = THREE.MathUtils.clamp(speed / 40, 0, 1);

      const baseGrip = onTrack ? (0.92 - speed01 * 0.35) : (0.80 - speed01 * 0.25);
      const hbGrip = handbrake ? 0.35 : 1.0; // lateral grip multiplier
      const hbForward = handbrake ? 0.75 : 1.0;

      const lateralGrip = baseGrip * hbGrip;
      const forwardGrip = (0.96 - speed01 * 0.12) * hbForward;

      // Steering changes yaw faster at speed
      const steerStrength = (onTrack ? 1.0 : 0.75);
      const yawRate = steerStrength * (1.35 + speed01 * 2.2) * carState.steer;
      carState.heading = wrapAngle(carState.heading + yawRate * dt);

      // Recompute forward/right after yaw
      forward.set(Math.sin(carState.heading), 0, Math.cos(carState.heading));
      right.set(forward.z, 0, -forward.x);

      // Target velocity components
      let targetForward = desiredForward;
      let targetLateral = lateral;

      // Steering “kick” for drift: adds lateral at speed, stronger with handbrake
      const driftKick = (0.55 + speed01 * 1.15) * (handbrake ? 1.8 : 1.0);
      targetLateral += carState.steer * driftKick * speed01;

      // Apply grip: move current components toward targets
      const newForward = THREE.MathUtils.lerp(currentForwardSpeed, targetForward, 1 - Math.pow(1 - forwardGrip, dt * 60));
      const newLateral = THREE.MathUtils.lerp(lateral, targetLateral, 1 - Math.pow(1 - lateralGrip, dt * 60));

      // Compose velocity
      carState.vel.copy(forward).multiplyScalar(newForward).addScaledVector(right, newLateral);

      // Drag (always)
      carState.vel.addScaledVector(carState.vel, -drag * dt);

      // Soft wall: push toward center if too far off
      if (!onTrack && distToCenter > TRACK_W * 0.85) {
        const ang = Math.atan2(carState.pos.z, carState.pos.x);
        const a = ang < 0 ? ang + Math.PI * 2 : ang;
        const p = centerAtAngle(a);
        const toCenter = p.clone().sub(new THREE.Vector3(carState.pos.x, 0, carState.pos.z));
        carState.vel.addScaledVector(toCenter.normalize(), 18 * dt);
      }

      // Move
      carState.pos.addScaledVector(carState.vel, dt);

      // Keep within big world bounds
      carState.pos.x = THREE.MathUtils.clamp(carState.pos.x, -260, 260);
      carState.pos.z = THREE.MathUtils.clamp(carState.pos.z, -260, 260);

      // Wheel rotation + front wheel steering
      const wheelSpin = speed * dt / 0.42;
      wRL.rotation.x += wheelSpin;
      wRR.rotation.x += wheelSpin;
      wFL.rotation.x += wheelSpin;
      wFR.rotation.x += wheelSpin;

      const wheelSteer = carState.steer * (0.55 - speed01 * 0.22);
      wFL.rotation.y = wheelSteer;
      wFR.rotation.y = wheelSteer;

      // Car transform
      car.position.set(carState.pos.x, 0, carState.pos.z);
      car.rotation.y = carState.heading;

      // Slight body roll / pitch for feel
      const fwdSpdNow = carState.vel.dot(forward);
      const latNow = carState.vel.dot(right);
      body.rotation.x = THREE.MathUtils.lerp(body.rotation.x, -fwdSpdNow * 0.004, 1 - Math.pow(0.001, dt));
      body.rotation.z = THREE.MathUtils.lerp(body.rotation.z,  latNow * 0.008, 1 - Math.pow(0.001, dt));

      // Drift scoring (based on slip angle)
      const velDir = carState.vel.clone();
      velDir.y = 0;
      const velLen = velDir.length();
      let slipDeg = 0;
      if (velLen > 2) {
        velDir.normalize();
        const f = forward.clone().normalize();
        const dot = THREE.MathUtils.clamp(velDir.dot(f), -1, 1);
        const ang = Math.acos(dot);
        slipDeg = ang * 180 / Math.PI;
      }

      const drifting = onTrack && velLen > 10 && slipDeg > 12;
      if (drifting) {
        const bonus = (slipDeg - 10) * (0.4 + speed01) * (handbrake ? 1.25 : 1.0);
        carState.driftScore += bonus * dt * 10;
        carState.totalScore += bonus * dt * 6;
      } else {
        carState.driftScore *= Math.pow(0.55, dt * 6);
      }

      // Lap counting
      const prevSide = carState._gateSide ?? signedToGatePlane(carState.pos);
      const curSide = signedToGatePlane(carState.pos);
      carState._gateSide = curSide;

      // Count lap when crossing from negative -> positive (driving forward through gate)
      if (!carState.crossedGate) {
        // initialize gate state once moving
        carState.crossedGate = true;
        carState._gateSide = curSide;
        carState._lastLapSide = curSide;
      } else {
        const crossed = (prevSide < 0 && curSide >= 0);
        if (crossed) {
          // Ensure car is near the gate AND heading roughly forward
          const nearGate = car.position.distanceTo(startLine.position) < 18;
          const headingOk = carState.vel.dot(gatePlaneNormal) > 6;
          if (nearGate && headingOk) {
            const now = performance.now();
            const lapTime = (now - carState.currentLapStart) / 1000;
            carState.currentLapStart = now;

            if (carState.lap > 0) carState.bestLap = Math.min(carState.bestLap, lapTime);
            carState.lap++;

            if (carState.lap > carState.lapsToWin) {
              // keep from spamming
              carState.lap = carState.lapsToWin;
            }
          }
        }
      }

      // Update HUD
      const lap = carState.lap;
      const lapsToWin = carState.lapsToWin;
      const lapTimeNow = (performance.now() - carState.currentLapStart) / 1000;

      const pct = Math.round((lap / lapsToWin) * 100);
      barEl.style.width = `${pct}%`;

      statsEl.innerHTML =
        `Speed: <b>${Math.max(0, Math.round(velLen * 3.6))}</b> km/h<br>` +
        `Slip: <b>${Math.round(slipDeg)}</b>° ${drifting ? "• <b style='color:#9ff'>DRIFT!</b>" : ""}<br>` +
        `Lap: <b>${lap}</b> / ${lapsToWin} • Lap time: <b>${lapTimeNow.toFixed(1)}s</b>` +
        (isFinite(carState.bestLap) ? ` • Best: <b>${carState.bestLap.toFixed(1)}s</b>` : "") +
        `<br>Drift score: <b>${Math.round(carState.totalScore)}</b>`;

      // Win message
      if (lap >= lapsToWin) {
        hintEl.innerHTML =
          `<div style="font-weight:900; font-size:20px; margin-bottom:6px;">Finished 🏁</div>` +
          `Total drift score: <b>${Math.round(carState.totalScore)}</b><br>` +
          `Press <b>R</b> to restart.`;
        hintEl.classList.remove("hidden");
      }

      updateCamera(dt);
    }

    // ---------- Decorative scenery (simple) ----------
    function rand(seed) {
      const x = Math.sin(seed) * 10000;
      return x - Math.floor(x);
    }

    // Neon pylons around the map
    for (let i = 0; i < 90; i++) {
      const a = (i / 90) * Math.PI * 2;
      const r = 95 + rand(i * 13.7) * 130;
      const x = Math.cos(a) * r;
      const z = Math.sin(a) * r;
      const h = 3 + rand(i * 9.1) * 12;

      const pylon = new THREE.Mesh(
        new THREE.CylinderGeometry(0.35, 0.6, h, 10),
        new THREE.MeshStandardMaterial({
          color: 0x0c1020,
          roughness: 0.55,
          metalness: 0.55,
          emissive: (i % 2 === 0) ? 0x112033 : 0x210b1d,
          emissiveIntensity: 0.35
        })
      );
      pylon.position.set(x, h / 2, z);
      pylon.castShadow = true;
      pylon.receiveShadow = true;
      scene.add(pylon);

      if (i % 6 === 0) {
        const glow = new THREE.PointLight(i % 12 === 0 ? 0xff77cc : 0x66ccff, 0.8, 18, 2);
        glow.position.set(x, h * 0.8, z);
        scene.add(glow);
      }
    }

    // ---------- Loop ----------
    function animate() {
      requestAnimationFrame(animate);
      const dt = Math.min(clock.getDelta(), 0.033);
      if (started && carState.lap < carState.lapsToWin) update(dt);
      renderer.render(scene, camera);
    }
    animate();

    // ---------- Resize ----------
    addEventListener("resize", () => {
      camera.aspect = innerWidth / innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(innerWidth, innerHeight);
      renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    });

    // Start prompt if not clicked
    statsEl.textContent = "Click to start. W/S accelerate/brake, A/D steer, Space handbrake drift, R reset.";
  </script>
</body>
</html>