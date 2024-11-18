<html><head><base href="https://www.konami.com/efootball/">
<title>eFootball™ 3D - The Ultimate Soccer Experience</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/cannon.js/0.6.2/cannon.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/howler/2.2.3/howler.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/stats.js/r17/Stats.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/dat-gui/0.7.7/dat.gui.min.js"></script>
<style>
body, html { margin: 0; padding: 0; overflow: hidden; font-family: Arial, sans-serif; }
#game-container { width: 100vw; height: 100vh; position: relative; }
#menu, #instructions, #pause-menu { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background-color: rgba(0,0,0,0.8); padding: 20px; border-radius: 10px; text-align: center; max-width: 80%; max-height: 80%; overflow-y: auto; color: white; }
button { margin: 10px; padding: 10px 20px; font-size: 16px; background-color: #4CAF50; color: white; border: none; border-radius: 5px; cursor: pointer; transition: background-color 0.3s; }
button:hover { background-color: #45a049; }
#hud { position: absolute; top: 10px; left: 10px; color: white; font-size: 18px; display: none; }
#stamina-bar { width: 200px; height: 20px; background-color: #333; margin-top: 10px; }
#stamina-fill { width: 100%; height: 100%; background-color: #4CAF50; transition: width 0.3s; }
#power-meter { position: absolute; bottom: 20px; left: 50%; transform: translateX(-50%); width: 200px; height: 20px; background-color: #333; display: none; }
#power-fill { width: 0; height: 100%; background-color: #ff4136; transition: width 0.1s; }
#minimap { position: absolute; bottom: 20px; right: 20px; width: 150px; height: 90px; background-color: rgba(0,0,0,0.5); border: 2px solid white; }
#loading-screen { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background-color: #000; display: flex; justify-content: center; align-items: center; z-index: 1000; }
#loading-bar { width: 50%; height: 20px; background-color: #333; border-radius: 10px; overflow: hidden; }
#loading-progress { width: 0%; height: 100%; background-color: #4CAF50; transition: width 0.3s; }
</style>
</head>
<body>
<div id="game-container">
    <div id="loading-screen">
        <div id="loading-bar">
            <div id="loading-progress"></div>
        </div>
    </div>
    <div id="menu">
        <h1>eFootball™ 3D</h1>
        <button id="start-game">Start Game</button>
        <button id="open-instructions">How to Play</button>
        <button id="open-settings">Settings</button>
    </div>
    <div id="instructions" style="display:none;">
        <h2>How to Play</h2>
        <p>Use Z, Q, S, D to move. A to pass, SPACE to shoot, E to switch player, F to tackle, SHIFT to sprint.</p>
        <button id="close-instructions">Close</button>
    </div>
    <div id="pause-menu" style="display:none;">
        <h2>Game Paused</h2>
        <button id="resume-game">Resume</button>
        <button id="restart-game">Restart</button>
        <button id="quit-game">Quit</button>
    </div>
    <div id="hud">
        Score: <span id="score">0 - 0</span> | Time: <span id="time">05:00</span>
        <div id="stamina-bar"><div id="stamina-fill"></div></div>
    </div>
    <div id="power-meter"><div id="power-fill"></div></div>
    <canvas id="minimap"></canvas>
</div>

<script type="module">
import * as CANNON from 'https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/dist/cannon-es.min.js';
import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/postprocessing/UnrealBloomPass.js';
import { SMAAPass } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/postprocessing/SMAAPass.js';
import { GLTFLoader } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/loaders/DRACOLoader.js';
import { KTX2Loader } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/loaders/KTX2Loader.js';
import { MeshoptDecoder } from 'https://cdn.jsdelivr.net/npm/three@0.128.0/examples/jsm/libs/meshopt_decoder.module.js';

let scene, camera, renderer, composer, world, ball, ballBody, players, playerBodies, pitch, light;
let gameActive = false, scores = {teamA: 0, teamB: 0}, selectedPlayer, gameTime = 300, difficulty = 'medium', stamina = 100;
let powerMeter = {active: false, power: 0}, minimapContext;
const loader = new THREE.TextureLoader();
const soundEffects = {
    kick: new Howl({src: ['https://example.com/sounds/kick.mp3']}),
    whistle: new Howl({src: ['https://example.com/sounds/whistle.mp3']}),
    crowd: new Howl({src: ['https://example.com/sounds/crowd.mp3'], loop: true})
};

const stats = new Stats();
document.body.appendChild(stats.dom);

const gui = new dat.GUI();
const settings = {
    graphics: 'high',
    sound: 'on',
    difficulty: 'medium'
};
gui.add(settings, 'graphics', ['low', 'medium', 'high']).onChange(updateGraphics);
gui.add(settings, 'sound', ['off', 'on']).onChange(updateSound);
gui.add(settings, 'difficulty', ['easy', 'medium', 'hard']).onChange(updateDifficulty);

const loadingManager = new THREE.LoadingManager();
loadingManager.onProgress = function(url, itemsLoaded, itemsTotal) {
    const progress = (itemsLoaded / itemsTotal) * 100;
    document.getElementById('loading-progress').style.width = progress + '%';
};
loadingManager.onLoad = function() {
    document.getElementById('loading-screen').style.display = 'none';
    init();
};

const textureLoader = new THREE.TextureLoader(loadingManager);
const spriteSheet = textureLoader.load('https://example.com/textures/sprite_sheet.ktx2');
const texturePositions = {
    grass: new THREE.Vector2(0, 0),
    ball: new THREE.Vector2(0.5, 0),
    player: new THREE.Vector2(0, 0.5)
};

const gltfLoader = new GLTFLoader(loadingManager);
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/v1/decoders/');
gltfLoader.setDRACOLoader(dracoLoader);

const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath('https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/libs/basis/');
ktx2Loader.detectSupport(renderer);
gltfLoader.setKTX2Loader(ktx2Loader);

let playerModel, ballModel, stadiumModel;

gltfLoader.load('https://example.com/models/player.glb', (gltf) => {
    playerModel = gltf.scene;
    playerModel.scale.set(0.1, 0.1, 0.1);
});

gltfLoader.load('https://example.com/models/ball.glb', (gltf) => {
    ballModel = gltf.scene;
    ballModel.scale.set(0.1, 0.1, 0.1);
});

gltfLoader.load('https://example.com/models/stadium.glb', (gltf) => {
    stadiumModel = gltf.scene;
    stadiumModel.scale.set(1, 1, 1);
});

function init() {
    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    renderer = new THREE.WebGLRenderer({antialias: true, powerPreference: "high-performance"});
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1;
    renderer.outputEncoding = THREE.sRGBEncoding;
    document.getElementById('game-container').appendChild(renderer.domElement);

    composer = new EffectComposer(renderer);
    const renderPass = new RenderPass(scene, camera);
    composer.addPass(renderPass);

    const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
    composer.addPass(bloomPass);

    const smaaPass = new SMAAPass(window.innerWidth * renderer.getPixelRatio(), window.innerHeight * renderer.getPixelRatio());
    composer.addPass(smaaPass);

    world = new CANNON.World();
    world.gravity.set(0, -9.82, 0);

    createPitch();
    createBall();
    createPlayers();
    createLighting();
    createStadium();

    camera.position.set(0, 20, 30);
    camera.lookAt(scene.position);

    minimapContext = document.getElementById('minimap').getContext('2d');

    const worker = new Worker('physics-worker.js');
    worker.onmessage = function(e) {
        updatePhysics(e.data);
    };

    animate();
}

function createPitch() {
    const pitchGeometry = new THREE.PlaneGeometry(100, 60);
    const pitchMaterial = new THREE.MeshStandardMaterial({
        map: spriteSheet,
        normalMap: textureLoader.load('https://example.com/textures/grass_normal.ktx2'),
        roughnessMap: textureLoader.load('https://example.com/textures/grass_roughness.ktx2'),
        aoMap: textureLoader.load('https://example.com/textures/grass_ao.ktx2'),
        color: 0x88cc88
    });
    pitchMaterial.map.repeat.set(0.5, 0.5);
    pitchMaterial.map.offset.copy(texturePositions.grass);
    pitch = new THREE.Mesh(pitchGeometry, pitchMaterial);
    pitch.rotation.x = -Math.PI / 2;
    scene.add(pitch);

    const pitchShape = new CANNON.Plane();
    const pitchBody = new CANNON.Body({mass: 0});
    pitchBody.addShape(pitchShape);
    pitchBody.quaternion.setFromAxisAngle(new CANNON.Vec3(1, 0, 0), -Math.PI / 2);
    world.addBody(pitchBody);

    addFieldMarkings();
}

function createBall() {
    ball = ballModel.clone();
    scene.add(ball);

    ballBody = new CANNON.Body({
        mass: 1,
        shape: new CANNON.Sphere(1),
        position: new CANNON.Vec3(0, 1, 0),
        material: new CANNON.Material({restitution: 0.8})
    });
    world.addBody(ballBody);
}

function createPlayers() {
    players = [];
    playerBodies = [];

    for (let i = 0; i < 22; i++) {
        const team = i < 11 ? 'A' : 'B';
        const x = team === 'A' ? -20 + (i % 11) * 4 : 20 - (i % 11) * 4;
        const z = (i % 2 === 0 ? 1 : -1) * (Math.random() * 20 + 5);

        const player = playerModel.clone();
        player.position.set(x, 1, z);
        player.team = team;
        scene.add(player);
        players.push(player);

        const playerShape = new CANNON.Cylinder(0.5, 0.5, 2, 16);
        const playerBody = new CANNON.Body({
            mass: 5,
            shape: playerShape,
            position: new CANNON.Vec3(x, 1, z),
            fixedRotation: true
        });
        world.addBody(playerBody);
        playerBodies.push(playerBody);
    }

    selectedPlayer = players[0];
}

function createLighting() {
    const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(0, 100, 0);
    directionalLight.castShadow = true;
    directionalLight.shadow.mapSize.width = 2048;
    directionalLight.shadow.mapSize.height = 2048;
    directionalLight.shadow.camera.near = 0.5;
    directionalLight.shadow.camera.far = 500;
    scene.add(directionalLight);

    const hemisphereLight = new THREE.HemisphereLight(0xffffbb, 0x080820, 0.5);
    scene.add(hemisphereLight);
}

function createStadium() {
    scene.add(stadiumModel);
}

function addFieldMarkings() {
    const lineMaterial = new THREE.LineBasicMaterial({color: 0xffffff});
    const centerCircle = new THREE.Line(
        new THREE.CircleGeometry(9.15, 32).setFromPoints(
            new THREE.CircleGeometry(9.15, 32).vertices
        ),
        lineMaterial
    );
    centerCircle.rotation.x = -Math.PI / 2;
    scene.add(centerCircle);

    const halfwayLine = new THREE.Line(
        new THREE.BufferGeometry().setFromPoints([
            new THREE.Vector3(0, 0.01, -30),
            new THREE.Vector3(0, 0.01, 30)
        ]),
        lineMaterial
    );
    scene.add(halfwayLine);

    // Add more field markings (penalty areas, corner arcs, etc.)
}

let lastTime = 0;
function animate(time) {
    if (!gameActive) return;
    requestAnimationFrame(animate);

    const deltaTime = time - lastTime;
    if (deltaTime > 16) {  // Cap at ~60 FPS
        lastTime = time;
        updateGame(deltaTime);
        composer.render();
        stats.update();
    }
}

function updateGame(deltaTime) {
    world.step(deltaTime / 1000);
    updatePlayerPositions();
    updateBallPosition();
    updateCamera();
    checkForGoals();
    updateMinimap();
    updateParticles();
}

function updatePlayerPositions() {
    players.forEach((player, index) => {
        if (player === selectedPlayer) {
            stamina = Math.max(0, stamina - (sprinting ? 0.5 : -0.1));
            document.getElementById('stamina-fill').style.width = `${stamina}%`;
        }

        if (player.team === 'B') {
            updateAI(player, index);
        }

        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);

        playerBodies[index].position.x = Math.max(-50, Math.min(50, playerBodies[index].position.x));
        playerBodies[index].position.z = Math.max(-30, Math.min(30, playerBodies[index].position.z));
    });
}

function updateAI(player, index) {
    const aiStrength = difficulty === 'easy' ? 0.3 : difficulty === 'medium' ? 0.6 : 0.9;
    const targetPosition = Math.random() < aiStrength ? ballBody.position : new CANNON.Vec3(20 - (index % 11) * 4, 1.5, (index % 2 === 0 ? 1 : -1) * (Math.random() * 20 + 5));
    const direction = targetPosition.vsub(playerBodies[index].position).unit().scale(aiStrength);
    playerBodies[index].velocity.vadd(direction, playerBodies[index].velocity);

    if (playerBodies[index].position.distanceTo(ballBody.position) < 3 && Math.random() < aiStrength * 0.1) {
        Math.random() < 0.7 ? passBall(player) : shootBall(player);
    }
}

function updateBallPosition() {
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);
    ballBody.position.x = Math.max(-50, Math.min(50, ballBody.position.x));
    ballBody.position.z = Math.max(-30, Math.min(30, ballBody.position.z));
    ballBody.velocity.scale(0.99, ballBody.velocity);
}

const cameraOffset = new THREE.Vector3(0, 15, 20);
const cameraLookAt = new THREE.Vector3();

function updateCamera() {
    const idealOffset = selectedPlayer.position.clone().add(cameraOffset);
    const idealLookat = selectedPlayer.position.clone();
    
    camera.position.lerp(idealOffset, 0.1);
    cameraLookAt.lerp(idealLookat, 0.1);
    camera.lookAt(cameraLookAt);
}

function checkForGoals() {
    if (Math.abs(ballBody.position.x) > 49 && Math.abs(ballBody.position.z) < 7.32 / 2) {
        ballBody.position.x > 0 ? scores.teamA++ : scores.teamB++;
        document.getElementById('score').textContent = `${scores.teamA} - ${scores.teamB}`;
        resetAfterGoal();
        soundEffects.whistle.play();
        createGoalCelebration();
    }
}

function resetAfterGoal() {
    ballBody.position.set(0, 1, 0);
    ballBody.velocity.set(0, 0, 0);
    ballBody.angularVelocity.set(0, 0, 0);

    players.forEach((player, index) => {
        const x = player.team === 'A' ? -20 + (index % 11) * 4 : 20 - (index % 11) * 4;
        const z = (index % 2 === 0 ? 1 : -1) * (Math.random() * 20 + 5);
        playerBodies[index].position.set(x, 1.5, z);
        playerBodies[index].velocity.set(0, 0, 0);
        player.position.copy(playerBodies[index].position);
    });
}

function updateMinimap() {
    minimapContext.clearRect(0, 0, 150, 90);
    minimapContext.fillStyle = '#4CAF50';
    minimapContext.fillRect(0, 0, 150, 90);

    players.forEach(player => {
        minimapContext.fillStyle = player.team === 'A' ? 'red' : 'blue';
        const x = (player.position.x + 50) * 1.5;
        const y = (player.position.z + 30) * 1.5;
        minimapContext.fillRect(x - 2, y - 2, 4, 4);
    });

    minimapContext.fillStyle = 'white';
    const ballX = (ball.position.x + 50) * 1.5;
    const ballY = (ball.position.z + 30) * 1.5;
    minimapContext.beginPath();
    minimapContext.arc(ballX, ballY, 2, 0, Math.PI * 2);
    minimapContext.fill();
}

let sprinting = false;

const keyMap = {
    'z': { action: 'moveForward', axis: 'z', value: -1 },
    's': { action: 'moveBackward', axis: 'z', value: 1 },
    'q': { action: 'moveLeft', axis: 'x', value: -1 },
    'd': { action: 'moveRight', axis: 'x', value: 1 },
    'e': { action: 'switchPlayer' },
    'a': { action: 'pass' },
    ' ': { action: 'shoot' },
    'f': { action: 'tackle' },
    'Shift': { action: 'sprint' },
    'p': { action: 'pause' }
};

document.addEventListener('keydown', handleKeyDown);
document.addEventListener('keyup', handleKeyUp);

function handleKeyDown(event) {
    if (!gameActive) return;
    const key = keyMap[event.key];
    if (key) {
        const moveSpeed = sprinting && stamina > 0 ? 15 : 10;
        const selectedPlayerBody = playerBodies[players.indexOf(selectedPlayer)];

        switch(key.action) {
            case 'moveForward':
            case 'moveBackward':
            case 'moveLeft':
            case 'moveRight':
                selectedPlayerBody.velocity[key.axis] = moveSpeed * key.value;
                break;
            case 'switchPlayer':
                switchPlayer();
                break;
            case 'pass':
            case 'shoot':
                startPowerMeter(key.action === 'pass' ? 'pass' : 'shoot');
                break;
            case 'tackle':
                tackle();
                break;
            case 'sprint':
                sprinting = true;
                break;
            case 'pause':
                togglePause();
                break;
        }
    }
}

function handleKeyUp(event) {
    if (!gameActive) return;
    const key = keyMap[event.key];
    if (key) {
        const selectedPlayerBody = playerBodies[players.indexOf(selectedPlayer)];

        switch(key.action) {
            case 'moveForward':
            case 'moveBackward':
            case 'moveLeft':
            case 'moveRight':
                selectedPlayerBody.velocity[key.axis] = 0;
                break;
            case 'pass':
            case 'shoot':
                if (powerMeter.active) {
                    powerMeter.type === 'pass' ? passBall(selectedPlayer, powerMeter.power) : shootBall(selectedPlayer, powerMeter.power);
                    stopPowerMeter();
                }
                break;
            case 'sprint':
                sprinting = false;
                break;
        }
    }
}

function switchPlayer() {
    let closestPlayer = null;
    let closestDistance = Infinity;
    players.forEach(player => {
        if (player.team === 'A' && player !== selectedPlayer) {
            const distance = player.position.distanceTo(ball.position);
            if (distance < closestDistance) {
                closestPlayer = player;
                closestDistance = distance;
            }
        }
    });
    if (closestPlayer) {
        selectedPlayer = closestPlayer;
        stamina = 100;
        document.getElementById('stamina-fill').style.width = '100%';
    }
}

function startPowerMeter(type) {
    powerMeter.active = true;
    powerMeter.power = 0;
    powerMeter.type = type;
    document.getElementById('power-meter').style.display = 'block';

    function increasePower() {
        if (powerMeter.active) {
            powerMeter.power = Math.min(100, powerMeter.power + 2);
            document.getElementById('power-fill').style.width = `${powerMeter.power}%`;
            requestAnimationFrame(increasePower);
        }
    }
    increasePower();
}

function stopPowerMeter() {
    powerMeter.active = false;
    document.getElementById('power-meter').style.display = 'none';
}

function passBall(player, power = 50) {
    const strength = 15 + (power / 100) * 15;
    const nearestTeammate = findNearestTeammate(player);
    if (nearestTeammate) {
        const direction = new CANNON.Vec3().copy(nearestTeammate.position).vsub(player.position).unit();
        ballBody.velocity.set(direction.x * strength, strength / 2, direction.z * strength);
        soundEffects.kick.play();
        createKickEffect(player.position);
    }
}

function shootBall(player, power = 50) {
    const strength = 20 + (power / 100) * 20;
    const goalDirection = player.team === 'A' ? 1 : -1;
    const direction = new CANNON.Vec3(goalDirection, 0.5, (Math.random() - 0.5) * 0.5).unit();
    ballBody.velocity.set(direction.x * strength, direction.y * strength, direction.z * strength);
    soundEffects.kick.play();
    createKickEffect(player.position);
}

function findNearestTeammate(player) {
    return players.reduce((nearest, p) => {
        if (p.team === player.team && p !== player) {
            const distance = p.position.distanceTo(player.position);
            return (!nearest || distance < nearest.distance) ? {player: p, distance: distance} : nearest;
        }
        return nearest;
    }, null)?.player;
}

function tackle() {
    const selectedPlayerBody = playerBodies[players.indexOf(selectedPlayer)];
    players.forEach((player, index) => {
        if (player.team !== selectedPlayer.team) {
            if (player.position.distanceTo(selectedPlayer.position) < 3) {
                const direction = new CANNON.Vec3().copy(playerBodies[index].position).vsub(selectedPlayerBody.position).unit();
                playerBodies[index].velocity.set(direction.x * 10, 5, direction.z * 10);
                if (Math.random() < 0.5) {
                    const ballDirection = new CANNON.Vec3().copy(selectedPlayerBody.position).vsub(ballBody.position).unit();
                    ballBody.velocity.set(ballDirection.x * 15, 5, ballDirection.z * 15);
                }
                createTackleEffect(selectedPlayer.position);
            }
        }
    });
}

function togglePause() {
    gameActive = !gameActive;
    clearInterval(gameInterval);
    document.getElementById('pause-menu').style.display = gameActive ? 'none' : 'block';
    if (gameActive) {
        startTimer();
        animate();
    }
}

