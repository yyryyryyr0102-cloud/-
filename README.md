<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>色觉共生 - 色盲视角模拟</title>
    <!-- 引入 Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- 设置 Tailwind 配置，使用 Inter 字体 -->
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                }
            }
        }
    </script>
    <style>
        /* 隐藏原始 Video 元素，只显示 Canvas */
        #video-feed { display: none; }
        
        /* 确保 Canvas 铺满容器并实现平滑的滤镜过渡 */
        #simulation-canvas {
            width: 100%;
            height: 100%;
            object-fit: cover; /* 关键：确保画面铺满并裁剪 */
            position: absolute; /* 确保它能覆盖整个 div */
            top: 0;
            left: 0;
            transition: filter 0.3s ease-in-out; /* 0.3秒平滑过渡动画 */
        }

        /* 悬浮标签样式 */
        .floating-label {
            position: absolute;
            background-color: rgba(30, 64, 175, 0.85); /* 蓝色背景 */
            color: white;
            padding: 4px 8px;
            border-radius: 8px;
            font-size: 0.75rem; /* text-xs */
            font-weight: 600; /* font-semibold */
            pointer-events: none; /* 允许点击穿透到下面的canvas */
            transform: translate(-50%, -100%); /* 居中并置于物体上方 */
            opacity: 0;
            transition: opacity 0.3s;
            z-index: 20;
        }

        /* 模拟模式的 CSS 滤镜定义 */
        .filter-normal { filter: none; }
        /* 红色盲 - 严重红绿差异减弱 */
        .filter-protanopia { filter: saturate(0.5) hue-rotate(-20deg); }
        /* 绿色盲 - 严重红绿差异减弱 */
        .filter-deuteranopia { filter: saturate(0.5) hue-rotate(20deg); }
        /* 蓝色盲 - 蓝黄差异减弱 */
        .filter-tritanopia { filter: saturate(0.7) hue-rotate(40deg); }
        /* 全色盲 - 黑白灰 */
        .filter-achromatopsia { filter: grayscale(1); }
        /* 红色弱 - 轻度 */
        .filter-protanomaly { filter: saturate(0.7) hue-rotate(-10deg); }
        /* 绿色弱 - 轻度 */
        .filter-deuteranomaly { filter: saturate(0.7) hue-rotate(10deg); }

        /* 用于显示颜色识别弹窗的背景，需要防止滚动 */
        .modal-overlay {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.6);
            display: flex;
            justify-content: center;
            align-items: center;
            z-index: 50;
        }

    </style>
