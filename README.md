<!DOCTYPE html>
<html>
<head>
    <title>Gesture Particles</title>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        body { margin: 0; overflow: hidden; background: #000; }
        #video-container { position: absolute; bottom: 10px; right: 10px; width: 150px; border: 2px solid white; z-index: 10; }
        canvas { display: block; }
    </style>
</head>
<body>

<video id="input_video" style="display:none;"></video>
<div id="video-container"><video id="guide_video" style="width:100%; transform: scaleX(-1);"></video></div>

<script>
/** 1. SHADERS **/
const _VS = `
    varying vec3 vColor;
    uniform float uSize;
    uniform float uTime;
    uniform float uMorph;
    attribute vec3 targetPos;

    void main() {
        // Interpolate between Sphere (Position) and Template (targetPos)
        vec3 morphed = mix(position, targetPos, uMorph);
        
        // Add a subtle wave animation
        morphed.y += sin(uTime + morphed.x) * 0.1;
        
        vec4 mvPosition = modelViewMatrix * vec4(morphed, 1.0);
        gl_PointSize = uSize * (300.0 / -mvPosition.z);
        gl_Position = projectionMatrix * mvPosition;
        vColor = vec3(0.5 + 0.5 * sin(uTime), 0.7, 1.0);
    }
`;

const _FS = `
    varying vec3 vColor;
    void main() {
        float d = distance(gl_PointCoord, vec2(0.5));
        if (d > 0.5) discard;
        gl_FragColor = vec4(vColor, 1.0 - (d * 2.0));
    }
`;

/** 2. THREE.JS SETUP **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const count = 5000;
const geo = new THREE.BufferGeometry();
const positions = new Float32Array(count * 3);
const heartPositions = new Float32Array(count * 3);

for (let i = 0; i < count; i++) {
    // Base Shape: Sphere
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.acos(2 * Math.random() - 1);
    positions[i*3] = Math.sin(phi) * Math.cos(theta) * 2;
    positions[i*3+1] = Math.sin(phi) * Math.sin(theta) * 2;
    positions[i*3+2] = Math.cos(phi) * 2;

    // Template Shape: Heart
    const t = Math.random() * Math.PI * 2;
    heartPositions[i*3] = (16 * Math.pow(Math.sin(t), 3)) / 10;
    heartPositions[i*3+1] = (13 * Math.cos(t) - 5 * Math.cos(2*t) - 2 * Math.cos(3*t) - Math.cos(4*t)) / 10;
    heartPositions[i*3+2] = (Math.random() - 0.5) * 0.5;
}

geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geo.setAttribute('targetPos', new THREE.BufferAttribute(heartPositions, 3));

const material = new THREE.ShaderMaterial({
    uniforms: { 
        uTime: { value: 0 }, 
        uSize: { value: 2.0 },
        uMorph: { value: 0.0 } 
    },
    vertexShader: _VS,
    fragmentShader: _FS,
    transparent: true
});

const points = new THREE.Points(geo, material);
scene.add(points);
camera.position.z = 5;

/** 3. MEDIAPIPE HAND TRACKING **/
const videoElement = document.getElementById('input_video');
const guideVideo = document.getElementById('guide_video');

function onResults(results) {
    if (results.multiHandLandmarks && results.multiHandLandmarks[0]) {
        const hand = results.multiHandLandmarks[0];
        
        // Calculate Pinch Distance (Thumb Tip to Index Tip)
        const dx = hand[8].x - hand[4].x;
        const dy = hand[8].y - hand[4].y;
        const dist = Math.sqrt(dx*dx + dy*dy);

        // Map distance to Morphing (Pinch = Sphere, Open = Heart)
        material.uniforms.uMorph.value = THREE.MathUtils.lerp(
            material.uniforms.uMorph.value, 
            dist > 0.1 ? 1.0 : 0.0, 
            0.1
        );
        
        // Move particles with hand
        points.position.x = (hand[9].x - 0.5) * -10;
        points.position.y = (hand[9].y - 0.5) * -10;
    }
}

const hands = new Hands({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`});
hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.5 });
hands.onResults(onResults);

const cameraUtils = new Camera(videoElement, {
    onFrame: async () => {
        await hands.send({image: videoElement});
        guideVideo.srcObject = videoElement.captureStream ? videoElement.captureStream() : videoElement.srcObject;
    },
    width: 640, height: 480
});
cameraUtils.start();

/** ANIMATION LOOP **/
function animate() {
    requestAnimationFrame(animate);
    material.uniforms.uTime.value += 0.05;
    renderer.render(scene, camera);
}
animate();

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>