let gameInterval;
function startTimer() {
    gameInterval = setInterval(() => {
        gameTime--;
        const minutes = Math.floor(gameTime / 60);
        const seconds = gameTime % 60;
        document.getElementById('time').textContent = `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
        if (gameTime <= 0) {
            clearInterval(gameInterval);
            endGame();
        }
    }, 1000);
}

function endGame() {
    gameActive = false;
    soundEffects.whistle.play();
    soundEffects.crowd.stop();
    alert(`Game Over! Final Score: Team A ${scores.teamA} - ${scores.teamB} Team B`);
    location.reload();
}

document.getElementById('start-game').addEventListener('click', startGame);
document.getElementById('open-instructions').addEventListener('click', openInstructions);
document.getElementById('close-instructions').addEventListener('click', closeInstructions);
document.getElementById('open-settings').addEventListener('click', openSettings);
document.getElementById('resume-game').addEventListener('click', togglePause);
document.getElementById('restart-game').addEventListener('click', () => location.reload());
document.getElementById('quit-game').addEventListener('click', quitGame);

function startGame() {
    document.getElementById('menu').style.display = 'none';
    document.getElementById('hud').style.display = 'block';
    gameActive = true;
    animate();
    startTimer();
    soundEffects.whistle.play();
    soundEffects.crowd.play();
}

function openInstructions() {
    document.getElementById('instructions').style.display = 'block';
    document.getElementById('menu').style.display = 'none';
}

function closeInstructions() {
    document.getElementById('instructions').style.display = 'none';
    document.getElementById('menu').style.display = 'block';
}

function openSettings() {
    // Implement settings functionality here
}

function quitGame() {
    document.getElementById('menu').style.display = 'block';
    document.getElementById('pause-menu').style.display = 'none';
    document.getElementById('hud').style.display = 'none';
    gameActive = false;
    clearInterval(gameInterval);
    scores = {teamA: 0, teamB: 0};
    gameTime = 300;
    document.getElementById('score').textContent = '0 - 0';
    document.getElementById('time').textContent = '05:00';
}

window.addEventListener('resize', on

WindowResize);

function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
    composer.setSize(window.innerWidth, window.innerHeight);
}

function updateGraphics(value) {
    switch(value) {
        case 'low':
            renderer.setPixelRatio(1);
            composer.removePass(composer.passes[1]); // Remove bloom pass
            break;
        case 'medium':
            renderer.setPixelRatio(window.devicePixelRatio);
            composer.addPass(new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1, 0.4, 0.85));
            break;
        case 'high':
            renderer.setPixelRatio(window.devicePixelRatio);
            composer.addPass(new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85));
            break;
    }
}

function updateSound(value) {
    Howler.mute(value === 'off');
}

function updateDifficulty(value) {
    difficulty = value;
}

function createParticleSystem() {
    const particleCount = 1000;
    const particles = new THREE.BufferGeometry();
    const positions = new Float32Array(particleCount * 3);
    const colors = new Float32Array(particleCount * 3);

    for (let i = 0; i < particleCount; i++) {
        positions[i * 3] = Math.random() * 100 - 50;
        positions[i * 3 + 1] = Math.random() * 50;
        positions[i * 3 + 2] = Math.random() * 60 - 30;

        colors[i * 3] = Math.random();
        colors[i * 3 + 1] = Math.random();
        colors[i * 3 + 2] = Math.random();
    }

    particles.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    particles.setAttribute('color', new THREE.BufferAttribute(colors, 3));

    const particleMaterial = new THREE.PointsMaterial({
        size: 0.1,
        vertexColors: true,
        blending: THREE.AdditiveBlending,
        transparent: true
    });

    const particleSystem = new THREE.Points(particles, particleMaterial);
    scene.add(particleSystem);

    return particleSystem;
}

const particleSystem = createParticleSystem();

function updateParticles() {
    const positions = particleSystem.geometry.attributes.position.array;
    for (let i = 0; i < positions.length; i += 3) {
        positions[i + 1] -= 0.1;
        if (positions[i + 1] < 0) {
            positions[i + 1] = 50;
        }
    }
    particleSystem.geometry.attributes.position.needsUpdate = true;
}

function createKickEffect(position) {
    const kickGeometry = new THREE.SphereGeometry(0.1, 32, 32);
    const kickMaterial = new THREE.MeshBasicMaterial({ color: 0xffff00 });
    const kickEffect = new THREE.Mesh(kickGeometry, kickMaterial);
    kickEffect.position.copy(position);
    scene.add(kickEffect);

    new TWEEN.Tween(kickEffect.scale)
        .to({ x: 3, y: 3, z: 3 }, 300)
        .easing(TWEEN.Easing.Quadratic.Out)
        .start();

    new TWEEN.Tween(kickEffect.material)
        .to({ opacity: 0 }, 300)
        .easing(TWEEN.Easing.Quadratic.Out)
        .onComplete(() => {
            scene.remove(kickEffect);
        })
        .start();
}

function createTackleEffect(position) {
    const tackleGeometry = new THREE.RingGeometry(0.5, 1, 32);
    const tackleMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000, side: THREE.DoubleSide });
    const tackleEffect = new THREE.Mesh(tackleGeometry, tackleMaterial);
    tackleEffect.position.copy(position);
    tackleEffect.rotation.x = -Math.PI / 2;
    scene.add(tackleEffect);

    new TWEEN.Tween(tackleEffect.scale)
        .to({ x: 3, y: 3, z: 3 }, 500)
        .easing(TWEEN.Easing.Quadratic.Out)
        .start();

    new TWEEN.Tween(tackleEffect.material)
        .to({ opacity: 0 }, 500)
        .easing(TWEEN.Easing.Quadratic.Out)
        .onComplete(() => {
            scene.remove(tackleEffect);
        })
        .start();
}

function createGoalCelebration() {
    const confettiCount = 500;
    const colors = [0xff0000, 0x00ff00, 0x0000ff, 0xffff00, 0xff00ff, 0x00ffff];

    for (let i = 0; i < confettiCount; i++) {
        const confettiGeometry = new THREE.PlaneGeometry(0.1, 0.1);
        const confettiMaterial = new THREE.MeshBasicMaterial({
            color: colors[Math.floor(Math.random() * colors.length)],
            side: THREE.DoubleSide
        });
        const confetti = new THREE.Mesh(confettiGeometry, confettiMaterial);

        confetti.position.set(
            Math.random() * 100 - 50,
            50,
            Math.random() * 60 - 30
        );

        scene.add(confetti);

        new TWEEN.Tween(confetti.position)
            .to({ y: -1 }, 3000)
            .easing(TWEEN.Easing.Quadratic.Out)
            .start();

        new TWEEN.Tween(confetti.rotation)
            .to({
                x: Math.random() * Math.PI * 2,
                y: Math.random() * Math.PI * 2,
                z: Math.random() * Math.PI * 2
            }, 3000)
            .easing(TWEEN.Easing.Linear.None)
            .start();

        setTimeout(() => {
            scene.remove(confetti);
        }, 3000);
    }
}

// Implement advanced AI using behavior trees
class BehaviorTree {
    constructor(root) {
        this.root = root;
    }

    run(context) {
        return this.root.run(context);
    }
}

class BehaviorNode {
    run(context) {
        throw new Error('run method must be implemented');
    }
}

class Sequence extends BehaviorNode {
    constructor(children) {
        super();
        this.children = children;
    }

    run(context) {
        for (const child of this.children) {
            if (!child.run(context)) {
                return false;
            }
        }
        return true;
    }
}

class Selector extends BehaviorNode {
    constructor(children) {
        super();
        this.children = children;
    }

    run(context) {
        for (const child of this.children) {
            if (child.run(context)) {
                return true;
            }
        }
        return false;
    }
}

// Example AI behaviors
const isCloseToBall = new BehaviorNode();
isCloseToBall.run = (context) => {
    return context.player.position.distanceTo(context.ball.position) < 5;
};

const moveTowardsBall = new BehaviorNode();
moveTowardsBall.run = (context) => {
    const direction = context.ball.position.clone().sub(context.player.position).normalize();
    context.player.position.add(direction.multiplyScalar(0.1));
    return true;
};

const kickBall = new BehaviorNode();
kickBall.run = (context) => {
    if (context.player.position.distanceTo(context.ball.position) < 2) {
        shootBall(context.player);
        return true;
    }
    return false;
};

const aiTree = new BehaviorTree(
    new Selector([
        new Sequence([isCloseToBall, kickBall]),
        moveTowardsBall
    ])
);

function updateAI(player, index) {
    const context = {
        player: player,
        ball: ball,
        team: player.team
    };

    aiTree.run(context);
}

// Implement game state management
const GameState = {
    MENU: 'menu',
    PLAYING: 'playing',
    PAUSED: 'paused',
    GAME_OVER: 'gameOver'
};

let currentState = GameState.MENU;

function changeState(newState) {
    currentState = newState;
    switch (newState) {
        case GameState.MENU:
            showMenu();
            break;
        case GameState.PLAYING:
            startGame();
            break;
        case GameState.PAUSED:
            pauseGame();
            break;
        case GameState.GAME_OVER:
            endGame();
            break;
    }
}

// Implement advanced ball physics
function updateBallPhysics() {
    // Apply air resistance
    const airResistance = ballBody.velocity.clone().negate().normalize().multiplyScalar(0.01);
    ballBody.applyForce(airResistance, ballBody.position);

    // Apply spin effect
    const spinEffect = new CANNON.Vec3(0, 0.1, 0).cross(ballBody.angularVelocity);
    ballBody.applyForce(spinEffect, ballBody.position);

    // Handle bouncing
    if (ballBody.position.y <= 1 && ballBody.velocity.y < 0) {
        ballBody.velocity.y *= -0.8; // Damping factor
    }
}

// Implement replays
const replayBuffer = [];
const REPLAY_BUFFER_SIZE = 600; // 10 seconds at 60 fps

function recordReplayFrame() {
    const frame = {
        ballPosition: ball.position.clone(),
        playerPositions: players.map(player => player.position.clone())
    };
    replayBuffer.push(frame);
    if (replayBuffer.length > REPLAY_BUFFER_SIZE) {
        replayBuffer.shift();
    }
}

function playReplay() {
    let frameIndex = 0;
    const replayInterval = setInterval(() => {
        if (frameIndex >= replayBuffer.length) {
            clearInterval(replayInterval);
            return;
        }
        const frame = replayBuffer[frameIndex];
        ball.position.copy(frame.ballPosition);
        players.forEach((player, index) => {
            player.position.copy(frame.playerPositions[index]);
        });
        frameIndex++;
    }, 1000 / 60);
}

// Implement weather effects
const weatherEffects = {
    rain: null,
    wind: new THREE.Vector3()
};

function createRainEffect() {
    const rainGeometry = new THREE.BufferGeometry();
    const rainCount = 15000;
    const positions = new Float32Array(rainCount * 3);
    const velocities = new Float32Array(rainCount * 3);

    for (let i = 0; i < rainCount * 3; i += 3) {
        positions[i] = Math.random() * 200 - 100;
        positions[i + 1] = Math.random() * 100 + 50;
        positions[i + 2] = Math.random() * 200 - 100;

        velocities[i] = 0;
        velocities[i + 1] = -Math.random() * 1.5 - 0.5;
        velocities[i + 2] = 0;
    }

    rainGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    rainGeometry.setAttribute('velocity', new THREE.BufferAttribute(velocities, 3));

    const rainMaterial = new THREE.PointsMaterial({
        color: 0xaaaaaa,
        size: 0.1,
        transparent: true
    });

    weatherEffects.rain = new THREE.Points(rainGeometry, rainMaterial);
    scene.add(weatherEffects.rain);
}

function updateRain() {
    if (!weatherEffects.rain) return;

    const positions = weatherEffects.rain.geometry.attributes.position.array;
    const velocities = weatherEffects.rain.geometry.attributes.velocity.array;

    for (let i = 0; i < positions.length; i += 3) {
        positions[i] += velocities[i] + weatherEffects.wind.x;
        positions[i + 1] += velocities[i + 1];
        positions[i + 2] += velocities[i + 2] + weatherEffects.wind.z;

        if (positions[i + 1] < 0) {
            positions[i] = Math.random() * 200 - 100;
            positions[i + 1] = 100;
            positions[i + 2] = Math.random() * 200 - 100;
        }
    }

    weatherEffects.rain.geometry.attributes.position.needsUpdate = true;
}

function setWeather(type) {
    switch (type) {
        case 'clear':
            if (weatherEffects.rain) {
                scene.remove(weatherEffects.rain);
                weatherEffects.rain = null;
            }
            weatherEffects.wind.set(0, 0, 0);
            break;
        case 'rain':
            if (!weatherEffects.rain) {
                createRainEffect();
            }
            weatherEffects.wind.set(Math.random() * 0.1 - 0.05, 0, Math.random() * 0.1 - 0.05);
            break;
        case 'windy':
            weatherEffects.wind.set(Math.random() * 0.2 - 0.1, 0, Math.random() * 0.2 - 0.1);
            break;
    }
}

// Main game loop
function gameLoop(time) {
    stats.begin();

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(time);
            break;
        case GameState.PAUSED:
            // Render the paused scene
            composer.render();
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            composer.render();
            break;
    }

    TWEEN.update(time);
    updateRain();
    updateBallPhysics();
    recordReplayFrame();

    stats.end();

    requestAnimationFrame(gameLoop);
}

gameLoop(0);

});

</script>
</body>
</html>

function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
    composer.setSize(window.innerWidth, window.innerHeight);
}

function updateGraphics(value) {
    switch(value) {
        case 'low':
            renderer.setPixelRatio(1);
            composer.removePass(composer.passes[1]); // Remove bloom pass
            break;
        case 'medium':
            renderer.setPixelRatio(window.devicePixelRatio);
            composer.addPass(new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1, 0.4, 0.85));
            break;
        case 'high':
            renderer.setPixelRatio(window.devicePixelRatio);
            composer.addPass(new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85));
            break;
    }
}

function updateSound(value) {
    Howler.mute(value === 'off');
}

function updateDifficulty(value) {
    difficulty = value;
}

function createParticleSystem() {
    const particleCount = 1000;
    const particles = new THREE.BufferGeometry();
    const positions = new Float32Array(particleCount * 3);
    const colors = new Float32Array(particleCount * 3);

    for (let i = 0; i < particleCount; i++) {
        positions[i * 3] = Math.random() * 100 - 50;
        positions[i * 3 + 1] = Math.random() * 50;
        positions[i * 3 + 2] = Math.random() * 60 - 30;

        colors[i * 3] = Math.random();
        colors[i * 3 + 1] = Math.random();
        colors[i * 3 + 2] = Math.random();
    }

    particles.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    particles.setAttribute('color', new THREE.BufferAttribute(colors, 3));

    const particleMaterial = new THREE.PointsMaterial({
        size: 0.1,
        vertexColors: true,
        blending: THREE.AdditiveBlending,
        transparent: true
    });

    const particleSystem = new THREE.Points(particles, particleMaterial);
    scene.add(particleSystem);

    return particleSystem;
}

const particleSystem = createParticleSystem();

function updateParticles() {
    const positions = particleSystem.geometry.attributes.position.array;
    for (let i = 0; i < positions.length; i += 3) {
        positions[i + 1] -= 0.1;
        if (positions[i + 1] < 0) {
            positions[i + 1] = 50;
        }
    }
    particleSystem.geometry.attributes.position.needsUpdate = true;
}

function createKickEffect(position) {
    const kickGeometry = new THREE.SphereGeometry(0.1, 32, 32);
    const kickMaterial = new THREE.MeshBasicMaterial({ color: 0xffff00 });
    const kickEffect = new THREE.Mesh(kickGeometry, kickMaterial);
    kickEffect.position.copy(position);
    scene.add(kickEffect);

    new TWEEN.Tween(kickEffect.scale)
        .to({ x: 3, y: 3, z: 3 }, 300)
        .easing(TWEEN.Easing.Quadratic.Out)
        .start();

    new TWEEN.Tween(kickEffect.material)
        .to({ opacity: 0 }, 300)
        .easing(TWEEN.Easing.Quadratic.Out)
        .onComplete(() => {
            scene.remove(kickEffect);
        })
        .start();
}

function createTackleEffect(position) {
    const tackleGeometry = new THREE.RingGeometry(0.5, 1, 32);
    const tackleMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000, side: THREE.DoubleSide });
    const tackleEffect = new THREE.Mesh(tackleGeometry, tackleMaterial);
    tackleEffect.position.copy(position);
    tackleEffect.rotation.x = -Math.PI / 2;
    scene.add(tackleEffect);

    new TWEEN.Tween(tackleEffect.scale)
        .to({ x: 3, y: 3, z: 3 }, 500)
        .easing(TWEEN.Easing.Quadratic.Out)
        .start();

    new TWEEN.Tween(tackleEffect.material)
        .to({ opacity: 0 }, 500)
        .easing(TWEEN.Easing.Quadratic.Out)
        .onComplete(() => {
            scene.remove(tackleEffect);
        })
        .start();
}

function createGoalCelebration() {
    const confettiCount = 500;
    const colors = [0xff0000, 0x00ff00, 0x0000ff, 0xffff00, 0xff00ff, 0x00ffff];

    for (let i = 0; i < confettiCount; i++) {
        const confettiGeometry = new THREE.PlaneGeometry(0.1, 0.1);
        const confettiMaterial = new THREE.MeshBasicMaterial({
            color: colors[Math.floor(Math.random() * colors.length)],
            side: THREE.DoubleSide
        });
        const confetti = new THREE.Mesh(confettiGeometry, confettiMaterial);

        confetti.position.set(
            Math.random() * 100 - 50,
            50,
            Math.random() * 60 - 30
        );

        scene.add(confetti);

        new TWEEN.Tween(confetti.position)
            .to({ y: -1 }, 3000)
            .easing(TWEEN.Easing.Quadratic.Out)
            .start();

        new TWEEN.Tween(confetti.rotation)
            .to({
                x: Math.random() * Math.PI * 2,
                y: Math.random() * Math.PI * 2,
                z: Math.random() * Math.PI * 2
            }, 3000)
            .easing(TWEEN.Easing.Linear.None)
            .start();

        setTimeout(() => {
            scene.remove(confetti);
        }, 3000);
    }
}

// Implement advanced AI using behavior trees
class BehaviorTree {
    constructor(root) {
        this.root = root;
    }

    run(context) {
        return this.root.run(context);
    }
}

class BehaviorNode {
    run(context) {
        throw new Error('run method must be implemented');
    }
}

class Sequence extends BehaviorNode {
    constructor(children) {
        super();
        this.children = children;
    }

    run(context) {
        for (const child of this.children) {
            if (!child.run(context)) {
                return false;
            }
        }
        return true;
    }
}

class Selector extends BehaviorNode {
    constructor(children) {
        super();
        this.children = children;
    }

    run(context) {
        for (const child of this.children) {
            if (child.run(context)) {
                return true;
            }
        }
        return false;
    }
}

// Example AI behaviors
const isCloseToBall = new BehaviorNode();
isCloseToBall.run = (context) => {
    return context.player.position.distanceTo(context.ball.position) < 5;
};

const moveTowardsBall = new BehaviorNode();
moveTowardsBall.run = (context) => {
    const direction = context.ball.position.clone().sub(context.player.position).normalize();
    context.player.position.add(direction.multiplyScalar(0.1));
    return true;
};

const kickBall = new BehaviorNode();
kickBall.run = (context) => {
    if (context.player.position.distanceTo(context.ball.position) < 2) {
        shootBall(context.player);
        return true;
    }
    return false;
};

const aiTree = new BehaviorTree(
    new Selector([
        new Sequence([isCloseToBall, kickBall]),
        moveTowardsBall
    ])
);

function updateAI(player, index) {
    const context = {
        player: player,
        ball: ball,
        team: player.team
    };

    aiTree.run(context);
}

// Implement game state management
const GameState = {
    MENU: 'menu',
    PLAYING: 'playing',
    PAUSED: 'paused',
    GAME_OVER: 'gameOver'
};

let currentState = GameState.MENU;

function changeState(newState) {
    currentState = newState;
    switch (newState) {
        case GameState.MENU:
            showMenu();
            break;
        case GameState.PLAYING:
            startGame();
            break;
        case GameState.PAUSED:
            pauseGame();
            break;
        case GameState.GAME_OVER:
            endGame();
            break;
    }
}

// Implement advanced ball physics
function updateBallPhysics() {
    // Apply air resistance
    const airResistance = ballBody.velocity.clone().negate().normalize().multiplyScalar(0.01);
    ballBody.applyForce(airResistance, ballBody.position);

    // Apply spin effect
    const spinEffect = new CANNON.Vec3(0, 0.1, 0).cross(ballBody.angularVelocity);
    ballBody.applyForce(spinEffect, ballBody.position);

    // Handle bouncing
    if (ballBody.position.y <= 1 && ballBody.velocity.y < 0) {
        ballBody.velocity.y *= -0.8; // Damping factor
    }
}

// Implement replays
const replayBuffer = [];
const REPLAY_BUFFER_SIZE = 600; // 10 seconds at 60 fps

function recordReplayFrame() {
    const frame = {
        ballPosition: ball.position.clone(),
        playerPositions: players.map(player => player.position.clone())
    };
    replayBuffer.push(frame);
    if (replayBuffer.length > REPLAY_BUFFER_SIZE) {
        replayBuffer.shift();
    }
}

function playReplay() {
    let frameIndex = 0;
    const replayInterval = setInterval(() => {
        if (frameIndex >= replayBuffer.length) {
            clearInterval(replayInterval);
            return;
        }
        const frame = replayBuffer[frameIndex];
        ball.position.copy(frame.ballPosition);
        players.forEach((player, index) => {
            player.position.copy(frame.playerPositions[index]);
        });
        frameIndex++;
    }, 1000 / 60);
}

// Implement weather effects
const weatherEffects = {
    rain: null,
    wind: new THREE.Vector3()
};

function createRainEffect() {
    const rainGeometry = new THREE.BufferGeometry();
    const rainCount = 15000;
    const positions = new Float32Array(rainCount * 3);
    const velocities = new Float32Array(rainCount * 3);

    for (let i = 0; i < rainCount * 3; i += 3) {
        positions[i] = Math.random() * 200 - 100;
        positions[i + 1] = Math.random() * 100 + 50;
        positions[i + 2] = Math.random() * 200 - 100;

        velocities[i] = 0;
        velocities[i + 1] = -Math.random() * 1.5 - 0.5;
        velocities[i + 2] = 0;
    }

    rainGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    rainGeometry.setAttribute('velocity', new THREE.BufferAttribute(velocities, 3));

    const rainMaterial = new THREE.PointsMaterial({
        color: 0xaaaaaa,
        size: 0.1,
        transparent: true
    });

    weatherEffects.rain = new THREE.Points(rainGeometry, rainMaterial);
    scene.add(weatherEffects.rain);
}

function updateRain() {
    if (!weatherEffects.rain) return;

    const positions = weatherEffects.rain.geometry.attributes.position.array;
    const velocities = weatherEffects.rain.geometry.attributes.velocity.array;

    for (let i = 0; i < positions.length; i += 3) {
        positions[i] += velocities[i] + weatherEffects.wind.x;
        positions[i + 1] += velocities[i + 1];
        positions[i + 2] += velocities[i + 2] + weatherEffects.wind.z;

        if (positions[i + 1] < 0) {
            positions[i] = Math.random() * 200 - 100;
            positions[i + 1] = 100;
            positions[i + 2] = Math.random() * 200 - 100;
        }
    }

    weatherEffects.rain.geometry.attributes.position.needsUpdate = true;
}

function setWeather(type) {
    switch (type) {
        case 'clear':
            if (weatherEffects.rain) {
                scene.remove(weatherEffects.rain);
                weatherEffects.rain = null;
            }
            weatherEffects.wind.set(0, 0, 0);
            break;
        case 'rain':
            if (!weatherEffects.rain) {
                createRainEffect();
            }
            weatherEffects.wind.set(Math.random() * 0.1 - 0.05, 0, Math.random() * 0.1 - 0.05);
            break;
        case 'windy':
            weatherEffects.wind.set(Math.random() * 0.2 - 0.1, 0, Math.random() * 0.2 - 0.1);
            break;
    }
}

// Main game loop
function gameLoop(time) {
    stats.begin();

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(time);
            break;
        case GameState.PAUSED:
            // Render the paused scene
            composer.render();
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            composer.render();
            break;
    }

    TWEEN.update(time);
    updateRain();
    updateBallPhysics();
    recordReplayFrame();

    stats.end();

    requestAnimationFrame(gameLoop);
}

gameLoop(0);

// Add post-processing effects
const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
composer.addPass(bloomPass);

const filmPass = new FilmPass(0.35, 0.5, 2048, false);
composer.addPass(filmPass);

// Implement dynamic lighting
function updateLighting() {
    const time = Date.now() * 0.001;
    const lightIntensity = Math.sin(time) * 0.5 + 1.5;
    directionalLight.intensity = lightIntensity;
}

// Add crowd sounds
const crowdSound = new THREE.Audio(listener);
const audioLoader = new THREE.AudioLoader();
audioLoader.load('sounds/crowd.mp3', function(buffer) {
    crowdSound.setBuffer(buffer);
    crowdSound.setLoop(true);
    crowdSound.setVolume(0.5);
    crowdSound.play();
});

// Implement player skills and attributes
class PlayerAttributes {
    constructor(speed, strength, accuracy, stamina) {
        this.speed = speed;
        this.strength = strength;
        this.accuracy = accuracy;
        this.stamina = stamina;
    }
}

players.forEach(player => {
    player.attributes = new PlayerAttributes(
        Math.random() * 0.5 + 0.5,
        Math.random() * 0.5 + 0.5,
        Math.random() * 0.5 + 0.5,
        Math.random() * 0.5 + 0.5
    );
});

function updatePlayerMovement(player, deltaTime) {
    const speed = player.attributes.speed * (sprinting ? 2 : 1);
    const movement = new THREE.Vector3().copy(player.movement).multiplyScalar(speed * deltaTime);
    player.position.add(movement);
}

// Implement advanced ball control
function updateBallControl() {
    players.forEach(player => {
        if (player.position.distanceTo(ball.position) < 2) {
            const controlProbability = player.attributes.accuracy * 0.5;
            if (Math.random() < controlProbability) {
                ball.position.copy(player.position).add(new THREE.Vector3(0, 1, 0));
                ballBody.position.copy(ball.position);
                ballBody.velocity.set(0, 0, 0);
            }
        }
    });
}

// Implement team strategies
const teamStrategies = {
    offensive: {
        formation: [
            { x: -40, z: 0 },
            { x: -30, z: -10 }, { x: -30, z: 10 },
            { x: -20, z: -20 }, { x: -20, z: 0 }, { x: -20, z: 20 },
            { x: -10, z: -15 }, { x: -10, z: 15 },
            { x: 0, z: -10 }, { x: 0, z: 10 },
            { x: 10, z: 0 }
        ],
        pressureIntensity: 0.8
    },
    defensive: {
        formation: [
            { x: -40, z: 0 },
            { x: -35, z: -10 }, { x: -35, z: 10 },
            { x: -30, z: -20 }, { x: -30, z: 0 }, { x: -30, z: 20 },
            { x: -25, z: -15 }, { x: -25, z: 15 },
            { x: -20, z: -10 }, { x: -20, z: 10 },
            { x: -15, z: 0 }
        ],
        pressureIntensity: 0.4
    }
};

function updateTeamFormation(team, strategy) {
    const teamPlayers = players.filter(player => player.team === team);
    const formation = teamStrategies[strategy].formation;
    teamPlayers.forEach((player, index) => {
        const targetPosition = new THREE.Vector3(formation[index].x, 1, formation[index].z);
        if (team === 'B') targetPosition.x *= -1;
        player.formationPosition = targetPosition;
    });
}

function updatePlayerAI(player, deltaTime) {
    const ballPosition = ball.position;
    const distanceToBall = player.position.distanceTo(ballPosition);

    if (distanceToBall < 10) {
        // Chase the ball
        const direction = ballPosition.clone().sub(player.position).normalize();
        player.position.add(direction.multiplyScalar(player.attributes.speed * deltaTime));
    } else {
        // Move towards formation position
        const direction = player.formationPosition.clone().sub(player.position).normalize();
        player.position.add(direction.multiplyScalar(player.attributes.speed * 0.5 * deltaTime));
    }

    // Apply team strategy
    const strategy = player.team === 'A' ? teamStrategies.offensive : teamStrategies.defensive;
    if (Math.random() < strategy.pressureIntensity) {
        const nearestOpponent = findNearestOpponent(player);
        if (nearestOpponent) {
            const direction = nearestOpponent.position.clone().sub(player.position).normalize();
            player.position.add(direction.multiplyScalar(player.attributes.speed * 0.3 * deltaTime));
        }
    }
}

function findNearestOpponent(player) {
    return players.reduce((nearest, p) => {
        if (p.team !== player.team) {
            const distance = p.position.distanceTo(player.position);
            return (!nearest || distance < nearest.distance) ? {player: p, distance: distance} : nearest;
        }
        return nearest;
    }, null)?.player;
}

// Implement advanced ball physics
function updateBallPhysics(deltaTime) {
    // Apply gravity
    ballBody.velocity.y -= 9.81 * deltaTime;

    // Apply air resistance
    const airResistance = ballBody.velocity.clone().negate().normalize().multiplyScalar(0.1 * deltaTime);
    ballBody.velocity.add(airResistance);

    // Apply Magnus effect (spin)
    const spinAxis = new CANNON.Vec3(0, 1, 0);
    const spinStrength = 0.5;
    const magnusForce = new CANNON.Vec3().copy(ballBody.velocity).cross(spinAxis).scale(spinStrength * deltaTime);
    ballBody.velocity.vadd(magnusForce);

    // Update position
    ballBody.position.vadd(ballBody.velocity.scale(deltaTime));

    // Handle ground collision
    if (ballBody.position.y < 1) {
        ballBody.position.y = 1;
        ballBody.velocity.y *= -0.8; // Bounce with some energy loss
    }

    // Update Three.js ball position
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);
}

// Implement advanced camera system
const cameraTargets = {
    ball: new THREE.Vector3(),
    activePlayer: new THREE.Vector3(),
    overview: new THREE.Vector3(0, 50, 0)
};

let currentCameraTarget = 'ball';

function updateCamera(deltaTime) {
    let targetPosition;
    switch (currentCameraTarget) {
        case 'ball':
            targetPosition = ball.position;
            break;
        case 'activePlayer':
            targetPosition = selectedPlayer.position;
            break;
        case 'overview':
            targetPosition = cameraTargets.overview;
            break;
    }

    camera.position.lerp(targetPosition.clone().add(new THREE.Vector3(0, 30, 40)), 0.1);
    camera.lookAt(targetPosition);
}

function switchCameraTarget() {
    const targets = ['ball', 'activePlayer', 'overview'];
    currentCameraTarget = targets[(targets.indexOf(currentCameraTarget) + 1) % targets.length];
}

// Implement advanced UI
function updateUI() {
    document.getElementById('score').textContent = `${scores.teamA} - ${scores.teamB}`;
    document.getElementById('time').textContent = formatTime(gameTime);
    document.getElementById('stamina-fill').style.width = `${stamina}%`;

    // Update player stats
    const playerStatsElement = document.getElementById('player-stats');
    playerStatsElement.innerHTML = '';
    players.forEach(player => {
        const statsDiv = document.createElement('div');
        statsDiv.textContent = `Player ${player.id}: Speed ${player.attributes.speed.toFixed(2)}, Stamina ${player.attributes.stamina.toFixed(2)}`;
        playerStatsElement.appendChild(statsDiv);
    });
}

function formatTime(seconds) {
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = seconds % 60;
    return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
}

// Implement match events
function createMatchEvent(type, data) {
    const event = { type, data, time: gameTime };
    matchEvents.push(event);
    displayMatchEvent(event);
}

function displayMatchEvent(event) {
    const eventElement = document.createElement('div');
    eventElement.classList.add('match-event');
    eventElement.textContent = `${formatTime(event.time)} - ${event.type}: ${JSON.stringify(event.data)}`;
    document.getElementById('match-events').appendChild(eventElement);
}

// Main game loop
function gameLoop(time) {
    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    if (currentState === GameState.PLAYING) {
        updateGame(deltaTime);
        updateCamera(deltaTime);
        updateUI();
        updateLighting();
        updateBallPhysics(deltaTime);
        updateBallControl();
        players.forEach(player => updatePlayerAI(player, deltaTime));
    }

    renderer.render(scene, camera);
    requestAnimationFrame(gameLoop);
}

// Initialize the game
init();
gameLoop(0);

// Optimize performance
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement occlusion culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateVisibility() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use instancing for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement level of detail (LOD)
    const playerLODs = [];
    players.forEach(player => {
        const highDetailGeometry = new THREE.SphereGeometry(1, 32, 32);
        const mediumDetailGeometry = new THREE.SphereGeometry(1, 16, 16);
        const lowDetailGeometry = new THREE.SphereGeometry(1, 8, 8);

        const playerLOD = new THREE.LOD();

        playerLOD.addLevel(new THREE.Mesh(highDetailGeometry, player.material), 0);
        playerLOD.addLevel(new THREE.Mesh(mediumDetailGeometry, player.material), 10);
        playerLOD.addLevel(new THREE.Mesh(lowDetailGeometry, player.material), 50);

        playerLOD.position.copy(player.position);
        scene.add(playerLOD);
        playerLODs.push(playerLOD);
    });

    function updateLODs() {
        playerLODs.forEach((playerLOD, index) => {
            playerLOD.position.copy(players[index].position);
            playerLOD.update(camera);
        });
    }

    // Use worker threads for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        ball.position.copy(ballPosition);
        ballBody.velocity.copy(ballVelocity);
        players.forEach((player, index) => {
            player.position.copy(playerPositions[index]);
        });
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Implement shader-based grass
    const grassShaderMaterial = new THREE.ShaderMaterial({
        vertexShader: `
            varying vec2 vUv;
            uniform float time;
            
            void main() {
                vUv = uv;
                vec3 pos = position;
                
                // Grass movement
                float movement = sin(time * 2.0 + position.x * 0.5 + position.z * 0.5) * 0.1;
                pos.x += movement;
                
                gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
            }
        `,
        fragmentShader: `
            varying vec2 vUv;
            uniform sampler2D grassTexture;
            
            void main() {
                vec4 texColor = texture2D(grassTexture, vUv);
                gl_FragColor = texColor;
            }
        `,
        uniforms: {
            time: { value: 0 },
            grassTexture: { value: new THREE.TextureLoader().load('textures/grass.jpg') }
        }
    });

    const grassMesh = new THREE.Mesh(
        new THREE.PlaneGeometry(100, 60, 100, 60),
        grassShaderMaterial
    );
    grassMesh.rotation.x = -Math.PI / 2;
    scene.add(grassMesh);

    // Update optimization functions
    function updateOptimizations(time) {
        updateVisibility();
        updateLODs();
        updatePhysics();
        grassShaderMaterial.uniforms.time.value = time;
    }

    return updateOptimizations;
}

const updateOptimizations = optimizePerformance();

// Enhanced game loop
function enhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            updateLighting();
            updateBallPhysics(deltaTime);
            updateBallControl();
            players.forEach(player => updatePlayerAI(player, deltaTime));
            updateOptimizations(time);
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoop);
}

// Start the enhanced game loop
enhancedGameLoop(0);

// Add more advanced features

// Implement a basic commentary system
const commentarySystem = {
    phrases: [
        "What a beautiful day for football!",
        "The crowd is electric today!",
        "That was a close call!",
        "Spectacular save by the goalkeeper!",
        "The striker is looking dangerous today!",
        "What a fantastic pass!",
        "The defense is holding strong!",
        "This match could go either way!",
        "The manager looks concerned on the sideline.",
        "That's a controversial decision by the referee!"
    ],
    lastCommentaryTime: 0,
    commentaryInterval: 10000, // 10 seconds

    update: function(time) {
        if (time - this.lastCommentaryTime > this.commentaryInterval) {
            this.generateCommentary();
            this.lastCommentaryTime = time;
        }
    },

    generateCommentary: function() {
        const commentary = this.phrases[Math.floor(Math.random() * this.phrases.length)];
        console.log("Commentary: " + commentary);
        // You could also display this in the UI or play it as audio
    }
};

// Implement a basic referee system
const refereeSystem = {
    position: new THREE.Vector3(),
    speed: 5,

    update: function(deltaTime) {
        // Move the referee towards the ball
        const direction = new THREE.Vector3().subVectors(ball.position, this.position).normalize();
        this.position.add(direction.multiplyScalar(this.speed * deltaTime));

        // Check for fouls
        this.checkForFouls();
    },

    checkForFouls: function() {
        players.forEach(player => {
            if (player.position.distanceTo(ball.position) < 2 && Math.random() < 0.01) {
                this.callFoul(player);
            }
        });
    },

    callFoul: function(player) {
        console.log(`Foul called on player ${player.id} from team ${player.team}!`);
        // Implement foul logic (free kicks, penalties, etc.)
    }
};

// Implement a basic crowd system
const crowdSystem = {
    excitement: 0.5,
    volumeMultiplier: 1,

    update: function(deltaTime) {
        // Increase excitement for goals, near misses, fouls, etc.
        if (Math.random() < 0.05) {
            this.excitement = Math.min(1, this.excitement + 0.1);
        } else {
            this.excitement = Math.max(0, this.excitement - 0.01 * deltaTime);
        }

        // Update crowd sound
        if (crowdSound) {
            crowdSound.setVolume(this.excitement * this.volumeMultiplier);
        }
    }
};

// Enhance the game loop with new systems
function enhancedGameLoopWithSystems(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            updateLighting();
            updateBallPhysics(deltaTime);
            updateBallControl();
            players.forEach(player => updatePlayerAI(player, deltaTime));
            updateOptimizations(time);

            // Update new systems
            commentarySystem.update(time);
            refereeSystem.update(deltaTime);
            crowdSystem.update(deltaTime);
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoopWithSystems);
}

// Start the enhanced game loop with new systems
enhancedGameLoopWithSystems(0);

// Implement dynamic weather system
const weatherSystem = {
    currentWeather: 'clear',
    transitionTime: 0,
    transitionDuration: 30000, // 30 seconds

    update: function(deltaTime) {
        this.transitionTime += deltaTime * 1000;
        if (this.transitionTime >= this.transitionDuration) {
            this.changeWeather();
        }
        this.updateWeatherEffects();
    },

    changeWeather: function() {
        const weathers = ['clear', 'cloudy', 'rainy', 'foggy'];
        this.currentWeather = weathers[Math.floor(Math.random() * weathers.length)];
        this.transitionTime = 0;
        console.log(`Weather changing to: ${this.currentWeather}`);
    },

    updateWeatherEffects: function() {
        const transitionProgress = this.transitionTime / this.transitionDuration;
        switch (this.currentWeather) {
            case 'clear':
                scene.fog.density = Math.max(0, scene.fog.density - 0.0001 * transitionProgress);
                break;
            case 'cloudy':
                // Adjust lighting for cloudy weather
                break;
            case 'rainy':
                if (!weatherEffects.rain) createRainEffect();
                weatherEffects.rain.visible = true;
                weatherEffects.rain.material.opacity = transitionProgress;
                break;
            case 'foggy':
                scene.fog.density = Math.min(0.05, scene.fog.density + 0.0001 * transitionProgress);
                break;
        }
    }
};

// Implement a basic injury system
const injurySystem = {
    injuredPlayers: new Set(),

    update: function(deltaTime) {
        players.forEach(player => {
            if (this.injuredPlayers.has(player)) {
                this.updateInjuredPlayer(player, deltaTime);
            } else if (Math.random() < 0.0001) {
                this.injurePlayer(player);
            }
        });
    },

    injurePlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} is injured!`);
        this.injuredPlayers.add(player);
        player.speed *= 0.5;
        // Visual indication of injury
        player.material.color.setHex(0xff0000);
    },

    updateInjuredPlayer: function(player, deltaTime) {
        player.injuryTime += deltaTime;
        if (player.injuryTime > 30) { // Recover after 30 seconds
            this.recoverPlayer(player);
        }
    },

    recoverPlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} has recovered!`);
        this.injuredPlayers.delete(player);
        player.speed *= 2;
        player.injuryTime = 0;
        player.material.color.setHex(0xffffff);
    }
};

// Implement a substitution system
const substitutionSystem = {
    benchPlayers: [],
    maxSubstitutions: 3,
    substitutionsMade: 0,

    initialize: function() {
        // Create bench players
        for (let i = 0; i < 7; i++) {
            const benchPlayer = createPlayer(Math.random() < 0.5 ? 'A' : 'B');
            benchPlayer.position.set(0, 0, -40); // Off the field
            this.benchPlayers.push(benchPlayer);
        }
    },

    update: function() {
        if (this.substitutionsMade < this.maxSubstitutions && Math.random() < 0.001) {
            this.makeSubstitution();
        }
    },

    makeSubstitution: function() {
        const teamToSubstitute = Math.random() < 0.5 ? 'A' : 'B';
        const playerToSubstitute = players.find(p => p.team === teamToSubstitute && p.energy < 30);
        const benchPlayer = this.benchPlayers.find(p => p.team === teamToSubstitute);

        if (playerToSubstitute && benchPlayer) {
            console.log(`Substitution: Player ${playerToSubstitute.id} is being replaced by bench player ${benchPlayer.id}`);
            const index = players.indexOf(playerToSubstitute);
            players[index] = benchPlayer;
            benchPlayer.position.copy(playerToSubstitute.position);
            this.benchPlayers = this.benchPlayers.filter(p => p !== benchPlayer);
            this.benchPlayers.push(playerToSubstitute);
            playerToSubstitute.position.set(0, 0, -40); // Move to bench
            this.substitutionsMade++;
        }
    }
};

// Enhance the game loop with new systems
function enhancedGameLoopWithAdvancedSystems(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            updateLighting();
            updateBallPhysics(deltaTime);
            updateBallControl();
            players.forEach(player => updatePlayerAI(player, deltaTime));
            updateOptimizations(time);

            // Update existing systems
            commentarySystem.update(time);
            refereeSystem.update(deltaTime);
            crowdSystem.update(deltaTime);

            // Update new advanced systems
            weatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoopWithAdvancedSystems);
}

// Initialize new systems
weatherSystem.changeWeather(); // Set initial weather
substitutionSystem.initialize();

// Start the enhanced game loop with advanced systems
enhancedGameLoopWithAdvancedSystems(0);

// Implement a more advanced AI system using a state machine
class PlayerAI {
    constructor(player) {
        this.player = player;
        this.currentState = this.idleState;
    }

    update(deltaTime) {
        this.currentState(deltaTime);
    }

    idleState(deltaTime) {
        if (this.player.position.distanceTo(ball.position) < 10) {
            this.currentState = this.chaseBallState;
        } else {
            this.moveToFormationPosition(deltaTime);
        }
    }

    chaseBallState(deltaTime) {
        const direction = ball.position.clone().sub(this.player.position).normalize();
        this.player.position.add(direction.multiplyScalar(this.player.attributes.speed * deltaTime));

        if (this.player.position.distanceTo(ball.position) < 2) {
            this.currentState = this.controlBallState;
        } else if (this.player.position.distanceTo(ball.position) > 15) {
            this.currentState = this.idleState;
        }
    }

    controlBallState(deltaTime) {
        if (this.player.position.distanceTo(ball.position) > 2) {
            this.currentState = this.chaseBallState;
            return;
        }

        const goalPosition = this.player.team === 'A' ? new THREE.Vector3(50, 0, 0) : new THREE.Vector3(-50, 0, 0);
        const directionToGoal = goalPosition.clone().sub(this.player.position).normalize();

        if (Math.random() < 0.05) { // Chance to shoot or pass
            if (this.player.position.distanceTo(goalPosition) < 30 && Math.random() < this.player.attributes.accuracy) {
                shootBall(this.player);
                this.currentState = this.idleState;
            } else {
                this.passBall();
                this.currentState = this.idleState;
            }
        } else {
            // Dribble towards the goal
            this.player.position.add(directionToGoal.multiplyScalar(this.player.attributes.speed * 0.5 * deltaTime));
            ball.position.copy(this.player.position).add(new THREE.Vector3(0, 1, 0));
        }
    }

    moveToFormationPosition(deltaTime) {
        const direction = this.player.formationPosition.clone().sub(this.player.position).normalize();
        this.player.position.add(direction.multiplyScalar(this.player.attributes.speed * 0.5 * deltaTime));
    }

    passBall() {
        const teammates = players.filter(p => p.team === this.player.team && p !== this.player);
        const nearestTeammate = teammates.reduce((nearest, p) => {
            const distance = p.position.distanceTo(this.player.position);
            return (!nearest || distance < nearest.distance) ? {player: p, distance: distance} : nearest;
        }, null).player;

        if (nearestTeammate) {
            const direction = nearestTeammate.position.clone().sub(this.player.position).normalize();
            const strength = 15 + Math.random() * 10;
            ballBody.velocity.set(direction.x * strength, strength / 2, direction.z * strength);
        }
    }
}

// Update player creation to include AI
function createPlayer(team) {
    // ... (existing player creation code)
    player.ai = new PlayerAI(player);
    return player;
}

// Update the player update function
function updatePlayers(deltaTime) {
    players.forEach(player => {
        player.ai.update(deltaTime);
        updatePlayerPhysics(player);
    });
}

// Implement a basic tactics system
const tacticsSystem = {
    currentTactics: {
        A: 'balanced',
        B: 'balanced'
    },
    tacticOptions: ['defensive', 'balanced', 'offensive'],

    update: function(deltaTime) {
        // Randomly change tactics every 5 minutes
        if (Math.random() < 0.001) {
            this.changeTactics('A');
            this.changeTactics('B');
        }
    },

    changeTactics: function(team) {
        const newTactic = this.tacticOptions[Math.floor(Math.random() * this.tacticOptions.length)];
        this.currentTactics[team] = newTactic;
        console.log(`Team ${team} changed tactics to ${newTactic}`);
        this.applyTactics(team);
    },

    applyTactics: function(team) {
        const teamPlayers = players.filter(p => p.team === team);
        const tactic = this.currentTactics[team];

        teamPlayers.forEach(player => {
            switch (tactic) {
                case 'defensive':
                    player.formationPosition.x *= 0.8; // Move closer to own goal
                    break;
                case 'offensive':
                    player.formationPosition.x *= 1.2; // Move closer to opponent's goal
                    break;
                // 'balanced' keeps the default formation
            }
        });
    }
};

// Implement a fatigue system
const fatigueSystem = {
    update: function(deltaTime) {
        players.forEach(player => {
            // Decrease energy based on movement and actions
            const movement = player.position.distanceTo(player.lastPosition || player.position);
            player.energy = Math.max(0, player.energy - movement * 0.1 - deltaTime * 0.5);

            // Recover energy when not moving much
            if (movement < 0.1) {
                player.energy = Math.min(100, player.energy + deltaTime * 2);
            }

            // Update player speed based on energy
            player.attributes.speed = player.baseSpeed * (0.5 + player.energy / 200);

            player.lastPosition = player.position.clone();
        });
    }
};

// Enhance the game loop with the new systems
function enhancedGameLoopWithTacticsAndFatigue(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            updateLighting();
            updateBallPhysics(deltaTime);
            updateBallControl();
            updatePlayers(deltaTime);
            updateOptimizations(time);

            // Update all systems
            commentarySystem.update(time);
            refereeSystem.update(deltaTime);
            crowdSystem.update(deltaTime);
            weatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            tacticsSystem.update(deltaTime);
            fatigueSystem.update(deltaTime);
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoopWithTacticsAndFatigue);
}

// Initialize players with energy and base speed
players.forEach(player => {
    player.energy = 100;
    player.baseSpeed = player.attributes.speed;
});

// Start the enhanced game loop with tactics and fatigue
enhancedGameLoopWithTacticsAndFatigue(0);

// Implement a more advanced ball physics system
const advancedBallPhysics = {
    update: function(deltaTime) {
        // Apply gravity
        ballBody.velocity.y -= 9.81 * deltaTime;

        // Apply air resistance
        const airDensity = 1.2; // kg/m^3
        const dragCoefficient = 0.47; // for a sphere
        const crossSectionalArea = Math.PI * 0.11 * 0.11; // for a standard soccer ball
        const airResistance = ballBody.velocity.clone().negate().normalize().multiplyScalar(
            0.5 * airDensity * dragCoefficient * crossSectionalArea * ballBody.velocity.lengthSq()
        );
        ballBody.applyForce(airResistance, ballBody.position);

        // Apply Magnus effect (spin)
        const liftCoefficient = 0.33; // Typical value for a soccer ball
        const angularVelocity = new CANNON.Vec3(0, 10, 0); // Example spin
        const magnusForce = new CANNON.Vec3().copy(ballBody.velocity).cross(angularVelocity).scale(
            0.5 * airDensity * liftCoefficient * crossSectionalArea * ballBody.velocity.length()
        );
        ballBody.applyForce(magnusForce, ballBody.position);

        // Update position
        ballBody.position.vadd(ballBody.velocity.scale(deltaTime));

        // Handle ground collision with realistic bounce
        if (ballBody.position.y < 0.11) { // Assuming ball radius is 0.11 meters
            ballBody.position.y = 0.11;
            const restitution = 0.8; // Coefficient of restitution for a soccer ball
            ballBody.velocity.y = -ballBody.velocity.y * restitution;

            // Apply friction
            const friction = 0.3;
            ballBody.velocity.x *= (1 - friction);
            ballBody.velocity.z *= (1 - friction);
        }

        // Update Three.js ball position and rotation
        ball.position.copy(ballBody.position);
        ball.quaternion.copy(ballBody.quaternion);
    }
};

// Implement a more realistic player movement system
const advancedPlayerMovement = {
    update: function(deltaTime, player) {
        // Calculate desired velocity
        const desiredVelocity = new THREE.Vector3();
        if (player.movement.forward) desiredVelocity.z -= 1;
        if (player.movement.backward) desiredVelocity.z += 1;
        if (player.movement.left) desiredVelocity.x -= 1;
        if (player.movement.right) desiredVelocity.x += 1;
        desiredVelocity.normalize().multiplyScalar(player.attributes.speed);

        // Apply acceleration
        const acceleration = new THREE.Vector3().subVectors(desiredVelocity, player.velocity);
        acceleration.multiplyScalar(deltaTime * 10); // Adjust for responsiveness

        player.velocity.add(acceleration);

        // Apply friction
        const friction = 0.9;
        player.velocity.multiplyScalar(Math.pow(friction, deltaTime));

        // Update position
        player.position.add(player.velocity.clone().multiplyScalar(deltaTime));

        // Contain player within the field
        player.position.x = Math.max(-50, Math.min(50, player.position.x));
        player.position.z = Math.max(-30, Math.min(30, player.position.z));

        // Update player's look direction
        if (player.velocity.lengthSq() > 0.01) {
            player.lookAt(player.position.clone().add(player.velocity));
        }
    }
};

// Implement a more advanced AI decision-making system
class AdvancedPlayerAI {
    constructor(player) {
        this.player = player;
        this.decisionCooldown = 0;
    }

    update(deltaTime) {
        this.decisionCooldown -= deltaTime;
        if (this.decisionCooldown <= 0) {
            this.makeDecision();
            this.decisionCooldown = 0.5 + Math.random() * 0.5; // Make decisions every 0.5 to 1 second
        }

        this.executeCurrentAction(deltaTime);
    }

    makeDecision() {
        const distanceToBall = this.player.position.distanceTo(ball.position);
        const nearestTeammate = this.findNearestTeammate();
        const nearestOpponent = this.findNearestOpponent();

        if (distanceToBall < 2 && this.player.team === ball.lastTouchedBy) {
            // Player has the ball
            if (this.canShoot()) {
                this.currentAction = 'shoot';
            } else if (this.shouldPass(nearestOpponent)) {
                this.currentAction = 'pass';
                this.passTarget = nearestTeammate;
            } else {
                this.currentAction = 'dribble';
            }
        } else if (distanceToBall < 10 && ball.lastTouchedBy !== this.player.team) {
            // Ball is close and possessed by opponent
            this.currentAction = 'tackle';
        } else if (distanceToBall < 20) {
            // Ball is within range
            this.currentAction = 'moveToBall';
        } else {
            // Ball is far away
            this.currentAction = 'moveToFormation';
        }
    }

    executeCurrentAction(deltaTime) {
        switch (this.currentAction) {
            case 'shoot':
                shootBall(this.player);
                break;
            case 'pass':
                this.passBall(this.passTarget);
                break;
            case 'dribble':
                this.dribble(deltaTime);
                break;
            case 'tackle':
                this.tackle();
                break;
            case 'moveToBall':
                this.moveTowards(ball.position, deltaTime);
                break;
            case 'moveToFormation':
                this.moveTowards(this.player.formationPosition, deltaTime);
                break;
        }
    }

    canShoot() {
        const goalPosition = this.player.team === 'A' ? new THREE.Vector3(50, 0, 0) : new THREE.Vector3(-50, 0, 0);
        return this.player.position.distanceTo(goalPosition) < 30 && Math.random() < this.player.attributes.accuracy;
    }

    shouldPass(nearestOpponent) {
        return nearestOpponent && nearestOpponent.position.distanceTo(this.player.position) < 5;
    }

    dribble(deltaTime) {
        const goalPosition = this.player.team === 'A' ? new THREE.Vector3(50, 0, 0) : new THREE.Vector3(-50, 0, 0);
        const direction = goalPosition.clone().sub(this.player.position).normalize();
        this.player.position.add(direction.multiplyScalar(this.player.attributes.speed * 0.7 * deltaTime));
        ball.position.copy(this.player.position).add(new THREE.Vector3(0, 1, 0));
    }

    tackle() {
        if (this.player.position.distanceTo(ball.position) < 2) {
            const tackleSuccess = Math.random() < this.player.attributes.tackling;
            if (tackleSuccess) {
                ball.lastTouchedBy = this.player.team;
                const kickStrength = 5 + Math.random() * 5;
                const kickDirection = new THREE.Vector3().subVectors(ball.position, this.player.position).normalize();
                ballBody.velocity.copy(kickDirection.multiplyScalar(kickStrength));
            }
        }
    }

    moveTowards(target, deltaTime) {
        const direction = target.clone().sub(this.player.position).normalize();
        this.player.position.add(direction.multiplyScalar(this.player.attributes.speed * deltaTime));
    }

    findNearestTeammate() {
        return players.reduce((nearest, p) => {
            if (p.team === this.player.team && p !== this.player) {
                const distance = p.position.distanceTo(this.player.position);
                return (!nearest || distance < nearest.distance) ? {player: p, distance: distance} : nearest;
            }
            return nearest;
        }, null)?.player;
    }

    findNearestOpponent() {
        return players.reduce((nearest, p) => {
            if (p.team !== this.player.team) {
                const distance = p.position.distanceTo(this.player.position);
                return (!nearest || distance < nearest.distance) ? {player: p, distance: distance} : nearest;
            }
            return nearest;
        }, null)?.player;
    }

    passBall(target) {
        if (target) {
            const direction = target.position.clone().sub(this.player.position).normalize();
            const strength = 15 + Math.random() * 10;
            ballBody.velocity.set(direction.x * strength, strength / 2, direction.z * strength);
            ball.lastTouchedBy = this.player.team;
        }
    }
}

// Update player creation to include advanced AI
function createAdvancedPlayer(team) {
    // ... (existing player creation code)
    player.ai = new AdvancedPlayerAI(player);
    return player;
}

// Update the player update function
function updateAdvancedPlayers(deltaTime) {
    players.forEach(player => {
        player.ai.update(deltaTime);
        advancedPlayerMovement.update(deltaTime, player);
    });
}

// Enhance the game loop with advanced physics and AI
function enhancedGameLoopWithAdvancedPhysicsAndAI(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            updateLighting();
            advancedBallPhysics.update(deltaTime);
            updateAdvancedPlayers(deltaTime);
            updateOptimizations(time);

            // Update all systems
            commentarySystem.update(time);
            refereeSystem.update(deltaTime);
            crowdSystem.update(deltaTime);
            weatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            tacticsSystem.update(deltaTime);
            fatigueSystem.update(deltaTime);
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoopWithAdvancedPhysicsAndAI);
}

// Initialize players with advanced AI and attributes
players = players.map(player => createAdvancedPlayer(player.team));

// Start the enhanced game loop with advanced physics and AI
enhancedGameLoopWithAdvancedPhysicsAndAI(0);

// Implement a more sophisticated team tactics system
const advancedTacticsSystem = {
    formations: {
        '4-4-2': [
            [-45, 0], [-30, -20], [-30, 20], [-15, -10], [-15, 10],
            [-5, -25], [-5, -8], [-5, 8], [-5, 25],
            [10, -10], [10, 10]
        ],
        '4-3-3': [
            [-45, 0], [-30, -20], [-30, 20], [-15, -10], [-15, 10],
            [-5, -20], [-5, 0], [-5, 20],
            [15, -20], [15, 0], [15, 20]
        ],
        '3-5-2': [
            [-45, 0], [-30, -15], [-30, 15],
            [-15, -25], [-15, -8], [-5, 0], [-15, 8], [-15, 25],
            [10, -10], [10, 10]
        ]
    },
    currentFormation: {
        A: '4-4-2',
        B: '4-4-2'
    },
    pressureStyles: ['low', 'medium', 'high'],
    currentPressure: {
        A: 'medium',
        B: 'medium'
    },
    possessionStyles: ['defensive', 'balanced', 'attacking'],
    currentPossessionStyle: {
        A: 'balanced',
        B: 'balanced'
    },

    update: function(deltaTime) {
        // Randomly change tactics every 10 minutes of game time
        if (Math.random() < 0.0005) {
            this.changeTactics('A');
            this.changeTactics('B');
        }
    },

    changeTactics: function(team) {
        const formations = Object.keys(this.formations);
        this.currentFormation[team] = formations[Math.floor(Math.random() * formations.length)];
        this.currentPressure[team] = this.pressureStyles[Math.floor(Math.random() * this.pressureStyles.length)];
        this.currentPossessionStyle[team] = this.possessionStyles[Math.floor(Math.random() * this.possessionStyles.length)];

        console.log(`Team ${team} changed tactics:`, 
            `Formation: ${this.currentFormation[team]},`,
            `Pressure: ${this.currentPressure[team]},`,
            `Possession Style: ${this.currentPossessionStyle[team]}`
        );

        this.applyTactics(team);
    },

    applyTactics: function(team) {
        const teamPlayers = players.filter(p => p.team === team);
        const formation = this.formations[this.currentFormation[team]];
        const pressureStyle = this.currentPressure[team];
        const possessionStyle = this.currentPossessionStyle[team];

        teamPlayers.forEach((player, index) => {
            let [x, z] = formation[index];
            if (team === 'B') {
                x = -x;
                z = -z;
            }

            // Adjust formation based on pressure style
            switch (pressureStyle) {
                case 'low':
                    x *= 0.8;
                    break;
                case 'high':
                    x *= 1.2;
                    break;
            }

            // Adjust formation based on possession style
            switch (possessionStyle) {
                case 'defensive':
                    x -= 5;
                    break;
                case 'attacking':
                    x += 5;
                    break;
            }

            player.formationPosition.set(x, 1, z);
        });
    }
};

// Implement an advanced referee system with dynamic difficulty adjustment
const advancedRefereeSystem = {
    position: new THREE.Vector3(),
    speed: 7,
    foulProbability: 0.01,
    cardProbability: 0.2,
    difficultyAdjustment: 0,

    update: function(deltaTime) {
        // Move the referee
        this.moveReferee(deltaTime);

        // Check for fouls
        this.checkForFouls();

        // Adjust difficulty based on score difference
        this.adjustDifficulty();
    },

    moveReferee: function(deltaTime) {
        const targetPosition = ball.position.clone();
        targetPosition.y = 1; // Keep referee on the ground
        const direction = targetPosition.sub(this.position).normalize();
        this.position.add(direction.multiplyScalar(this.speed * deltaTime));
    },

    checkForFouls: function() {
        players.forEach(player => {
            if (player.position.distanceTo(ball.position) < 2 && Math.random() < this.foulProbability + this.difficultyAdjustment) {
                this.callFoul(player);
            }
        });
    },

    callFoul: function(player) {
        console.log(`Foul called on player ${player.id} from team ${player.team}!`);

        if (Math.random() < this.cardProbability) {
            this.showCard(player);
        } else {
            this.awardFreeKick(player);
        }
    },

    showCard: function(player) {
        if (!player.yellowCard) {
            player.yellowCard = true;
            console.log(`Yellow card shown to player ${player.id} from team ${player.team}!`);
        } else {
            console.log(`Red card shown to player ${player.id} from team ${player.team}! Player sent off!`);
            this.sendPlayerOff(player);
        }
    },

    awardFreeKick: function(player) {
        const foulPosition = ball.position.clone();
        ball.position.copy(foulPosition);
        ballBody.velocity.set(0, 0, 0);
        ballBody.angularVelocity.set(0, 0, 0);

        // TODO: Implement free kick mechanism
    },

    sendPlayerOff: function(player) {
        // Remove player from the game
        const index = players.indexOf(player);
        if (index > -1) {
            players.splice(index, 1);
        }
        scene.remove(player);
    },

    adjustDifficulty: function() {
        const scoreDifference = Math.abs(scores.teamA - scores.teamB);
        this.difficultyAdjustment = scoreDifference * 0.005; // Increase foul probability for the leading team
    }
};

// Implement an advanced commentary system
const advancedCommentarySystem = {
    phrases: {
        goalScored: [
            "GOAL! What a fantastic finish!",
            "He's done it! The ball is in the back of the net!",
            "An absolute screamer! The crowd goes wild!"
        ],
        nearMiss: [
            "Oh, so close! The goalkeeper was beaten there.",
            "What a chance! He'll be disappointed not to score.",
            "The woodwork comes to the rescue! What a let-off for the defense."
        ],
        greatPass: [
            "What a pass! That's vision for you.",
            "Brilliant ball through the defense!",
            "He's split the defense wide open with that pass!"
        ],
        tackle: [
            "Crunching tackle! But he's won the ball cleanly.",
            "Great defensive work there.",
            "He's not going to let the attacker through easily!"
        ],
        save: [
            "What a save! The goalkeeper's keeping his team in this.",
            "Brilliant reflexes from the keeper!",
            "He's denied what looked like a certain goal!"
        ]
    },
    lastCommentaryTime: 0,
    commentaryInterval: 10000, // 10 seconds

    update: function(time) {
        if (time - this.lastCommentaryTime > this.commentaryInterval) {
            this.generateCommentary();
            this.lastCommentaryTime = time;
        }
    },

    generateCommentary: function() {
        const events = this.detectEvents();
        if (events.length > 0) {
            const event = events[Math.floor(Math.random() * events.length)];
            const phrases = this.phrases[event];
            const commentary = phrases[Math.floor(Math.random() * phrases.length)];
            console.log("Commentary: " + commentary);
            // You could also display this in the UI or play it as audio
        }
    },

    detectEvents: function() {
        const events = [];
        if (this.isGoalScored()) events.push('goalScored');
        if (this.isNearMiss()) events.push('nearMiss');
        if (this.isGreatPass()) events.push('greatPass');
        if (this.isTackle()) events.push('tackle');
        if (this.isGreatSave()) events.push('save');
        return events;
    },

    isGoalScored: function() {
        // Check if a goal was scored recently
        return Math.abs(ball.position.x) > 45 && Math.abs(ball.position.z) < 7.32 / 2;
    },

    isNearMiss: function() {
        // Check if the ball narrowly missed the goal
        return Math.abs(ball.position.x) > 40 && Math.abs(ball.position.z) < 10 && Math.abs(ball.position.y) < 3;
    },

    isGreatPass: function() {
        // Check if a long pass was completed successfully
        // This is a simplified check and could be improved
        return ballBody.velocity.length() > 20;
    },

    isTackle: function() {
        // Check if a tackle was made recently
        // This would require keeping track of recent tackles in the game logic
        return false; // Placeholder
    },

    isGreatSave: function() {
        // Check if the goalkeeper made a save recently
        // This would require implementing goalkeeper logic and tracking saves
        return false; // Placeholder
    }
};

// Implement an advanced crowd system
const advancedCrowdSystem = {
    excitement: 0.5,
    homeTeamSupport: 0.6, // 60% of the crowd supports the home team
    awayTeamSupport: 0.4, // 40% supports the away team
    baseVolume: 0.3,
    maxAdditionalVolume: 0.7,
    lastChantTime: 0,
    chantInterval: 30000, // 30 seconds

    update: function(deltaTime, time) {
        this.updateExcitement(deltaTime);
        this.updateCrowdSound();
        this.triggerChants(time);
    },

    updateExcitement: function(deltaTime) {
        // Increase excitement for goals, near misses, fouls, etc.
        if (this.isExcitingEvent()) {
            this.excitement = Math.min(1, this.excitement + 0.2);
        } else {
            this.excitement = Math.max(0.1, this.excitement - 0.02 * deltaTime);
        }
    },

    updateCrowdSound: function() {
        if (crowdSound) {
            const volume = this.baseVolume + this.maxAdditionalVolume * this.excitement;
            crowdSound.setVolume(volume);

            // Adjust stereo panning based on ball position
            const pan = THREE.MathUtils.clamp(ball.position.x / 50, -1, 1);
            crowdSound.stereo(pan);
        }
    },

    triggerChants: function(time) {
        if (time - this.lastChantTime > this.chantInterval) {
            if (Math.random() < this.excitement) {
                this.playChant();
                this.lastChantTime = time;
            }
        }
    },

    playChant: function() {
        const isHomeChant = Math.random() < this.homeTeamSupport;
        const chant = isHomeChant ? this.selectHomeChant() : this.selectAwayChant();
        console.log(`Crowd Chant: ${chant}`);
        // Here you would trigger the actual audio for the chant
    },

    selectHomeChant: function() {
        const homeChants = [
            "Home team, home team!",
            "We are the champions!",
            "Defense! Defense!"
        ];
        return homeChants[Math.floor(Math.random() * homeChants.length)];
    },

    selectAwayChant: function() {
        const awayChants = [
            "Away day! Away day!",
            "We're gonna win!",
            "You'll never walk alone!"
        ];
        return awayChants[Math.floor(Math.random() * awayChants.length)];
    },

    isExcitingEvent: function() {
        // Check for exciting events like goals, near misses, etc.
        // This should be tied into the game events system
        return Math.random() < 0.05; // Placeholder implementation
    }
};

// Enhance the game loop with all advanced systems
function ultimateEnhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            updateLighting();
            advancedBallPhysics.update(deltaTime);
            updateAdvancedPlayers(deltaTime);
            updateOptimizations(time);

            // Update all advanced systems
            advancedTacticsSystem.update(deltaTime);
            advancedRefereeSystem.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            weatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            fatigueSystem.update(deltaTime);
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(ultimateEnhancedGameLoop);
}

// Initialize all advanced systems
advancedTacticsSystem.changeTactics('A');
advancedTacticsSystem.changeTactics('B');

// Start the ultimate enhanced game loop
ultimateEnhancedGameLoop(0);

// Implement dynamic camera system
const dynamicCameraSystem = {
    modes: ['follow', 'tactical', 'cinematic'],
    currentMode: 'follow',
    transitionDuration: 1000,
    lastTransitionTime: 0,

    update: function(deltaTime, time) {
        switch (this.currentMode) {
            case 'follow':
                this.updateFollowCamera(deltaTime);
                break;
            case 'tactical':
                this.updateTacticalCamera(deltaTime);
                break;
            case 'cinematic':
                this.updateCinematicCamera(deltaTime, time);
                break;
        }

        // Randomly switch camera modes
        if (time - this.lastTransitionTime > 30000) { // Switch every 30 seconds
            this.switchCameraMode();
            this.lastTransitionTime = time;
        }
    },

    updateFollowCamera: function(deltaTime) {
        const targetPosition = ball.position.clone().add(new THREE.Vector3(0, 10, 20));
        camera.position.lerp(targetPosition, 0.1);
        camera.lookAt(ball.position);
    },

    updateTacticalCamera: function(deltaTime) {
        const targetPosition = new THREE.Vector3(0, 50, 0);
        camera.position.lerp(targetPosition, 0.05);
        camera.lookAt(new THREE.Vector3(0, 0, 0));
    },

    updateCinematicCamera: function(deltaTime, time) {
        const angle = time * 0.001;
        const radius = 40;
        const height = 20 + Math.sin(time * 0.0005) * 10;
        const x = Math.cos(angle) * radius;
        const z = Math.sin(angle) * radius;
        camera.position.set(x, height, z);
        camera.lookAt(ball.position);
    },

    switchCameraMode: function() {
        const newMode = this.modes[(this.modes.indexOf(this.currentMode) + 1) % this.modes.length];
        console.log(`Switching camera mode to: ${newMode}`);
        this.currentMode = newMode;
    }
};

// Implement advanced player animations
const playerAnimationSystem = {
    animations: {
        idle: null,
        run: null,
        kick: null,
        tackle: null,
        celebrate: null
    },

    loadAnimations: function() {
        const loader = new THREE.FBXLoader();
        loader.load('animations/idle.fbx', (anim) => this.animations.idle = anim);
        loader.load('animations/run.fbx', (anim) => this.animations.run = anim);
        loader.load('animations/kick.fbx', (anim) => this.animations.kick = anim);
        loader.load('animations/tackle.fbx', (anim) => this.animations.tackle = anim);
        loader.load('animations/celebrate.fbx', (anim) => this.animations.celebrate = anim);
    },

    update: function(deltaTime) {
        players.forEach(player => {
            this.updatePlayerAnimation(player, deltaTime);
        });
    },

    updatePlayerAnimation: function(player, deltaTime) {
        if (player.velocity.length() > 0.1) {
            this.playAnimation(player, 'run');
        } else {
            this.playAnimation(player, 'idle');
        }

        // Special animations
        if (player.isKicking) {
            this.playAnimation(player, 'kick');
        } else if (player.isTackling) {
            this.playAnimation(player, 'tackle');
        } else if (player.isCelebrating) {
            this.playAnimation(player, 'celebrate');
        }

        if (player.mixer) {
            player.mixer.update(deltaTime);
        }
    },

    playAnimation: function(player, animationName) {
        if (player.currentAnimation !== animationName) {
            const animation = this.animations[animationName];
            if (animation) {
                if (!player.mixer) {
                    player.mixer = new THREE.AnimationMixer(player);
                }
                const action = player.mixer.clipAction(animation);
                action.reset().play();
                player.currentAnimation = animationName;
            }
        }
    }
};

// Implement advanced ball control system
const advancedBallControlSystem = {
    update: function(deltaTime) {
        players.forEach(player => {
            if (this.isPlayerControllingBall(player)) {
                this.updateBallControl(player, deltaTime);
            }
        });
    },

    isPlayerControllingBall: function(player) {
        return player.position.distanceTo(ball.position) < 2 && ball.lastTouchedBy === player.team;
    },

    updateBallControl: function(player, deltaTime) {
        const controlQuality = player.attributes.ballControl;
        const chanceToLoseBall = 0.1 - (controlQuality * 0.09); // 1% to 10% chance to lose ball per second

        if (Math.random() < chanceToLoseBall * deltaTime) {
            this.loseBallControl(player);
        } else {
            this.moveBallWithPlayer(player, deltaTime);
        }
    },

    loseBallControl: function(player) {
        const kickDirection = new THREE.Vector3(Math.random() - 0.5, 0, Math.random() - 0.5).normalize();
        const kickStrength = 5 + Math.random() * 5;
        ballBody.velocity.copy(kickDirection.multiplyScalar(kickStrength));
        ball.lastTouchedBy = null;
    },

    moveBallWithPlayer: function(player, deltaTime) {
        const offset = new THREE.Vector3(player.direction.x, 0.5, player.direction.z).normalize().multiplyScalar(2);
        ball.position.copy(player.position).add(offset);
        ballBody.position.copy(ball.position);
        ballBody.velocity.set(0, 0, 0);
    }
};

// Implement a more advanced UI system
const advancedUISystem = {
    init: function() {
        this.createScoreboard();
        this.createPlayerStats();
        this.createMatchEvents();
    },

    createScoreboard: function() {
        const scoreboard = document.createElement('div');
        scoreboard.id = 'scoreboard';
        scoreboard.style.position = 'absolute';
        scoreboard.style.top = '10px';
        scoreboard.style.left = '50%';
        scoreboard.style.transform = 'translateX(-50%)';
        scoreboard.style.background = 'rgba(0, 0, 0, 0.7)';
        scoreboard.style.color = 'white';
        scoreboard.style.padding = '10px';
        scoreboard.style.borderRadius = '5px';
        document.body.appendChild(scoreboard);
    },

    createPlayerStats: function() {
        const playerStats = document.createElement('div');
        playerStats.id = 'player-stats';
        playerStats.style.position = 'absolute';
        playerStats.style.top = '10px';
        playerStats.style.right = '10px';
        playerStats.style.background = 'rgba(0, 0, 0, 0.7)';
        playerStats.style.color = 'white';
        playerStats.style.padding = '10px';
        playerStats.style.borderRadius = '5px';
        document.body.appendChild(playerStats);
    },

    createMatchEvents: function() {
        const matchEvents = document.createElement('div');
        matchEvents.id = 'match-events';
        matchEvents.style.position = 'absolute';
        matchEvents.style.bottom = '10px';
        matchEvents.style.left = '10px';
        matchEvents.style.background = 'rgba(0, 0, 0, 0.7)';
        matchEvents.style.color = 'white';
        matchEvents.style.padding = '10px';
        matchEvents.style.borderRadius = '5px';
        matchEvents.style.maxHeight = '150px';
        matchEvents.style.overflowY = 'auto';
        document.body.appendChild(matchEvents);
    },

    update: function() {
        this.updateScoreboard();
        this.updatePlayerStats();
        this.updateMatchEvents();
    },

    updateScoreboard: function() {
        const scoreboard = document.getElementById('scoreboard');
        scoreboard.innerHTML = `Team A ${scores.teamA} - ${scores.teamB} Team B | Time: ${formatTime(gameTime)}`;
    },

    updatePlayerStats: function() {
        const playerStats = document.getElementById('player-stats');
        playerStats.innerHTML = '';
        const selectedPlayer = players.find(p => p.isSelected);
        if (selectedPlayer) {
            playerStats.innerHTML = `
                <h3>Player ${selectedPlayer.id}</h3>
                <p>Team: ${selectedPlayer.team}</p>
                <p>Speed: ${selectedPlayer.attributes.speed.toFixed(2)}</p>
                <p>Stamina: ${selectedPlayer.attributes.stamina.toFixed(2)}</p>
                <p>Ball Control: ${selectedPlayer.attributes.ballControl.toFixed(2)}</p>
            `;
        }
    },

    updateMatchEvents: function() {
        const matchEvents = document.getElementById('match-events');
        // Add new match events here
    }
};

// Enhance the ultimate game loop with new systems
function supremeEnhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            updateLighting();
            advancedBallPhysics.update(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);
            updateOptimizations(time);

            // Update all advanced systems
            advancedTacticsSystem.update(deltaTime);
            advancedRefereeSystem.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            weatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            fatigueSystem.update(deltaTime);
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(supremeEnhancedGameLoop);
}

// Initialize all systems
playerAnimationSystem.loadAnimations();
advancedUISystem.init();

// Start the supreme enhanced game loop
supremeEnhancedGameLoop(0);

// Implement a dynamic weather system
const dynamicWeatherSystem = {
    currentWeather: 'clear',
    transitionProgress: 0,
    transitionDuration: 60000, // 1 minute for weather transitions
    weatherTypes: ['clear', 'cloudy', 'rainy', 'foggy', 'windy'],
    
    update: function(deltaTime) {
        this.transitionProgress += deltaTime;
        if (this.transitionProgress >= this.transitionDuration) {
            this.changeWeather();
        }
        this.updateWeatherEffects(deltaTime);
    },

    changeWeather: function() {
        const newWeather = this.weatherTypes[Math.floor(Math.random() * this.weatherTypes.length)];
        console.log(`Weather changing from ${this.currentWeather} to ${newWeather}`);
        this.currentWeather = newWeather;
        this.transitionProgress = 0;
    },

    updateWeatherEffects: function(deltaTime) {
        const transitionAlpha = this.transitionProgress / this.transitionDuration;
        
        switch (this.currentWeather) {
            case 'clear':
                this.updateClearWeather(transitionAlpha);
                break;
            case 'cloudy':
                this.updateCloudyWeather(transitionAlpha);
                break;
            case 'rainy':
                this.updateRainyWeather(transitionAlpha, deltaTime);
                break;
            case 'foggy':
                this.updateFoggyWeather(transitionAlpha);
                break;
            case 'windy':
                this.updateWindyWeather(transitionAlpha, deltaTime);
                break;
        }
    },

    updateClearWeather: function(alpha) {
        scene.fog.density = Math.max(0, scene.fog.density - 0.0001 * alpha);
        directionalLight.intensity = 1;
    },

    updateCloudyWeather: function(alpha) {
        directionalLight.intensity = 0.7;
        // Add cloud particles or adjust sky texture
    },

    updateRainyWeather: function(alpha, deltaTime) {
        if (!this.rainParticles) {
            this.createRainParticles();
        }
        this.rainParticles.visible = true;
        this.rainParticles.material.opacity = alpha;
        this.animateRainParticles(deltaTime);
        
        // Adjust field texture for wet appearance
        pitch.material.roughness = 0.7 - 0.4 * alpha;
    },

    updateFoggyWeather: function(alpha) {
        scene.fog.density = Math.min(0.05, scene.fog.density + 0.0001 * alpha);
        directionalLight.intensity = 0.5;
    },

    updateWindyWeather: function(alpha, deltaTime) {
        // Animate grass and trees
        if (grassMesh && grassMesh.material.uniforms) {
            grassMesh.material.uniforms.windStrength.value = alpha * 2;
        }
        
        // Affect ball physics
        const windForce = new CANNON.Vec3(alpha * 10, 0, alpha * 10);
        ballBody.applyForce(windForce, ballBody.position);
    },

    createRainParticles: function() {
        const rainGeometry = new THREE.BufferGeometry();
        const rainVertices = [];
        for (let i = 0; i < 15000; i++) {
            rainVertices.push(
                Math.random() * 200 - 100,
                Math.random() * 200 - 100,
                Math.random() * 200 - 100
            );
        }
        rainGeometry.setAttribute('position', new THREE.Float32BufferAttribute(rainVertices, 3));
        const rainMaterial = new THREE.PointsMaterial({
            color: 0xaaaaaa,
            size: 0.1,
            transparent: true
        });
        this.rainParticles = new THREE.Points(rainGeometry, rainMaterial);
        scene.add(this.rainParticles);
    },

    animateRainParticles: function(deltaTime) {
        const positions = this.rainParticles.geometry.attributes.position.array;
        for (let i = 0; i < positions.length; i += 3) {
            positions[i + 1] -= 10 * deltaTime; // Fall speed
            if (positions[i + 1] < -100) {
                positions[i + 1] = 100;
            }
        }
        this.rainParticles.geometry.attributes.position.needsUpdate = true;
    }
};

// Implement advanced ball physics
const advancedBallPhysics = {
    update: function(deltaTime) {
        // Apply Magnus effect (spin)
        const spinAxis = new CANNON.Vec3(1, 0, 0); // Assume spin around x-axis
        const spinStrength = 0.5;
        const velocity = ballBody.velocity;
        const magnusForce = new CANNON.Vec3();
        magnusForce.cross(spinAxis, velocity);
        magnusForce.scale(spinStrength * velocity.length() * deltaTime);
        ballBody.applyForce(magnusForce, ballBody.position);

        // Apply air resistance
        const airDensity = 1.2; // kg/m^3
        const dragCoefficient = 0.47; // for a sphere
        const crossSectionalArea = Math.PI * 0.11 * 0.11; // for a standard soccer ball
        const airResistance = velocity.clone().negate().normalize().scale(
            0.5 * airDensity * dragCoefficient * crossSectionalArea * velocity.lengthSquared()
        );
        ballBody.applyForce(airResistance, ballBody.position);

        // Handle bouncing with energy loss
        if (ballBody.position.y < 0.11 && ballBody.velocity.y < 0) {
            ballBody.velocity.y = -ballBody.velocity.y * 0.8; // 20% energy loss
            // Apply friction
            const friction = 0.3;
            ballBody.velocity.x *= (1 - friction);
            ballBody.velocity.z *= (1 - friction);
        }
    }
};

// Implement an advanced AI decision-making system
class AdvancedAI {
    constructor(player) {
        this.player = player;
        this.decisionTree = this.buildDecisionTree();
    }

    update(deltaTime) {
        const decision = this.decisionTree.makeDecision();
        this.executeDecision(decision, deltaTime);
    }

    buildDecisionTree() {
        return new DecisionTree(
            new DecisionNode('isNearBall',
                new DecisionNode('hasPossession',
                    new DecisionNode('isNearGoal',
                        new ActionNode(() => this.shoot()),
                        new DecisionNode('hasOpenTeammate',
                            new ActionNode(() => this.pass()),
                            new ActionNode(() => this.dribble())
                        )
                    ),
                    new ActionNode(() => this.tackle())
                ),
                new ActionNode(() => this.moveTowardsBall())
            )
        );
    }

    executeDecision(decision, deltaTime) {
        switch (decision) {
            case 'shoot':
                this.shoot();
                break;
            case 'pass':
                this.pass();
                break;
            case 'dribble':
                this.dribble(deltaTime);
                break;
            case 'tackle':
                this.tackle();
                break;
            case 'moveTowardsBall':
                this.moveTowardsBall(deltaTime);
                break;
        }
    }

    isNearBall() {
        return this.player.position.distanceTo(ball.position) < 5;
    }

    hasPossession() {
        return this.player.team === ball.lastTouchedBy;
    }

    isNearGoal() {
        const ownGoal = this.player.team === 'A' ? new THREE.Vector3(50, 0, 0) : new THREE.Vector3(-50, 0, 0);
        return this.player.position.distanceTo(ownGoal) < 30;
    }

    hasOpenTeammate() {
        const teammates = players.filter(p => p.team === this.player.team && p !== this.player);
        return teammates.some(teammate => {
            const direction = teammate.position.clone().sub(this.player.position).normalize();
            const ray = new THREE.Raycaster(this.player.position, direction);
            const intersects = ray.intersectObjects(players.filter(p => p.team !== this.player.team));
            return intersects.length === 0;
        });
    }

    shoot() {
        const ownGoal = this.player.team === 'A' ? new THREE.Vector3(50, 0, 0) : new THREE.Vector3(-50, 0, 0);
        const direction = ownGoal.clone().sub(this.player.position).normalize();
        const strength = 25 + Math.random() * 10;
        ballBody.velocity.copy(direction.multiplyScalar(strength));
        ball.lastTouchedBy = this.player.team;
    }

    pass() {
        const teammates = players.filter(p => p.team === this.player.team && p !== this.player);
        const nearestTeammate = teammates.reduce((nearest, t) => {
            const distance = t.position.distanceTo(this.player.position);
            return (!nearest || distance < nearest.distance) ? {player: t, distance: distance} : nearest;
        }, null).player;

        const direction = nearestTeammate.position.clone().sub(this.player.position).normalize();
        const strength = 15 + Math.random() * 10;
        ballBody.velocity.copy(direction.multiplyScalar(strength));
        ball.lastTouchedBy = this.player.team;
    }

    dribble(deltaTime) {
        const ownGoal = this.player.team === 'A' ? new THREE.Vector3(50, 0, 0) : new THREE.Vector3(-50, 0, 0);
        const direction = ownGoal.clone().sub(this.player.position).normalize();
        this.player.position.add(direction.multiplyScalar(this.player.attributes.speed * deltaTime));
        ball.position.copy(this.player.position).add(new THREE.Vector3(0, 1, 0));
        ballBody.position.copy(ball.position);
        ballBody.velocity.set(0, 0, 0);
    }

    tackle() {
        if (this.player.position.distanceTo(ball.position) < 2) {
            const tackleSuccess = Math.random() < this.player.attributes.tackling;
            if (tackleSuccess) {
                ball.lastTouchedBy = this.player.team;
                const kickStrength = 5 + Math.random() * 5;
                const kickDirection = new THREE.Vector3().subVectors(ball.position, this.player.position).normalize();
                ballBody.velocity.copy(kickDirection.multiplyScalar(kickStrength));
            }
        }
    }

    moveTowardsBall(deltaTime) {
        const direction = ball.position.clone().sub(this.player.position).normalize();
        this.player.position.add(direction.multiplyScalar(this.player.attributes.speed * deltaTime));
    }
}

class DecisionTree {
    constructor(root) {
        this.root = root;
    }

    makeDecision() {
        return this.root.makeDecision();
    }
}

class DecisionNode {
    constructor(condition, trueBranch, falseBranch) {
        this.condition = condition;
        this.trueBranch = trueBranch;
        this.falseBranch = falseBranch;
    }

    makeDecision() {
        if (this[this.condition]()) {
            return this.trueBranch.makeDecision();
        } else {
            return this.falseBranch.makeDecision();
        }
    }
}

class ActionNode {
    constructor(action) {
        this.action = action;
    }

    makeDecision() {
        return this.action();
    }
}

// Update player creation to use advanced AI
function createAdvancedPlayer(team) {
    const player = createPlayer(team);
    player.ai = new AdvancedAI(player);
    return player;
}

// Implement a more sophisticated team tactics system
const advancedTeamTacticsSystem = {
    formations: {
        '4-4-2': [
            [-45, 0], [-30, -20], [-30, 20], [-15, -10], [-15, 10],
            [-5, -25], [-5, -8], [-5, 8], [-5, 25],
            [10, -10], [10, 10]
        ],
        '4-3-3': [
            [-45, 0], [-30, -20], [-30, 20], [-15, -10], [-15, 10],
            [-5, -20], [-5, 0], [-5, 20],
            [15, -20], [15, 0], [15, 20]
        ],
        '3-5-2': [
            [-45, 0], [-30, -15], [-30, 15],
            [-15, -25], [-15, -8], [-5, 0], [-15, 8], [-15, 25],
            [10, -10], [10, 10]
        ]
    },
    currentFormation: {
        A: '4-4-2',
        B: '4-4-2'
    },
    playstyles: ['defensive', 'balanced', 'attacking'],
    currentPlaystyle: {
        A: 'balanced',
        B: 'balanced'
    },

    update: function(deltaTime) {
        if (Math.random() < 0.001) { // Chance to change tactics every frame
            this.changeTactics('A');
            this.changeTactics('B');
        }
        this.applyTactics('A');
        this.applyTactics('B');
    },

    changeTactics: function(team) {
        const formations = Object.keys(this.formations);
        this.currentFormation[team] = formations[Math.floor(Math.random() * formations.length)];
        this.currentPlaystyle[team] = this.playstyles[Math.floor(Math.random() * this.playstyles.length)];
        console.log(`Team ${team} changed tactics: ${this.currentFormation[team]}, ${this.currentPlaystyle[team]}`);
    },

    applyTactics: function(team) {
        const teamPlayers = players.filter(p => p.team === team);
        const formation = this.formations[this.currentFormation[team]];
        const playstyle = this.currentPlaystyle[team];

        teamPlayers.forEach((player, index) => {
            let [x, z] = formation[index];
            if (team === 'B') {
                x = -x;
                z = -z;
            }

            // Adjust positions based on playstyle
            switch (playstyle) {
                case 'defensive':
                    x *= 0.8;
                    break;
                case 'attacking':
                    x *= 1.2;
                    break;
            }

            player.formationPosition = new THREE.Vector3(x, 1, z);
        });
    }
};

// Enhance the supreme game loop with new systems
function ultimateEnhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            updateLighting();
            advancedBallPhysics.update(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);
            updateOptimizations(time);

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedRefereeSystem.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            fatigueSystem.update(deltaTime);
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(ultimateEnhancedGameLoop);
}

// Initialize all systems
playerAnimationSystem.loadAnimations();
advancedUISystem.init();
dynamicWeatherSystem.changeWeather();

// Start the ultimate enhanced game loop
ultimateEnhancedGameLoop(0);

// Add post-processing effects
const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
composer.addPass(bloomPass);

const filmPass = new FilmPass(0.35, 0.5, 2048, false);
composer.addPass(filmPass);

// Implement shader-based grass
const grassShaderMaterial = new THREE.ShaderMaterial({
    vertexShader: `
        uniform float time;
        uniform float windStrength;
        varying vec2 vUv;
        
        void main() {
            vUv = uv;
            vec3 pos = position;
            
            // Grass movement
            float movement = sin(time * 2.0 + position.x * 0.5 + position.z * 0.5) * windStrength;
            pos.x += movement;
            
            gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
        }
    `,
    fragmentShader: `
        uniform sampler2D grassTexture;
        varying vec2 vUv;
        
        void main() {
            vec4 texColor = texture2D(grassTexture, vUv);
            gl_FragColor = texColor;
        }
    `,
    uniforms: {
        time: { value: 0 },
        windStrength: { value: 0.1 },
        grassTexture: { value: new THREE.TextureLoader().load('textures/grass.jpg') }
    }
});

const grassMesh = new THREE.Mesh(
    new THREE.PlaneGeometry(100, 60, 100, 60),
    grassShaderMaterial
);
grassMesh.rotation.x = -Math.PI / 2;
scene.add(grassMesh);

// Implement advanced crowd rendering
const crowdSystem = {
    spectators: [],
    textures: [],

    init: function() {
        // Load spectator textures
        for (let i = 0; i < 5; i++) {
            this.textures.push(new THREE.TextureLoader().load(`textures/spectator${i}.jpg`));
        }

        // Create spectators
        for (let i = 0; i < 10000; i++) {
            const spectator = this.createSpectator();
            this.spectators.push(spectator);
            scene.add(spectator);
        }
    },

    createSpectator: function() {
        const geometry = new THREE.PlaneGeometry(1, 2);
        const material = new THREE.MeshBasicMaterial({
            map: this.textures[Math.floor(Math.random() * this.textures.length)],
            side: THREE.DoubleSide
        });
        const spectator = new THREE.Mesh(geometry, material);

        // Position spectator in the stands
        const angle = Math.random() * Math.PI * 2;
        const radius = 60 + Math.random() * 20;
        spectator.position.set(
            Math.cos(angle) * radius,
            10 + Math.random() * 10,
            Math.sin(angle) * radius
        );
        spectator.lookAt(new THREE.Vector3(0, 0, 0));

        return spectator;
    },

    update: function(deltaTime) {
        // Animate spectators (e.g., slight movement, color change on goals)
        this.spectators.forEach(spectator => {
            spectator.position.y += Math.sin(Date.now() * 0.001 + spectator.position.x) * 0.01;
        });
    }
};

// Initialize crowd system
crowdSystem.init();

// Enhance the ultimate game loop with crowd system
function supremeEnhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            updateLighting();
            advancedBallPhysics.update(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);
            updateOptimizations(time);

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedRefereeSystem.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            fatigueSystem.update(deltaTime);
            crowdSystem.update(deltaTime);

            // Update grass shader
            grassShaderMaterial.uniforms.time.value = time * 0.001;
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(supremeEnhancedGameLoop);
}

// Start the supreme enhanced game loop
supremeEnhancedGameLoop(0);

// Implement a dynamic lighting system
const dynamicLightingSystem = {
    sunLight: null,
    ambientLight: null,
    time: 0,

    init: function() {
        this.sunLight = new THREE.DirectionalLight(0xffffff, 1);
        this.sunLight.position.set(0, 100, 0);
        this.sunLight.castShadow = true;
        scene.add(this.sunLight);

        this.ambientLight = new THREE.AmbientLight(0x404040, 0.5);
        scene.add(this.ambientLight);
    },

    update: function(deltaTime) {
        this.time += deltaTime;

        // Simulate day-night cycle
        const angle = (this.time % 300) / 300 * Math.PI * 2;
        const height = Math.sin(angle) * 100;
        const horizontal = Math.cos(angle) * 100;

        this.sunLight.position.set(horizontal, height, 0);

        // Adjust light intensity based on sun position
        const intensity = Math.max(0, (height + 100) / 200);
        this.sunLight.intensity = intensity;
        this.ambientLight.intensity = 0.2 + intensity * 0.3;

        // Adjust shadow properties
        if (this.sunLight.shadow) {
            this.sunLight.shadow.mapSize.width = 2048;
            this.sunLight.shadow.mapSize.height = 2048;
            this.sunLight.shadow.camera.near = 0.5;
            this.sunLight.shadow.camera.far = 500;
        }
    }
};

// Initialize dynamic lighting system
dynamicLightingSystem.init();

// Implement a replay system
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 30 * 60, // 30 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballVelocity: ballBody.velocity.clone(),
            players: players.map(player => ({
                position: player.position.clone(),
                rotation: player.rotation.clone()
            }))
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ballBody.velocity.copy(frame.ballVelocity);

            players.forEach((player, index) => {
                player.position.copy(frame.players[index].position);
                player.rotation.copy(frame.players[index].rotation);
            });

            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Implement advanced sound system
const advancedSoundSystem = {
    listener: null,
    sounds: {},

    init: function() {
        this.listener = new THREE.AudioListener();
        camera.add(this.listener);

        this.loadSound('crowd', 'sounds/crowd_ambient.mp3', true, 0.5);
        this.loadSound('kick', 'sounds/ball_kick.mp3', false, 1);
        this.loadSound('whistle', 'sounds/whistle.mp3', false, 1);
        this.loadSound('goal', 'sounds/goal_cheer.mp3', false, 1);
    },

    loadSound: function(name, file, loop, volume) {
        const sound = new THREE.Audio(this.listener);
        const audioLoader = new THREE.AudioLoader();
        audioLoader.load(file, (buffer) => {
            sound.setBuffer(buffer);
            sound.setLoop(loop);
            sound.setVolume(volume);
        });
        this.sounds[name] = sound;
    },

    play: function(name) {
        if (this.sounds[name]) {
            this.sounds[name].play();
        }
    },

    update: function() {
        // Update 3D sound positions
        if (this.sounds.kick) {
            this.sounds.kick.position.copy(ball.position);
        }
    }
};

// Initialize advanced sound system
advancedSoundSystem.init();

// Implement particle effects system
const particleSystem = {
    systems: [],

    createEffect: function(position, color, count, speed, size, duration) {
        const particles = new THREE.Geometry();
        const pMaterial = new THREE.PointsMaterial({
            color: color,
            size: size,
            transparent: true,
            opacity: 0.8
        });

        for (let i = 0; i < count; i++) {
            const particle = new THREE.Vector3(
                position.x + Math.random() - 0.5,
                position.y + Math.random() - 0.5,
                position.z + Math.random() - 0.5
            );
            particle.velocity = new THREE.Vector3(
                (Math.random() - 0.5) * speed,
                (Math.random() - 0.5) * speed,
                (Math.random() - 0.5) * speed
            );
            particles.vertices.push(particle);
        }

        const particleSystem = new THREE.Points(particles, pMaterial);
        particleSystem.startTime = Date.now();
        particleSystem.duration = duration;

        this.systems.push(particleSystem);
        scene.add(particleSystem);
    },

    update: function(deltaTime) {
        this.systems.forEach((system, index) => {
            const particles = system.geometry.vertices;
            particles.forEach(particle => {
                particle.add(particle.velocity.clone().multiplyScalar(deltaTime));
            });
            system.geometry.verticesNeed

Update = true;

            // Remove expired systems
            if (Date.now() - system.startTime > system.duration) {
                scene.remove(system);
                this.systems.splice(index, 1);
            }
        });
    },

    createGoalEffect: function() {
        const goalPosition = ball.position.clone();
        this.createEffect(goalPosition, 0xFFD700, 1000, 10, 0.1, 3000);
    }
};

// Enhance the supreme game loop with new systems
function ultimateEnhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            advancedBallPhysics.update(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);
            updateOptimizations(time);

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedRefereeSystem.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            fatigueSystem.update(deltaTime);
            crowdSystem.update(deltaTime);
            advancedSoundSystem.update();
            particleSystem.update(deltaTime);

            // Record replay frame
            replaySystem.recordFrame();

            // Update grass shader
            grassShaderMaterial.uniforms.time.value = time * 0.001;
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(ultimateEnhancedGameLoop);
}

// Implement a more advanced goal detection system
function checkForGoals() {
    const goalPosts = [
        { x: -50, z: -3.66, team: 'B' },
        { x: -50, z: 3.66, team: 'B' },
        { x: 50, z: -3.66, team: 'A' },
        { x: 50, z: 3.66, team: 'A' }
    ];

    const ballRadius = 0.11; // Assuming the ball radius is 11cm

    goalPosts.forEach(post => {
        const distanceToPost = Math.sqrt(
            Math.pow(ball.position.x - post.x, 2) +
            Math.pow(ball.position.z - post.z, 2)
        );

        if (distanceToPost <= ballRadius && Math.abs(ball.position.y - 1.22) <= 1.22) {
            // Goal scored
            const scoringTeam = post.team === 'A' ? 'B' : 'A';
            scores[scoringTeam]++;
            advancedCommentarySystem.announceGoal(scoringTeam);
            advancedSoundSystem.play('goal');
            particleSystem.createGoalEffect();
            resetAfterGoal();
        }
    });
}

// Implement an advanced ball spin system
const ballSpinSystem = {
    spinAxis: new THREE.Vector3(1, 0, 0),
    spinStrength: 0,

    update: function(deltaTime) {
        // Apply spin to the ball's rotation
        const rotationDelta = new THREE.Quaternion().setFromAxisAngle(
            this.spinAxis,
            this.spinStrength * deltaTime
        );
        ball.quaternion.multiply(rotationDelta);

        // Gradually reduce spin over time
        this.spinStrength *= 0.99;
    },

    applyKickSpin: function(kickDirection, kickStrength) {
        // Calculate spin axis perpendicular to kick direction
        this.spinAxis.crossVectors(kickDirection, new THREE.Vector3(0, 1, 0)).normalize();
        
        // Set spin strength based on kick strength
        this.spinStrength = kickStrength * 5; // Adjust multiplier as needed
    }
};

// Enhance the ball physics update
function updateBallPhysics(deltaTime) {
    advancedBallPhysics.update(deltaTime);
    ballSpinSystem.update(deltaTime);
}

// Implement an offside detection system
const offsideSystem = {
    checkOffside: function(team) {
        const attackingPlayers = players.filter(p => p.team === team);
        const defendingPlayers = players.filter(p => p.team !== team);

        // Find the second-last defender
        const sortedDefenders = defendingPlayers.sort((a, b) => {
            const goalLine = team === 'A' ? 50 : -50;
            return Math.abs(b.position.x - goalLine) - Math.abs(a.position.x - goalLine);
        });
        const secondLastDefenderX = sortedDefenders[1].position.x;

        // Check if any attacking player is offside
        attackingPlayers.forEach(player => {
            const isOffside = team === 'A' ?
                player.position.x > secondLastDefenderX && player.position.x > ball.position.x :
                player.position.x < secondLastDefenderX && player.position.x < ball.position.x;

            if (isOffside) {
                player.isOffside = true;
                console.log(`Player ${player.id} from team ${team} is offside!`);
            } else {
                player.isOffside = false;
            }
        });
    },

    update: function() {
        this.checkOffside('A');
        this.checkOffside('B');
    }
};

// Implement a more sophisticated referee decision-making system
const advancedRefereeSystem = {
    position: new THREE.Vector3(),
    speed: 7,
    foulProbability: 0.01,
    cardProbability: 0.2,
    VAR: {
        isReviewing: false,
        reviewDuration: 5, // seconds
        reviewTimer: 0,
    },

    update: function(deltaTime) {
        this.moveReferee(deltaTime);
        this.checkForFouls();
        this.updateVAR(deltaTime);
    },

    moveReferee: function(deltaTime) {
        const targetPosition = ball.position.clone();
        targetPosition.y = 1; // Keep referee on the ground
        const direction = targetPosition.sub(this.position).normalize();
        this.position.add(direction.multiplyScalar(this.speed * deltaTime));
    },

    checkForFouls: function() {
        players.forEach(player => {
            if (player.position.distanceTo(ball.position) < 2 && Math.random() < this.foulProbability) {
                this.callFoul(player);
            }
        });
    },

    callFoul: function(player) {
        console.log(`Foul called on player ${player.id} from team ${player.team}!`);

        if (Math.random() < this.cardProbability) {
            this.showCard(player);
        } else {
            this.awardFreeKick(player);
        }
    },

    showCard: function(player) {
        if (!player.yellowCard) {
            player.yellowCard = true;
            console.log(`Yellow card shown to player ${player.id} from team ${player.team}!`);
        } else {
            console.log(`Red card shown to player ${player.id} from team ${player.team}! Player sent off!`);
            this.sendPlayerOff(player);
        }
    },

    awardFreeKick: function(player) {
        const foulPosition = ball.position.clone();
        ball.position.copy(foulPosition);
        ballBody.velocity.set(0, 0, 0);
        ballBody.angularVelocity.set(0, 0, 0);

        // TODO: Implement free kick mechanism
    },

    sendPlayerOff: function(player) {
        // Remove player from the game
        const index = players.indexOf(player);
        if (index > -1) {
            players.splice(index, 1);
        }
        scene.remove(player);
    },

    initiateVARReview: function() {
        this.VAR.isReviewing = true;
        this.VAR.reviewTimer = this.VAR.reviewDuration;
        console.log("VAR review initiated!");
    },

    updateVAR: function(deltaTime) {
        if (this.VAR.isReviewing) {
            this.VAR.reviewTimer -= deltaTime;
            if (this.VAR.reviewTimer <= 0) {
                this.concludeVARReview();
            }
        }
    },

    concludeVARReview: function() {
        this.VAR.isReviewing = false;
        const decision = Math.random() < 0.5 ? "upheld" : "overturned";
        console.log(`VAR decision: ${decision}`);
        // Implement the effects of the VAR decision
    }
};

// Enhance the game loop with the new systems
function ultimateEnhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            updateBallPhysics(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);
            updateOptimizations(time);

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedRefereeSystem.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            fatigueSystem.update(deltaTime);
            crowdSystem.update(deltaTime);
            advancedSoundSystem.update();
            particleSystem.update(deltaTime);
            offsideSystem.update();

            // Record replay frame
            replaySystem.recordFrame();

            // Update grass shader
            grassShaderMaterial.uniforms.time.value = time * 0.001;

            // Check for goals
            checkForGoals();
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(ultimateEnhancedGameLoop);
}

// Start the ultimate enhanced game loop
ultimateEnhancedGameLoop(0);

// Implement advanced player fatigue system
const advancedFatigueSystem = {
    update: function(deltaTime) {
        players.forEach(player => {
            // Calculate energy expenditure based on player's actions
            const distanceMoved = player.position.distanceTo(player.lastPosition || player.position);
            const energyExpended = distanceMoved * 0.1 + (player.isSprinting ? 0.2 : 0) + (player.isKicking ? 0.5 : 0);

            // Reduce player's stamina
            player.stamina = Math.max(0, player.stamina - energyExpended);

            // Affect player's attributes based on stamina
            player.attributes.speed = player.baseSpeed * (0.5 + player.stamina / 200);
            player.attributes.accuracy = player.baseAccuracy * (0.7 + player.stamina / 300);

            // Slowly regenerate stamina when player is not moving much
            if (distanceMoved < 0.1) {
                player.stamina = Math.min(100, player.stamina + deltaTime * 2);
            }

            player.lastPosition = player.position.clone();
        });
    }
};

// Implement advanced ball control system
const advancedBallControlSystem = {
    update: function(deltaTime) {
        players.forEach(player => {
            if (this.isPlayerControllingBall(player)) {
                this.updateBallControl(player, deltaTime);
            }
        });
    },

    isPlayerControllingBall: function(player) {
        return player.position.distanceTo(ball.position) < 2 && ball.lastTouchedBy === player.team;
    },

    updateBallControl: function(player, deltaTime) {
        const controlQuality = player.attributes.ballControl * (player.stamina / 100);
        const chanceToLoseBall = 0.1 - (controlQuality * 0.09); // 1% to 10% chance to lose ball per second

        if (Math.random() < chanceToLoseBall * deltaTime) {
            this.loseBallControl(player);
        } else {
            this.moveBallWithPlayer(player, deltaTime);
        }
    },

    loseBallControl: function(player) {
        const kickDirection = new THREE.Vector3(Math.random() - 0.5, 0, Math.random() - 0.5).normalize();
        const kickStrength = 5 + Math.random() * 5;
        ballBody.velocity.copy(kickDirection.multiplyScalar(kickStrength));
        ball.lastTouchedBy = null;
        ballSpinSystem.applyKickSpin(kickDirection, kickStrength);
    },

    moveBallWithPlayer: function(player, deltaTime) {
        const offset = new THREE.Vector3(player.direction.x, 0.5, player.direction.z).normalize().multiplyScalar(2);
        ball.position.copy(player.position).add(offset);
        ballBody.position.copy(ball.position);
        ballBody.velocity.set(0, 0, 0);
    }
};

// Implement advanced AI for goalkeeper
class GoalkeeperAI extends AdvancedAI {
    constructor(player) {
        super(player);
    }

    update(deltaTime) {
        const decision = this.makeDecision();
        this.executeDecision(decision, deltaTime);
    }

    makeDecision() {
        if (this.isBallInDangerZone()) {
            return this.decideSaveAction();
        } else {
            return this.decidePositioning();
        }
    }

    isBallInDangerZone() {
        const dangerZoneWidth = 20;
        const dangerZoneDepth = 10;
        const goalX = this.player.team === 'A' ? -50 : 50;
        
        return Math.abs(ball.position.x - goalX) < dangerZoneDepth &&
               Math.abs(ball.position.z) < dangerZoneWidth / 2;
    }

    decideSaveAction() {
        const distanceToBall = this.player.position.distanceTo(ball.position);
        if (distanceToBall < 3) {
            return 'dive';
        } else {
            return 'rush';
        }
    }

    decidePositioning() {
        return 'position';
    }

    executeDecision(decision, deltaTime) {
        switch (decision) {
            case 'dive':
                this.dive();
                break;
            case 'rush':
                this.rushToBall(deltaTime);
                break;
            case 'position':
                this.positionInGoal(deltaTime);
                break;
        }
    }

    dive() {
        const diveDirection = ball.position.clone().sub(this.player.position).normalize();
        this.player.position.add(diveDirection.multiplyScalar(5));
        // Trigger dive animation
        playerAnimationSystem.playAnimation(this.player, 'dive');
    }

    rushToBall(deltaTime) {
        const direction = ball.position.clone().sub(this.player.position).normalize();
        this.player.position.add(direction.multiplyScalar(this.player.attributes.speed * 1.5 * deltaTime));
    }

    positionInGoal(deltaTime) {
        const goalX = this.player.team === 'A' ? -50 : 50;
        const targetPosition = new THREE.Vector3(goalX, 1, ball.position.z * 0.5); // Stay halfway between center and ball's z position
        const direction = targetPosition.sub(this.player.position).normalize();
        this.player.position.add(direction.multiplyScalar(this.player.attributes.speed * deltaTime));
    }
}

// Update player creation to include goalkeeper AI
function createAdvancedPlayer(team, isGoalkeeper = false) {
    const player = createPlayer(team);
    player.ai = isGoalkeeper ? new GoalkeeperAI(player) : new AdvancedAI(player);
    if (isGoalkeeper) {
        player.attributes.reflexes = Math.random() * 0.5 + 0.5;
        player.isGoalkeeper = true;
    }
    return player;
}

// Implement advanced set-piece system
const setPieceSystem = {
    currentSetPiece: null,

    startCornerKick: function(team) {
        this.currentSetPiece = {
            type: 'corner',
            team: team,
            position: new THREE.Vector3(team === 'A' ? 50 : -50, 0, 30 * (Math.random() > 0.5 ? 1 : -1))
        };
        this.positionPlayersForCorner(team);
    },

    startFreeKick: function(team, position) {
        this.currentSetPiece = {
            type: 'freekick',
            team: team,
            position: position
        };
        this.positionPlayersForFreeKick(team, position);
    },

    positionPlayersForCorner: function(team) {
        const attackingPlayers = players.filter(p => p.team === team);
        const defendingPlayers = players.filter(p => p.team !== team);

        // Position attacking players in the box
        attackingPlayers.forEach((player, index) => {
            player.position.set(
                this.currentSetPiece.position.x + (team === 'A' ? -10 : 10),
                1,
                (index - 5) * 3
            );
        });

        // Position defending players
        defendingPlayers.forEach((player, index) => {
            player.position.set(
                this.currentSetPiece.position.x + (team === 'A' ? -5 : 5),
                1,
                (index - 5) * 2.5
            );
        });
    },

    positionPlayersForFreeKick: function(team, position) {
        const attackingPlayers = players.filter(p => p.team === team);
        const defendingPlayers = players.filter(p => p.team !== team);

        // Position attacking players
        attackingPlayers.forEach((player, index) => {
            player.position.set(
                position.x + (team === 'A' ? 5 : -5),
                1,
                position.z + (index - 5) * 2
            );
        });

        // Position defending players in a wall if close to goal
        const isCloseToGoal = Math.abs(position.x) > 35;
        if (isCloseToGoal) {
            defendingPlayers.forEach((player, index) => {
                player.position.set(
                    position.x + (team === 'A' ? -2 : 2),
                    1,
                    position.z + (index - 2) * 0.5
                );
            });
        } else {
            // Otherwise, position them normally
            defendingPlayers.forEach((player, index) => {
                player.position.set(
                    position.x + (team === 'A' ? -10 : 10),
                    1,
                    position.z + (index - 5) * 2
                );
            });
        }
    },

    executeSetPiece: function() {
        if (!this.currentSetPiece) return;

        const kickingPlayer = players.find(p => p.team === this.currentSetPiece.team);
        ball.position.copy(this.currentSetPiece.position);
        ballBody.position.copy(ball.position);
        ballBody.velocity.set(0, 0, 0);

        // Determine kick direction and strength
        const kickDirection = new THREE.Vector3(
            this.currentSetPiece.team === 'A' ? 1 : -1,
            0.5,
            (Math.random() - 0.5) * 0.5
        ).normalize();
        const kickStrength = 20 + Math.random() * 10;

        // Apply kick
        ballBody.velocity.copy(kickDirection.multiplyScalar(kickStrength));
        ballSpinSystem.applyKickSpin(kickDirection, kickStrength);

        this.currentSetPiece = null;
    }
};

// Enhance the game loop with new systems
function ultimateEnhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            updateBallPhysics(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);
            updateOptimizations(time);

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedRefereeSystem.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            injurySystem.update(deltaTime);
            substitutionSystem.update();
            advancedFatigueSystem.update(deltaTime);
            crowdSystem.update(deltaTime);
            advancedSoundSystem.update();
            particleSystem.update(deltaTime);
            offsideSystem.update();

            // Handle set pieces
            if (setPieceSystem.currentSetPiece) {
                setPieceSystem.executeSetPiece();
            }

            // Record replay frame
            replaySystem.recordFrame();

            // Update grass shader
            grassShaderMaterial.uniforms.time.value = time * 0.001;

            // Check for goals
            checkForGoals();
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(ultimateEnhancedGameLoop);
}

// Implement advanced player injuries
const advancedInjurySystem = {
    injuredPlayers: new Set(),

    update: function(deltaTime) {
        players.forEach(player => {
            if (this.injuredPlayers.has(player)) {
                this.updateInjuredPlayer(player, deltaTime);
            } else if (Math.random() < 0.0001) { // Small chance of injury each frame
                this.injurePlayer(player);
            }
        });
    },

    injurePlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} is injured!`);
        this.injuredPlayers.add(player);
        player.attributes.speed *= 0.5;
        player.attributes.stamina *= 0.5;
        player.injuryTime = 0;
        player.injurySeverity = Math.random(); // 0 to 1, where 1 is most severe

        // Visual indication of injury
        player.material.color.setHex(0xff0000);

        // Trigger injury animation
        playerAnimationSystem.playAnimation(player, 'injured');

        // Notify commentary system
        advancedCommentarySystem.announceInjury(player);

        // Check if substitution is needed
        if (player.injurySeverity > 0.7) {
            substitutionSystem.requestSubstitution(player.team);
        }
    },

    updateInjuredPlayer: function(player, deltaTime) {
        player.injuryTime += deltaTime;
        const recoveryTime = 30 + player.injurySeverity * 90; // 30 to 120 seconds recovery time

        if (player.injuryTime > recoveryTime) {
            this.recoverPlayer(player);
        }
    },

    recoverPlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} has recovered!`);
        this.injuredPlayers.delete(player);
        player.attributes.speed /= 0.5;
        player.attributes.stamina /= 0.5;
        player.injuryTime = 0;
        player.injurySeverity = 0;
        player.material.color.setHex(0xffffff);

        // Notify commentary system
        advancedCommentarySystem.announceRecovery(player);
    }
};

// Implement advanced substitution system
const advancedSubstitutionSystem = {
    substitutionsLeft: { A: 3, B: 3 },
    benchPlayers: { A: [], B: [] },

    initialize: function() {
        // Create bench players
        for (let team of ['A', 'B']) {
            for (let i = 0; i < 7; i++) {
                const benchPlayer = createAdvancedPlayer(team);
                benchPlayer.position.set(0, 0, -40); // Off the field
                this.benchPlayers[team].push(benchPlayer);
            }
        }
    },

    requestSubstitution: function(team) {
        if (this.substitutionsLeft[team] > 0) {
            const playerToSubstitute = this.selectPlayerToSubstitute(team);
            const benchPlayer = this.selectBenchPlayer(team);

            if (playerToSubstitute && benchPlayer) {
                this.performSubstitution(playerToSubstitute, benchPlayer);
                this.substitutionsLeft[team]--;
                advancedCommentarySystem.announceSubstitution(playerToSubstitute, benchPlayer);
            }
        }
    },

    selectPlayerToSubstitute: function(team) {
        // Select the most tired or injured player
        return players
            .filter(p => p.team === team && !p.isGoalkeeper)
            .sort((a, b) => (a.stamina + a.attributes.speed) - (b.stamina + b.attributes.speed))[0];
    },

    selectBenchPlayer: function(team) {
        // Select the most rested bench player
        return this.benchPlayers[team]
            .sort((a, b) => b.stamina - a.stamina)[0];
    },

    performSubstitution: function(playerOut, playerIn) {
        const index = players.indexOf(playerOut);
        players[index] = playerIn;
        playerIn.position.copy(playerOut.position);
        this.benchPlayers[playerOut.team] = this.benchPlayers[playerOut.team].filter(p => p !== playerIn);
        this.benchPlayers[playerOut.team].push(playerOut);
        playerOut.position.set(0, 0, -40); // Move to bench

        console.log(`Substitution: Player ${playerOut.id} is replaced by Player ${playerIn.id} in team ${playerOut.team}`);
    }
};

// Initialize advanced systems
advancedSubstitutionSystem.initialize();

// Implement advanced ball physics with air resistance and Magnus effect
const advancedBallPhysics = {
    update: function(deltaTime) {
        // Apply gravity
        ballBody.velocity.y -= 9.81 * deltaTime;

        // Apply air resistance
        const airDensity = 1.2; // kg/m^3
        const dragCoefficient = 0.47; // for a sphere
        const crossSectionalArea = Math.PI * 0.11 * 0.11; // for a standard soccer ball
        const airResistance = ballBody.velocity.clone().negate().normalize().multiplyScalar(
            0.5 * airDensity * dragCoefficient * crossSectionalArea * ballBody.velocity.lengthSq()
        );
        ballBody.applyForce(airResistance, ballBody.position);

        // Apply Magnus effect (spin)
        const liftCoefficient = 0.33; // Typical value for a soccer ball
        const magnusForce = new CANNON.Vec3().copy(ballBody.velocity).cross(ballSpinSystem.spinAxis).scale(
            0.5 * airDensity * liftCoefficient * crossSectionalArea * ballBody.velocity.length() * ballSpinSystem.spinStrength
        );
        ballBody.applyForce(magnusForce, ballBody.position);

        // Update position
        ballBody.position.vadd(ballBody.velocity.scale(deltaTime));

        // Handle ground collision with realistic bounce
        if (ballBody.position.y < 0.11) { // Assuming ball radius is 0.11 meters
            ballBody.position.y = 0.11;
            const restitution = 0.8; // Coefficient of restitution for a soccer ball
            ballBody.velocity.y = -ballBody.velocity.y * restitution;

            // Apply friction
            const friction = 0.3;
            ballBody.velocity.x *= (1 - friction);
            ballBody.velocity.z *= (1 - friction);
        }

        // Update Three.js ball position and rotation
        ball.position.copy(ballBody.position);
        ball.quaternion.copy(ballBody.quaternion);
    }
};

// Implement advanced referee AI
class AdvancedRefereeAI {
    constructor() {
        this.position = new THREE.Vector3();
        this.speed = 7;
        this.decisionMakingAbility = Math.random() * 0.4 + 0.6; // 0.6 to 1.0
        this.consistencyFactor = Math.random() * 0.4 + 0.6; // 0.6 to 1.0
        this.cardThreshold = 0.7;
        this.foulDetectionRadius = 5;
    }

    update(deltaTime) {
        this.moveReferee(deltaTime);
        this.detectFouls();
        this.updateCardDecisions();
    }

    moveReferee(deltaTime) {
        const targetPosition = ball.position.clone();
        targetPosition.y = 1; // Keep referee on the ground
        const direction = targetPosition.sub(this.position).normalize();
        this.position.add(direction.multiplyScalar(this.speed * deltaTime));
    }

    detectFouls() {
        players.forEach(player => {
            if (player.position.distanceTo(ball.position) < this.foulDetectionRadius) {
                const foulProbability = this.calculateFoulProbability(player);
                if (Math.random() < foulProbability) {
                    this.callFoul(player);
                }
            }
        });
    }

    calculateFoulProbability(player) {
        const baseProbability = 0.01;
        const distanceFactor = 1 - (player.position.distanceTo(ball.position) / this.foulDetectionRadius);
        const speedFactor = player.velocity.length() / 10; // Assuming max speed is 10
        return baseProbability * distanceFactor * speedFactor * this.decisionMakingAbility;
    }

    callFoul(player) {
        console.log(`Foul called on player ${player.id} from team ${player.team}!`);
        advancedCommentarySystem.announceFoul(player);
        this.decidePunishment(player);
    }

    decidePunishment(player) {
        const severityScore = Math.random() * this.consistencyFactor;
        if (severityScore > this.cardThreshold) {
            this.showCard(player, severityScore > 0.9 ? 'red' : 'yellow');
        } else {
            setPieceSystem.startFreeKick(player.team === 'A' ? 'B' : 'A', ball.position.clone());
        }
    }

    showCard(player, cardType) {
        if (cardType === 'yellow') {
            if (!player.yellowCard) {
                player.yellowCard = true;
                console.log(`Yellow card shown to player ${player.id} from team ${player.team}!`);
                advancedCommentarySystem.announceCard(player, 'yellow');
            } else {
                this.showCard(player, 'red');
            }
        } else if (cardType === 'red') {
            console.log(`Red card shown to player ${player.id} from team ${player.team}! Player sent off!`);
            advancedCommentarySystem.announceCard(player, 'red');
            this.sendPlayerOff(player);
        }
    }

    sendPlayerOff(player) {
        const index = players.indexOf(player);
        if (index > -1) {
            players.splice(index, 1);
        }
        scene.remove(player);
        advancedCommentarySystem.announcePlayerSentOff(player);
    }

    updateCardDecisions() {
        // Slightly adjust card threshold based on game time and score difference
        const gameTimeEffect = gameTime / 5400; // 90 minutes = 5400 seconds
        const scoreDifference = Math.abs(scores.teamA - scores.teamB);
        this.cardThreshold = 0.7 + (gameTimeEffect * 0.1) - (scoreDifference * 0.05);
        this.cardThreshold = Math.max(0.5, Math.min(0.9, this.cardThreshold));
    }
}

// Create and use the advanced referee
const advancedReferee = new AdvancedRefereeAI();

// Update the game loop to use the advanced referee
function updateGame(deltaTime) {
    // ... (previous update code)

    advancedReferee.update(deltaTime);

    // ... (rest of update code)
}

// Implement a more sophisticated commentary system
const advancedCommentarySystem = {
    commentators: [
        { name: "John", enthusiasm: 0.8, knowledge: 0.9, bias: 0.1 },
        { name: "Sarah", enthusiasm: 0.7, knowledge: 0.95, bias: -0.1 }
    ],
    lastCommentTime: 0,
    commentCooldown: 5, // seconds

    update: function(time) {
        if (time - this.lastCommentTime > this.commentCooldown) {
            const events = this.detectEvents();
            if (events.length > 0) {
                const event = this.selectEvent(events);
                this.generateCommentary(event);
                this.lastCommentTime = time;
            }
        }
    },

    detectEvents: function() {
        const events = [];
        if (this.isGoalScored()) events.push('goal');
        if (this.isNearMiss()) events.push('nearMiss');
        if (this.isGreatPass()) events.push('greatPass');
        if (this.isGreatSave()) events.push('greatSave');
        return events;
    },

    selectEvent: function(events) {
        // Prioritize more exciting events
        const eventPriority = { 'goal': 4, 'nearMiss': 3, 'greatSave': 2, 'greatPass': 1 };
        return events.reduce((a, b) => eventPriority[a] > eventPriority[b] ? a : b);
    },

    generateCommentary: function(event) {
        const commentator = this.selectCommentator(event);
        const commentary = this.createComment(commentator, event);
        console.log(`${commentator.name}: ${commentary}`);
        // Here you could also trigger audio playback of the commentary
    },

    selectCommentator: function(event) {
        // Select commentator based on the event and their characteristics
        return this.commentators.reduce((a, b) => {
            const aScore = a.enthusiasm * (event === 'goal' ? 2 : 1) + a.knowledge;
            const bScore = b.enthusiasm * (event === 'goal' ? 2 : 1) + b.knowledge;
            return aScore > bScore ? a : b;
        });
    },

    createComment: function(commentator, event) {
        const templates = {
            goal: [
                "What a goal! Absolutely fantastic!",
                "He's done it! The ball is in the back of the net!",
                "Goal! You won't see many better than that!"
            ],
            nearMiss: [
                "Oh, so close! The goalkeeper was beaten there.",
                "What a chance! He'll be disappointed not to score.",
                "Inches wide! That was a golden opportunity!"
            ],
            greatPass: [
                "What a pass! That's vision for you.",
                "Brilliant ball through the defense!",
                "He's split the defense wide open with that pass!"
            ],
            greatSave: [
                "What a save! The goalkeeper's keeping his team in this.",
                "Brilliant reflexes from the keeper!",
                "He's denied what looked like a certain goal!"
            ]
        };

        const template = templates[event][Math.floor(Math.random() * templates[event].length)];
        return this.addCommentatorFlavor(commentator, template);
    },

    addCommentatorFlavor: function(commentator, comment) {
        if (commentator.enthusiasm > 0.8) {
            comment = comment.toUpperCase() + "!";
        }
        if (commentator.knowledge > 0.9) {
            comment += " You know, I haven't seen anything like that since the World Cup final of '98.";
        }
        if (Math.abs(commentator.bias) > 0.2) {
            const favoredTeam = commentator.bias > 0 ? 'A' : 'B';
            comment += ` Team ${favoredTeam} is really showing their quality today.`;
        }
        return comment;
    },

    isGoalScored: function() {
        // Implementation depends on your goal detection system
        return Math.abs(ball.position.x) > 45 && Math.abs(ball.position.z) < 7.32 / 2;
    },

    isNearMiss: function() {
        return Math.abs(ball.position.x) > 40 && Math.abs(ball.position.z) < 10 && Math.abs(ball.position.y) < 3;
    },

    isGreatPass: function() {
        // This is a simplified check and could be improved
        return ballBody.velocity.length() > 20;
    },

    isGreatSave: function() {
        // This would require implementing goalkeeper logic and tracking saves
        const goalkeeper = players.find(p => p.isGoalkeeper && p.position.distanceTo(ball.position) < 3);
        return goalkeeper && ballBody.velocity.length() > 15;
    },

    announceInjury: function(player) {
        const commentator = this.selectCommentator('injury');
        const comment = `Oh no, player ${player.id} from team ${player.team} seems to be injured. This could be a big blow for the team.`;
        console.log(`${commentator.name}: ${this.addCommentatorFlavor(commentator, comment)}`);
    },

    announceRecovery: function(player) {
        const commentator = this.selectCommentator('recovery');
        const comment = `Good news for team ${player.team}, player ${player.id} has recovered and is back in the game.`;
        console.log(`${commentator.name}: ${this.addCommentatorFlavor(commentator, comment)}`);
    },

    announceSubstitution: function(playerOut, playerIn) {
        const commentator = this.selectCommentator('substitution');
        const comment = `We have a substitution for team ${playerOut.team}. Player ${playerOut.id} is coming off, replaced by player ${playerIn.id}.`;
        console.log(`${commentator.name}: ${this.addCommentatorFlavor(commentator, comment)}`);
    },

    announceFoul: function(player) {
        const commentator = this.selectCommentator('foul');
        const comment = `Foul by player ${player.id} from team ${player.team}. The referee has blown the whistle.`;
        console.log(`${commentator.name}: ${this.addCommentatorFlavor(commentator, comment)}`);
    },

    announceCard: function(player, cardType) {
        const commentator = this.selectCommentator('card');
        const comment = `${cardType.charAt(0).toUpperCase() + cardType.slice(1)} card shown to player ${player.id} from team ${player.team}. That's a big moment in the game.`;
        console.log(`${commentator.name}: ${this.addCommentatorFlavor(commentator, comment)}`);
    },

    announcePlayerSentOff: function(player) {
        const commentator = this.selectCommentator('sentOff');
        const comment = `Player ${player.id} from team ${player.team} has been sent off! They'll have to finish the game with 10 men.`;
        console.log(`${commentator.name}: ${this.addCommentatorFlavor(commentator, comment)}`);
    }
};

