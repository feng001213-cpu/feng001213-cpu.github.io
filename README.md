[新建 文本文档.html](https://github.com/user-attachments/files/23885092/default.html)
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI 手势交互粒子系统</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #050505; font-family: 'Segoe UI', sans-serif; }
        canvas { display: block; }
        
        /* 视频元素隐藏，用于后台处理 */
        #input_video { display: none; }

        /* UI 容器 */
        #ui-container {
            position: absolute;
            top: 20px;
            right: 20px;
            z-index: 100;
        }

        /* 全屏按钮 */
        #fullscreen-btn {
            position: absolute;
            bottom: 30px;
            right: 30px;
            padding: 12px 24px;
            background: rgba(255, 255, 255, 0.1);
            border: 1px solid rgba(255, 255, 255, 0.3);
            color: white;
            border-radius: 30px;
            cursor: pointer;
            backdrop-filter: blur(10px);
            transition: all 0.3s;
            z-index: 100;
            font-size: 14px;
            letter-spacing: 1px;
            text-transform: uppercase;
        }
        #fullscreen-btn:hover { background: rgba(255, 255, 255, 0.3); }

        /* 加载提示 */
        #loader {
            position: absolute;
            top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            color: rgba(0, 255, 255, 0.8);
            font-size: 24px;
            pointer-events: none;
            transition: opacity 0.5s;
        }
        
        /* 摄像头状态指示 */
        #cam-status {
            position: absolute;
            top: 20px;
            left: 20px;
            display: flex;
            align-items: center;
            color: #fff;
            background: rgba(0,0,0,0.5);
            padding: 8px 15px;
            border-radius: 20px;
            font-size: 12px;
        }
        .dot { width: 10px; height: 10px; background: #ff3b30; border-radius: 50%; margin-right: 8px; }
        .dot.active { background: #34c759; box-shadow: 0 0 10px #34c759; }
    </style>
    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/",
                "@mediapipe/hands": "https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js",
                "@mediapipe/camera_utils": "https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"
            }
        }
    </script>
