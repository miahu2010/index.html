# index.html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>手势控制粒子太阳系 - 改良版</title>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>

    <style>
        body { margin: 0; overflow: hidden; background-color: #000; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        canvas { display: block; }
        #ui-container {
            position: absolute; top: 20px; left: 20px; width: 280px;
            background: rgba(20,20,30,0.7); backdrop-filter: blur(10px);
            padding: 20px; border-radius: 15px; border: 1px solid rgba(255,255,255,0.08);
            color: white; z-index: 10; transition: all 0.5s ease;
        }
        h2 { margin: 0 0 15px 0; font-size:1.2rem; font-weight:300; }
        .control-group { margin-bottom: 15px; }
        label { display:block; margin-bottom:5px; font-size:0.9rem; color:#aaa; }
        .planet-grid { display:grid; grid-template-columns: repeat(3,1fr); gap:8px; }
        .planet-btn { background: rgba(255,255,255,0.08); border:none; color:#fff; padding:8px 0; border-radius:6px; cursor:pointer; font-size:0.8rem; }
        .planet-btn:hover, .planet-btn.active { background: rgba(0,150,255,0.6); }
        input[type="color"] { width:100%; height:35px; border:none; background:none; cursor:pointer; }
        #status { position:absolute; bottom:20px; left:50%; transform:translateX(-50%); color:rgba(255,255,255,0.6); font-size:0.9rem; background: rgba(0,0,0,0.5); padding:5px 15px; border-radius:20px; pointer-events:none; }
        #input_video { position:absolute; bottom:10px; right:10px; width:160px; height:120px; transform:scaleX(-1); border-radius:8px; opacity:0.5; z-index:5; }
        #fullscreen-btn, #toggle-ui-btn { position:absolute; top:20px; right:20px; background: rgba(0,0,0,0.5); border:1px solid rgba(255,255,255,0.3); color:white; width:40px; height:40px; border-radius:50%; cursor:pointer; z-index:10; display:flex; align-items:center; justify-content:center; }
        #toggle-ui-btn { right:70px; }
        #info-panel { position:absolute; top:20px; right:20px; width:300px; background: rgba(30,30,50,0.85); backdrop-filter: blur(8px); padding:15px; border-radius:10px; color:#fff; z-index:10; transition: all 0.5s ease; }
        #info-panel h3 { color:#f9d71c; border-bottom:1px solid #444; padding-bottom:5px; margin-top:0; }
        @media (max-width:600px) {
            #ui-container { width:calc(100% - 40px); top:auto; bottom:60px; }
            #input_video { display:none; }
            #info-panel { top:10px; left:10px; right:auto; width:calc(100% - 20px); }
            #fullscreen-btn { top:10px; right:10px; }
            #toggle-ui-btn { top:10px; right:60px; }
        }
    </style>
</head>
<body>
    <video id="input_video"></video>
    <div id="canvas-container"></div>

    <div id="ui-container">
        <h2>手势控制粒子太阳系（改良）</h2>
        <div class="control-group">
            <label>选择天体 / 聚焦</label>
            <div class="planet-grid" id="planet-selector"></div>
        </div>
        <div class="control-group" id="color-control-group">
            <label>星云颜色</label>
            <input type="color" id="color-picker" value="#ffffff">
        </div>
        <div style="font-size:0.8rem;color:#666;line-height:1.4;">
            交互说明:<br>
            👋 <b>左右移动手掌:</b> 调整视角 / 旋转聚焦<br>
            👐 <b>双手张开/合拢:</b> 拉近/拉远相机或让星云扩散
        </div>
    </div>

    <div id="info-panel">
        <h3>星球简介：<span id="planet-name"></span></h3>
        <p>🔭 <b>一般观测星等：</b><span id="planet-magnitude"></span></p>
        <p>👁️ <b>如何肉眼观测：</b><span id="planet-obs"></span></p>
        <p>💡 <b>对生活的影响：</b><span id="planet-impact"></span></p>
        <p>📜 <b>有趣的故事：</b><span id="planet-story"></span></p>
    </div>

    <div id="status">正在初始化摄像头与AI模型...</div>
    <button id="fullscreen-btn" onclick="toggleFullScreen()">⛶</button>
    <button id="toggle-ui-btn" onclick="toggleUI()">✖</button>

<script>
/* ================= 配置数据（同你原始数据，稍作调整） */
const planets = [
    { name: 'Sun', color: '#FDB813', size: 1.6, distance: 0, speed: 0, magnitude: -26.74, obs: '白天可见（切勿直视）', impact: '提供能量与光照', story: '太阳神阿波罗的战车', focusDistance: 5 },
    { name: 'Mercury', color: '#A57C5B', size: 0.12, distance: 3.0, speed: 0.015, magnitude: -1.9, obs: '黎明或黄昏低空可见', impact: '最靠近太阳', story: '墨丘利', focusDistance: 1.5 },
    { name: 'Venus', color: '#E3BB76', size: 0.2, distance: 4.5, speed: 0.012, magnitude: -4.9, obs: '极亮：晨星/昏星', impact: '极端温室效应示例', story: '维纳斯', focusDistance: 1.8 },
    { name: 'Earth', color: '#22A6F2', size: 0.22, distance: 6.0, speed: 0.01, magnitude: -3.99, obs: '我们的家园', impact: '生命之源', story: '古人曾以地心说', focusDistance: 1.8 },
    { name: 'Mars', color: '#DF4925', size: 0.15, distance: 8.5, speed: 0.008, magnitude: -2.91, obs: '红色显眼', impact: '未来殖民焦点', story: '玛尔斯', focusDistance: 1.6 },
    { name: 'Jupiter', color: '#C88B3A', size: 0.6, distance: 15.0, speed: 0.005, magnitude: -2.94, obs: '非常明亮', impact: '引力护盾', story: '朱庇特', focusDistance: 3.0 },
    { name: 'Saturn', color: '#C5AB6E', size: 0.5, distance: 20.0, speed: 0.004, magnitude: 0.47, obs: '需要望远镜看光环', impact: '壮观光环', story: '萨图尔努斯', focusDistance: 2.5 },
    { name: 'Uranus', color: '#4FD0E7', size: 0.4, distance: 25.0, speed: 0.003, magnitude: 5.9, obs: '近极限肉眼可见', impact: '侧卧旋转轴', story: '乌拉诺斯', focusDistance: 2.2 },
    { name: 'Neptune', color: '#3E54E8', size: 0.4, distance: 30.0, speed: 0.002, magnitude: 7.8, obs: '需望远镜', impact: '风速极高', story: '尼普顿', focusDistance: 2.2 }
];

/* ================= Three.js 初始化 ================= */
const scene = new THREE.Scene();
scene.fog = new THREE.FogExp2(0x000000, 0.005);

const camera = new THREE.PerspectiveCamera(60, window.innerWidth/window.innerHeight, 0.1, 1000);
camera.position.set(0, 10, 20);

const renderer = new THREE.WebGLRenderer({ antialias:true, alpha:true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.getElementById('canvas-container').appendChild(renderer.domElement);

/* 背景星星（不变） */
function addStars() {
    const starGeo = new THREE.BufferGeometry();
    const starPos = [];
    for (let i=0;i<2500;i++){
        starPos.push((Math.random()-0.5)*600, (Math.random()-0.5)*600, (Math.random()-0.5)*600);
    }
    starGeo.setAttribute('position', new THREE.Float32BufferAttribute(starPos,3));
    const starMat = new THREE.PointsMaterial({ color: 0xffffff, size: 0.12, transparent:true, opacity:0.7 });
    const stars = new THREE.Points(starGeo, starMat);
    scene.add(stars);
}
addStars();

/* ====== 状态与参数 ====== */
let currentMode = 'overview';
let currentFocusPlanet = planets[0];
let particleColor = planets[0].color;
let particleSystem = null;
let particleData = [];
const MAIN_PARTICLE_COUNT = 38000; // 可根据机性能调节

// 相机平滑控制
let targetCameraPos = new THREE.Vector3(0,10,20);
let targetCameraLookAt = new THREE.Vector3(0,0,0);

/* ====== 生成软圆粒子纹理（canvas） ====== */
function makeSprite(size=128) {
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = size;
    const ctx = canvas.getContext('2d');

    const grad = ctx.createRadialGradient(size/2,size/2,0,size/2,size/2,size/2);
    grad.addColorStop(0.0, 'rgba(255,255,255,1.0)');
    grad.addColorStop(0.3, 'rgba(255,255,255,0.85)');
    grad.addColorStop(0.6, 'rgba(200,200,200,0.4)');
    grad.addColorStop(1.0, 'rgba(0,0,0,0.0)');

    ctx.fillStyle = grad;
    ctx.fillRect(0,0,size,size);

    const texture = new THREE.CanvasTexture(canvas);
    texture.minFilter = THREE.LinearFilter;
    texture.magFilter = THREE.LinearFilter;
    texture.needsUpdate = true;
    return texture;
}
const particleSprite = makeSprite(128);

/* ====== 着色器（支持 per-vertex size & color & texture） ====== */
const vertexShader = `
    attribute float size;
    attribute vec3 customColor;
    varying vec3 vColor;
    void main() {
        vColor = customColor;
        vec4 mvPosition = modelViewMatrix * vec4( position, 1.0 );
        // 深度衰减，用于视觉上在远处粒子更小
        gl_PointSize = size * ( 200.0 / -mvPosition.z );
        gl_Position = projectionMatrix * mvPosition;
    }
`;
const fragmentShader = `
    uniform sampler2D pointTexture;
    varying vec3 vColor;
    void main() {
        vec4 tex = texture2D(pointTexture, gl_PointCoord);
        // 将顶点颜色乘以纹理值，从而获得柔和边缘与色彩
        vec3 col = vColor;
        gl_FragColor = vec4(col * tex.rgb, tex.a);
        if (gl_FragColor.a < 0.02) discard;
    }
`;

/* ====== 采样函数：在球体内均匀采样（体积分布） ====== */
function randomPointInSphere(radius) {
    // 均匀填充球体体积：方向均匀，半径 r ~ cbrt(u)*R
    const u = Math.random();
    const costheta = Math.random()*2 - 1;
    const phi = Math.random() * Math.PI * 2;
    const r = Math.cbrt(u) * radius;
    const sintheta = Math.sqrt(1 - costheta*costheta);
    const x = r * sintheta * Math.cos(phi);
    const y = r * sintheta * Math.sin(phi);
    const z = r * costheta;
    return new THREE.Vector3(x,y,z);
}

/* ====== 创建粒子系统：主体（body）与星云（cloud）都为体积分布 ====== */
function createSolarSystemParticles() {
    // 清理旧的
    if (particleSystem) { scene.remove(particleSystem); particleSystem.geometry.dispose(); particleSystem.material.dispose(); }
    particleData = [];

    // 临时数组，随后写入 Buffer
    const temp = [];

    planets.forEach(p => {
        const coreColor = new THREE.Color(p.color);
        const cloudBase = coreColor.clone().multiplyScalar(0.75);

        // 主体粒子数按行星大小比例增加
        const bodyCount = (p.name === 'Sun') ? 4000 : Math.max(600, Math.round(1200 * p.size));
        for (let i=0;i<bodyCount;i++){
            // 稍微向心：主体更密集，使用 radius = size * (0.6)
            const v = randomPointInSphere(p.size * 0.6);
            // jitter color slightly
            const col = coreColor.clone().multiplyScalar(0.9 + Math.random()*0.2);
            temp.push({
                pos: new THREE.Vector3().copy(v).add(new THREE.Vector3(p.distance,0,0)),
                originalPos: v.clone(), // 用于自转
                id: p.name,
                type: 'body',
                color: col,
                orbitDistance: p.distance,
                speed: p.speed,
                angle: Math.random()*Math.PI*2,
                size: (p.name==='Sun') ? 6.0 : 2.0  // 注意：这里是基准 size，会被 shader 深度衰减控制
            });
        }

        // 星云/环绕：范围更大、更稀疏
        const cloudCount = (p.name === 'Sun') ? 8000 : Math.max(1500, Math.round(3000 * p.size));
        for (let i=0;i<cloudCount;i++){
            const radius = p.size * (1.0 + Math.random()*2.0); // 更大的外扩
            // 采样时尽量让云有层次感：随机生成方向 & radius^(1/2) 保持外层更多点
            const v = randomPointInSphere(radius);
            const c = cloudBase.clone().multiplyScalar(0.6 + Math.random()*0.6);
            temp.push({
                pos: new THREE.Vector3().copy(v).add(new THREE.Vector3(p.distance,0,0)),
                originalPos: v.clone(),
                id: p.name,
                type: 'cloud',
                color: c,
                orbitDistance: p.distance,
                speed: p.speed * (0.9 + Math.random()*0.4),
                angle: Math.random()*Math.PI*2,
                size: 0.6 + Math.random()*1.2
            });
        }
    });

    // 裁剪或填充到 MAIN_PARTICLE_COUNT
    const total = Math.min(temp.length, MAIN_PARTICLE_COUNT);
    particleData = temp.slice(0, total);

    // 建立 BufferGeometry
    const geometry = new THREE.BufferGeometry();
    const positions = new Float32Array(MAIN_PARTICLE_COUNT * 3);
    const colors = new Float32Array(MAIN_PARTICLE_COUNT * 3);
    const sizes = new Float32Array(MAIN_PARTICLE_COUNT);

    // 写入实际粒子
    for (let i=0;i<total;i++){
        const p = particleData[i];
        positions[i*3] = p.pos.x;
        positions[i*3+1] = p.pos.y;
        positions[i*3+2] = p.pos.z;

        colors[i*3] = p.color.r;
        colors[i*3+1] = p.color.g;
        colors[i*3+2] = p.color.b;

        sizes[i] = p.size;
    }
    // 填充剩余
    for (let i=total;i<MAIN_PARTICLE_COUNT;i++){
        positions[i*3]=positions[i*3+1]=positions[i*3+2]=0;
        colors[i*3]=colors[i*3+1]=colors[i*3+2]=0;
        sizes[i]=0;
    }

    geometry.setAttribute('position', new THREE.BufferAttribute(positions,3));
    geometry.setAttribute('customColor', new THREE.BufferAttribute(colors,3));
    geometry.setAttribute('size', new THREE.BufferAttribute(sizes,1));

    // ShaderMaterial
    const material = new THREE.ShaderMaterial({
        uniforms: {
            pointTexture: { value: particleSprite }
        },
        vertexShader,
        fragmentShader,
        transparent: true,
        depthWrite: false,
        blending: THREE.AdditiveBlending,
        vertexColors: true
    });

    particleSystem = new THREE.Points(geometry, material);
    scene.add(particleSystem);

    // 默认更新 info panel
    updatePlanetInfo(currentFocusPlanet);
}

/* ====== UI 构建（保留你原始逻辑） ====== */
const planetSelector = document.getElementById('planet-selector');
const overviewBtn = document.createElement('button');
overviewBtn.className = 'planet-btn active';
overviewBtn.innerText = '总览太阳系';
overviewBtn.onclick = () => {
    document.querySelectorAll('.planet-btn').forEach(b=>b.classList.remove('active'));
    overviewBtn.classList.add('active');
    currentMode = 'overview';
    currentFocusPlanet = planets[0];
    document.getElementById('color-control-group').style.visibility = 'hidden';
    targetCameraPos.set(0,10,20);
    updatePlanetInfo(currentFocusPlanet);
};
planetSelector.appendChild(overviewBtn);
planets.filter(p=>p.name!=='Sun').forEach(p=>{
    const btn = document.createElement('button');
    btn.className = 'planet-btn';
    btn.innerText = p.name;
    btn.onclick = () => {
        document.querySelectorAll('.planet-btn').forEach(b=>b.classList.remove('active'));
        btn.classList.add('active');
        currentMode = 'focus';
        currentFocusPlanet = p;
        particleColor = p.color;
        document.getElementById('color-control-group').style.visibility = 'visible';
        document.getElementById('color-picker').value = p.color;
        updatePlanetInfo(p);
    };
    planetSelector.appendChild(btn);
});
document.getElementById('color-control-group').style.visibility = 'hidden';
document.getElementById('color-picker').addEventListener('input', e=>{
    particleColor = e.target.value;
});

/* ====== 更新 info panel ====== */
function updatePlanetInfo(pd){
    document.getElementById('planet-name').textContent = pd.name;
    document.getElementById('planet-magnitude').textContent = pd.magnitude ? `${pd.magnitude} M` : 'N/A';
    document.getElementById('planet-obs').textContent = pd.obs;
    document.getElementById('planet-impact').textContent = pd.impact;
    document.getElementById('planet-story').textContent = pd.story;
}

/* ====== 初始化粒子系统 ====== */
createSolarSystemParticles();

/* ====== MediaPipe 手势控制（保留并稍作调整） ====== */
const videoElement = document.getElementById('input_video');
const statusElement = document.getElementById('status');

let targetX = 0, currentX = 0;
let targetScale = 20, currentScale = 20;

const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
hands.setOptions({ maxNumHands: 2, modelComplexity: 1, minDetectionConfidence: 0.5, minTrackingConfidence: 0.5 });
hands.onResults(onResults);

const cameraUtils = new Camera(videoElement, {
    onFrame: async () => { await hands.send({ image: videoElement }); },
    width: 640, height: 480
});
cameraUtils.start().catch(err=>{ statusElement.innerText = "摄像头访问失败"; console.error(err); });

function onResults(results){
    statusElement.innerText = "系统运行中 - 请移动双手";
    statusElement.style.color = "#4FD0E7";

    if (!particleSystem) return;

    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        const landmarks = results.multiHandLandmarks;
        let avgX = 0;
        landmarks.forEach(h => avgX += h[9].x);
        avgX /= landmarks.length;
        const screenX = (1 - avgX) * 2 - 1;
        targetX = screenX * 1;

        if (landmarks.length === 2) {
            const h1 = landmarks[0][0], h2 = landmarks[1][0];
            const dist = Math.sqrt((h1.x-h2.x)**2 + (h1.y-h2.y)**2);
            if (currentMode === 'focus') {
                // 控制星云扩散（不放太大）
                targetScale = 0.8 + (dist * 2.0);
                targetScale = Math.min(Math.max(targetScale, 0.6), 2.2);
            } else {
                // 总览模式：控制相机 Z
                const minZ = 12, maxZ = 40;
                const adjusted = Math.max(0, Math.min(1, 1 - dist));
                targetScale = minZ + adjusted * (maxZ - minZ);
            }
        } else {
            targetScale = (currentMode === 'focus') ? 1.0 : 20.0;
        }
    } else {
        targetX = 0;
        targetScale = (currentMode === 'focus') ? 1.0 : 20.0;
        statusElement.innerText = "未检测到手部";
        statusElement.style.color = "#aaa";
    }
}

/* ====== 渲染循环 ====== */
function animate() {
    requestAnimationFrame(animate);
    if (!particleSystem) return;

    const positions = particleSystem.geometry.attributes.position.array;
    const colors = particleSystem.geometry.attributes.customColor.array;
    const sizes = particleSystem.geometry.attributes.size.array;

    // 平滑参数
    currentScale += (targetScale - currentScale) * 0.12;
    currentX += (targetX - currentX) * 0.08;

    // 求中心位置（主体粒子平均）用于聚焦计算
    let focusX = 0, focusY = 0, focusZ = 0;
    let focusBodyCount = 0;

    // 更新每个粒子的位置（公转 + 自转 + 聚焦缩放）
    for (let i=0;i<particleData.length;i++){
        const p = particleData[i];

        // 公转：让行星绕原点转
        if (p.orbitDistance > 0) {
            p.angle += p.speed * 0.8; // 放慢一点以便观感更柔和
            const orbitX = p.orbitDistance * Math.cos(p.angle);
            const orbitZ = p.orbitDistance * Math.sin(p.angle);
            // 自转：基于 originalPos 旋转（更柔和）
            const rot = Date.now() * 0.0001 * (p.id === 'Sun' ? 0.3 : 1.0);
            const ox = p.originalPos.x * Math.cos(rot) - p.originalPos.z * Math.sin(rot);
            const oz = p.originalPos.x * Math.sin(rot) + p.originalPos.z * Math.cos(rot);

            // 聚焦时：只有当前聚焦行星的粒子放大少量
            const isFocused = p.id === currentFocusPlanet.name;
            const focusMul = (isFocused && currentMode === 'focus') ? (1.0 + (currentScale - 1.0) * 0.3) : 1.0;

            positions[i*3] = orbitX + ox * focusMul;
            positions[i*3+1] = p.originalPos.y * focusMul;
            positions[i*3+2] = orbitZ + oz * focusMul;

            // 聚焦时记录主体位置平均（用于相机）
            if (isFocused && p.type === 'body') {
                focusX += positions[i*3];
                focusY += positions[i*3+1];
                focusZ += positions[i*3+2];
                focusBodyCount++;
            }

            // 云颜色在聚焦时微调为用户选择颜色（但不会突兀）
            if (currentMode === 'focus' && isFocused && p.type === 'cloud') {
                const tc = new THREE.Color(particleColor);
                // 只部分混合到顶点颜色（不会瞬间替换）
                const idx = i*3;
                colors[idx] = THREE.MathUtils.lerp(colors[idx], tc.r * 0.8, 0.06);
                colors[idx+1] = THREE.MathUtils.lerp(colors[idx+1], tc.g * 0.8, 0.06);
                colors[idx+2] = THREE.MathUtils.lerp(colors[idx+2], tc.b * 0.8, 0.06);
            }

            // 大小：主体略大，云较小。聚焦时放大幅度控制在较小范围。
            sizes[i] = p.size * (p.type === 'body' ? 1.1 : 0.9) * (currentMode === 'focus' && p.id === currentFocusPlanet.name ? 1.05 : 1.0);
        } else {
            // 太阳中心保持
            const rot = Date.now() * 0.00008;
            const ox = p.originalPos.x * Math.cos(rot) - p.originalPos.z * Math.sin(rot);
            const oz = p.originalPos.x * Math.sin(rot) + p.originalPos.z * Math.cos(rot);
            positions[i*3] = ox;
            positions[i*3+1] = p.originalPos.y;
            positions[i*3+2] = oz;
        }
    }

    // 更新 geometry 标志
    particleSystem.geometry.attributes.position.needsUpdate = true;
    particleSystem.geometry.attributes.customColor.needsUpdate = true;
    particleSystem.geometry.attributes.size.needsUpdate = true;

    // 相机平滑控制
    if (focusBodyCount > 0 && currentMode === 'focus') {
        focusX /= focusBodyCount; focusY /= focusBodyCount; focusZ /= focusBodyCount;
        targetCameraLookAt.set(focusX, focusY, focusZ);
        const orbitRadius = currentFocusPlanet.focusDistance * 1.4;
        const camAngle = currentX * 0.6;
        targetCameraPos.set(
            focusX + orbitRadius * Math.sin(camAngle),
            focusY + 0.7,
            focusZ + orbitRadius * Math.cos(camAngle)
        );
    } else {
        // 总览模式：让手势控制绕 Y 轴旋转并控制 Z 距离
        targetCameraLookAt.set(0,0,0);
        scene.rotation.y = currentX * 0.4;
        targetCameraPos.set(0, 10, targetScale);
    }

    camera.position.lerp(targetCameraPos, 0.08);
    camera.lookAt(targetCameraLookAt);

    renderer.render(scene, camera);
}
animate();

/* ====== 窗口与 UI 控制 ====== */
window.addEventListener('resize', ()=>{
    camera.aspect = window.innerWidth/window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

let isUIHidden = false;
function toggleUI() {
    const uiPanel = document.getElementById('ui-container');
    const infoPanel = document.getElementById('info-panel');
    const toggleBtn = document.getElementById('toggle-ui-btn');
    if (isUIHidden) {
        uiPanel.style.transform = 'translateX(0)'; infoPanel.style.transform = 'translateX(0)'; toggleBtn.innerText='✖';
    } else {
        uiPanel.style.transform = 'translateX(-350px)'; infoPanel.style.transform = 'translateX(350px)'; toggleBtn.innerText='☰';
    }
    isUIHidden = !isUIHidden;
}
function toggleFullScreen(){
    if (!document.fullscreenElement) document.documentElement.requestFullscreen();
    else if (document.exitFullscreen) document.exitFullscreen();
}

/* ====== 小提示：如果需要重新生成粒子（例如调 MAIN_PARTICLE_COUNT），调用 createSolarSystemParticles() ====== */

</script>
</body>
</html>