// Enhance the game loop with the advanced commentary system
function ultimateEnhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            updateBallPhysics(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);
            updateOptimizations(time);

            // Update all advanced systems
            adv
ancedTeamTacticsSystem.update(deltaTime);
            advancedReferee.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            advancedInjurySystem.update(deltaTime);
            advancedSubstitutionSystem.update();
            advancedFatigueSystem.update(deltaTime);
            crowdSystem.update(deltaTime);
            advancedSoundSystem.update();
            particleSystem.update(deltaTime);
            offsideSystem.update();

            // Handle set pieces
            if (setPieceSystem.currentSetPiece) {
                setPieceSystem.executeSetPiece();
            }

            // Record replay frame
            replaySystem.recordFrame();

            // Update grass shader
            grassShaderMaterial.uniforms.time.value = time * 0.001;

            // Check for goals
            checkForGoals();

            // Check for game end
            if (gameTime <= 0) {
                endGame();
            }
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            displayFinalResults();
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(ultimateEnhancedGameLoop);
}

function endGame() {
    currentState = GameState.GAME_OVER;
    advancedCommentarySystem.announceGameEnd();
    advancedSoundSystem.play('finalWhistle');
}

function displayFinalResults() {
    const resultScreen = document.createElement('div');
    resultScreen.style.position = 'absolute';
    resultScreen.style.top = '50%';
    resultScreen.style.left = '50%';
    resultScreen.style.transform = 'translate(-50%, -50%)';
    resultScreen.style.backgroundColor = 'rgba(0, 0, 0, 0.8)';
    resultScreen.style.color = 'white';
    resultScreen.style.padding = '20px';
    resultScreen.style.borderRadius = '10px';
    resultScreen.style.textAlign = 'center';

    const resultTitle = document.createElement('h2');
    resultTitle.textContent = 'Final Score';
    resultScreen.appendChild(resultTitle);

    const scoreDisplay = document.createElement('p');
    scoreDisplay.textContent = `Team A ${scores.teamA} - ${scores.teamB} Team B`;
    resultScreen.appendChild(scoreDisplay);

    const winnerDisplay = document.createElement('p');
    if (scores.teamA > scores.teamB) {
        winnerDisplay.textContent = 'Team A wins!';
    } else if (scores.teamB > scores.teamA) {
        winnerDisplay.textContent = 'Team B wins!';
    } else {
        winnerDisplay.textContent = 'It\'s a draw!';
    }
    resultScreen.appendChild(winnerDisplay);

    const restartButton = document.createElement('button');
    restartButton.textContent = 'Play Again';
    restartButton.onclick = () => location.reload();
    resultScreen.appendChild(restartButton);

    document.body.appendChild(resultScreen);
}