</head>
<body>

    <div id="loader">系统初始化中... 请允许摄像头权限</div>
    <div id="cam-status"><div class="dot" id="cam-dot"></div><span id="cam-text">等待摄像头...</span></div>
    <video id="input_video"></video>
    <button id="fullscreen-btn">进入全屏</button>
    
    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
        import { GUI } from 'three/addons/libs/lil-gui.module.min.js';

        // --- 全局变量 ---
        let scene, camera, renderer, particles, geometry, material;
        let controls;
        const particleCount = 15000; // 粒子数量
        let currentShape = 'heart';
        let targetPositions = []; // 存储目标形状的坐标
        let handInteractionFactor = 1.0; // 手势控制缩放系数
        let handTargetColor = new THREE.Color(0xffffff);

        // 状态管理
        const params = {
            shape: 'heart',
            color: '#ff0055',
            particleSize: 0.15,
            rotationSpeed: 0.002,
            interactionStrength: 2.0 // 手势影响强度
        };

        // --- 1. Three.js 初始化 ---
        function initThree() {
            scene = new THREE.Scene();
            scene.fog = new THREE.FogExp2(0x050505, 0.02);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.z = 30;

            renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            document.body.appendChild(renderer.domElement);

            controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;

            // 创建粒子系统
            geometry = new THREE.BufferGeometry();
            const positions = new Float32Array(particleCount * 3);
            const initialColors = new Float32Array(particleCount * 3);

            // 初始化随机位置
            for (let i = 0; i < particleCount; i++) {
                positions[i * 3] = (Math.random() - 0.5) * 100;
                positions[i * 3 + 1] = (Math.random() - 0.5) * 100;
                positions[i * 3 + 2] = (Math.random() - 0.5) * 100;

                initialColors[i * 3] = 1;
                initialColors[i * 3 + 1] = 1;
                initialColors[i * 3 + 2] = 1;
            }

            geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
            geometry.setAttribute('color', new THREE.BufferAttribute(initialColors, 3));

            // 使用自定义着色器或点材质
            const sprite = new THREE.TextureLoader().load('https://threejs.org/examples/textures/sprites/disc.png');
            material = new THREE.PointsMaterial({
                size: params.particleSize,
                map: sprite,
                vertexColors: true,
                blending: THREE.AdditiveBlending,
                depthWrite: false,
                transparent: true,
                opacity: 0.8
            });

            particles = new THREE.Points(geometry, material);
            scene.add(particles);

            // 生成初始形状
            generateTargetPositions(params.shape);

            // 事件监听
            window.addEventListener('resize', onWindowResize);
            
            // 全屏按钮
            document.getElementById('fullscreen-btn').addEventListener('click', () => {
                if (!document.fullscreenElement) {
                    document.documentElement.requestFullscreen();
                    document.getElementById('fullscreen-btn').textContent = "退出全屏";
                } else {
                    document.exitFullscreen();
                    document.getElementById('fullscreen-btn').textContent = "进入全屏";
                }
            });
        }

        // --- 2. 形状生成逻辑 (数学核心) ---
        function generateTargetPositions(shapeType) {
            targetPositions = new Float32Array(particleCount * 3);
            const scale = 12;

            if (shapeType === 'heart') {
                for (let i = 0; i < particleCount; i++) {
                    let t = Math.random() * Math.PI * 2;
                    // 分布优化：让粒子更多在内部而不是仅仅边缘
                    let r = Math.sqrt(Math.random()); 
                    // 爱心方程
                    let x = 16 * Math.pow(Math.sin(t), 3);
                    let y = 13 * Math.cos(t) - 5 * Math.cos(2*t) - 2 * Math.cos(3*t) - Math.cos(4*t);
                    let z = (Math.random() - 0.5) * 4; // 增加厚度
                    
                    // 随机填充内部
                    x *= r; y *= r; z *= r;

                    targetPositions[i * 3] = x * 0.8;
                    targetPositions[i * 3 + 1] = y * 0.8;
                    targetPositions[i * 3 + 2] = z;
                }
            } 
            else if (shapeType === 'saturn') {
                for (let i = 0; i < particleCount; i++) {
                    const isRing = i > particleCount * 0.6; // 40% 行星，60% 环
                    if (isRing) {
                        const angle = Math.random() * Math.PI * 2;
                        const radius = 10 + Math.random() * 8;
                        targetPositions[i * 3] = Math.cos(angle) * radius;
                        targetPositions[i * 3 + 1] = (Math.random() - 0.5) * 0.5;
                        targetPositions[i * 3 + 2] = Math.sin(angle) * radius;
                    } else {
                        const phi = Math.acos(-1 + (2 * i) / (particleCount * 0.6));
                        const theta = Math.sqrt((particleCount * 0.6) * Math.PI) * phi;
                        const r = 7;
                        targetPositions[i * 3] = r * Math.cos(theta) * Math.sin(phi);
                        targetPositions[i * 3 + 1] = r * Math.sin(theta) * Math.sin(phi);
                        targetPositions[i * 3 + 2] = r * Math.cos(phi);
                    }
                }
            }
            else if (shapeType === 'cake') {
                for (let i = 0; i < particleCount; i++) {
                    const layer = Math.random();
                    let r, y;
                    if (layer < 0.4) { // 底座
                        r = Math.sqrt(Math.random()) * 8;
                        y = -5 + Math.random() * 4;
                    } else if (layer < 0.7) { // 中层
                        r = Math.sqrt(Math.random()) * 6;
                        y = -1 + Math.random() * 3;
                    } else if (layer < 0.9) { // 顶层
                        r = Math.sqrt(Math.random()) * 4;
                        y = 2 + Math.random() * 2;
                    } else { // 蜡烛/火焰
                        r = Math.random() * 0.5;
                        y = 4 + Math.random() * 3;
                    }
                    const theta = Math.random() * Math.PI * 2;
                    targetPositions[i * 3] = r * Math.cos(theta);
                    targetPositions[i * 3 + 1] = y;
                    targetPositions[i * 3 + 2] = r * Math.sin(theta);
                }
            }
            else if (shapeType === 'number19') {
                // 使用 Canvas 采样生成文字点云
                createTextParticles("19");
                return; // 这里的逻辑比较特殊，单独处理
            }
            else if (shapeType === 'fireworks') {
                for (let i = 0; i < particleCount; i++) {
                     // 球形爆炸
                    const theta = Math.random() * Math.PI * 2;
                    const phi = Math.acos((Math.random() * 2) - 1);
                    const r = 2 + Math.random() * 18; // 很大的扩散范围
                    
                    targetPositions[i * 3] = r * Math.sin(phi) * Math.cos(theta);
                    targetPositions[i * 3 + 1] = r * Math.sin(phi) * Math.sin(theta);
                    targetPositions[i * 3 + 2] = r * Math.cos(phi);
                }
            }
        }

        // 辅助：文字转粒子
        function createTextParticles(text) {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            canvas.width = 200;
            canvas.height = 200;
            ctx.fillStyle = '#000000';
            ctx.fillRect(0,0,200,200);
            ctx.fillStyle = '#ffffff';
            ctx.font = 'bold 120px Arial';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText(text, 100, 100);

            const imageData = ctx.getImageData(0, 0, 200, 200);
            const data = imageData.data;
            const validPoints = [];

            for(let y=0; y<200; y+=2) {
                for(let x=0; x<200; x+=2) {
                    if(data[(y*200+x)*4] > 128) {
                        validPoints.push({
                            x: (x - 100) / 4,
                            y: (100 - y) / 4 // 翻转Y轴
                        });
                    }
                }
            }

            // 将采样点映射到所有粒子
            for(let i=0; i<particleCount; i++) {
                const p = validPoints[i % validPoints.length];
                targetPositions[i*3] = p.x * 1.5 + (Math.random()-0.5);
                targetPositions[i*3+1] = p.y * 1.5 + (Math.random()-0.5);
                targetPositions[i*3+2] = (Math.random()-0.5) * 2;
            }
        }

        // --- 3. UI 初始化 ---
        function initGUI() {
            const gui = new GUI({ title: "粒子控制台" });
            
            // 模型选择
            gui.add(params, 'shape', ['heart', 'saturn', 'cake', 'number19', 'fireworks'])
               .name('选择模型')
               .onChange(val => {
                    currentShape = val;
                    generateTargetPositions(val);
                    // 切换模型时重置颜色（如果是特殊的可以加逻辑）
               });
            
            // 颜色选择
            gui.addColor(params, 'color').name('粒子颜色').onChange(val => {
                const c = new THREE.Color(val);
                handTargetColor = c;
            });

            gui.add(params, 'particleSize', 0.05, 0.5).name('粒子大小').onChange(v => material.size = v);
            gui.add(params, 'rotationSpeed', 0, 0.01).name('自转速度');
            gui.add(params, 'interactionStrength', 0.1, 5.0).name('手势灵敏度');
        }

        // --- 4. MediaPipe Hands 初始化 ---
        async function initMediaPipe() {
            // 动态导入库 (通过 global window 对象，因为 CDN 方式导入非模块化 JS)
            // 注意：上面的 importmap 只解决了 three.js，MediaPipe 需要通过 script 标签或 global 访问
            // 但为了兼容性，我们假设脚本已加载。我们使用 dynamic import 或者等待 window 对象
            
            // 实际使用 CDN 链接的简单方式:
            const { Hands } = window; // MediaPipe 将挂载在 window 上
            const { Camera } = window;

            if(!Hands || !Camera) {
                // 等待一下脚本加载（简易处理）
                setTimeout(initMediaPipe, 500);
                return;
            }

            const hands = new Hands({locateFile: (file) => {
                return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
            }});

            hands.setOptions({
                maxNumHands: 2,
                modelComplexity: 1,
                minDetectionConfidence: 0.5,
                minTrackingConfidence: 0.5
            });

            hands.onResults(onHandResults);

            const videoElement = document.getElementById('input_video');
            const cameraUtils = new Camera(videoElement, {
                onFrame: async () => {
                    await hands.send({image: videoElement});
                },
                width: 640,
                height: 480
            });

            cameraUtils.start()
                .then(() => {
                    document.getElementById('loader').style.opacity = 0;
                    document.getElementById('cam-dot').classList.add('active');
                    document.getElementById('cam-text').textContent = "手势追踪中";
                })
                .catch(err => {
                    console.error(err);
                    document.getElementById('loader').innerText = "摄像头启动失败 (请使用 HTTPS)";
                    document.getElementById('cam-text').textContent = "摄像头错误";
                });
        }

        // 手势处理核心逻辑
        function onHandResults(results) {
            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                // 逻辑：
                // 1. 如果有两只手，计算两手中心点的距离 -> 控制扩散/缩放
                // 2. 如果有一只手，计算拇指和食指指尖距离 -> 备用缩放
                
                let dist = 0;

                if (results.multiHandLandmarks.length === 2) {
                    const hand1 = results.multiHandLandmarks[0][9]; // 手掌中心附近点
                    const hand2 = results.multiHandLandmarks[1][9];
                    // 简单的欧几里得距离 (x, y 都是 0-1 归一化坐标)
                    const dx = hand1.x - hand2.x;
                    const dy = hand1.y - hand2.y;
                    dist = Math.sqrt(dx*dx + dy*dy);
                    
                    // 映射距离到缩放因子 (一般两手距离 0.1 - 0.8)
                    // 目标 scale: 0.5 (合拢) 到 2.5 (张开)
                    handInteractionFactor = THREE.MathUtils.mapLinear(dist, 0.1, 0.7, 0.5, 3.0);
                    
                } else if (results.multiHandLandmarks.length === 1) {
                    // 单手捏合
                    const hand = results.multiHandLandmarks[0];
                    const thumb = hand[4];
                    const index = hand[8];
                    const dx = thumb.x - index.x;
                    const dy = thumb.y - index.y;
                    dist = Math.sqrt(dx*dx + dy*dy);
                    
                    // 单手距离通常 0.02 - 0.2
                    handInteractionFactor = THREE.MathUtils.mapLinear(dist, 0.02, 0.2, 0.5, 2.0);
                }

                // 限制范围
                handInteractionFactor = THREE.MathUtils.clamp(handInteractionFactor, 0.5, 4.0);
            } else {
                // 如果没有手，缓慢恢复到默认大小
                handInteractionFactor = THREE.MathUtils.lerp(handInteractionFactor, 1.0, 0.05);
            }
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        // --- 5. 动画循环 ---
        function animate() {
            requestAnimationFrame(animate);

            const positions = particles.geometry.attributes.position.array;
            const colors = particles.geometry.attributes.color.array;
            const targetColor = new THREE.Color(params.color);

            // 粒子插值运动 (Lerp)
            for (let i = 0; i < particleCount; i++) {
                const px = i * 3;
                const py = i * 3 + 1;
                const pz = i * 3 + 2;

                // 获取基础目标位置
                const tx = targetPositions[px];
                const ty = targetPositions[py];
                const tz = targetPositions[pz];

                // 应用手势缩放 (将目标位置向外推)
                // 烟花模式下，扩散更剧烈
                let scale = handInteractionFactor;
                if(currentShape === 'fireworks') scale *= 1.5;

                const finalTx = tx * scale;
                const finalTy = ty * scale;
                const finalTz = tz * scale;

                // 移动粒子
                // 速度 0.05，制造拖尾和流动感
                positions[px] += (finalTx - positions[px]) * 0.08;
                positions[py] += (finalTy - positions[py]) * 0.08;
                positions[pz] += (finalTz - positions[pz]) * 0.08;

                // 颜色渐变
                colors[px] += (targetColor.r - colors[px]) * 0.05;
                colors[py] += (targetColor.g - colors[py]) * 0.05;
                colors[pz] += (targetColor.b - colors[pz]) * 0.05;
            }

            particles.geometry.attributes.position.needsUpdate = true;
            particles.geometry.attributes.color.needsUpdate = true;

            // 整体旋转
            particles.rotation.y += params.rotationSpeed;
            // 随时间微小的上下浮动
            particles.position.y = Math.sin(Date.now() * 0.001) * 0.5;

            controls.update();
            renderer.render(scene, camera);
        }

        // --- 启动程序 ---
        window.onload = () => {
            initThree();
            initGUI();
            // 延迟加载 MediaPipe 以确保 DOM 准备就绪
            // 这里为了简单直接加载脚本文件到 head
            const script1 = document.createElement('script');
            script1.src = "https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js";
            script1.crossOrigin = "anonymous";
            script1.onload = () => {
                const script2 = document.createElement('script');
                script2.src = "https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js";
                script2.crossOrigin = "anonymous";
                script2.onload = () => {
                    initMediaPipe();
                }
                document.head.appendChild(script2);
            };
            document.head.appendChild(script1);
            
            animate();
        };

    </script>
</body>
</html>