</head>
<body class="bg-gray-100 font-sans h-screen flex flex-col overflow-hidden">

    <!-- 主应用容器 -->
    <div id="app-container" class="relative flex-grow flex flex-col items-center justify-center overflow-hidden">
        
        <!-- 1. 视频和 Canvas 区域 -->
        <div class="relative w-full h-full bg-black">
            <!-- 摄像头源 (隐藏) -->
            <video id="video-feed" playsinline autoplay muted></video>
            
            <!-- 模拟画布 (应用滤镜) -->
            <canvas id="simulation-canvas" class="filter-normal"></canvas>

            <!-- 2. 实时识别悬浮标签 (Placeholder) -->
            <div id="object-labels-container">
                <!-- 标签将由 JS 动态添加，例如: -->
                <!-- <div class="floating-label" style="top: 50%; left: 50%;">真实颜色：深红色</div> -->
            </div>
            
        </div>

        <!-- 3. 当前模式显示 -->
        <div class="absolute top-4 left-4 bg-white/70 backdrop-blur-sm px-3 py-1 rounded-full shadow-lg text-sm font-semibold z-10">
            当前模式: <span id="current-mode-display">正常色觉</span>
        </div>

        <!-- 4. 底部悬浮菜单栏 -->
        <div class="absolute bottom-0 w-full p-4 bg-white/80 backdrop-blur-md shadow-2xl rounded-t-xl z-30">
            <div class="flex justify-around items-center max-w-lg mx-auto">
                
                <!-- 视角切换按钮 -->
                <button id="toggle-menu-btn" class="flex flex-col items-center p-2 rounded-xl text-indigo-700 hover:bg-indigo-100 transition">
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2H6a2 2 0 01-2-2V6zM14 6a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2V6zM4 16a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2H6a2 2 0 01-2-2v-2zM14 16a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2v-2z"></path></svg>
                    <span class="text-xs mt-1">视角切换</span>
                </button>

                <!-- 语音告知开关 -->
                <button id="toggle-speech-btn" class="flex flex-col items-center p-2 rounded-xl text-gray-600 hover:bg-gray-100 transition">
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.536 8.464a5 5 0 010 7.072m2.121-7.072a8 8 0 010 11.314m-12.728-4.243c.09-.345.247-.665.45-.964.536-.789 1.48-1.282 2.65-1.282s2.114.493 2.65 1.282c.203.299.36.619.45.964m-.45 0c0 .942-.423 1.83-1.15 2.5l-.25-.25a.707.707 0 01-1 0l-.25.25c-.727-.67-1.15-1.558-1.15-2.5zm4.243-7.778L13 19l4.949-4.949m-2.122-2.121L13 14.879l-2.828-2.828"></path></svg>
                    <span class="text-xs mt-1">语音告知 (关)</span>
                </button>

                <!-- 帮助/信息 -->
                <button id="info-btn" class="flex flex-col items-center p-2 rounded-xl text-gray-600 hover:bg-gray-100 transition">
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path></svg>
                    <span class="text-xs mt-1">帮助</span>
                </button>
            </div>
        </div>
    </div>

    <!-- 5. 视角切换模式选择面板 -->
    <div id="mode-panel" class="fixed inset-x-0 bottom-0 bg-white p-6 shadow-2xl rounded-t-2xl transform translate-y-full transition-transform duration-300 ease-in-out z-40">
        <h2 class="text-lg font-bold mb-4 text-gray-800">选择色觉模拟模式</h2>
        <div class="grid grid-cols-2 gap-4">
            <!-- 模式列表 -->
        </div>
        <button id="close-panel-btn" class="mt-6 w-full py-3 bg-gray-200 text-gray-700 font-semibold rounded-xl hover:bg-gray-300 transition">关闭</button>
    </div>

    <!-- 6. 颜色识别弹窗 (手动查询) -->
    <div id="color-modal-overlay" class="modal-overlay hidden">
        <div class="bg-white p-6 rounded-2xl shadow-xl max-w-sm w-11/12 transform transition duration-300 scale-100">
            <h3 class="text-xl font-bold mb-4 text-indigo-800">手动颜色识别结果</h3>
            <div class="flex items-center space-x-4 mb-4">
                <div id="color-swatch" class="w-12 h-12 rounded-full border-2 border-gray-300 shadow-inner"></div>
                <div>
                    <p class="text-sm text-gray-500">颜色名称 (真实)</p>
                    <p id="color-name-display" class="text-2xl font-extrabold text-gray-800">加载中...</p>
                </div>
            </div>
            <p class="text-sm text-gray-600 mb-4">RGB 值: <span id="color-rgb-display" class="font-mono text-indigo-600">--</span></p>

            <!-- Gemini API 增强功能: 颜色描述与情感联想 -->
            <button id="gemini-describe-btn" class="w-full py-2 mb-4 bg-purple-600 text-white font-semibold rounded-xl hover:bg-purple-700 transition flex items-center justify-center space-x-2">
                <svg class="w-5 h-5 text-yellow-300" fill="currentColor" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg"><path d="M10 2a8 8 0 100 16 8 8 0 000-16zM8 11.5a1.5 1.5 0 113 0 1.5 1.5 0 01-3 0zm4.5-4.5a.5.5 0 00-1 0v.5h-1v-.5a.5.5 0 00-1 0V9h3V7a.5.5 0 00-.5-.5z"/></svg>
                <span>描述颜色与情感联想</span>
            </button>
            
            <div id="gemini-output" class="p-3 bg-gray-50 border border-gray-200 rounded-lg hidden">
                <p class="text-xs text-center text-gray-500 hidden" id="gemini-loading-text">AI 正在联想中，请稍候...</p>
                <p id="gemini-description-text" class="text-sm italic text-gray-700 leading-relaxed"></p>
            </div>
            
            <button id="close-modal-btn" class="w-full py-2 mt-4 bg-indigo-600 text-white font-semibold rounded-xl hover:bg-indigo-700 transition">确定</button>
        </div>
    </div>

    <script>
        // --- Gemini API 配置 ---
        const apiKey = ""; 
        const apiUrlBase = "https://generativelanguage.googleapis.com/v1beta/models/";
        const modelName = "gemini-2.5-flash-preview-09-2025";
        const maxRetries = 3; 

        // --- 核心变量 ---
        const video = document.getElementById('video-feed');
        const canvas = document.getElementById('simulation-canvas');
        const ctx = canvas.getContext('2d', { willReadFrequently: true });
        const currentModeDisplay = document.getElementById('current-mode-display');
        const modePanel = document.getElementById('mode-panel');
        const modesContainer = modePanel.querySelector('.grid');
        const toggleMenuBtn = document.getElementById('toggle-menu-btn');
        const closePanelBtn = document.getElementById('close-panel-btn');
        const toggleSpeechBtn = document.getElementById('toggle-speech-btn');
        const objectLabelsContainer = document.getElementById('object-labels-container');
        const colorModalOverlay = document.getElementById('color-modal-overlay');
        const colorSwatch = document.getElementById('color-swatch');
        const colorNameDisplay = document.getElementById('color-name-display');
        const colorRGBDisplay = document.getElementById('color-rgb-display');
        const closeModalBtn = document.getElementById('close-modal-btn'); 
        
        // 新增 Gemini 元素引用
        const geminiDescribeBtn = document.getElementById('gemini-describe-btn');
        const geminiOutputDiv = document.getElementById('gemini-output');
        const geminiDescriptionText = document.getElementById('gemini-description-text');
        const geminiLoadingText = document.getElementById('gemini-loading-text');

        let currentMode = 'normal';
        let isSpeechEnabled = false;
        let isCameraActive = false;
        let lastSpeakTime = 0; // 用于自动播报节流
        let longPressTimer = null; // 用于长按识别
        const LONG_PRESS_DURATION = 500; // 500ms 长按时间

        // 色觉模式定义 (键: 内部标识, name: 显示名称, filterClass: CSS类名)
        const visionModes = [
            { id: 'normal', name: '正常色觉 (默认)', filterClass: 'filter-normal' },
            { id: 'protanopia', name: '红色盲 (Protanopia)', filterClass: 'filter-protanopia' },
            { id: 'deuteranopia', name: '绿色盲 (Deuteranopia)', filterClass: 'filter-deuteranopia' },
            { id: 'tritanopia', name: '蓝色盲 (Tritanopia)', filterClass: 'filter-tritanopia' },
            { id: 'achromatopsia', name: '全色盲 (Achromatopsia)', filterClass: 'filter-achromatopsia' },
            { id: 'protanomaly', name: '红色弱 (Protanomaly)', filterClass: 'filter-protanomaly' },
            { id: 'deuteranomaly', name: '绿色弱 (Deuteranomaly)', filterClass: 'filter-deuteranomaly' },
        ];

        // --- 实用工具函数 ---

        /**
         * 带有指数退避的 Fetch 重试机制
         */
        async function fetchWithRetry(url, options, retries = 0) {
            try {
                const response = await fetch(url, options);
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                return response;
            } catch (error) {
                if (retries < maxRetries) {
                    const delay = Math.pow(2, retries) * 1000;
                    // Note: Removed console.warn logging for retries as per guidelines
                    await new Promise(resolve => setTimeout(resolve, delay));
                    return fetchWithRetry(url, options, retries + 1);
                }
                throw new Error("API request failed after multiple retries.");
            }
        }

        /**
         * 自定义模态框函数 (替代 alert)
         */
        function showModal(title, message) {
            const tempModalId = 'temp-info-modal';
            let modal = document.getElementById(tempModalId);
            if (!modal) {
                modal = document.createElement('div');
                modal.id = tempModalId;
                modal.className = 'modal-overlay';
                modal.innerHTML = `
                    <div class="bg-white p-6 rounded-2xl shadow-xl max-w-sm w-11/12">
                        <h3 class="text-xl font-bold mb-4 text-indigo-800">${title}</h3>
                        <p class="text-gray-700 whitespace-pre-line mb-6">${message}</p>
                        <button class="w-full py-2 bg-indigo-600 text-white font-semibold rounded-xl hover:bg-indigo-700 transition" onclick="document.getElementById('${tempModalId}').classList.add('hidden')">关闭</button>
                    </div>
                `;
                document.body.appendChild(modal);
            } else {
                modal.querySelector('h3').textContent = title;
                modal.querySelector('p').textContent = message;
            }
            modal.classList.remove('hidden');
        }

        /**
         * 模拟颜色名称查找 (实际应用中可能需要更复杂的查找表或模型)
         * @param {number[]} rgb - [r, g, b] 数组
         * @returns {string} 颜色名称
         */
        function mockColorName(rgb) {
            const [r, g, b] = rgb;
            if (r > 200 && g < 100 && b < 100) return '鲜红色';
            if (g > 200 && r < 100 && b < 100) return '翠绿色';
            if (b > 200 && r < 100 && g < 100) return '蔚蓝色';
            if (r > 150 && g > 150 && b < 100) return '暖黄色';
            if (r < 50 && g < 50 && b < 50) return '深黑色';
            if (r > 200 && g > 200 && b > 200) return '纯白色';
            if (r > 150 && g < 100 && b > 150) return '紫罗兰色';
            // 更多颜色识别逻辑...
            
            // 默认返回一个明确的值，供 LLM 使用
            if (r > 100 && g > 100 && b < 100) return '橄榄绿';
            if (r < 100 && g < 100 && b > 100) return '深海蓝';

            return '混合色';
        }

        /**
         * 模拟物体识别结果 (占位逻辑)
         * 实际应用中需要调用深度学习模型 (如TensorFlow.js COCO-SSD)
         * @returns {Array} 模拟的识别结果
         */
        function mockObjectRecognition() {
            // 返回一个模拟的物体列表，包含屏幕坐标和真实颜色
            const currentTime = Date.now();
            // 模拟识别到的物体会随机出现在屏幕的三个位置
            return [
                { name: '苹果', color: '深红色', x: canvas.width * 0.25, y: canvas.height * 0.4 },
                { name: '天空', color: '蔚蓝色', x: canvas.width * 0.75, y: canvas.height * 0.2 },
                { name: '树叶', color: '墨绿色', x: canvas.width * 0.5, y: canvas.height * 0.7 }
            ].filter((_, index) => (currentTime + index) % 5 < 3); // 随机显示一些
        }

        /**
         * 语音播报函数
         * @param {string} text - 要播报的文本
         */
        function speak(text) {
            if ('speechSynthesis' in window) {
                const utterance = new SpeechSynthesisUtterance(text);
                // 默认选择一个中文女声，实际可由用户在设置中选择
                utterance.lang = 'zh-CN'; 
                // utterance.rate = 1.0; // 语速调节
                window.speechSynthesis.speak(utterance);
            } else {
                console.warn('浏览器不支持 Web Speech API 语音播报。');
            }
        }

        // --- Gemini API 集成功能 ---

        /**
         * 使用 Gemini LLM 生成颜色描述和情感联想
         * @param {string} colorName - 颜色名称
         * @param {number[]} rgb - [r, g, b] 颜色数组
         */
        async function generateColorDescription(colorName, rgb) {
            // 重置输出区
            geminiOutputDiv.classList.remove('hidden');
            geminiDescriptionText.textContent = '';
            geminiLoadingText.classList.remove('hidden');

            const systemPrompt = "你是一位富有诗意和情感的色彩诗人。请根据提供的颜色名称和RGB值，用中文生成一段简短（不超过40字）、富有想象力、包含情感联想和具象比喻的描述。不要使用任何标题或Markdown格式，只需纯文本描述。";
            const userQuery = `颜色名称: ${colorName}, RGB: ${rgb.join(',')}。请用中文描述这段颜色。`;

            const payload = {
                contents: [{ parts: [{ text: userQuery }] }],
                systemInstruction: { parts: [{ text: systemPrompt }] },
            };
            
            const apiUrl = `${apiUrlBase}${modelName}:generateContent?key=${apiKey}`;

            try {
                const response = await fetchWithRetry(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });

                const result = await response.json();
                const text = result.candidates?.[0]?.content?.parts?.[0]?.text || "抱歉，未能生成描述。";
                
                geminiDescriptionText.textContent = text.trim();
                if (isSpeechEnabled) {
                     speak(`颜色描述已生成：${text.trim()}`);
                }


            } catch (error) {
                geminiDescriptionText.textContent = `生成失败: ${error.message}`;
                console.error("Gemini API Error:", error);
                if (isSpeechEnabled) {
                     speak(`AI 描述功能调用失败。`);
                }
            } finally {
                // 隐藏加载状态
                geminiLoadingText.classList.add('hidden');
            }
        }

        // --- 核心功能实现 ---

        /**
         * 1. 初始化摄像头
         * 修正：简化了视频约束条件，增强了错误提示和 Fallback 机制。
         */
        async function initCamera() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ 
                    video: true // 保持兼容性优先
                });
                video.srcObject = stream;
                video.onloadedmetadata = () => {
                    video.play();
                    isCameraActive = true;
                    // 仅设置 Canvas 的内部像素尺寸与视频源尺寸一致
                    canvas.width = video.videoWidth;
                    canvas.height = video.videoHeight;
                    // CSS (object-fit: cover) 处理响应式布局
                    
                    requestAnimationFrame(drawLoop);
                    startRecognitionInterval();
                };
            } catch (err) {
                isCameraActive = false; // 确保后续的绘制和识别不会尝试依赖不存在的视频流
                console.error("摄像头访问失败: ", err);

                // --- 摄像头访问失败的 Fallback 渲染 ---
                
                // 设置 Canvas 尺寸为当前窗口尺寸，以便进行错误提示绘制
                // 注意：在 canvas 父容器使用 flex-grow w-full h-full 的情况下，这里使用 clientWidth/Height 获取实际渲染尺寸
                const appContainer = document.getElementById('app-container');
                canvas.width = appContainer.clientWidth;
                canvas.height = appContainer.clientHeight;

                let errorMessage = '浏览器或运行环境拒绝了摄像头权限。这通常是由于此应用在非安全或受限（如 iFrame）环境中运行导致的。请检查权限设置。';

                // 绘制错误提示
                ctx.fillStyle = '#1f2937'; // 灰黑色背景
                ctx.fillRect(0, 0, canvas.width, canvas.height); // 填充背景
                
                ctx.fillStyle = '#fefefe'; // 白色字体
                ctx.font = '24px Inter, sans-serif';
                ctx.textAlign = 'center';
                
                const centerX = canvas.width / 2;
                const centerY = canvas.height / 2;

                ctx.fillText('🔴 摄像头访问失败 🔴', centerX, centerY - 30);
                
                ctx.font = '16px Inter, sans-serif';
                ctx.fillStyle = '#d1d5db'; // 浅灰色字体
                ctx.fillText('(功能受限，请在支持 HTTPS 和权限的环境中运行)', centerX, centerY + 10);
                
                showModal('摄像头访问失败', errorMessage);
            }
        }

        /**
         * 渲染循环：将视频帧绘制到 Canvas 上并保持滤镜效果
         */
        function drawLoop() {
            if (isCameraActive && video.readyState === video.HAVE_ENOUGH_DATA) {
                ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
            }
            // 只有当 isCameraActive 为 true 时才继续请求下一帧
            if (isCameraActive) {
                requestAnimationFrame(drawLoop);
            }
        }

        /**
         * 2. 切换色觉模拟模式
         * @param {string} modeId - 模式的ID
         */
        function setVisionMode(modeId) {
            const mode = visionModes.find(m => m.id === modeId);
            if (!mode) return;

            // 移除旧的滤镜类，添加新的滤镜类
            canvas.className = '';
            canvas.classList.add(mode.filterClass);
            
            // 更新显示
            currentMode = modeId;
            currentModeDisplay.textContent = mode.name;
            
            // 隐藏模式选择面板
            modePanel.classList.remove('translate-y-0');
            modePanel.classList.add('translate-y-full');
        }

        /**
         * 3. 实时物体识别与标签/语音展示 (占位逻辑)
         */
        function updateObjectRecognition() {
            // 只有在摄像头活跃时才进行物体识别模拟
            if (!isCameraActive) return;

            // 清除旧标签
            objectLabelsContainer.innerHTML = '';
            
            const objects = mockObjectRecognition();

            objects.forEach(obj => {
                // 3.1 悬浮标签展示
                const label = document.createElement('div');
                label.className = 'floating-label';
                label.textContent = `真实颜色：${obj.color} (${obj.name})`;
                
                // 将视频坐标转换为屏幕像素坐标
                // 这里的转换需要根据 canvas 实际显示比例进行调整，简化为中心点定位
                const rect = canvas.getBoundingClientRect();
                const xPx = obj.x * rect.width / canvas.width;
                const yPx = obj.y * rect.height / canvas.height;

                label.style.left = `${xPx}px`;
                label.style.top = `${yPx}px`;
                label.style.opacity = '1';

                objectLabelsContainer.appendChild(label);

                // 3.2 语音播报 (每3秒自动触发一次)
                if (isSpeechEnabled) {
                    const now = Date.now();
                    if (now - lastSpeakTime > 3000) {
                        speak(`检测到 ${obj.name}，真实颜色是 ${obj.color}。`);
                        lastSpeakTime = now;
                    }
                }

                // 3.3 模拟点击物体触发语音
                label.onclick = (e) => {
                    e.stopPropagation();
                    speak(`您点击了 ${obj.name}，真实颜色是 ${obj.color}。`);
                };
            });
        }

        /**
         * 启动自动识别计时器
         */
        function startRecognitionInterval() {
            // 模拟每 2000ms 执行一次物体识别更新
            setInterval(updateObjectRecognition, 2000); 
        }


        // --- 4. 手动颜色识别 (长按/点击) ---

        /**
         * 显示颜色识别结果弹窗
         * @param {number[]} rgb - [r, g, b] 颜色数组
         * @param {string} name - 颜色名称
         */
        function showColorModal(rgb, name) {
            const hex = '#' + rgb.map(x => x.toString(16).padStart(2, '0')).join('');
            
            colorSwatch.style.backgroundColor = hex;
            colorNameDisplay.textContent = name;
            colorRGBDisplay.textContent = `${rgb[0]}, ${rgb[1]}, ${rgb[2]} (HEX: ${hex})`;
            
            // 每次打开时，隐藏 Gemini 输出并清除内容
            geminiOutputDiv.classList.add('hidden');
            geminiDescriptionText.textContent = '';
            geminiLoadingText.classList.add('hidden');

            colorModalOverlay.classList.remove('hidden');
        }

        /**
         * 获取点击点的像素颜色
         * @param {number} clientX - 屏幕X坐标
         * @param {number} clientY - 屏幕Y坐标
         */
        function getPixelColor(clientX, clientY) {
            if (!isCameraActive) {
                showModal('功能受限', '摄像头未启用，无法进行实时颜色识别。请尝试在支持摄像头的环境中运行。');
                return;
            }
            
            const rect = canvas.getBoundingClientRect();
            // 1. 计算在 Canvas CSS 尺寸上的相对坐标
            const x = clientX - rect.left;
            const y = clientY - rect.top;

            // 2. 将 CSS 坐标映射到 Canvas 内部像素坐标 (考虑 Canvas 可能会被拉伸或缩放)
            const canvasX = Math.floor(x * (canvas.width / rect.width));
            const canvasY = Math.floor(y * (canvas.height / rect.height));

            // 3. 从 Canvas 获取像素数据
            const pixelData = ctx.getImageData(canvasX, canvasY, 1, 1).data;
            const rgb = [pixelData[0], pixelData[1], pixelData[2]];

            // 4. 获取颜色名称并显示模态框
            const name = mockColorName(rgb);
            showColorModal(rgb, name);

            // 触发语音播报 (如果是点击触发)
            if (isSpeechEnabled) {
                speak(`手动识别到颜色：${name}，RGB值为 ${rgb.join(', ')}。`);
            }
        }


        // --- 事件监听器 ---

        document.addEventListener('DOMContentLoaded', () => {
            initCamera();
            
            // 渲染模式选择面板
            modesContainer.innerHTML = visionModes.map(mode => `
                <button data-mode-id="${mode.id}" 
                        class="mode-btn p-4 rounded-xl text-center shadow-md transition ${currentMode === mode.id ? 'bg-indigo-600 text-white shadow-indigo-400/50' : 'bg-white text-gray-700 hover:bg-indigo-50 hover:text-indigo-700'}">
                    ${mode.name}
                </button>
            `).join('');

            // 模式选择按钮点击事件
            modesContainer.addEventListener('click', (e) => {
                const btn = e.target.closest('.mode-btn');
                if (btn) {
                    setVisionMode(btn.dataset.modeId);
                    // 重新渲染按钮状态
                    document.querySelectorAll('.mode-btn').forEach(b => {
                        b.classList.remove('bg-indigo-600', 'text-white', 'shadow-indigo-400/50');
                        b.classList.add('bg-white', 'text-gray-700', 'hover:bg-indigo-50', 'hover:text-indigo-700');
                    });
                    btn.classList.add('bg-indigo-600', 'text-white', 'shadow-indigo-400/50');
                    btn.classList.remove('bg-white', 'text-gray-700', 'hover:bg-indigo-50', 'hover:text-indigo-700');
                }
            });

            // 视角切换菜单开关
            toggleMenuBtn.onclick = () => {
                modePanel.classList.toggle('translate-y-full');
                modePanel.classList.toggle('translate-y-0');
            };
            closePanelBtn.onclick = () => {
                modePanel.classList.add('translate-y-full');
                modePanel.classList.remove('translate-y-0');
            };

            // 语音告知开关
            toggleSpeechBtn.onclick = () => {
                isSpeechEnabled = !isSpeechEnabled;
                const icon = toggleSpeechBtn.querySelector('svg');
                const text = toggleSpeechBtn.querySelector('span');

                if (isSpeechEnabled) {
                    text.textContent = '语音告知 (开)';
                    toggleSpeechBtn.classList.remove('text-gray-600');
                    toggleSpeechBtn.classList.add('text-green-600');
                    // 切换图标为麦克风开启
                    icon.innerHTML = '<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 11a7 7 0 01-7 7m0 0a7 7 0 01-7-7m7 7v4m0 0H8m4 0h4m-4-8a4 4 0 01-4-4V6a4 4 0 014-4v0z"></path>';
                    speak('实时语音告知功能已开启。');
                } else {
                    text.textContent = '语音告知 (关)';
                    toggleSpeechBtn.classList.remove('text-green-600');
                    toggleSpeechBtn.classList.add('text-gray-600');
                    // 切换图标为默认
                    icon.innerHTML = '<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.536 8.464a5 5 0 010 7.072m2.121-7.072a8 8 0 010 11.314m-12.728-4.243c.09-.345.247-.665.45-.964.536-.789 1.48-1.282 2.65-1.282s2.114.493 2.65 1.282c.203.299.36.619.45.964m-.45 0c0 .942-.423 1.83-1.15 2.5l-.25-.25a.707.707 0 01-1 0l-.25.25c-.727-.67-1.15-1.558-1.15-2.5zm4.243-7.778L13 19l4.949-4.949m-2.122-2.121L13 14.879l-2.828-2.828"></path>';
                    if ('speechSynthesis' in window) window.speechSynthesis.cancel();
                }
            };
            
            // 关闭颜色识别弹窗
            closeModalBtn.onclick = () => {
                colorModalOverlay.classList.add('hidden');
            };
            
            // 触发 Gemini 颜色描述
            geminiDescribeBtn.onclick = () => {
                // 获取当前颜色名称和 RGB 值
                const colorName = colorNameDisplay.textContent;
                // 从显示文本中提取 RGB 值，跳过 HEX 部分
                const rgbMatch = colorRGBDisplay.textContent.match(/(\d+), (\d+), (\d+)/);
                if (colorName && colorName !== '加载中...' && rgbMatch) {
                    const rgb = [parseInt(rgbMatch[1], 10), parseInt(rgbMatch[2], 10), parseInt(rgbMatch[3], 10)];
                    generateColorDescription(colorName, rgb);
                } else {
                    geminiDescriptionText.textContent = '请先识别颜色。';
                    geminiOutputDiv.classList.remove('hidden');
                }
            };


            // --- 长按/触摸事件：手动颜色识别 ---

            // 鼠标按下/触摸开始
            canvas.addEventListener('mousedown', startLongPress);
            canvas.addEventListener('touchstart', startLongPress);

            // 鼠标抬起/触摸结束/鼠标移出
            canvas.addEventListener('mouseup', cancelLongPress);
            canvas.addEventListener('touchend', cancelLongPress);
            canvas.addEventListener('touchmove', cancelLongPress);

            // 开始长按计时
            function startLongPress(e) {
                // 阻止默认行为（如移动端缩放）
                e.preventDefault(); 
                // 清除之前的计时器
                if (longPressTimer) clearTimeout(longPressTimer);
                
                // 启动新的计时器
                longPressTimer = setTimeout(() => {
                    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
                    const clientY = e.touches ? e.touches[0].clientY : e.clientY;
                    getPixelColor(clientX, clientY);
                    longPressTimer = null; // 触发后清空
                }, LONG_PRESS_DURATION);
            }

            // 取消长按计时
            function cancelLongPress() {
                if (longPressTimer) {
                    clearTimeout(longPressTimer);
                    longPressTimer = null;
                }
            }

            // --- 帮助信息弹窗 (替代 alert) ---
            document.getElementById('info-btn').onclick = () => {
                showModal('帮助信息', '欢迎使用色觉共生模拟应用！\n\n- **视角切换:** 点击底部“视角切换”按钮选择不同的色盲或色弱模式。\n- **手动查询:** 长按屏幕任意区域，可以识别该点的真实颜色名称和RGB值。\n- **✨ 颜色描述:** 在手动识别弹窗中点击紫色按钮，由 AI 生成该颜色的诗意描述。\n- **语音告知:** 点击“语音告知”按钮开启，将自动或手动播报颜色信息。');
            }
        });
    </script>
</body>
</html>