// Start the ultimate enhanced game loop
ultimateEnhancedGameLoop(0);

// Final touches and optimizations

// Implement WebGL 2.0 features for better performance
if (renderer.capabilities.isWebGL2) {
    renderer.outputEncoding = THREE.sRGBEncoding;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.0;
}

// Use Web Workers for physics calculations
const physicsWorker = new Worker('physics-worker.js');
physicsWorker.onmessage = function(e) {
    const { ballPosition, ballVelocity, playerPositions } = e.data;
    updatePhysicsResults(ballPosition, ballVelocity, playerPositions);
};

function updatePhysicsResults(ballPosition, ballVelocity, playerPositions) {
    ball.position.copy(ballPosition);
    ballBody.velocity.copy(ballVelocity);
    players.forEach((player, index) => {
        player.position.copy(playerPositions[index]);
    });
}

// Implement Level of Detail (LOD) for players
players.forEach(player => {
    const geometry = new THREE.SphereGeometry(0.5, 32, 32);
    const material = new THREE.MeshStandardMaterial({ color: player.team === 'A' ? 0xff0000 : 0x0000ff });
    
    const highDetailMesh = new THREE.Mesh(geometry, material);
    const mediumDetailMesh = new THREE.Mesh(new THREE.SphereGeometry(0.5, 16, 16), material);
    const lowDetailMesh = new THREE.Mesh(new THREE.SphereGeometry(0.5, 8, 8), material);

    const lod = new THREE.LOD();
    lod.addLevel(highDetailMesh, 0);
    lod.addLevel(mediumDetailMesh, 10);
    lod.addLevel(lowDetailMesh, 50);

    lod.position.copy(player.position);
    scene.add(lod);
    player.lod = lod;
});

