<!DOCTYPE html>
<html>
<head>
    <title>3D Chrome Racer</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: sans-serif; }
        #ui { position: absolute; top: 20px; left: 20px; color: white; pointer-events: none; }
        .controls { font-size: 12px; opacity: 0.7; }
    </style>
</head>
<body>
    <div id="ui">
        <h1>3D PRO RACER</h1>
        <div id="speed">Speed: 0 km/h</div>
        <div class="controls">WASD / Arrows to Drive | R to Reset</div>
    </div>

    <script type="module">
        import * as THREE from 'https://cdn.skypack.dev/three@0.136.0';
        import { OrbitControls } from 'https://cdn.skypack.dev/three@0.136.0/examples/jsm/controls/OrbitControls.js';

        // --- Configuration ---
        let scene, camera, renderer, world, carBody, carMesh;
        let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false;
        const velocity = new THREE.Vector3();
        
        init();
        animate();

        function init() {
            // 1. Scene & Camera
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x87ceeb); // Sky blue
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, 5, 10);

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.shadowMap.enabled = true;
            document.body.appendChild(renderer.domElement);

            // 2. Lighting
            const sun = new THREE.DirectionalLight(0xffffff, 1);
            sun.position.set(10, 20, 10);
            sun.castShadow = true;
            scene.add(sun);
            scene.add(new THREE.AmbientLight(0x404040));

            // 3. The Track (Procedural Loop)
            const trackMaterial = new THREE.MeshStandardMaterial({ color: 0x333333 });
            const trackGeometry = new THREE.TorusGeometry(50, 6, 16, 100);
            const track = new THREE.Mesh(trackGeometry, trackMaterial);
            track.rotation.x = Math.PI / 2;
            track.receiveShadow = true;
            scene.add(track);

            // Ground plane (Grass)
            const grass = new THREE.Mesh(
                new THREE.PlaneGeometry(500, 500),
                new THREE.MeshStandardMaterial({ color: 0x228b22 })
            );
            grass.rotation.x = -Math.PI / 2;
            grass.position.y = -0.1;
            scene.add(grass);

            // 4. The Car (Simplified Physics Body)
            const carGroup = new THREE.Group();
            
            // Car Body
            const bodyGeo = new THREE.BoxGeometry(2, 0.5, 4);
            const bodyMat = new THREE.MeshStandardMaterial({ color: 0xff0000 });
            const bodyMesh = new THREE.Mesh(bodyGeo, bodyMat);
            bodyMesh.castShadow = true;
            carGroup.add(bodyMesh);

            // Car Cabin
            const cabin = new THREE.Mesh(new THREE.BoxGeometry(1.5, 0.5, 2), new THREE.MeshStandardMaterial({color: 0x333333}));
            cabin.position.y = 0.5;
            carGroup.add(cabin);

            carMesh = carGroup;
            scene.add(carMesh);

            // Input Listeners
            window.addEventListener('keydown', (e) => updateControls(e.code, true));
            window.addEventListener('keyup', (e) => updateControls(e.code, false));
        }

        function updateControls(code, state) {
            if (code === 'KeyW' || code === 'ArrowUp') moveForward = state;
            if (code === 'KeyS' || code === 'ArrowDown') moveBackward = state;
            if (code === 'KeyA' || code === 'ArrowLeft') moveLeft = state;
            if (code === 'KeyD' || code === 'ArrowRight') moveRight = state;
            if (code === 'KeyR' && state) resetCar();
        }

        function resetCar() {
            carMesh.position.set(50, 0.5, 0);
            carMesh.rotation.set(0, 0, 0);
            velocity.set(0, 0, 0);
        }

        // Basic Physics & Movement Logic
        let speed = 0;
        let angle = 0;
        const maxSpeed = 0.8;
        const acceleration = 0.02;
        const friction = 0.98;

        function animate() {
            requestAnimationFrame(animate);

            // Physics Calculation
            if (moveForward) speed += acceleration;
            if (moveBackward) speed -= acceleration;
            
            speed *= friction; // Air resistance/friction
            
            if (Math.abs(speed) > 0.01) {
                if (moveLeft) angle += 0.03;
                if (moveRight) angle -= 0.03;
            }

            // Update Mesh Position
            carMesh.position.x += Math.sin(angle) * speed;
            carMesh.position.z += Math.cos(angle) * speed;
            carMesh.rotation.y = angle;

            // Camera follow (Third person)
            const camOffset = new THREE.Vector3(0, 3, -10).applyQuaternion(carMesh.quaternion);
            camera.position.lerp(carMesh.position.clone().add(camOffset), 0.1);
            camera.lookAt(carMesh.position);

            // Update UI
            document.getElementById('speed').innerText = `Speed: ${Math.floor(speed * 200)} km/h`;

            renderer.render(scene, camera);
        }

        // Handle Resize
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