// Implement Occlusion Culling
const frustum = new THREE.Frustum();
const projScreenMatrix = new THREE.Matrix4();

function updateVisibility() {
    camera.updateMatrixWorld();
    projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
    frustum.setFromProjectionMatrix(projScreenMatrix);

    players.forEach(player => {
        player.lod.visible = frustum.intersectsObject(player.lod);
    });
}

// Add post-processing effects
const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
composer.addPass(bloomPass);

const fxaaPass = new ShaderPass(FXAAShader);
fxaaPass.uniforms['resolution'].value.set(1 / window.innerWidth, 1 / window.innerHeight);
composer.addPass(fxaaPass);

// Implement Dynamic Resolution Scaling
let resolutionScale = 1;
function updateResolutionScale() {
    const fps = 1 / deltaTime;
    if (fps < 30 && resolutionScale > 0.5) {
        resolutionScale -= 0.1;
    } else if (fps > 60 && resolutionScale < 1) {
        resolutionScale += 0.1;
    }
    renderer.setPixelRatio(window.devicePixelRatio * resolutionScale);
    composer.setPixelRatio(window.devicePixelRatio * resolutionScale);
}

// Final game loop update
function ultimateEnhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            updateBallPhysics(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);
            updateOptimizations(time);

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedReferee.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            advancedInjurySystem.update(deltaTime);
            advancedSubstitutionSystem.update();
            advancedFatigueSystem.update(deltaTime);
            crowdSystem.update(deltaTime);
            advancedSoundSystem.update();
            particleSystem.update(deltaTime);
            offsideSystem.update();

            // Handle set pieces
            if (setPieceSystem.currentSetPiece) {
                setPieceSystem.executeSetPiece();
            }

            // Record replay frame
            replaySystem.recordFrame();

            // Update grass shader
            grassShaderMaterial.uniforms.time.value = time * 0.001;

            // Check for goals
            checkForGoals();

            // Update visibility and LOD
            updateVisibility();
            players.forEach(player => player.lod.update(camera));

            // Update resolution scale
            updateResolutionScale();

            // Check for game end
            if (gameTime <= 0) {
                endGame();
            }
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            displayFinalResults();
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(ultimateEnhancedGameLoop);
}

// Start the game
initializeGame();
ultimateEnhancedGameLoop(0);

// Performance optimizations
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement instanced rendering for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement texture atlases
    const textureAtlas = new THREE.TextureLoader().load('textures/texture_atlas.jpg');
    const atlasedMaterial = new THREE.MeshBasicMaterial({ map: textureAtlas });

    // Use shared geometries and materials
    const sharedPlayerGeometry = new THREE.SphereGeometry(0.5, 32, 32);
    const sharedTeamAMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
    const sharedTeamBMaterial = new THREE.MeshStandardMaterial({ color: 0x0000ff });

    players.forEach(player => {
        player.geometry = sharedPlayerGeometry;
        player.material = player.team === 'A' ? sharedTeamAMaterial : sharedTeamBMaterial;
    });

    // Implement mesh merging for static objects
    const staticGeometries = [];
    scene.traverse(object => {
        if (object.isStatic && object.isMesh) {
            staticGeometries.push(object.geometry);
        }
    });
    const mergedGeometry = BufferGeometryUtils.mergeBufferGeometries(staticGeometries);
    const mergedMesh = new THREE.Mesh(mergedGeometry, new THREE.MeshStandardMaterial());
    scene.add(mergedMesh);

    // Use sprite sheets for animations
    const spriteSheet = new THREE.TextureLoader().load('textures/player_animations.png');
    const spriteSheetMaterial = new THREE.SpriteMaterial({ map: spriteSheet });

    function updatePlayerSprite(player, frameIndex) {
        const spriteSize = 64; // Assuming 64x64 pixel sprites
        const framesPerRow = 8;
        const u = (frameIndex % framesPerRow) / framesPerRow;
        const v = Math.floor(frameIndex / framesPerRow) / framesPerRow;
        player.material.map.offset.set(u, v);
        player.material.map.repeat.set(1 / framesPerRow, 1 / framesPerRow);
    }

    // Implement frustum culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateFrustumCulling() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use Web Workers for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        updatePhysicsResults(ballPosition, ballVelocity, playerPositions);
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Implement occlusion culling
    const occlusionCulling = new THREE.OcclusionCulling(camera, scene);

    function updateOcclusionCulling() {
        occlusionCulling.update();
    }

    // Use GPU instancing for crowd
    const crowdGeometry = new THREE.BoxGeometry(0.5, 1.8, 0.5);
    const crowdMaterial = new THREE.MeshPhongMaterial({ color: 0xcccccc });
    const crowdInstancedMesh = new THREE.InstancedMesh(crowdGeometry, crowdMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 120,
            1,
            (Math.random() - 0.5) * 80
        );
        const scale = new THREE.Vector3(1, 0.9 + Math.random() * 0.2, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI * 2, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        crowdInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(crowdInstancedMesh);

    // Implement dynamic LOD system
    const lodSystem = new THREE.LOD();

    players.forEach(player => {
        const highDetail = new THREE.Mesh(sharedPlayerGeometry, player.material);
        const mediumDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 16, 16), player.material);
        const lowDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 8, 8), player.material);

        lodSystem.addLevel(highDetail, 0);
        lodSystem.addLevel(mediumDetail, 20);
        lodSystem.addLevel(lowDetail, 50);

        player.add(lodSystem);
    });

    // Implement shader-based grass
    const grassShaderMaterial = new THREE.ShaderMaterial({
        vertexShader: `
            uniform float time;
            varying vec2 vUv;
            void main() {
                vUv = uv;
                vec3 pos = position;
                pos.y += sin(pos.x * 10.0 + time) * 0.1;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
            }
        `,
        fragmentShader: `
            uniform sampler2D grassTexture;
            varying vec2 vUv;
            void main() {
                gl_FragColor = texture2D(grassTexture, vUv);
            }
        `,
        uniforms: {
            time: { value: 0 },
            grassTexture: { value: new THREE.TextureLoader().load('textures/grass.jpg') }
        }
    });

    const grassMesh = new THREE.Mesh(
        new THREE.PlaneGeometry(100, 60, 100, 60),
        grassShaderMaterial
    );
    grassMesh.rotation.x = -Math.PI / 2;
    scene.add(grassMesh);

    // Implement dynamic resolution scaling
    let resolutionScale = 1;
    function updateResolutionScale() {
        const fps = 1 / deltaTime;
        if (fps < 30 && resolutionScale > 0.5) {
            resolutionScale -= 0.1;
        } else if (fps > 60 && resolutionScale < 1) {
            resolutionScale += 0.1;
        }
        renderer.setPixelRatio(window.devicePixelRatio * resolutionScale);
        composer.setPixelRatio(window.devicePixelRatio * resolutionScale);
    }

    // Update functions
    return {
        updateParticles: function() {
            particleSystem.geometry.attributes.position.needsUpdate = true;
        },
        updateGrass: function(time) {
            grassShaderMaterial.uniforms.time.value = time;
        },
        updateFrustumCulling: updateFrustumCulling,
        updateOcclusionCulling: updateOcclusionCulling,
        updatePhysics: updatePhysics,
        updateResolutionScale: updateResolutionScale
    };
}

const optimizations = optimizePerformance();

// Enhance the game loop with optimizations
function enhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            updateBallPhysics(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);

            // Apply optimizations
            optimizations.updateParticles();
            optimizations.updateGrass(time);
            optimizations.updateFrustumCulling();
            optimizations.updateOcclusionCulling();
            optimizations.updatePhysics();
            optimizations.updateResolutionScale();

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedReferee.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            advancedInjurySystem.update(deltaTime);
            advancedSubstitutionSystem.update();
            advancedFatigueSystem.update(deltaTime);
            particleSystem.update(deltaTime);
            offsideSystem.update();

            // Handle set pieces
            if (setPieceSystem.currentSetPiece) {
                setPieceSystem.executeSetPiece();
            }

            // Record replay frame
            replaySystem.recordFrame();

            // Check for goals
            checkForGoals();

            // Check for game end
            if (gameTime <= 0) {
                endGame();
            }
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            displayFinalResults();
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoop);
}

// Start the enhanced game loop
enhancedGameLoop(0);

// Final touches

// Implement advanced audio system with 3D positional audio
const audioListener = new THREE.AudioListener();
camera.add(audioListener);

const audioLoader = new THREE.AudioLoader();
const soundEffects = {
    kick: new THREE.PositionalAudio(audioListener),
    whistle: new THREE.PositionalAudio(audioListener),
    crowd: new THREE.Audio(audioListener)
};

audioLoader.load('sounds/kick.mp3', buffer => {
    soundEffects.kick.setBuffer(buffer);
    soundEffects.kick.setRefDistance(20);
});

audioLoader.load('sounds/whistle.mp3', buffer => {
    soundEffects.whistle.setBuffer(buffer);
    soundEffects.whistle.setRefDistance(50);
});

audioLoader.load('sounds/crowd.mp3', buffer => {
    soundEffects.crowd.setBuffer(buffer);
    soundEffects.crowd.setLoop(true);
    soundEffects.crowd.setVolume(0.5);
    soundEffects.crowd.play();
});

// Implement cinematic replay system
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 10 * 60, // 10 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballRotation: ball.quaternion.clone(),
            playerPositions: players.map(player => player.position.clone()),
            playerRotations: players.map(player => player.quaternion.clone()),
            cameraPosition: camera.position.clone(),
            cameraRotation: camera.quaternion.clone()
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        const replayCamera = camera.clone();
        scene.add(replayCamera);

        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                scene.remove(replayCamera);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ball.quaternion.copy(frame.ballRotation);

            players.forEach((player, index) => {
                player.position.copy(frame.playerPositions[index]);
                player.quaternion.copy(frame.playerRotations[index]);
            });

            replayCamera.position.copy(frame.cameraPosition);
            replayCamera.quaternion.copy(frame.cameraRotation);

            composer.render();
            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Implement advanced particle systems
const particleSystem = {
    systems: [],

    createEffect: function(position, color, count, speed, size, duration) {
        const particles = new THREE.BufferGeometry();
        const positions = new Float32Array(count * 3);
        const colors = new Float32Array(count * 3);
        const sizes = new Float32Array(count);

        for (let i = 0; i < count; i++) {
            positions[i * 3] = position.x;
            positions[i * 3 + 1] = position.y;
            positions[i * 3 + 2] = position.z;

            colors[i * 3] = color.r;
            colors[i * 3 + 1] = color.g;
            colors[i * 3 + 2] = color.b;

            sizes[i] = size;
        }

        particles.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        particles.setAttribute('color', new THREE.BufferAttribute(colors, 3));
        particles.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

        const material = new THREE.ShaderMaterial({
            uniforms: {
                time: { value: 0 },
                speed: { value: speed }
            },
            vertexShader: `
                uniform float time;
                uniform float speed;
                attribute float size;
                varying vec3 vColor;
                void main() {
                    vColor = color;
                    vec3 pos = position;
                    pos.y += speed * time;
                    vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
                    gl_PointSize = size * (300.0 / -mvPosition.z);
                    gl_Position = projectionMatrix * mvPosition;
                }
            `,
            fragmentShader: `
                varying vec3 vColor;
                void main() {
                    if (length(gl_PointCoord - vec2(0.5, 0.5)) > 0.5) discard;
                    gl_FragColor = vec4(vColor, 1.0);
                }
            `,
            blending: THREE.AdditiveBlending,
            depthTest: false,
            transparent: true
        });

        const particleSystem = new THREE.Points(particles, material);
        this.systems.push({ system: particleSystem, startTime: Date.now(), duration: duration });
        scene.add(particleSystem);
    },

    update: function(deltaTime) {
        this.systems.forEach((system, index) => {
            system.system.material.uniforms.time.value += deltaTime;

            if (Date.now() - system.startTime > system.duration) {
                scene.remove(system.system);
                this.systems.splice(index, 1);
            }
        });
    }
};

// Implement advanced post-processing effects
const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
composer.addPass(bloomPass);

const filmPass = new FilmPass(0.35, 0.5, 2048, false);
composer.addPass(filmPass);

const smaaPass = new SMAAPass(window.innerWidth * renderer.getPixelRatio(), window.innerHeight * renderer.getPixelRatio());
composer.addPass(smaaPass);

// Implement dynamic environment mapping
const pmremGenerator = new THREE.PMREMGenerator(renderer);
let envMap;

function updateEnvironmentMap() {
    const generatedCubeRenderTarget = pmremGenerator.fromScene(scene);
    envMap = generatedCubeRenderTarget.texture;

    scene.traverse((object) => {
        if (object.isMesh && object.material.envMap) {
            object.material.envMap = envMap;
            object.material.needsUpdate = true;
        }
    });

    pmremGenerator.dispose();
}

// Final game initialization
function initGame() {
    createScene();
    createLights();
    createPlayers();
    createBall();
    createStadium();
    createCrowd();
    initPhysics();
    initAudio();
    initUI();

    updateEnvironmentMap();

    // Start the game loop
    enhancedGameLoop(0);
}

// Start the game
initGame();

// Remove initial loading screen
document.getElementById('loading-screen').style.display = 'none';

// Event listeners
window.addEventListener('resize', onWindowResize);
document.addEventListener('keydown', onKeyDown);
document.addEventListener('keyup', onKeyUp);

// Game loop
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updatePhysics(deltaTime);
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

function updateGame(deltaTime) {
    gameTime -= deltaTime;
    if (gameTime <= 0) {
        endGame();
    }

    updatePlayers(deltaTime);
    updateBall(deltaTime);
    checkForGoals();
    checkForFouls();
    updateTactics(deltaTime);
}

function updatePhysics(deltaTime) {
    world.step(deltaTime);
    
    // Update Three.js objects with Cannon.js data
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);

    players.forEach((player, index) => {
        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);
    });
}

function updateAI(deltaTime) {
    players.forEach(player => {
        if (player.team !== humanTeam) {
            player.ai.update(deltaTime);
        }
    });
}

function updateParticles(deltaTime) {
    particleSystem.update(deltaTime);
}

function updateSound() {
    // Update 3D sound positions
    soundEffects.kick.position.copy(ball.position);
    soundEffects.whistle.position.copy(refereeSystem.position);

    // Update crowd sound based on excitement
    const excitement = calculateCrowdExcitement();
    soundEffects.crowd.setVolume(0.5 + excitement * 0.5);
}

function updateWeather(deltaTime) {
    weatherSystem.update(deltaTime);
}

function updateCamera(deltaTime) {
    const cameraTarget = getCameraTarget();
    camera.position.lerp(cameraTarget.position, 0.1);
    camera.lookAt(cameraTarget.lookAt);
}

function updateUI() {
    document.getElementById('score').textContent = `${scores.teamA} - ${scores.teamB}`;
    document.getElementById('time').textContent = formatTime(Math.max(0, gameTime));
    document.getElementById('stamina-bar').style.width = `${selectedPlayer.stamina}%`;
}

function render() {
    composer.render();
}

function endGame() {
    currentState = GameState.GAME_OVER;
    displayFinalScore();
    // Additional end game logic
}

// Helper functions
function formatTime(seconds) {
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = Math.floor(seconds % 60);
    return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
}

function calculateCrowdExcitement() {
    // Calculate crowd excitement based on game events
    // Return a value between 0 and 1
}

function getCameraTarget() {
    // Determine camera target based on current game state
    // Return an object with position and lookAt properties
}

// Start the game loop
gameLoop(0);

// Additional game systems

// Referee system
const refereeSystem = {
    position: new THREE.Vector3(),
    speed: 5,

    update: function(deltaTime) {
        // Move referee towards the ball
        const direction = new THREE.Vector3().subVectors(ball.position, this.position).normalize();
        this.position.add(direction.multiplyScalar(this.speed * deltaTime));

        // Check for fouls
        this.checkForFouls();
    },

    checkForFouls: function() {
        // Foul detection logic
    },

    callFoul: function(player) {
        console.log(`Foul called on player ${player.id} from team ${player.team}`);
        // Implement foul consequences
    }
};

// Tactics system
const tacticsSystem = {
    formations: {
        '4-4-2': [/* player positions */],
        '4-3-3': [/* player positions */],
        '3-5-2': [/* player positions */]
    },

    updateFormation: function(team, formation) {
        const teamPlayers = players.filter(p => p.team === team);
        const positions = this.formations[formation];

        teamPlayers.forEach((player, index) => {
            player.formationPosition.copy(positions[index]);
        });
    },

    updateTactics: function(deltaTime) {
        // Update team tactics based on game state
    }
};

// Weather system
const weatherSystem = {
    currentWeather: 'clear',
    transitionTime: 0,
    transitionDuration: 30, // seconds

    update: function(deltaTime) {
        this.transitionTime += deltaTime;
        if (this.transitionTime >= this.transitionDuration) {
            this.changeWeather();
        }
        this.updateWeatherEffects(deltaTime);
    },

    changeWeather: function() {
        const weathers = ['clear', 'cloudy', 'rainy', 'foggy'];
        this.currentWeather = weathers[Math.floor(Math.random() * weathers.length)];
        this.transitionTime = 0;
        console.log(`Weather changing to: ${this.currentWeather}`);
    },

    updateWeatherEffects: function(deltaTime) {
        // Update weather effects based on current weather
    }
};

// Commentary system
const commentarySystem = {
    phrases: {
        goal: [
            "GOAL! What a fantastic finish!",
            "He's done it! The ball is in the back of the net!",
            "An absolute screamer! The crowd goes wild!"
        ],
        miss: [
            "Oh, so close! The goalkeeper was beaten there.",
            "What a chance! He'll be disappointed not to score.",
            "Inches wide! That was a golden opportunity!"
        ],
        // Add more phrase categories
    },

    generateCommentary: function(event) {
        const phrases = this.phrases[event];
        const commentary = phrases[Math.floor(Math.random() * phrases.length)];
        console.log(`Commentary: ${commentary}`);
        // You could also trigger audio playback of the commentary
    }
};

// Injury system
const injurySystem = {
    injuredPlayers: new Set(),

    update: function(deltaTime) {
        players.forEach(player => {
            if (this.injuredPlayers.has(player)) {
                this.updateInjuredPlayer(player, deltaTime);
            } else if (Math.random() < 0.0001) { // Small chance of injury each frame
                this.injurePlayer(player);
            }
        });
    },

    injurePlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} is injured!`);
        this.injuredPlayers.add(player);
        player.speed *= 0.5;
        // Visual indication of injury
        player.material.color.setHex(0xff0000);
    },

    updateInjuredPlayer: function(player, deltaTime) {
        player.injuryTime += deltaTime;
        if (player.injuryTime > 30) { // Recover after 30 seconds
            this.recoverPlayer(player);
        }
    },

    recoverPlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} has recovered!`);
        this.injuredPlayers.delete(player);
        player.speed *= 2;
        player.injuryTime = 0;
        player.material.color.setHex(0xffffff);
    }
};

// Substitution system
const substitutionSystem = {
    substitutionsLeft: { A: 3, B: 3 },
    benchPlayers: { A: [], B: [] },

    initialize: function() {
        // Create bench players
        for (let team of ['A', 'B']) {
            for (let i = 0; i < 7; i++) {
                const benchPlayer = createPlayer(team);
                benchPlayer.position.set(0, 0, -40); // Off the field
                this.benchPlayers[team].push(benchPlayer);
            }
        }
    },

    requestSubstitution: function(team) {
        if (this.substitutionsLeft[team] > 0) {
            const playerToSubstitute = this.selectPlayerToSubstitute(team);
            const benchPlayer = this.selectBenchPlayer(team);

            if (playerToSubstitute && benchPlayer) {
                this.performSubstitution(playerToSubstitute, benchPlayer);
                this.substitutionsLeft[team]--;
            }
        }
    },

    selectPlayerToSubstitute: function(team) {
        // Select the most tired player
        return players
            .filter(p => p.team === team)
            .sort((a, b) => a.stamina - b.stamina)[0];
    },

    selectBenchPlayer: function(team) {
        // Select the most rested bench player
        return this.benchPlayers[team]
            .sort((a, b) => b.stamina - a.stamina)[0];
    },

    performSubstitution: function(playerOut, playerIn) {
        const index = players.indexOf(playerOut);
        players[index] = playerIn;
        playerIn.position.copy(playerOut.position);
        this.benchPlayers[playerOut.team] = this.benchPlayers[playerOut.team].filter(p => p !== playerIn);
        this.benchPlayers[playerOut.team].push(playerOut);
        playerOut.position.set(0, 0, -40); // Move to bench

        console.log(`Substitution: Player ${playerOut.id} is replaced by Player ${playerIn.id} in team ${playerOut.team}`);
    }
};

// Initialize additional systems
substitutionSystem.initialize();

// Add these systems to the game loop
function updateGame(deltaTime) {
    gameTime -= deltaTime;
    if (gameTime <= 0) {
        endGame();
    }

    updatePlayers(deltaTime);
    updateBall(deltaTime);
    checkForGoals();
    refereeSystem.update(deltaTime);
    tacticsSystem.updateTactics(deltaTime);
    weatherSystem.update(deltaTime);
    injurySystem.update(deltaTime);
    substitutionSystem.update();
}

// Final touches

// Implement replays
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 10 * 60, // 10 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballRotation: ball.quaternion.clone(),
            playerPositions: players.map(player => player.position.clone()),
            playerRotations: players.map(player => player.quaternion.clone()),
            cameraPosition: camera.position.clone(),
            cameraRotation: camera.quaternion.clone()
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        const replayCamera = camera.clone();
        scene.add(replayCamera);

        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                scene.remove(replayCamera);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ball.quaternion.copy(frame.ballRotation);

            players.forEach((player, index) => {
                player.position.copy(frame.playerPositions[index]);
                player.quaternion.copy(frame.playerRotations[index]);
            });

            replayCamera.position.copy(frame.cameraPosition);
            replayCamera.quaternion.copy(frame.cameraRotation);

            composer.render();
            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Add replay recording to the game loop
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updatePhysics(deltaTime);
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            replaySystem.recordFrame();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

// Implement cinematic camera for replays
const cinematicCamera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
scene.add(cinematicCamera);

function updateCinematicCamera(deltaTime) {
    // Implement cinematic camera movement
    const time = Date.now() * 0.001;
    const radius = 50;
    cinematicCamera.position.x = Math.cos(time * 0.5) * radius;
    cinematicCamera.position.z = Math.sin(time * 0.5) * radius;
    cinematicCamera.position.y = 20 + Math.sin(time) * 10;
    cinematicCamera.lookAt(ball.position);
}

// Add cinematic camera update to replay system
replaySystem.playReplay = function() {
    let frameIndex = 0;
    const replayInterval = setInterval(() => {
        if (frameIndex >= this.recordedFrames.length) {
            clearInterval(replayInterval);
            camera = mainCamera; // Switch back to main camera
            return;
        }

        const frame = this.recordedFrames[frameIndex];
        ball.position.copy(frame.ballPosition);
        ball.quaternion.copy(frame.ballRotation);

        players.forEach((player, index) => {
            player.position.copy(frame.playerPositions[index]);
            player.quaternion.copy(frame.playerRotations[index]);
        });

        updateCinematicCamera(1/60); // Assume 60 fps
        camera = cinematicCamera; // Use cinematic camera for replay

        composer.render();
        frameIndex++;
    }, 1000 / 60); // Play at 60 fps
};

// Implement slow motion
let timeScale = 1;

function updatePhysics(deltaTime) {
    world.step(deltaTime * timeScale);
    
    // Update Three.js objects with Cannon.js data
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);

    players.forEach((player, index) => {
        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);
    });
}

function setSlowMotion(enable) {
    timeScale = enable ? 0.2 : 1;
}

// Add slow motion to replay system
replaySystem.playReplay = function() {
    setSlowMotion(true);
    // ... existing replay code ...
    // When replay ends:
    setSlowMotion(false);
};

// Implement spectator reactions
const spectatorSystem = {
    spectators: [],
    excitement: 0,

    initialize: function() {
        // Create spectators
        for (let i = 0; i < 1000; i++) {
            const spectator = new THREE.Mesh(
                new THREE.BoxGeometry(0.5, 1.8, 0.5),
                new THREE.MeshLambertMaterial({ color: 0xcccccc })
            );
            spectator.position.set(
                (Math.random() - 0.5) * 120,
                1,
                (Math.random() - 0.5) * 80
            );
            scene.add(spectator);
            this.spectators.push(spectator);
        }
    },

    update: function(deltaTime) {
        this.excitement = Math.max(0, this.excitement - deltaTime * 0.1);
        this.spectators.forEach(spectator => {
            spectator.position.y = 1 + Math.sin(Date.now() * 0.01 + spectator.position.x) * 0.1 * this.excitement;
        });
    },

    cheer: function() {
        this.excitement = 1;
    }
};

spectatorSystem.initialize();

// Add spectator update to game loop
function updateGame(deltaTime) {
    // ... existing game update code ...
    spectatorSystem.update(deltaTime);
}

// Make spectators cheer on goals
function checkForGoals() {
    // ... existing goal check code ...
    if (goalScored) {
        spectatorSystem.cheer();
    }
}

// Final optimization
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement frustum culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateFrustumCulling() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use instancing for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement level of detail (LOD)
    const playerLODs = [];
    players.forEach(player => {
        const highDetailGeometry = new THREE.SphereGeometry(1, 32, 32);
        const mediumDetailGeometry = new THREE.SphereGeometry(1, 16, 16);
        const lowDetailGeometry = new THREE.SphereGeometry(1, 8, 8);

        const playerLOD = new THREE.LOD();

        playerLOD.addLevel(new THREE.Mesh(highDetailGeometry, player.material), 0);
        playerLOD.addLevel(new THREE.Mesh(mediumDetailGeometry, player.material), 10);
        playerLOD.addLevel(new THREE.Mesh(lowDetailGeometry, player.material), 50);

        playerLOD.position.copy(player.position);
        scene.add(playerLOD);
        playerLODs.push(playerLOD);
    });

    function updateLODs() {
        playerLODs.forEach((playerLOD, index) => {
            playerLOD.position.copy(players[index].position);
            playerLOD.update(camera);
        });
    }

    // Use worker threads for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        ball.position.copy(ballPosition);
        ballBody.velocity.copy(ballVelocity);
        players.forEach((player, index) => {
            player.position.copy(playerPositions[index]);
        });
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Update optimization functions
    return {
        updateFrustumCulling: updateFrustumCulling,
        updateLODs: updateLODs,
        updatePhysics: updatePhysics
    };
}

const optimizations = optimizePerformance();

// Update game loop with optimizations
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            optimizations.updatePhysics();
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            replaySystem.recordFrame();
            optimizations.updateFrustumCulling();
            optimizations.updateLODs();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

// Start the game
initGame();
gameLoop(0);

// Performance optimizations
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement occlusion culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateVisibility() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use instancing for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement texture atlases
    const textureAtlas = new THREE.TextureLoader().load('textures/texture_atlas.jpg');
    const atlasedMaterial = new THREE.MeshBasicMaterial({ map: textureAtlas });

    // Use shared geometries and materials
    const sharedPlayerGeometry = new THREE.SphereGeometry(0.5, 32, 32);
    const sharedTeamAMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
    const sharedTeamBMaterial = new THREE.MeshStandardMaterial({ color: 0x0000ff });

    players.forEach(player => {
        player.geometry = sharedPlayerGeometry;
        player.material = player.team === 'A' ? sharedTeamAMaterial : sharedTeamBMaterial;
    });

    // Implement mesh merging for static objects
    const staticGeometries = [];
    scene.traverse(object => {
        if (object.isStatic && object.isMesh) {
            staticGeometries.push(object.geometry);
        }
    });
    const mergedGeometry = BufferGeometryUtils.mergeBufferGeometries(staticGeometries);
    const mergedMesh = new THREE.Mesh(mergedGeometry, new THREE.MeshStandardMaterial());
    scene.add(mergedMesh);

    // Use sprite sheets for animations
    const spriteSheet = new THREE.TextureLoader().load('textures/player_animations.png');
    const spriteSheetMaterial = new THREE.SpriteMaterial({ map: spriteSheet });

    function updatePlayerSprite(player, frameIndex) {
        const spriteSize = 64; // Assuming 64x64 pixel sprites
        const framesPerRow = 8;
        const u = (frameIndex % framesPerRow) / framesPerRow;
        const v = Math.floor(frameIndex / framesPerRow) / framesPerRow;
        player.material.map.offset.set(u, v);
        player.material.map.repeat.set(1 / framesPerRow, 1 / framesPerRow);
    }

    // Implement frustum culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateFrustumCulling() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use Web Workers for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        updatePhysicsResults(ballPosition, ballVelocity, playerPositions);
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Implement occlusion culling
    const occlusionCulling = new THREE.OcclusionCulling(camera, scene);

    function updateOcclusionCulling() {
        occlusionCulling.update();
    }

    // Use GPU instancing for crowd
    const crowdGeometry = new THREE.BoxGeometry(0.5, 1.8, 0.5);
    const crowdMaterial = new THREE.MeshPhongMaterial({ color: 0xcccccc });
    const crowdInstancedMesh = new THREE.InstancedMesh(crowdGeometry, crowdMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 120,
            1,
            (Math.random() - 0.5) * 80
        );
        const scale = new THREE.Vector3(1, 0.9 + Math.random() * 0.2, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI * 2, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        crowdInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(crowdInstancedMesh);

    // Implement dynamic LOD system
    const lodSystem = new THREE.LOD();

    players.forEach(player => {
        const highDetail = new THREE.Mesh(sharedPlayerGeometry, player.material);
        const mediumDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 16, 16), player.material);
        const lowDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 8, 8), player.material);

        lodSystem.addLevel(highDetail, 0);
        lodSystem.addLevel(mediumDetail, 20);
        lodSystem.addLevel(lowDetail, 50);

        player.add(lodSystem);
    });

    // Implement shader-based grass
    const grassShaderMaterial = new THREE.ShaderMaterial({
        vertexShader: `
            uniform float time;
            varying vec2 vUv;
            void main() {
                vUv = uv;
                vec3 pos = position;
                pos.y += sin(pos.x * 10.0 + time) * 0.1;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
            }
        `,
        fragmentShader: `
            uniform sampler2D grassTexture;
            varying vec2 vUv;
            void main() {
                gl_FragColor = texture2D(grassTexture, vUv);
            }
        `,
        uniforms: {
            time: { value: 0 },
            grassTexture: { value: new THREE.TextureLoader().load('textures/grass.jpg') }
        }
    });

    const grassMesh = new THREE.Mesh(
        new THREE.PlaneGeometry(100, 60, 100, 60),
        grassShaderMaterial
    );
    grassMesh.rotation.x = -Math.PI / 2;
    scene.add(grassMesh);

    // Implement dynamic resolution scaling
    let resolutionScale = 1;
    function updateResolutionScale() {
        const fps = 1 / deltaTime;
        if (fps < 30 && resolutionScale > 0.5) {
            resolutionScale -= 0.1;
        } else if (fps > 60 && resolutionScale < 1) {
            resolutionScale += 0.1;
        }
        renderer.setPixelRatio(window.devicePixelRatio * resolutionScale);
        composer.setPixelRatio(window.devicePixelRatio * resolutionScale);
    }

    // Update functions
    return {
        updateParticles: function() {
            particleSystem.geometry.attributes.position.needsUpdate = true;
        },
        updateGrass: function(time) {
            grassShaderMaterial.uniforms.time.value = time;
        },
        updateFrustumCulling: updateFrustumCulling,
        updateOcclusionCulling: updateOcclusionCulling,
        updatePhysics: updatePhysics,
        updateResolutionScale: updateResolutionScale
    };
}

const optimizations = optimizePerformance();

// Enhance the game loop with optimizations
function enhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            updateBallPhysics(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);

            // Apply optimizations
            optimizations.updateParticles();
            optimizations.updateGrass(time);
            optimizations.updateFrustumCulling();
            optimizations.updateOcclusionCulling();
            optimizations.updatePhysics();
            optimizations.updateResolutionScale();

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedReferee.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            advancedInjurySystem.update(deltaTime);
            advancedSubstitutionSystem.update();
            advancedFatigueSystem.update(deltaTime);
            particleSystem.update(deltaTime);
            offsideSystem.update();

            // Handle set pieces
            if (setPieceSystem.currentSetPiece) {
                setPieceSystem.executeSetPiece();
            }

            // Record replay frame
            replaySystem.recordFrame();

            // Check for goals
            checkForGoals();

            // Check for game end
            if (gameTime <= 0) {
                endGame();
            }
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            displayFinalResults();
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoop);
}

// Start the enhanced game loop
enhancedGameLoop(0);

// Final touches

// Implement advanced audio system with 3D positional audio
const audioListener = new THREE.AudioListener();
camera.add(audioListener);

const audioLoader = new THREE.AudioLoader();
const soundEffects = {
    kick: new THREE.PositionalAudio(audioListener),
    whistle: new THREE.PositionalAudio(audioListener),
    crowd: new THREE.Audio(audioListener)
};

audioLoader.load('sounds/kick.mp3', buffer => {
    soundEffects.kick.setBuffer(buffer);
    soundEffects.kick.setRefDistance(20);
});

audioLoader.load('sounds/whistle.mp3', buffer => {
    soundEffects.whistle.setBuffer(buffer);
    soundEffects.whistle.setRefDistance(50);
});

audioLoader.load('sounds/crowd.mp3', buffer => {
    soundEffects.crowd.setBuffer(buffer);
    soundEffects.crowd.setLoop(true);
    soundEffects.crowd.setVolume(0.5);
    soundEffects.crowd.play();
});

// Implement cinematic replay system
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 10 * 60, // 10 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballRotation: ball.quaternion.clone(),
            playerPositions: players.map(player => player.position.clone()),
            playerRotations: players.map(player => player.quaternion.clone()),
            cameraPosition: camera.position.clone(),
            cameraRotation: camera.quaternion.clone()
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        const replayCamera = camera.clone();
        scene.add(replayCamera);

        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                scene.remove(replayCamera);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ball.quaternion.copy(frame.ballRotation);

            players.forEach((player, index) => {
                player.position.copy(frame.playerPositions[index]);
                player.quaternion.copy(frame.playerRotations[index]);
            });

            replayCamera.position.copy(frame.cameraPosition);
            replayCamera.quaternion.copy(frame.cameraRotation);

            composer.render();
            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Implement advanced particle systems
const particleSystem = {
    systems: [],

    createEffect: function(position, color, count, speed, size, duration) {
        const particles = new THREE.BufferGeometry();
        const positions = new Float32Array(count * 3);
        const colors = new Float32Array(count * 3);
        const sizes = new Float32Array(count);

        for (let i = 0; i < count; i++) {
            positions[i * 3] = position.x;
            positions[i * 3 + 1] = position.y;
            positions[i * 3 + 2] = position.z;

            colors[i * 3] = color.r;
            colors[i * 3 + 1] = color.g;
            colors[i * 3 + 2] = color.b;

            sizes[i] = size;
        }

        particles.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        particles.setAttribute('color', new THREE.BufferAttribute(colors, 3));
        particles.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

        const material = new THREE.ShaderMaterial({
            uniforms: {
                time: { value: 0 },
                speed: { value: speed }
            },
            vertexShader: `
                uniform float time;
                uniform float speed;
                attribute float size;
                varying vec3 vColor;
                void main() {
                    vColor = color;
                    vec3 pos = position;
                    pos.y += speed * time;
                    vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
                    gl_PointSize = size * (300.0 / -mvPosition.z);
                    gl_Position = projectionMatrix * mvPosition;
                }
            `,
            fragmentShader: `
                varying vec3 vColor;
                void main() {
                    if (length(gl_PointCoord - vec2(0.5, 0.5)) > 0.5) discard;
                    gl_FragColor = vec4(vColor, 1.0);
                }
            `,
            blending: THREE.AdditiveBlending,
            depthTest: false,
            transparent: true
        });

        const particleSystem = new THREE.Points(particles, material);
        this.systems.push({ system: particleSystem, startTime: Date.now(), duration: duration });
        scene.add(particleSystem);
    },

    update: function(deltaTime) {
        this.systems.forEach((system, index) => {
            system.system.material.uniforms.time.value += deltaTime;

            if (Date.now() - system.startTime > system.duration) {
                scene.remove(system.system);
                this.systems.splice(index, 1);
            }
        });
    }
};

// Implement advanced post-processing effects
const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
composer.addPass(bloomPass);

const filmPass = new FilmPass(0.35, 0.5, 2048, false);
composer.addPass(filmPass);

const smaaPass = new SMAAPass(window.innerWidth * renderer.getPixelRatio(), window.innerHeight * renderer.getPixelRatio());
composer.addPass(smaaPass);

// Implement dynamic environment mapping
const pmremGenerator = new THREE.PMREMGenerator(renderer);
let envMap;

function updateEnvironmentMap() {
    const generatedCubeRenderTarget = pmremGenerator.fromScene(scene);
    envMap = generatedCubeRenderTarget.texture;

    scene.traverse((object) => {
        if (object.isMesh && object.material.envMap) {
            object.material.envMap = envMap;
            object.material.needsUpdate = true;
        }
    });

    pmremGenerator.dispose();
}

// Final game initialization
function initGame() {
    createScene();
    createLights();
    createPlayers();
    createBall();
    createStadium();
    createCrowd();
    initPhysics();
    initAudio();
    initUI();

    updateEnvironmentMap();

    // Start the game loop
    enhancedGameLoop(0);
}

// Start the game
initGame();

// Remove initial loading screen
document.getElementById('loading-screen').style.display = 'none';

// Event listeners
window.addEventListener('resize', onWindowResize);
document.addEventListener('keydown', onKeyDown);
document.addEventListener('keyup', onKeyUp);

// Game loop
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updatePhysics(deltaTime);
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

function updateGame(deltaTime) {
    gameTime -= deltaTime;
    if (gameTime <= 0) {
        endGame();
    }

    updatePlayers(deltaTime);
    updateBall(deltaTime);
    checkForGoals();
    checkForFouls();
    updateTactics(deltaTime);
}

function updatePhysics(deltaTime) {
    world.step(deltaTime);
    
    // Update Three.js objects with Cannon.js data
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);

    players.forEach((player, index) => {
        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);
    });
}

function updateAI(deltaTime) {
    players.forEach(player => {
        if (player.team !== humanTeam) {
            player.ai.update(deltaTime);
        }
    });
}

function updateParticles(deltaTime) {
    particleSystem.update(deltaTime);
}

function updateSound() {
    // Update 3D sound positions
    soundEffects.kick.position.copy(ball.position);
    soundEffects.whistle.position.copy(refereeSystem.position);

    // Update crowd sound based on excitement
    const excitement = calculateCrowdExcitement();
    soundEffects.crowd.setVolume(0.5 + excitement * 0.5);
}

function updateWeather(deltaTime) {
    weatherSystem.update(deltaTime);
}

function updateCamera(deltaTime) {
    const cameraTarget = getCameraTarget();
    camera.position.lerp(cameraTarget.position, 0.1);
    camera.lookAt(cameraTarget.lookAt);
}

function updateUI() {
    document.getElementById('score').textContent = `${scores.teamA} - ${scores.teamB}`;
    document.getElementById('time').textContent = formatTime(Math.max(0, gameTime));
    document.getElementById('stamina-bar').style.width = `${selectedPlayer.stamina}%`;
}

function render() {
    composer.render();
}

function endGame() {
    currentState = GameState.GAME_OVER;
    displayFinalScore();
    // Additional end game logic
}

// Helper functions
function formatTime(seconds) {
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = Math.floor(seconds % 60);
    return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
}

function calculateCrowdExcitement() {
    // Calculate crowd excitement based on game events
    // Return a value between 0 and 1
}

function getCameraTarget() {
    // Determine camera target based on current game state
    // Return an object with position and lookAt properties
}

// Start the game loop
gameLoop(0);

// Additional game systems

// Referee system
const refereeSystem = {
    position: new THREE.Vector3(),
    speed: 5,

    update: function(deltaTime) {
        // Move referee towards the ball
        const direction = new THREE.Vector3().subVectors(ball.position, this.position).normalize();
        this.position.add(direction.multiplyScalar(this.speed * deltaTime));

        // Check for fouls
        this.checkForFouls();
    },

    checkForFouls: function() {
        // Foul detection logic
    },

    callFoul: function(player) {
        console.log(`Foul called on player ${player.id} from team ${player.team}`);
        // Implement foul consequences
    }
};

// Tactics system
const tacticsSystem = {
    formations: {
        '4-4-2': [/* player positions */],
        '4-3-3': [/* player positions */],
        '3-5-2': [/* player positions */]
    },

    updateFormation: function(team, formation) {
        const teamPlayers = players.filter(p => p.team === team);
        const positions = this.formations[formation];

        teamPlayers.forEach((player, index) => {
            player.formationPosition.copy(positions[index]);
        });
    },

    updateTactics: function(deltaTime) {
        // Update team tactics based on game state
    }
};

// Weather system
const weatherSystem = {
    currentWeather: 'clear',
    transitionTime: 0,
    transitionDuration: 30, // seconds

    update: function(deltaTime) {
        this.transitionTime += deltaTime;
        if (this.transitionTime >= this.transitionDuration) {
            this.changeWeather();
        }
        this.updateWeatherEffects(deltaTime);
    },

    changeWeather: function() {
        const weathers = ['clear', 'cloudy', 'rainy', 'foggy'];
        this.currentWeather = weathers[Math.floor(Math.random() * weathers.length)];
        this.transitionTime = 0;
        console.log(`Weather changing to: ${this.currentWeather}`);
    },

    updateWeatherEffects: function(deltaTime) {
        // Update weather effects based on current weather
    }
};

// Commentary system
const commentarySystem = {
    phrases: {
        goal: [
            "GOAL! What a fantastic finish!",
            "He's done it! The ball is in the back of the net!",
            "An absolute screamer! The crowd goes wild!"
        ],
        miss: [
            "Oh, so close! The goalkeeper was beaten there.",
            "What a chance! He'll be disappointed not to score.",
            "Inches wide! That was a golden opportunity!"
        ],
        // Add more phrase categories
    },

    generateCommentary: function(event) {
        const phrases = this.phrases[event];
        const commentary = phrases[Math.floor(Math.random() * phrases.length)];
        console.log(`Commentary: ${commentary}`);
        // You could also trigger audio playback of the commentary
    }
};

// Injury system
const injurySystem = {
    injuredPlayers: new Set(),

    update: function(deltaTime) {
        players.forEach(player => {
            if (this.injuredPlayers.has(player)) {
                this.updateInjuredPlayer(player, deltaTime);
            } else if (Math.random() < 0.0001) { // Small chance of injury each frame
                this.injurePlayer(player);
            }
        });
    },

    injurePlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} is injured!`);
        this.injuredPlayers.add(player);
        player.speed *= 0.5;
        // Visual indication of injury
        player.material.color.setHex(0xff0000);
    },

    updateInjuredPlayer: function(player, deltaTime) {
        player.injuryTime += deltaTime;
        if (player.injuryTime > 30) { // Recover after 30 seconds
            this.recoverPlayer(player);
        }
    },

    recoverPlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} has recovered!`);
        this.injuredPlayers.delete(player);
        player.speed *= 2;
        player.injuryTime = 0;
        player.material.color.setHex(0xffffff);
    }
};

// Substitution system
const substitutionSystem = {
    substitutionsLeft: { A: 3, B: 3 },
    benchPlayers: { A: [], B: [] },

    initialize: function() {
        // Create bench players
        for (let team of ['A', 'B']) {
            for (let i = 0; i < 7; i++) {
                const benchPlayer = createPlayer(team);
                benchPlayer.position.set(0, 0, -40); // Off the field
                this.benchPlayers[team].push(benchPlayer);
            }
        }
    },

    requestSubstitution: function(team) {
        if (this.substitutionsLeft[team] > 0) {
            const playerToSubstitute = this.selectPlayerToSubstitute(team);
            const benchPlayer = this.selectBenchPlayer(team);

            if (playerToSubstitute && benchPlayer) {
                this.performSubstitution(playerToSubstitute, benchPlayer);
                this.substitutionsLeft[team]--;
            }
        }
    },

    selectPlayerToSubstitute: function(team) {
        // Select the most tired player
        return players
            .filter(p => p.team === team)
            .sort((a, b) => a.stamina - b.stamina)[0];
    },

    selectBenchPlayer: function(team) {
        // Select the most rested bench player
        return this.benchPlayers[team]
            .sort((a, b) => b.stamina - a.stamina)[0];
    },

    performSubstitution: function(playerOut, playerIn) {
        const index = players.indexOf(playerOut);
        players[index] = playerIn;
        playerIn.position.copy(playerOut.position);
        this.benchPlayers[playerOut.team] = this.benchPlayers[playerOut.team].filter(p => p !== playerIn);
        this.benchPlayers[playerOut.team].push(playerOut);
        playerOut.position.set(0, 0, -40); // Move to bench

        console.log(`Substitution: Player ${playerOut.id} is replaced by Player ${playerIn.id} in team ${playerOut.team}`);
    }
};

// Initialize additional systems
substitutionSystem.initialize();

// Add these systems to the game loop
function updateGame(deltaTime) {
    gameTime -= deltaTime;
    if (gameTime <= 0) {
        endGame();
    }

    updatePlayers(deltaTime);
    updateBall(deltaTime);
    checkForGoals();
    refereeSystem.update(deltaTime);
    tacticsSystem.updateTactics(deltaTime);
    weatherSystem.update(deltaTime);
    injurySystem.update(deltaTime);
    substitutionSystem.update();
}

// Final touches

// Implement replays
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 10 * 60, // 10 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballRotation: ball.quaternion.clone(),
            playerPositions: players.map(player => player.position.clone()),
            playerRotations: players.map(player => player.quaternion.clone()),
            cameraPosition: camera.position.clone(),
            cameraRotation: camera.quaternion.clone()
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        const replayCamera = camera.clone();
        scene.add(replayCamera);

        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                scene.remove(replayCamera);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ball.quaternion.copy(frame.ballRotation);

            players.forEach((player, index) => {
                player.position.copy(frame.playerPositions[index]);
                player.quaternion.copy(frame.playerRotations[index]);
            });

            replayCamera.position.copy(frame.cameraPosition);
            replayCamera.quaternion.copy(frame.cameraRotation);

            composer.render();
            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Add replay recording to the game loop
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updatePhysics(deltaTime);
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            replaySystem.recordFrame();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

// Implement cinematic camera for replays
const cinematicCamera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
scene.add(cinematicCamera);

function updateCinematicCamera(deltaTime) {
    // Implement cinematic camera movement
    const time = Date.now() * 0.001;
    const radius = 50;
    cinematicCamera.position.x = Math.cos(time * 0.5) * radius;
    cinematicCamera.position.z = Math.sin(time * 0.5) * radius;
    cinematicCamera.position.y = 20 + Math.sin(time) * 10;
    cinematicCamera.lookAt(ball.position);
}

// Add cinematic camera update to replay system
replaySystem.playReplay = function() {
    let frameIndex = 0;
    const replayInterval = setInterval(() => {
        if (frameIndex >= this.recordedFrames.length) {
            clearInterval(replayInterval);
            camera = mainCamera; // Switch back to main camera
            return;
        }

        const frame = this.recordedFrames[frameIndex];
        ball.position.copy(frame.ballPosition);
        ball.quaternion.copy(frame.ballRotation);

        players.forEach((player, index) => {
            player.position.copy(frame.playerPositions[index]);
            player.quaternion.copy(frame.playerRotations[index]);
        });

        updateCinematicCamera(1/60); // Assume 60 fps
        camera = cinematicCamera; // Use cinematic camera for replay

        composer.render();
        frameIndex++;
    }, 1000 / 60); // Play at 60 fps
};

// Implement slow motion
let timeScale = 1;

function updatePhysics(deltaTime) {
    world.step(deltaTime * timeScale);
    
    // Update Three.js objects with Cannon.js data
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);

    players.forEach((player, index) => {
        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);
    });
}

function setSlowMotion(enable) {
    timeScale = enable ? 0.2 : 1;
}

// Add slow motion to replay system
replaySystem.playReplay = function() {
    setSlowMotion(true);
    // ... existing replay code ...
    // When replay ends:
    setSlowMotion(false);
};

// Implement spectator reactions
const spectatorSystem = {
    spectators: [],
    excitement: 0,

    initialize: function() {
        // Create spectators
        for (let i = 0; i < 1000; i++) {
            const spectator = new THREE.Mesh(
                new THREE.BoxGeometry(0.5, 1.8, 0.5),
                new THREE.MeshLambertMaterial({ color: 0xcccccc })
            );
            spectator.position.set(
                (Math.random() - 0.5) * 120,
                1,
                (Math.random() - 0.5) * 80
            );
            scene.add(spectator);
            this.spectators.push(spectator);
        }
    },

    update: function(deltaTime) {
        this.excitement = Math.max(0, this.excitement - deltaTime * 0.1);
        this.spectators.forEach(spectator => {
            spectator.position.y = 1 + Math.sin(Date.now() * 0.01 + spectator.position.x) * 0.1 * this.excitement;
        });
    },

    cheer: function() {
        this.excitement = 1;
    }
};

spectatorSystem.initialize();

// Add spectator update to game loop
function updateGame(deltaTime) {
    // ... existing game update code ...
    spectatorSystem.update(deltaTime);
}

// Make spectators cheer on goals
function checkForGoals() {
    // ... existing goal check code ...
    if (goalScored) {
        spectatorSystem.cheer();
    }
}

// Final optimization
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement frustum culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateFrustumCulling() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use instancing for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement level of detail (LOD)
    const playerLODs = [];
    players.forEach(player => {
        const highDetailGeometry = new THREE.SphereGeometry(1, 32, 32);
        const mediumDetailGeometry = new THREE.SphereGeometry(1, 16, 16);
        const lowDetailGeometry = new THREE.SphereGeometry(1, 8, 8);

        const playerLOD = new THREE.LOD();

        playerLOD.addLevel(new THREE.Mesh(highDetailGeometry, player.material), 0);
        playerLOD.addLevel(new THREE.Mesh(mediumDetailGeometry, player.material), 10);
        playerLOD.addLevel(new THREE.Mesh(lowDetailGeometry, player.material), 50);

        playerLOD.position.copy(player.position);
        scene.add(playerLOD);
        playerLODs.push(playerLOD);
    });

    function updateLODs() {
        playerLODs.forEach((playerLOD, index) => {
            playerLOD.position.copy(players[index].position);
            playerLOD.update(camera);
        });
    }

    // Use worker threads for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        ball.position.copy(ballPosition);
        ballBody.velocity.copy(ballVelocity);
        players.forEach((player, index) => {
            player.position.copy(playerPositions[index]);
        });
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Update optimization functions
    return {
        updateFrustumCulling: updateFrustumCulling,
        updateLODs: updateLODs,
        updatePhysics: updatePhysics
    };
}

const optimizations = optimizePerformance();

// Update game loop with optimizations
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            optimizations.updatePhysics();
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            replaySystem.recordFrame();
            optimizations.updateFrustumCulling();
            optimizations.updateLODs();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

// Start the game
initGame();
gameLoop(0);

// Performance optimizations
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement occlusion culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateVisibility() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use instancing for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement texture atlases
    const textureAtlas = new THREE.TextureLoader().load('textures/texture_atlas.jpg');
    const atlasedMaterial = new THREE.MeshBasicMaterial({ map: textureAtlas });

    // Use shared geometries and materials
    const sharedPlayerGeometry = new THREE.SphereGeometry(0.5, 32, 32);
    const sharedTeamAMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
    const sharedTeamBMaterial = new THREE.MeshStandardMaterial({ color: 0x0000ff });

    players.forEach(player => {
        player.geometry = sharedPlayerGeometry;
        player.material = player.team === 'A' ? sharedTeamAMaterial : sharedTeamBMaterial;
    });

    // Implement mesh merging for static objects
    const staticGeometries = [];
    scene.traverse(object => {
        if (object.isStatic && object.isMesh) {
            staticGeometries.push(object.geometry);
        }
    });
    const mergedGeometry = BufferGeometryUtils.mergeBufferGeometries(staticGeometries);
    const mergedMesh = new THREE.Mesh(mergedGeometry, new THREE.MeshStandardMaterial());
    scene.add(mergedMesh);

    // Use sprite sheets for animations
    const spriteSheet = new THREE.TextureLoader().load('textures/player_animations.png');
    const spriteSheetMaterial = new THREE.SpriteMaterial({ map: spriteSheet });

    function updatePlayerSprite(player, frameIndex) {
        const spriteSize = 64; // Assuming 64x64 pixel sprites
        const framesPerRow = 8;
        const u = (frameIndex % framesPerRow) / framesPerRow;
        const v = Math.floor(frameIndex / framesPerRow) / framesPerRow;
        player.material.map.offset.set(u, v);
        player.material.map.repeat.set(1 / framesPerRow, 1 / framesPerRow);
    }

    // Implement frustum culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateFrustumCulling() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use Web Workers for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        updatePhysicsResults(ballPosition, ballVelocity, playerPositions);
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Implement occlusion culling
    const occlusionCulling = new THREE.OcclusionCulling(camera, scene);

    function updateOcclusionCulling() {
        occlusionCulling.update();
    }

    // Use GPU instancing for crowd
    const crowdGeometry = new THREE.BoxGeometry(0.5, 1.8, 0.5);
    const crowdMaterial = new THREE.MeshPhongMaterial({ color: 0xcccccc });
    const crowdInstancedMesh = new THREE.InstancedMesh(crowdGeometry, crowdMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 120,
            1,
            (Math.random() - 0.5) * 80
        );
        const scale = new THREE.Vector3(1, 0.9 + Math.random() * 0.2, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI * 2, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        crowdInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(crowdInstancedMesh);

    // Implement dynamic LOD system
    const lodSystem = new THREE.LOD();

    players.forEach(player => {
        const highDetail = new THREE.Mesh(sharedPlayerGeometry, player.material);
        const mediumDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 16, 16), player.material);
        const lowDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 8, 8), player.material);

        lodSystem.addLevel(highDetail, 0);
        lodSystem.addLevel(mediumDetail, 20);
        lodSystem.addLevel(lowDetail, 50);

        player.add(lodSystem);
    });

    // Implement shader-based grass
    const grassShaderMaterial = new THREE.ShaderMaterial({
        vertexShader: `
            uniform float time;
            varying vec2 vUv;
            void main() {
                vUv = uv;
                vec3 pos = position;
                pos.y += sin(pos.x * 10.0 + time) * 0.1;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
            }
        `,
        fragmentShader: `
            uniform sampler2D grassTexture;
            varying vec2 vUv;
            void main() {
                gl_FragColor = texture2D(grassTexture, vUv);
            }
        `,
        uniforms: {
            time: { value: 0 },
            grassTexture: { value: new THREE.TextureLoader().load('textures/grass.jpg') }
        }
    });

    const grassMesh = new THREE.Mesh(
        new THREE.PlaneGeometry(100, 60, 100, 60),
        grassShaderMaterial
    );
    grassMesh.rotation.x = -Math.PI / 2;
    scene.add(grassMesh);

    // Implement dynamic resolution scaling
    let resolutionScale = 1;
    function updateResolutionScale() {
        const fps = 1 / deltaTime;
        if (fps < 30 && resolutionScale > 0.5) {
            resolutionScale -= 0.1;
        } else if (fps > 60 && resolutionScale < 1) {
            resolutionScale += 0.1;
        }
        renderer.setPixelRatio(window.devicePixelRatio * resolutionScale);
        composer.setPixelRatio(window.devicePixelRatio * resolutionScale);
    }

    // Update functions
    return {
        updateParticles: function() {
            particleSystem.geometry.attributes.position.needsUpdate = true;
        },
        updateGrass: function(time) {
            grassShaderMaterial.uniforms.time.value = time;
        },
        updateFrustumCulling: updateFrustumCulling,
        updateOcclusionCulling: updateOcclusionCulling,
        updatePhysics: updatePhysics,
        updateResolutionScale: updateResolutionScale
    };
}

const optimizations = optimizePerformance();

// Enhance the game loop with optimizations
function enhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            updateBallPhysics(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);

            // Apply optimizations
            optimizations.updateParticles();
            optimizations.updateGrass(time);
            optimizations.updateFrustumCulling();
            optimizations.updateOcclusionCulling();
            optimizations.updatePhysics();
            optimizations.updateResolutionScale();

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedReferee.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            advancedInjurySystem.update(deltaTime);
            advancedSubstitutionSystem.update();
            advancedFatigueSystem.update(deltaTime);
            particleSystem.update(deltaTime);
            offsideSystem.update();

            // Handle set pieces
            if (setPieceSystem.currentSetPiece) {
                setPieceSystem.executeSetPiece();
            }

            // Record replay frame
            replaySystem.recordFrame();

            // Check for goals
            checkForGoals();

            // Check for game end
            if (gameTime <= 0) {
                endGame();
            }
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            displayFinalResults();
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoop);
}

// Start the enhanced game loop
enhancedGameLoop(0);

// Final touches

// Implement advanced audio system with 3D positional audio
const audioListener = new THREE.AudioListener();
camera.add(audioListener);

const audioLoader = new THREE.AudioLoader();
const soundEffects = {
    kick: new THREE.PositionalAudio(audioListener),
    whistle: new THREE.PositionalAudio(audioListener),
    crowd: new THREE.Audio(audioListener)
};

audioLoader.load('sounds/kick.mp3', buffer => {
    soundEffects.kick.setBuffer(buffer);
    soundEffects.kick.setRefDistance(20);
});

audioLoader.load('sounds/whistle.mp3', buffer => {
    soundEffects.whistle.setBuffer(buffer);
    soundEffects.whistle.setRefDistance(50);
});

audioLoader.load('sounds/crowd.mp3', buffer => {
    soundEffects.crowd.setBuffer(buffer);
    soundEffects.crowd.setLoop(true);
    soundEffects.crowd.setVolume(0.5);
    soundEffects.crowd.play();
});

// Implement cinematic replay system
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 10 * 60, // 10 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballRotation: ball.quaternion.clone(),
            playerPositions: players.map(player => player.position.clone()),
            playerRotations: players.map(player => player.quaternion.clone()),
            cameraPosition: camera.position.clone(),
            cameraRotation: camera.quaternion.clone()
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        const replayCamera = camera.clone();
        scene.add(replayCamera);

        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                scene.remove(replayCamera);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ball.quaternion.copy(frame.ballRotation);

            players.forEach((player, index) => {
                player.position.copy(frame.playerPositions[index]);
                player.quaternion.copy(frame.playerRotations[index]);
            });

            replayCamera.position.copy(frame.cameraPosition);
            replayCamera.quaternion.copy(frame.cameraRotation);

            composer.render();
            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Implement advanced particle systems
const particleSystem = {
    systems: [],

    createEffect: function(position, color, count, speed, size, duration) {
        const particles = new THREE.BufferGeometry();
        const positions = new Float32Array(count * 3);
        const colors = new Float32Array(count * 3);
        const sizes = new Float32Array(count);

        for (let i = 0; i < count; i++) {
            positions[i * 3] = position.x;
            positions[i * 3 + 1] = position.y;
            positions[i * 3 + 2] = position.z;

            colors[i * 3] = color.r;
            colors[i * 3 + 1] = color.g;
            colors[i * 3 + 2] = color.b;

            sizes[i] = size;
        }

        particles.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        particles.setAttribute('color', new THREE.BufferAttribute(colors, 3));
        particles.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

        const material = new THREE.ShaderMaterial({
            uniforms: {
                time: { value: 0 },
                speed: { value: speed }
            },
            vertexShader: `
                uniform float time;
                uniform float speed;
                attribute float size;
                varying vec3 vColor;
                void main() {
                    vColor = color;
                    vec3 pos = position;
                    pos.y += speed * time;
                    vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
                    gl_PointSize = size * (300.0 / -mvPosition.z);
                    gl_Position = projectionMatrix * mvPosition;
                }
            `,
            fragmentShader: `
                varying vec3 vColor;
                void main() {
                    if (length(gl_PointCoord - vec2(0.5, 0.5)) > 0.5) discard;
                    gl_FragColor = vec4(vColor, 1.0);
                }
            `,
            blending: THREE.AdditiveBlending,
            depthTest: false,
            transparent: true
        });

        const particleSystem = new THREE.Points(particles, material);
        this.systems.push({ system: particleSystem, startTime: Date.now(), duration: duration });
        scene.add(particleSystem);
    },

    update: function(deltaTime) {
        this.systems.forEach((system, index) => {
            system.system.material.uniforms.time.value += deltaTime;

            if (Date.now() - system.startTime > system.duration) {
                scene.remove(system.system);
                this.systems.splice(index, 1);
            }
        });
    }
};

// Implement advanced post-processing effects
const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
composer.addPass(bloomPass);

const filmPass = new FilmPass(0.35, 0.5, 2048, false);
composer.addPass(filmPass);

const smaaPass = new SMAAPass(window.innerWidth * renderer.getPixelRatio(), window.innerHeight * renderer.getPixelRatio());
composer.addPass(smaaPass);

// Implement dynamic environment mapping
const pmremGenerator = new THREE.PMREMGenerator(renderer);
let envMap;

function updateEnvironmentMap() {
    const generatedCubeRenderTarget = pmremGenerator.fromScene(scene);
    envMap = generatedCubeRenderTarget.texture;

    scene.traverse((object) => {
        if (object.isMesh && object.material.envMap) {
            object.material.envMap = envMap;
            object.material.needsUpdate = true;
        }
    });

    pmremGenerator.dispose();
}

// Final game initialization
function initGame() {
    createScene();
    createLights();
    createPlayers();
    createBall();
    createStadium();
    createCrowd();
    initPhysics();
    initAudio();
    initUI();

    updateEnvironmentMap();

    // Start the game loop
    enhancedGameLoop(0);
}

// Start the game
initGame();

// Remove initial loading screen
document.getElementById('loading-screen').style.display = 'none';

// Event listeners
window.addEventListener('resize', onWindowResize);
document.addEventListener('keydown', onKeyDown);
document.addEventListener('keyup', onKeyUp);

// Game loop
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updatePhysics(deltaTime);
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

function updateGame(deltaTime) {
    gameTime -= deltaTime;
    if (gameTime <= 0) {
        endGame();
    }

    updatePlayers(deltaTime);
    updateBall(deltaTime);
    checkForGoals();
    checkForFouls();
    updateTactics(deltaTime);
}

function updatePhysics(deltaTime) {
    world.step(deltaTime);
    
    // Update Three.js objects with Cannon.js data
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);

    players.forEach((player, index) => {
        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);
    });
}

function updateAI(deltaTime) {
    players.forEach(player => {
        if (player.team !== humanTeam) {
            player.ai.update(deltaTime);
        }
    });
}

function updateParticles(deltaTime) {
    particleSystem.update(deltaTime);
}

function updateSound() {
    // Update 3D sound positions
    soundEffects.kick.position.copy(ball.position);
    soundEffects.whistle.position.copy(refereeSystem.position);

    // Update crowd sound based on excitement
    const excitement = calculateCrowdExcitement();
    soundEffects.crowd.setVolume(0.5 + excitement * 0.5);
}

function updateWeather(deltaTime) {
    weatherSystem.update(deltaTime);
}

function updateCamera(deltaTime) {
    const cameraTarget = getCameraTarget();
    camera.position.lerp(cameraTarget.position, 0.1);
    camera.lookAt(cameraTarget.lookAt);
}

function updateUI() {
    document.getElementById('score').textContent = `${scores.teamA} - ${scores.teamB}`;
    document.getElementById('time').textContent = formatTime(Math.max(0, gameTime));
    document.getElementById('stamina-bar').style.width = `${selectedPlayer.stamina}%`;
}

function render() {
    composer.render();
}

function endGame() {
    currentState = GameState.GAME_OVER;
    displayFinalScore();
    // Additional end game logic
}

// Helper functions
function formatTime(seconds) {
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = Math.floor(seconds % 60);
    return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
}

function calculateCrowdExcitement() {
    // Calculate crowd excitement based on game events
    // Return a value between 0 and 1
}

function getCameraTarget() {
    // Determine camera target based on current game state
    // Return an object with position and lookAt properties
}

// Start the game loop
gameLoop(0);

// Additional game systems

// Referee system
const refereeSystem = {
    position: new THREE.Vector3(),
    speed: 5,

    update: function(deltaTime) {
        // Move referee towards the ball
        const direction = new THREE.Vector3().subVectors(ball.position, this.position).normalize();
        this.position.add(direction.multiplyScalar(this.speed * deltaTime));

        // Check for fouls
        this.checkForFouls();
    },

    checkForFouls: function() {
        // Foul detection logic
    },

    callFoul: function(player) {
        console.log(`Foul called on player ${player.id} from team ${player.team}`);
        // Implement foul consequences
    }
};

// Tactics system
const tacticsSystem = {
    formations: {
        '4-4-2': [/* player positions */],
        '4-3-3': [/* player positions */],
        '3-5-2': [/* player positions */]
    },

    updateFormation: function(team, formation) {
        const teamPlayers = players.filter(p => p.team === team);
        const positions = this.formations[formation];

        teamPlayers.forEach((player, index) => {
            player.formationPosition.copy(positions[index]);
        });
    },

    updateTactics: function(deltaTime) {
        // Update team tactics based on game state
    }
};

// Weather system
const weatherSystem = {
    currentWeather: 'clear',
    transitionTime: 0,
    transitionDuration: 30, // seconds

    update: function(deltaTime) {
        this.transitionTime += deltaTime;
        if (this.transitionTime >= this.transitionDuration) {
            this.changeWeather();
        }
        this.updateWeatherEffects(deltaTime);
    },

    changeWeather: function() {
        const weathers = ['clear', 'cloudy', 'rainy', 'foggy'];
        this.currentWeather = weathers[Math.floor(Math.random() * weathers.length)];
        this.transitionTime = 0;
        console.log(`Weather changing to: ${this.currentWeather}`);
    },

    updateWeatherEffects: function(deltaTime) {
        // Update weather effects based on current weather
    }
};

// Commentary system
const commentarySystem = {
    phrases: {
        goal: [
            "GOAL! What a fantastic finish!",
            "He's done it! The ball is in the back of the net!",
            "An absolute screamer! The crowd goes wild!"
        ],
        miss: [
            "Oh, so close! The goalkeeper was beaten there.",
            "What a chance! He'll be disappointed not to score.",
            "Inches wide! That was a golden opportunity!"
        ],
        // Add more phrase categories
    },

    generateCommentary: function(event) {
        const phrases = this.phrases[event];
        const commentary = phrases[Math.floor(Math.random() * phrases.length)];
        console.log(`Commentary: ${commentary}`);
        // You could also trigger audio playback of the commentary
    }
};

// Injury system
const injurySystem = {
    injuredPlayers: new Set(),

    update: function(deltaTime) {
        players.forEach(player => {
            if (this.injuredPlayers.has(player)) {
                this.updateInjuredPlayer(player, deltaTime);
            } else if (Math.random() < 0.0001) { // Small chance of injury each frame
                this.injurePlayer(player);
            }
        });
    },

    injurePlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} is injured!`);
        this.injuredPlayers.add(player);
        player.speed *= 0.5;
        // Visual indication of injury
        player.material.color.setHex(0xff0000);
    },

    updateInjuredPlayer: function(player, deltaTime) {
        player.injuryTime += deltaTime;
        if (player.injuryTime > 30) { // Recover after 30 seconds
            this.recoverPlayer(player);
        }
    },

    recoverPlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} has recovered!`);
        this.injuredPlayers.delete(player);
        player.speed *= 2;
        player.injuryTime = 0;
        player.material.color.setHex(0xffffff);
    }
};

// Substitution system
const substitutionSystem = {
    substitutionsLeft: { A: 3, B: 3 },
    benchPlayers: { A: [], B: [] },

    initialize: function() {
        // Create bench players
        for (let team of ['A', 'B']) {
            for (let i = 0; i < 7; i++) {
                const benchPlayer = createPlayer(team);
                benchPlayer.position.set(0, 0, -40); // Off the field
                this.benchPlayers[team].push(benchPlayer);
            }
        }
    },

    requestSubstitution: function(team) {
        if (this.substitutionsLeft[team] > 0) {
            const playerToSubstitute = this.selectPlayerToSubstitute(team);
            const benchPlayer = this.selectBenchPlayer(team);

            if (playerToSubstitute && benchPlayer) {
                this.performSubstitution(playerToSubstitute, benchPlayer);
                this.substitutionsLeft[team]--;
            }
        }
    },

    selectPlayerToSubstitute: function(team) {
        // Select the most tired player
        return players
            .filter(p => p.team === team)
            .sort((a, b) => a.stamina - b.stamina)[0];
    },

    selectBenchPlayer: function(team) {
        // Select the most rested bench player
        return this.benchPlayers[team]
            .sort((a, b) => b.stamina - a.stamina)[0];
    },

    performSubstitution: function(playerOut, playerIn) {
        const index = players.indexOf(playerOut);
        players[index] = playerIn;
        playerIn.position.copy(playerOut.position);
        this.benchPlayers[playerOut.team] = this.benchPlayers[playerOut.team].filter(p => p !== playerIn);
        this.benchPlayers[playerOut.team].push(playerOut);
        playerOut.position.set(0, 0, -40); // Move to bench

        console.log(`Substitution: Player ${playerOut.id} is replaced by Player ${playerIn.id} in team ${playerOut.team}`);
    }
};

// Initialize additional systems
substitutionSystem.initialize();

// Add these systems to the game loop
function updateGame(deltaTime) {
    gameTime -= deltaTime;
    if (gameTime <= 0) {
        endGame();
    }

    updatePlayers(deltaTime);
    updateBall(deltaTime);
    checkForGoals();
    refereeSystem.update(deltaTime);
    tacticsSystem.updateTactics(deltaTime);
    weatherSystem.update(deltaTime);
    injurySystem.update(deltaTime);
    substitutionSystem.update();
}

// Final touches

// Implement replays
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 10 * 60, // 10 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballRotation: ball.quaternion.clone(),
            playerPositions: players.map(player => player.position.clone()),
            playerRotations: players.map(player => player.quaternion.clone()),
            cameraPosition: camera.position.clone(),
            cameraRotation: camera.quaternion.clone()
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        const replayCamera = camera.clone();
        scene.add(replayCamera);

        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                scene.remove(replayCamera);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ball.quaternion.copy(frame.ballRotation);

            players.forEach((player, index) => {
                player.position.copy(frame.playerPositions[index]);
                player.quaternion.copy(frame.playerRotations[index]);
            });

            replayCamera.position.copy(frame.cameraPosition);
            replayCamera.quaternion.copy(frame.cameraRotation);

            composer.render();
            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Add replay recording to the game loop
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updatePhysics(deltaTime);
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            replaySystem.recordFrame();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

// Implement cinematic camera for replays
const cinematicCamera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
scene.add(cinematicCamera);

function updateCinematicCamera(deltaTime) {
    // Implement cinematic camera movement
    const time = Date.now() * 0.001;
    const radius = 50;
    cinematicCamera.position.x = Math.cos(time * 0.5) * radius;
    cinematicCamera.position.z = Math.sin(time * 0.5) * radius;
    cinematicCamera.position.y = 20 + Math.sin(time) * 10;
    cinematicCamera.lookAt(ball.position);
}

// Add cinematic camera update to replay system
replaySystem.playReplay = function() {
    let frameIndex = 0;
    const replayInterval = setInterval(() => {
        if (frameIndex >= this.recordedFrames.length) {
            clearInterval(replayInterval);
            camera = mainCamera; // Switch back to main camera
            return;
        }

        const frame = this.recordedFrames[frameIndex];
        ball.position.copy(frame.ballPosition);
        ball.quaternion.copy(frame.ballRotation);

        players.forEach((player, index) => {
            player.position.copy(frame.playerPositions[index]);
            player.quaternion.copy(frame.playerRotations[index]);
        });

        updateCinematicCamera(1/60); // Assume 60 fps
        camera = cinematicCamera; // Use cinematic camera for replay

        composer.render();
        frameIndex++;
    }, 1000 / 60); // Play at 60 fps
};

// Implement slow motion
let timeScale = 1;

function updatePhysics(deltaTime) {
    world.step(deltaTime * timeScale);
    
    // Update Three.js objects with Cannon.js data
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);

    players.forEach((player, index) => {
        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);
    });
}

function setSlowMotion(enable) {
    timeScale = enable ? 0.2 : 1;
}

// Add slow motion to replay system
replaySystem.playReplay = function() {
    setSlowMotion(true);
    // ... existing replay code ...
    // When replay ends:
    setSlowMotion(false);
};

// Implement spectator reactions
const spectatorSystem = {
    spectators: [],
    excitement: 0,

    initialize: function() {
        // Create spectators
        for (let i = 0; i < 1000; i++) {
            const spectator = new THREE.Mesh(
                new THREE.BoxGeometry(0.5, 1.8, 0.5),
                new THREE.MeshLambertMaterial({ color: 0xcccccc })
            );
            spectator.position.set(
                (Math.random() - 0.5) * 120,
                1,
                (Math.random() - 0.5) * 80
            );
            scene.add(spectator);
            this.spectators.push(spectator);
        }
    },

    update: function(deltaTime) {
        this.excitement = Math.max(0, this.excitement - deltaTime * 0.1);
        this.spectators.forEach(spectator => {
            spectator.position.y = 1 + Math.sin(Date.now() * 0.01 + spectator.position.x) * 0.1 * this.excitement;
        });
    },

    cheer: function() {
        this.excitement = 1;
    }
};

spectatorSystem.initialize();

// Add spectator update to game loop
function updateGame(deltaTime) {
    // ... existing game update code ...
    spectatorSystem.update(deltaTime);
}

// Make spectators cheer on goals
function checkForGoals() {
    // ... existing goal check code ...
    if (goalScored) {
        spectatorSystem.cheer();
    }
}

// Final optimization
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement frustum culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateFrustumCulling() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use instancing for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement level of detail (LOD)
    const playerLODs = [];
    players.forEach(player => {
        const highDetailGeometry = new THREE.SphereGeometry(1, 32, 32);
        const mediumDetailGeometry = new THREE.SphereGeometry(1, 16, 16);
        const lowDetailGeometry = new THREE.SphereGeometry(1, 8, 8);

        const playerLOD = new THREE.LOD();

        playerLOD.addLevel(new THREE.Mesh(highDetailGeometry, player.material), 0);
        playerLOD.addLevel(new THREE.Mesh(mediumDetailGeometry, player.material), 10);
        playerLOD.addLevel(new THREE.Mesh(lowDetailGeometry, player.material), 50);

        playerLOD.position.copy(player.position);
        scene.add(playerLOD);
        playerLODs.push(playerLOD);
    });

    function updateLODs() {
        playerLODs.forEach((playerLOD, index) => {
            playerLOD.position.copy(players[index].position);
            playerLOD.update(camera);
        });
    }

    // Use worker threads for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        ball.position.copy(ballPosition);
        ballBody.velocity.copy(ballVelocity);
        players.forEach((player, index) => {
            player.position.copy(playerPositions[index]);
        });
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Update optimization functions
    return {
        updateFrustumCulling: updateFrustumCulling,
        updateLODs: updateLODs,
        updatePhysics: updatePhysics
    };
}

const optimizations = optimizePerformance();

// Update game loop with optimizations
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            optimizations.updatePhysics();
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            replaySystem.recordFrame();
            optimizations.updateFrustumCulling();
            optimizations.updateLODs();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

// Start the game
initGame();
gameLoop(0);

// Performance optimizations
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement occlusion culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateVisibility() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use instancing for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement texture atlases
    const textureAtlas = new THREE.TextureLoader().load('textures/texture_atlas.jpg');
    const atlasedMaterial = new THREE.MeshBasicMaterial({ map: textureAtlas });

    // Use shared geometries and materials
    const sharedPlayerGeometry = new THREE.SphereGeometry(0.5, 32, 32);
    const sharedTeamAMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
    const sharedTeamBMaterial = new THREE.MeshStandardMaterial({ color: 0x0000ff });

    players.forEach(player => {
        player.geometry = sharedPlayerGeometry;
        player.material = player.team === 'A' ? sharedTeamAMaterial : sharedTeamBMaterial;
    });

    // Implement mesh merging for static objects
    const staticGeometries = [];
    scene.traverse(object => {
        if (object.isStatic && object.isMesh) {
            staticGeometries.push(object.geometry);
        }
    });
    const mergedGeometry = BufferGeometryUtils.mergeBufferGeometries(staticGeometries);
    const mergedMesh = new THREE.Mesh(mergedGeometry, new THREE.MeshStandardMaterial());
    scene.add(mergedMesh);

    // Use sprite sheets for animations
    const spriteSheet = new THREE.TextureLoader().load('textures/player_animations.png');
    const spriteSheetMaterial = new THREE.SpriteMaterial({ map: spriteSheet });

    function updatePlayerSprite(player, frameIndex) {
        const spriteSize = 64; // Assuming 64x64 pixel sprites
        const framesPerRow = 8;
        const u = (frameIndex % framesPerRow) / framesPerRow;
        const v = Math.floor(frameIndex / framesPerRow) / framesPerRow;
        player.material.map.offset.set(u, v);
        player.material.map.repeat.set(1 / framesPerRow, 1 / framesPerRow);
    }

    // Implement frustum culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateFrustumCulling() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use Web Workers for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        updatePhysicsResults(ballPosition, ballVelocity, playerPositions);
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Implement occlusion culling
    const occlusionCulling = new THREE.OcclusionCulling(camera, scene);

    function updateOcclusionCulling() {
        occlusionCulling.update();
    }

    // Use GPU instancing for crowd
    const crowdGeometry = new THREE.BoxGeometry(0.5, 1.8, 0.5);
    const crowdMaterial = new THREE.MeshPhongMaterial({ color: 0xcccccc });
    const crowdInstancedMesh = new THREE.InstancedMesh(crowdGeometry, crowdMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 120,
            1,
            (Math.random() - 0.5) * 80
        );
        const scale = new THREE.Vector3(1, 0.9 + Math.random() * 0.2, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI * 2, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        crowdInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(crowdInstancedMesh);

    // Implement dynamic LOD system
    const lodSystem = new THREE.LOD();

    players.forEach(player => {
        const highDetail = new THREE.Mesh(sharedPlayerGeometry, player.material);
        const mediumDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 16, 16), player.material);
        const lowDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 8, 8), player.material);

        lodSystem.addLevel(highDetail, 0);
        lodSystem.addLevel(mediumDetail, 20);
        lodSystem.addLevel(lowDetail, 50);

        player.add(lodSystem);
    });

    // Implement shader-based grass
    const grassShaderMaterial = new THREE.ShaderMaterial({
        vertexShader: `
            uniform float time;
            varying vec2 vUv;
            void main() {
                vUv = uv;
                vec3 pos = position;
                pos.y += sin(pos.x * 10.0 + time) * 0.1;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
            }
        `,
        fragmentShader: `
            uniform sampler2D grassTexture;
            varying vec2 vUv;
            void main() {
                gl_FragColor = texture2D(grassTexture, vUv);
            }
        `,
        uniforms: {
            time: { value: 0 },
            grassTexture: { value: new THREE.TextureLoader().load('textures/grass.jpg') }
        }
    });

    const grassMesh = new THREE.Mesh(
        new THREE.PlaneGeometry(100, 60, 100, 60),
        grassShaderMaterial
    );
    grassMesh.rotation.x = -Math.PI / 2;
    scene.add(grassMesh);

    // Implement dynamic resolution scaling
    let resolutionScale = 1;
    function updateResolutionScale() {
        const fps = 1 / deltaTime;
        if (fps < 30 && resolutionScale > 0.5) {
            resolutionScale -= 0.1;
        } else if (fps > 60 && resolutionScale < 1) {
            resolutionScale += 0.1;
        }
        renderer.setPixelRatio(window.devicePixelRatio * resolutionScale);
        composer.setPixelRatio(window.devicePixelRatio * resolutionScale);
    }

    // Update functions
    return {
        updateParticles: function() {
            particleSystem.geometry.attributes.position.needsUpdate = true;
        },
        updateGrass: function(time) {
            grassShaderMaterial.uniforms.time.value = time;
        },
        updateFrustumCulling: updateFrustumCulling,
        updateOcclusionCulling: updateOcclusionCulling,
        updatePhysics: updatePhysics,
        updateResolutionScale: updateResolutionScale
    };
}

const optimizations = optimizePerformance();

// Enhance the game loop with optimizations
function enhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            updateBallPhysics(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);

            // Apply optimizations
            optimizations.updateParticles();
            optimizations.updateGrass(time);
            optimizations.updateFrustumCulling();
            optimizations.updateOcclusionCulling();
            optimizations.updatePhysics();
            optimizations.updateResolutionScale();

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedReferee.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            advancedInjurySystem.update(deltaTime);
            advancedSubstitutionSystem.update();
            advancedFatigueSystem.update(deltaTime);
            particleSystem.update(deltaTime);
            offsideSystem.update();

            // Handle set pieces
            if (setPieceSystem.currentSetPiece) {
                setPieceSystem.executeSetPiece();
            }

            // Record replay frame
            replaySystem.recordFrame();

            // Check for goals
            checkForGoals();

            // Check for game end
            if (gameTime <= 0) {
                endGame();
            }
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            displayFinalResults();
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoop);
}

// Start the enhanced game loop
enhancedGameLoop(0);

// Final touches

// Implement advanced audio system with 3D positional audio
const audioListener = new THREE.AudioListener();
camera.add(audioListener);

const audioLoader = new THREE.AudioLoader();
const soundEffects = {
    kick: new THREE.PositionalAudio(audioListener),
    whistle: new THREE.PositionalAudio(audioListener),
    crowd: new THREE.Audio(audioListener)
};

audioLoader.load('sounds/kick.mp3', buffer => {
    soundEffects.kick.setBuffer(buffer);
    soundEffects.kick.setRefDistance(20);
});

audioLoader.load('sounds/whistle.mp3', buffer => {
    soundEffects.whistle.setBuffer(buffer);
    soundEffects.whistle.setRefDistance(50);
});

audioLoader.load('sounds/crowd.mp3', buffer => {
    soundEffects.crowd.setBuffer(buffer);
    soundEffects.crowd.setLoop(true);
    soundEffects.crowd.setVolume(0.5);
    soundEffects.crowd.play();
});

// Implement cinematic replay system
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 10 * 60, // 10 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballRotation: ball.quaternion.clone(),
            playerPositions: players.map(player => player.position.clone()),
            playerRotations: players.map(player => player.quaternion.clone()),
            cameraPosition: camera.position.clone(),
            cameraRotation: camera.quaternion.clone()
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        const replayCamera = camera.clone();
        scene.add(replayCamera);

        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                scene.remove(replayCamera);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ball.quaternion.copy(frame.ballRotation);

            players.forEach((player, index) => {
                player.position.copy(frame.playerPositions[index]);
                player.quaternion.copy(frame.playerRotations[index]);
            });

            replayCamera.position.copy(frame.cameraPosition);
            replayCamera.quaternion.copy(frame.cameraRotation);

            composer.render();
            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Implement advanced particle systems
const particleSystem = {
    systems: [],

    createEffect: function(position, color, count, speed, size, duration) {
        const particles = new THREE.BufferGeometry();
        const positions = new Float32Array(count * 3);
        const colors = new Float32Array(count * 3);
        const sizes = new Float32Array(count);

        for (let i = 0; i < count; i++) {
            positions[i * 3] = position.x;
            positions[i * 3 + 1] = position.y;
            positions[i * 3 + 2] = position.z;

            colors[i * 3] = color.r;
            colors[i * 3 + 1] = color.g;
            colors[i * 3 + 2] = color.b;

            sizes[i] = size;
        }

        particles.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        particles.setAttribute('color', new THREE.BufferAttribute(colors, 3));
        particles.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

        const material = new THREE.ShaderMaterial({
            uniforms: {
                time: { value: 0 },
                speed: { value: speed }
            },
            vertexShader: `
                uniform float time;
                uniform float speed;
                attribute float size;
                varying vec3 vColor;
                void main() {
                    vColor = color;
                    vec3 pos = position;
                    pos.y += speed * time;
                    vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
                    gl_PointSize = size * (300.0 / -mvPosition.z);
                    gl_Position = projectionMatrix * mvPosition;
                }
            `,
            fragmentShader: `
                varying vec3 vColor;
                void main() {
                    if (length(gl_PointCoord - vec2(0.5, 0.5)) > 0.5) discard;
                    gl_FragColor = vec4(vColor, 1.0);
                }
            `,
            blending: THREE.AdditiveBlending,
            depthTest: false,
            transparent: true
        });

        const particleSystem = new THREE.Points(particles, material);
        this.systems.push({ system: particleSystem, startTime: Date.now(), duration: duration });
        scene.add(particleSystem);
    },

    update: function(deltaTime) {
        this.systems.forEach((system, index) => {
            system.system.material.uniforms.time.value += deltaTime;

            if (Date.now() - system.startTime > system.duration) {
                scene.remove(system.system);
                this.systems.splice(index, 1);
            }
        });
    }
};

// Implement advanced post-processing effects
const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
composer.addPass(bloomPass);

const filmPass = new FilmPass(0.35, 0.5, 2048, false);
composer.addPass(filmPass);

const smaaPass = new SMAAPass(window.innerWidth * renderer.getPixelRatio(), window.innerHeight * renderer.getPixelRatio());
composer.addPass(smaaPass);

// Implement dynamic environment mapping
const pmremGenerator = new THREE.PMREMGenerator(renderer);
let envMap;

function updateEnvironmentMap() {
    const generatedCubeRenderTarget = pmremGenerator.fromScene(scene);
    envMap = generatedCubeRenderTarget.texture;

    scene.traverse((object) => {
        if (object.isMesh && object.material.envMap) {
            object.material.envMap = envMap;
            object.material.needsUpdate = true;
        }
    });

    pmremGenerator.dispose();
}

// Final game initialization
function initGame() {
    createScene();
    createLights();
    createPlayers();
    createBall();
    createStadium();
    createCrowd();
    initPhysics();
    initAudio();
    initUI();

    updateEnvironmentMap();

    // Start the game loop
    enhancedGameLoop(0);
}

// Start the game
initGame();

// Remove initial loading screen
document.getElementById('loading-screen').style.display = 'none';

// Event listeners
window.addEventListener('resize', onWindowResize);
document.addEventListener('keydown', onKeyDown);
document.addEventListener('keyup', onKeyUp);

// Game loop
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updatePhysics(deltaTime);
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

function updateGame(deltaTime) {
    gameTime -= deltaTime;
    if (gameTime <= 0) {
        endGame();
    }

    updatePlayers(deltaTime);
    updateBall(deltaTime);
    checkForGoals();
    checkForFouls();
    updateTactics(deltaTime);
}

function updatePhysics(deltaTime) {
    world.step(deltaTime);
    
    // Update Three.js objects with Cannon.js data
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);

    players.forEach((player, index) => {
        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);
    });
}

function updateAI(deltaTime) {
    players.forEach(player => {
        if (player.team !== humanTeam) {
            player.ai.update(deltaTime);
        }
    });
}

function updateParticles(deltaTime) {
    particleSystem.update(deltaTime);
}

function updateSound() {
    // Update 3D sound positions
    soundEffects.kick.position.copy(ball.position);
    soundEffects.whistle.position.copy(refereeSystem.position);

    // Update crowd sound based on excitement
    const excitement = calculateCrowdExcitement();
    soundEffects.crowd.setVolume(0.5 + excitement * 0.5);
}

function updateWeather(deltaTime) {
    weatherSystem.update(deltaTime);
}

function updateCamera(deltaTime) {
    const cameraTarget = getCameraTarget();
    camera.position.lerp(cameraTarget.position, 0.1);
    camera.lookAt(cameraTarget.lookAt);
}

function updateUI() {
    document.getElementById('score').textContent = `${scores.teamA} - ${scores.teamB}`;
    document.getElementById('time').textContent = formatTime(Math.max(0, gameTime));
    document.getElementById('stamina-bar').style.width = `${selectedPlayer.stamina}%`;
}

function render() {
    composer.render();
}

function endGame() {
    currentState = GameState.GAME_OVER;
    displayFinalScore();
    // Additional end game logic
}

// Helper functions
function formatTime(seconds) {
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = Math.floor(seconds % 60);
    return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
}

function calculateCrowdExcitement() {
    // Calculate crowd excitement based on game events
    // Return a value between 0 and 1
}

function getCameraTarget() {
    // Determine camera target based on current game state
    // Return an object with position and lookAt properties
}

// Start the game loop
gameLoop(0);

// Additional game systems

// Referee system
const refereeSystem = {
    position: new THREE.Vector3(),
    speed: 5,

    update: function(deltaTime) {
        // Move referee towards the ball
        const direction = new THREE.Vector3().subVectors(ball.position, this.position).normalize();
        this.position.add(direction.multiplyScalar(this.speed * deltaTime));

        // Check for fouls
        this.checkForFouls();
    },

    checkForFouls: function() {
        // Foul detection logic
    },

    callFoul: function(player) {
        console.log(`Foul called on player ${player.id} from team ${player.team}`);
        // Implement foul consequences
    }
};

// Tactics system
const tacticsSystem = {
    formations: {
        '4-4-2': [/* player positions */],
        '4-3-3': [/* player positions */],
        '3-5-2': [/* player positions */]
    },

    updateFormation: function(team, formation) {
        const teamPlayers = players.filter(p => p.team === team);
        const positions = this.formations[formation];

        teamPlayers.forEach((player, index) => {
            player.formationPosition.copy(positions[index]);
        });
    },

    updateTactics: function(deltaTime) {
        // Update team tactics based on game state
    }
};

// Weather system
const weatherSystem = {
    currentWeather: 'clear',
    transitionTime: 0,
    transitionDuration: 30, // seconds

    update: function(deltaTime) {
        this.transitionTime += deltaTime;
        if (this.transitionTime >= this.transitionDuration) {
            this.changeWeather();
        }
        this.updateWeatherEffects(deltaTime);
    },

    changeWeather: function() {
        const weathers = ['clear', 'cloudy', 'rainy', 'foggy'];
        this.currentWeather = weathers[Math.floor(Math.random() * weathers.length)];
        this.transitionTime = 0;
        console.log(`Weather changing to: ${this.currentWeather}`);
    },

    updateWeatherEffects: function(deltaTime) {
        // Update weather effects based on current weather
    }
};

// Commentary system
const commentarySystem = {
    phrases: {
        goal: [
            "GOAL! What a fantastic finish!",
            "He's done it! The ball is in the back of the net!",
            "An absolute screamer! The crowd goes wild!"
        ],
        miss: [
            "Oh, so close! The goalkeeper was beaten there.",
            "What a chance! He'll be disappointed not to score.",
            "Inches wide! That was a golden opportunity!"
        ],
        // Add more phrase categories
    },

    generateCommentary: function(event) {
        const phrases = this.phrases[event];
        const commentary = phrases[Math.floor(Math.random() * phrases.length)];
        console.log(`Commentary: ${commentary}`);
        // You could also trigger audio playback of the commentary
    }
};

// Injury system
const injurySystem = {
    injuredPlayers: new Set(),

    update: function(deltaTime) {
        players.forEach(player => {
            if (this.injuredPlayers.has(player)) {
                this.updateInjuredPlayer(player, deltaTime);
            } else if (Math.random() < 0.0001) { // Small chance of injury each frame
                this.injurePlayer(player);
            }
        });
    },

    injurePlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} is injured!`);
        this.injuredPlayers.add(player);
        player.speed *= 0.5;
        // Visual indication of injury
        player.material.color.setHex(0xff0000);
    },

    updateInjuredPlayer: function(player, deltaTime) {
        player.injuryTime += deltaTime;
        if (player.injuryTime > 30) { // Recover after 30 seconds
            this.recoverPlayer(player);
        }
    },

    recoverPlayer: function(player) {
        console.log(`Player ${player.id} from team ${player.team} has recovered!`);
        this.injuredPlayers.delete(player);
        player.speed *= 2;
        player.injuryTime = 0;
        player.material.color.setHex(0xffffff);
    }
};

// Substitution system
const substitutionSystem = {
    substitutionsLeft: { A: 3, B: 3 },
    benchPlayers: { A: [], B: [] },

    initialize: function() {
        // Create bench players
        for (let team of ['A', 'B']) {
            for (let i = 0; i < 7; i++) {
                const benchPlayer = createPlayer(team);
                benchPlayer.position.set(0, 0, -40); // Off the field
                this.benchPlayers[team].push(benchPlayer);
            }
        }
    },

    requestSubstitution: function(team) {
        if (this.substitutionsLeft[team] > 0) {
            const playerToSubstitute = this.selectPlayerToSubstitute(team);
            const benchPlayer = this.selectBenchPlayer(team);

            if (playerToSubstitute && benchPlayer) {
                this.performSubstitution(playerToSubstitute, benchPlayer);
                this.substitutionsLeft[team]--;
            }
        }
    },

    selectPlayerToSubstitute: function(team) {
        // Select the most tired player
        return players
            .filter(p => p.team === team)
            .sort((a, b) => a.stamina - b.stamina)[0];
    },

    selectBenchPlayer: function(team) {
        // Select the most rested bench player
        return this.benchPlayers[team]
            .sort((a, b) => b.stamina - a.stamina)[0];
    },

    performSubstitution: function(playerOut, playerIn) {
        const index = players.indexOf(playerOut);
        players[index] = playerIn;
        playerIn.position.copy(playerOut.position);
        this.benchPlayers[playerOut.team] = this.benchPlayers[playerOut.team].filter(p => p !== playerIn);
        this.benchPlayers[playerOut.team].push(playerOut);
        playerOut.position.set(0, 0, -40); // Move to bench

        console.log(`Substitution: Player ${playerOut.id} is replaced by Player ${playerIn.id} in team ${playerOut.team}`);
    }
};

// Initialize additional systems
substitutionSystem.initialize();

// Add these systems to the game loop
function updateGame(deltaTime) {
    gameTime -= deltaTime;
    if (gameTime <= 0) {
        endGame();
    }

    updatePlayers(deltaTime);
    updateBall(deltaTime);
    checkForGoals();
    refereeSystem.update(deltaTime);
    tacticsSystem.updateTactics(deltaTime);
    weatherSystem.update(deltaTime);
    injurySystem.update(deltaTime);
    substitutionSystem.update();
}

// Final touches

// Implement replays
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 10 * 60, // 10 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballRotation: ball.quaternion.clone(),
            playerPositions: players.map(player => player.position.clone()),
            playerRotations: players.map(player => player.quaternion.clone()),
            cameraPosition: camera.position.clone(),
            cameraRotation: camera.quaternion.clone()
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        const replayCamera = camera.clone();
        scene.add(replayCamera);

        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                scene.remove(replayCamera);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ball.quaternion.copy(frame.ballRotation);

            players.forEach((player, index) => {
                player.position.copy(frame.playerPositions[index]);
                player.quaternion.copy(frame.playerRotations[index]);
            });

            replayCamera.position.copy(frame.cameraPosition);
            replayCamera.quaternion.copy(frame.cameraRotation);

            composer.render();
            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Add replay recording to the game loop
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updatePhysics(deltaTime);
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            replaySystem.recordFrame();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

// Implement cinematic camera for replays
const cinematicCamera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
scene.add(cinematicCamera);

function updateCinematicCamera(deltaTime) {
    // Implement cinematic camera movement
    const time = Date.now() * 0.001;
    const radius = 50;
    cinematicCamera.position.x = Math.cos(time * 0.5) * radius;
    cinematicCamera.position.z = Math.sin(time * 0.5) * radius;
    cinematicCamera.position.y = 20 + Math.sin(time) * 10;
    cinematicCamera.lookAt(ball.position);
}

// Add cinematic camera update to replay system
replaySystem.playReplay = function() {
    let frameIndex = 0;
    const replayInterval = setInterval(() => {
        if (frameIndex >= this.recordedFrames.length) {
            clearInterval(replayInterval);
            camera = mainCamera; // Switch back to main camera
            return;
        }

        const frame = this.recordedFrames[frameIndex];
        ball.position.copy(frame.ballPosition);
        ball.quaternion.copy(frame.ballRotation);

        players.forEach((player, index) => {
            player.position.copy(frame.playerPositions[index]);
            player.quaternion.copy(frame.playerRotations[index]);
        });

        updateCinematicCamera(1/60); // Assume 60 fps
        camera = cinematicCamera; // Use cinematic camera for replay

        composer.render();
        frameIndex++;
    }, 1000 / 60); // Play at 60 fps
};

// Implement slow motion
let timeScale = 1;

function updatePhysics(deltaTime) {
    world.step(deltaTime * timeScale);
    
    // Update Three.js objects with Cannon.js data
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);

    players.forEach((player, index) => {
        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);
    });
}

function setSlowMotion(enable) {
    timeScale = enable ? 0.2 : 1;
}

// Add slow motion to replay system
replaySystem.playReplay = function() {
    setSlowMotion(true);
    // ... existing replay code ...
    // When replay ends:
    setSlowMotion(false);
};

// Implement spectator reactions
const spectatorSystem = {
    spectators: [],
    excitement: 0,

    initialize: function() {
        // Create spectators
        for (let i = 0; i < 1000; i++) {
            const spectator = new THREE.Mesh(
                new THREE.BoxGeometry(0.5, 1.8, 0.5),
                new THREE.MeshLambertMaterial({ color: 0xcccccc })
            );
            spectator.position.set(
                (Math.random() - 0.5) * 120,
                1,
                (Math.random() - 0.5) * 80
            );
            scene.add(spectator);
            this.spectators.push(spectator);
        }
    },

    update: function(deltaTime) {
        this.excitement = Math.max(0, this.excitement - deltaTime * 0.1);
        this.spectators.forEach(spectator => {
            spectator.position.y = 1 + Math.sin(Date.now() * 0.01 + spectator.position.x) * 0.1 * this.excitement;
        });
    },

    cheer: function() {
        this.excitement = 1;
    }
};

spectatorSystem.initialize();

// Add spectator update to game loop
function updateGame(deltaTime) {
    // ... existing game update code ...
    spectatorSystem.update(deltaTime);
}

// Make spectators cheer on goals
function checkForGoals() {
    // ... existing goal check code ...
    if (goalScored) {
        spectatorSystem.cheer();
    }
}

// Final optimization
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement frustum culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateFrustumCulling() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use instancing for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement level of detail (LOD)
    const playerLODs = [];
    players.forEach(player => {
        const highDetailGeometry = new THREE.SphereGeometry(1, 32, 32);
        const mediumDetailGeometry = new THREE.SphereGeometry(1, 16, 16);
        const lowDetailGeometry = new THREE.SphereGeometry(1, 8, 8);

        const playerLOD = new THREE.LOD();

        playerLOD.addLevel(new THREE.Mesh(highDetailGeometry, player.material), 0);
        playerLOD.addLevel(new THREE.Mesh(mediumDetailGeometry, player.material), 10);
        playerLOD.addLevel(new THREE.Mesh(lowDetailGeometry, player.material), 50);

        playerLOD.position.copy(player.position);
        scene.add(playerLOD);
        playerLODs.push(playerLOD);
    });

    function updateLODs() {
        playerLODs.forEach((playerLOD, index) => {
            playerLOD.position.copy(players[index].position);
            playerLOD.update(camera);
        });
    }

    // Use worker threads for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        ball.position.copy(ballPosition);
        ballBody.velocity.copy(ballVelocity);
        players.forEach((player, index) => {
            player.position.copy(playerPositions[index]);
        });
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Update optimization functions
    return {
        updateFrustumCulling: updateFrustumCulling,
        updateLODs: updateLODs,
        updatePhysics: updatePhysics
    };
}

const optimizations = optimizePerformance();

// Update game loop with optimizations
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            optimizations.updatePhysics();
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            replaySystem.recordFrame();
            optimizations.updateFrustumCulling();
            optimizations.updateLODs();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

// Start the game
initGame();
gameLoop(0);

// Performance optimizations
function optimizePerformance() {
    // Use object pooling for particle effects
    const particlePool = [];
    const PARTICLE_POOL_SIZE = 1000;

    for (let i = 0; i < PARTICLE_POOL_SIZE; i++) {
        const particle = new THREE.Mesh(
            new THREE.SphereGeometry(0.1, 8, 8),
            new THREE.MeshBasicMaterial({ color: 0xffffff })
        );
        particle.visible = false;
        scene.add(particle);
        particlePool.push(particle);
    }

    function getParticle() {
        return particlePool.find(particle => !particle.visible) || particlePool[0];
    }

    // Implement occlusion culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateVisibility() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use instancing for grass
    const grassGeometry = new THREE.PlaneGeometry(1, 1);
    const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x33aa33, side: THREE.DoubleSide });
    const grassInstancedMesh = new THREE.InstancedMesh(grassGeometry, grassMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 100,
            0,
            (Math.random() - 0.5) * 60
        );
        const scale = new THREE.Vector3(0.5, 0.5 + Math.random() * 0.5, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        grassInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(grassInstancedMesh);

    // Implement texture atlases
    const textureAtlas = new THREE.TextureLoader().load('textures/texture_atlas.jpg');
    const atlasedMaterial = new THREE.MeshBasicMaterial({ map: textureAtlas });

    // Use shared geometries and materials
    const sharedPlayerGeometry = new THREE.SphereGeometry(0.5, 32, 32);
    const sharedTeamAMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
    const sharedTeamBMaterial = new THREE.MeshStandardMaterial({ color: 0x0000ff });

    players.forEach(player => {
        player.geometry = sharedPlayerGeometry;
        player.material = player.team === 'A' ? sharedTeamAMaterial : sharedTeamBMaterial;
    });

    // Implement mesh merging for static objects
    const staticGeometries = [];
    scene.traverse(object => {
        if (object.isStatic && object.isMesh) {
            staticGeometries.push(object.geometry);
        }
    });
    const mergedGeometry = BufferGeometryUtils.mergeBufferGeometries(staticGeometries);
    const mergedMesh = new THREE.Mesh(mergedGeometry, new THREE.MeshStandardMaterial());
    scene.add(mergedMesh);

    // Use sprite sheets for animations
    const spriteSheet = new THREE.TextureLoader().load('textures/player_animations.png');
    const spriteSheetMaterial = new THREE.SpriteMaterial({ map: spriteSheet });

    function updatePlayerSprite(player, frameIndex) {
        const spriteSize = 64; // Assuming 64x64 pixel sprites
        const framesPerRow = 8;
        const u = (frameIndex % framesPerRow) / framesPerRow;
        const v = Math.floor(frameIndex / framesPerRow) / framesPerRow;
        player.material.map.offset.set(u, v);
        player.material.map.repeat.set(1 / framesPerRow, 1 / framesPerRow);
    }

    // Implement frustum culling
    const frustum = new THREE.Frustum();
    const projScreenMatrix = new THREE.Matrix4();

    function updateFrustumCulling() {
        camera.updateMatrixWorld();
        projScreenMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
        frustum.setFromProjectionMatrix(projScreenMatrix);

        scene.traverse(object => {
            if (object.isMesh) {
                object.visible = frustum.intersectsObject(object);
            }
        });
    }

    // Use Web Workers for physics calculations
    const physicsWorker = new Worker('physics-worker.js');
    physicsWorker.onmessage = function(e) {
        const { ballPosition, ballVelocity, playerPositions } = e.data;
        updatePhysicsResults(ballPosition, ballVelocity, playerPositions);
    };

    function updatePhysics() {
        const playerPositions = players.map(player => player.position);
        physicsWorker.postMessage({
            ballPosition: ball.position,
            ballVelocity: ballBody.velocity,
            playerPositions: playerPositions
        });
    }

    // Implement occlusion culling
    const occlusionCulling = new THREE.OcclusionCulling(camera, scene);

    function updateOcclusionCulling() {
        occlusionCulling.update();
    }

    // Use GPU instancing for crowd
    const crowdGeometry = new THREE.BoxGeometry(0.5, 1.8, 0.5);
    const crowdMaterial = new THREE.MeshPhongMaterial({ color: 0xcccccc });
    const crowdInstancedMesh = new THREE.InstancedMesh(crowdGeometry, crowdMaterial, 10000);

    for (let i = 0; i < 10000; i++) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 120,
            1,
            (Math.random() - 0.5) * 80
        );
        const scale = new THREE.Vector3(1, 0.9 + Math.random() * 0.2, 1);
        const rotation = new THREE.Euler(0, Math.random() * Math.PI * 2, 0);

        const matrix = new THREE.Matrix4()
            .makeRotationFromEuler(rotation)
            .scale(scale)
            .setPosition(position);

        crowdInstancedMesh.setMatrixAt(i, matrix);
    }

    scene.add(crowdInstancedMesh);

    // Implement dynamic LOD system
    const lodSystem = new THREE.LOD();

    players.forEach(player => {
        const highDetail = new THREE.Mesh(sharedPlayerGeometry, player.material);
        const mediumDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 16, 16), player.material);
        const lowDetail = new THREE.Mesh(new THREE.SphereGeometry(0.5, 8, 8), player.material);

        lodSystem.addLevel(highDetail, 0);
        lodSystem.addLevel(mediumDetail, 20);
        lodSystem.addLevel(lowDetail, 50);

        player.add(lodSystem);
    });

    // Implement shader-based grass
    const grassShaderMaterial = new THREE.ShaderMaterial({
        vertexShader: `
            uniform float time;
            varying vec2 vUv;
            void main() {
                vUv = uv;
                vec3 pos = position;
                pos.y += sin(pos.x * 10.0 + time) * 0.1;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
            }
        `,
        fragmentShader: `
            uniform sampler2D grassTexture;
            varying vec2 vUv;
            void main() {
                gl_FragColor = texture2D(grassTexture, vUv);
            }
        `,
        uniforms: {
            time: { value: 0 },
            grassTexture: { value: new THREE.TextureLoader().load('textures/grass.jpg') }
        }
    });

    const grassMesh = new THREE.Mesh(
        new THREE.PlaneGeometry(100, 60, 100, 60),
        grassShaderMaterial
    );
    grassMesh.rotation.x = -Math.PI / 2;
    scene.add(grassMesh);

    // Implement dynamic resolution scaling
    let resolutionScale = 1;
    function updateResolutionScale() {
        const fps = 1 / deltaTime;
        if (fps < 30 && resolutionScale > 0.5) {
            resolutionScale -= 0.1;
        } else if (fps > 60 && resolutionScale < 1) {
            resolutionScale += 0.1;
        }
        renderer.setPixelRatio(window.devicePixelRatio * resolutionScale);
        composer.setPixelRatio(window.devicePixelRatio * resolutionScale);
    }

    // Update functions
    return {
        updateParticles: function() {
            particleSystem.geometry.attributes.position.needsUpdate = true;
        },
        updateGrass: function(time) {
            grassShaderMaterial.uniforms.time.value = time;
        },
        updateFrustumCulling: updateFrustumCulling,
        updateOcclusionCulling: updateOcclusionCulling,
        updatePhysics: updatePhysics,
        updateResolutionScale: updateResolutionScale
    };
}

const optimizations = optimizePerformance();

// Enhance the game loop with optimizations
function enhancedGameLoop(time) {
    stats.begin();

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            dynamicCameraSystem.update(deltaTime, time);
            advancedUISystem.update();
            dynamicLightingSystem.update(deltaTime);
            updateBallPhysics(deltaTime);
            updateAdvancedPlayers(deltaTime);
            playerAnimationSystem.update(deltaTime);
            advancedBallControlSystem.update(deltaTime);

            // Apply optimizations
            optimizations.updateParticles();
            optimizations.updateGrass(time);
            optimizations.updateFrustumCulling();
            optimizations.updateOcclusionCulling();
            optimizations.updatePhysics();
            optimizations.updateResolutionScale();

            // Update all advanced systems
            advancedTeamTacticsSystem.update(deltaTime);
            advancedReferee.update(deltaTime);
            advancedCommentarySystem.update(time);
            advancedCrowdSystem.update(deltaTime, time);
            dynamicWeatherSystem.update(deltaTime);
            advancedInjurySystem.update(deltaTime);
            advancedSubstitutionSystem.update();
            advancedFatigueSystem.update(deltaTime);
            particleSystem.update(deltaTime);
            offsideSystem.update();

            // Handle set pieces
            if (setPieceSystem.currentSetPiece) {
                setPieceSystem.executeSetPiece();
            }

            // Record replay frame
            replaySystem.recordFrame();

            // Check for goals
            checkForGoals();

            // Check for game end
            if (gameTime <= 0) {
                endGame();
            }
            break;
        case GameState.PAUSED:
            // Render the paused scene
            break;
        case GameState.GAME_OVER:
            // Render the game over scene
            displayFinalResults();
            break;
    }

    composer.render();
    stats.end();

    requestAnimationFrame(enhancedGameLoop);
}

// Start the enhanced game loop
enhancedGameLoop(0);

// Final touches

// Implement advanced audio system with 3D positional audio
const audioListener = new THREE.AudioListener();
camera.add(audioListener);

const audioLoader = new THREE.AudioLoader();
const soundEffects = {
    kick: new THREE.PositionalAudio(audioListener),
    whistle: new THREE.PositionalAudio(audioListener),
    crowd: new THREE.Audio(audioListener)
};

audioLoader.load('sounds/kick.mp3', buffer => {
    soundEffects.kick.setBuffer(buffer);
    soundEffects.kick.setRefDistance(20);
});

audioLoader.load('sounds/whistle.mp3', buffer => {
    soundEffects.whistle.setBuffer(buffer);
    soundEffects.whistle.setRefDistance(50);
});

audioLoader.load('sounds/crowd.mp3', buffer => {
    soundEffects.crowd.setBuffer(buffer);
    soundEffects.crowd.setLoop(true);
    soundEffects.crowd.setVolume(0.5);
    soundEffects.crowd.play();
});

// Implement cinematic replay system
const replaySystem = {
    isRecording: false,
    recordedFrames: [],
    maxRecordTime: 10 * 60, // 10 seconds at 60 fps

    startRecording: function() {
        this.isRecording = true;
        this.recordedFrames = [];
    },

    stopRecording: function() {
        this.isRecording = false;
    },

    recordFrame: function() {
        if (!this.isRecording) return;

        const frame = {
            ballPosition: ball.position.clone(),
            ballRotation: ball.quaternion.clone(),
            playerPositions: players.map(player => player.position.clone()),
            playerRotations: players.map(player => player.quaternion.clone()),
            cameraPosition: camera.position.clone(),
            cameraRotation: camera.quaternion.clone()
        };

        this.recordedFrames.push(frame);

        if (this.recordedFrames.length > this.maxRecordTime) {
            this.recordedFrames.shift();
        }
    },

    playReplay: function() {
        const replayCamera = camera.clone();
        scene.add(replayCamera);

        let frameIndex = 0;
        const replayInterval = setInterval(() => {
            if (frameIndex >= this.recordedFrames.length) {
                clearInterval(replayInterval);
                scene.remove(replayCamera);
                return;
            }

            const frame = this.recordedFrames[frameIndex];
            ball.position.copy(frame.ballPosition);
            ball.quaternion.copy(frame.ballRotation);

            players.forEach((player, index) => {
                player.position.copy(frame.playerPositions[index]);
                player.quaternion.copy(frame.playerRotations[index]);
            });

            replayCamera.position.copy(frame.cameraPosition);
            replayCamera.quaternion.copy(frame.cameraRotation);

            composer.render();
            frameIndex++;
        }, 1000 / 60); // Play at 60 fps
    }
};

// Implement advanced particle systems
const particleSystem = {
    systems: [],

    createEffect: function(position, color, count, speed, size, duration) {
        const particles = new THREE.BufferGeometry();
        const positions = new Float32Array(count * 3);
        const colors = new Float32Array(count * 3);
        const sizes = new Float32Array(count);

        for (let i = 0; i < count; i++) {
            positions[i * 3] = position.x;
            positions[i * 3 + 1] = position.y;
            positions[i * 3 + 2] = position.z;

            colors[i * 3] = color.r;
            colors[i * 3 + 1] = color.g;
            colors[i * 3 + 2] = color.b;

            sizes[i] = size;
        }

        particles.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        particles.setAttribute('color', new THREE.BufferAttribute(colors, 3));
        particles.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

        const material = new THREE.ShaderMaterial({
            uniforms: {
                time: { value: 0 },
                speed: { value: speed }
            },
            vertexShader: `
                uniform float time;
                uniform float speed;
                attribute float size;
                varying vec3 vColor;
                void main() {
                    vColor = color;
                    vec3 pos = position;
                    pos.y += speed * time;
                    vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
                    gl_PointSize = size * (300.0 / -mvPosition.z);
                    gl_Position = projectionMatrix * mvPosition;
                }
            `,
            fragmentShader: `
                varying vec3 vColor;
                void main() {
                    if (length(gl_PointCoord - vec2(0.5, 0.5)) > 0.5) discard;
                    gl_FragColor = vec4(vColor, 1.0);
                }
            `,
            blending: THREE.AdditiveBlending,
            depthTest: false,
            transparent: true
        });

        const particleSystem = new THREE.Points(particles, material);
        this.systems.push({ system: particleSystem, startTime: Date.now(), duration: duration });
        scene.add(particleSystem);
    },

    update: function(deltaTime) {
        this.systems.forEach((system, index) => {
            system.system.material.uniforms.time.value += deltaTime;

            if (Date.now() - system.startTime > system.duration) {
                scene.remove(system.system);
                this.systems.splice(index, 1);
            }
        });
    }
};

// Implement advanced post-processing effects
const composer = new EffectComposer(renderer);
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
composer.addPass(bloomPass);

const filmPass = new FilmPass(0.35, 0.5, 2048, false);
composer.addPass(filmPass);

const smaaPass = new SMAAPass(window.innerWidth * renderer.getPixelRatio(), window.innerHeight * renderer.getPixelRatio());
composer.addPass(smaaPass);

// Implement dynamic environment mapping
const pmremGenerator = new THREE.PMREMGenerator(renderer);
let envMap;

function updateEnvironmentMap() {
    const generatedCubeRenderTarget = pmremGenerator.fromScene(scene);
    envMap = generatedCubeRenderTarget.texture;

    scene.traverse((object) => {
        if (object.isMesh && object.material.envMap) {
            object.material.envMap = envMap;
            object.material.needsUpdate = true;
        }
    });

    pmremGenerator.dispose();
}

// Final game initialization
function initGame() {
    createScene();
    createLights();
    createPlayers();
    createBall();
    createStadium();
    createCrowd();
    initPhysics();
    initAudio();
    initUI();

    updateEnvironmentMap();

    // Start the game loop
    enhancedGameLoop(0);
}

// Start the game
initGame();

// Remove initial loading screen
document.getElementById('loading-screen').style.display = 'none';

// Event listeners
window.addEventListener('resize', onWindowResize);
document.addEventListener('keydown', onKeyDown);
document.addEventListener('keyup', onKeyUp);

// Game loop
function gameLoop(time) {
    requestAnimationFrame(gameLoop);

    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;

    switch (currentState) {
        case GameState.PLAYING:
            updateGame(deltaTime);
            updatePhysics(deltaTime);
            updateAI(deltaTime);
            updateParticles(deltaTime);
            updateSound();
            updateWeather(deltaTime);
            updateCamera(deltaTime);
            updateUI();
            break;
        case GameState.PAUSED:
            // Handle pause state
            break;
        case GameState.GAME_OVER:
            // Handle game over state
            break;
    }

    render();
    stats.update();
}

function updateGame(deltaTime) {
    gameTime -= deltaTime;
    if (gameTime <= 0) {
        endGame();
    }

    updatePlayers(deltaTime);
    updateBall(deltaTime);
    checkForGoals();
    checkForFouls();
    updateTactics(deltaTime);
}

function updatePhysics(deltaTime) {
    world.step(deltaTime);
    
    // Update Three.js objects with Cannon.js data
    ball.position.copy(ballBody.position);
    ball.quaternion.copy(ballBody.quaternion);

    players.forEach((player, index) => {
        player.position.copy(playerBodies[index].position);
        player.quaternion.copy(playerBodies[index].quaternion);
    });
}

function updateAI(deltaTime) {
    players.forEach(player => {
        if (player.team !== humanTeam) {
            player.ai.update(deltaTime);
        }
    });
}

function updateParticles(deltaTime) {
    particleSystem.update(deltaTime);
}

function updateSound() {
    // Update 3D sound positions
    soundEffects.kick.position.copy(ball.position);
    soundEffects.whistle.position.copy(refereeSystem.position);

    // Update crowd sound based on excitement
    const excitement = calculateCrowdExcitement();
    soundEffects.crowd.setVolume(0.5 + excitement * 0.5);
}

function updateWeather(deltaTime) {
    weatherSystem.update(deltaTime);
}

function updateCamera(deltaTime) {
    const cameraTarget = getCameraTarget();
    camera.position.lerp(cameraTarget.position, 0.1);
    camera.lookAt(cameraTarget.lookAt);
}

function updateUI() {
    document.getElementById('score').textContent = `${scores.teamA} - ${scores.teamB}`;
    document.getElementById('time').textContent = formatTime(Math.max(0, gameTime));
    document.getElementById('stamina-bar').style.width = `${selectedPlayer.stamina}%`;
}

function render() {
    composer.render();
}

function endGame() {
    currentState = GameState.GAME_OVER;
    displayFinalScore();
    // Additional end game logic
}

// Helper functions
function formatTime(seconds) {
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = Math.floor(seconds % 60);
    return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
}

function calculateCrowdExcitement() {
    // Calculate crowd excitement based on game events
    // Return a value between 0 and 1
}

function getCameraTarget() {
    // Determine camera target based on current game state
    // Return an object with position and lookAt properties
}

// Start the game loop
gameLoop(0);

// Additional game systems

// Referee system
const refereeSystem = {
    position: new THREE.Vector3(),
    speed: 5,

    update: function(deltaTime) {
        // Move referee towards the ball
        const direction = new THREE.Vector3().subVectors(ball.position, this.position).normalize();
        this.position.add(direction.multiplyScalar(this.speed * deltaTime));

        // Check for fouls
        this.checkForFouls();
    },

    checkForFouls: function() {
        // Foul detection logic
    },

    callFoul: function(player) {
        console.log(`Foul called on player ${player.id} from team ${player.team}`);
        // Implement foul consequences
    }
};

// Tactics system
const tacticsSystem = {
    formations: {
        '4-4-2': [/* player positions */],
        '4-3-3': [/* player positions */],
        '3-5-2': [/* player positions */]
    },

    updateFormation: function(team, formation) {
        const teamPlayers = players.filter(p => p.team === team);
        const positions = this.formations[formation];

        teamPlayers.forEach((player, index) => {
            player.formationPosition.copy(positions[index]);
        });
    },

    updateTactics: function(deltaTime) {
        // Update team tactics based on game state
    }
};

// Weather system
const weatherSystem = {
    currentWeather: 'clear',
    transitionTime: 0,
    transitionDuration: 30, // seconds

    update: function(deltaTime) {
        this.transitionTime += deltaTime;
        if (this.transitionTime >= this.transitionDuration) {
            this.changeWeather();
        }
        this.updateWeatherEffects(deltaTime);
    },

    changeWeather: function() {
        const weathers = ['clear', 'cloudy', 'rainy', 'foggy'];
        this.currentWeather = weathers[Math.floor(Math.random() * weathers.length)];
        this.transitionTime = 0;
        console.log(`Weather changing to: ${this.currentWeather}`);
    },

    updateWeatherEffects: function(deltaTime) {
        // Update weather effects based on current weather
    }
};

// Commentary system
const commentarySystem = {
    phrases: {
        goal: [
            "GOAL! What a fantastic finish!",
            "He's done it! The ball is in the back of the net!",
            "An absolute screamer! The crowd goes wild!"
        ],
        miss: [
            "Oh, so close! The goalkeeper was beaten there.",
            "What a chance! He'll be disappointed not to score.",
            "Inches wide! That was a golden opportunity!"
        ],
        // Add more phrase categories
    },

    generateCommentary: function(event) {
        const phrases = this.phrases[event];
        const commentary = phrases[Math.floor(Math.random() * phrases.length)];
        console.log(`Commentary: ${commentary}`);
        // You could also trigger audio playback of the commentary
    }
};

// Injury system
const injurySystem = {
    injuredPlayers: new Set(),

    update: function(deltaTime) {
        players.forEach(player => {
            if (this.injuredPlayers.has(player)) {
                this.updateInjuredPlayer(player, deltaTime);
            } else if (Math.random() < 0.0001) { // Small chance of injury each frame
                this.injurePlayer(player);
            }
        });
    },

    injurePlayer: function(player) {
        console.log(`Player
