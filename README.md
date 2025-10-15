<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>나이아드 물 절약 시스템</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Arial', sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            color: white;
        }

        .header {
            text-align: center;
            padding: 20px;
            background: rgba(0,0,0,0.1);
            backdrop-filter: blur(10px);
        }

        .header h1 {
            font-size: 24px;
            margin-bottom: 5px;
        }

        .main-container {
            flex: 1;
            padding: 20px;
            max-width: 400px;
            margin: 0 auto;
            width: 100%;
        }

        .measurement-card {
            background: rgba(255,255,255,0.1);
            backdrop-filter: blur(10px);
            border-radius: 20px;
            padding: 20px;
            margin-bottom: 20px;
            text-align: center;
        }

        .flow-display {
            font-size: 48px;
            font-weight: bold;
            margin: 20px 0;
            color: #4CAF50;
            transition: all 0.3s ease;
        }

        .flow-display.flowing {
            color: #ff9800;
            animation: glow 1s infinite;
        }

        @keyframes glow {
            0%, 100% { transform: scale(1); }
            50% { transform: scale(1.05); }
        }

        .angle-display {
            font-size: 24px;
            margin: 10px 0;
            opacity: 0.9;
        }

        .stats-grid {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 15px;
            margin-bottom: 20px;
        }

        .stat-card {
            background: rgba(255,255,255,0.1);
            backdrop-filter: blur(10px);
            border-radius: 15px;
            padding: 15px;
            text-align: center;
        }

        .stat-value {
            font-size: 24px;
            font-weight: bold;
            margin-bottom: 5px;
        }

        .stat-label {
            font-size: 12px;
            opacity: 0.8;
        }

        .status-indicator {
            display: inline-block;
            width: 12px;
            height: 12px;
            border-radius: 50%;
            background: #4CAF50;
            animation: pulse 2s infinite;
            margin-right: 8px;
        }

        @keyframes pulse {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.5; }
        }

        .feature-display {
            background: rgba(0,0,0,0.2);
            border-radius: 10px;
            padding: 10px;
            margin-top: 10px;
            font-size: 11px;
            font-family: monospace;
            text-align: left;
        }

        .control-btn {
            background: rgba(255,255,255,0.2);
            border: none;
            border-radius: 25px;
            padding: 15px 30px;
            color: white;
            cursor: pointer;
            font-size: 16px;
            width: 100%;
            margin: 10px 0;
            transition: all 0.3s ease;
        }

        .control-btn:hover {
            background: rgba(255,255,255,0.3);
            transform: translateY(-2px);
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>💧 나이아드 물 절약 시스템</h1>
        <p>실시간 음향 기반 유량 측정</p>
    </div>

    <div class="main-container">
        <!-- 측정 결과 카드 -->
        <div class="measurement-card">
            <h3><span class="status-indicator"></span>실시간 모니터링</h3>
            <div class="flow-display" id="flowDisplay">0.0 ml/s</div>
            <div class="angle-display" id="angleDisplay">각도: 0°</div>
            
            <div class="feature-display" id="featureDisplay">
                Centroid: 0 Hz | Peak: 0 Hz<br>
                Energy: 0.00e+00 | Distance: 0.0000
            </div>
        </div>

        <!-- 통계 -->
        <div class="stats-grid">
            <div class="stat-card">
                <div class="stat-value" id="currentFlow">0.0</div>
                <div class="stat-label">현재 유량 (ml/s)</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="totalVolume">0</div>
                <div class="stat-label">오늘 사용량 (ml)</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="savedAmount">0</div>
                <div class="stat-label">절약량 (ml)</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="sessionTime">0</div>
                <div class="stat-label">사용 시간 (초)</div>
            </div>
        </div>

        <button class="control-btn" onclick="resetDaily()">
            🔄 일일 초기화
        </button>
    </div>

    <script>
        // 학습된 각도-유량 매핑
        const angleFlowMap = {
            8: { flow: 24.75, centroid: 2373.0, peakFreq: 120.0, energy: 2.90e+08 },
            16: { flow: 30.94, centroid: 2075.7, peakFreq: 120.0, energy: 6.50e+09 },
            24: { flow: 40.84, centroid: 2779.4, peakFreq: 119.9, energy: 1.99e+09 },
            32: { flow: 61.88, centroid: 5005.7, peakFreq: 383.5, energy: 8.70e+08 },
            40: { flow: 123.76, centroid: 4177.7, peakFreq: 120.0, energy: 1.92e+09 }
        };

        // 전역 변수
        let audioContext;
        let analyser;
        let microphone;
        let scriptProcessor;
        let totalVolume = 0;
        let savedAmount = 0;
        let sessionTime = 0;
        let lastUpdateTime = Date.now();
        let currentFlow = 0;
        let isFlowing = false;

        // 페이지 로드 시 자동 시작
        window.addEventListener('load', async () => {
            loadStoredData();
            await startRealTimeAnalysis();
        });

        // 실시간 분석 시작
        async function startRealTimeAnalysis() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ 
                    audio: {
                        echoCancellation: false,
                        noiseSuppression: false,
                        autoGainControl: false
                    } 
                });
                
                audioContext = new (window.AudioContext || window.webkitAudioContext)();
                analyser = audioContext.createAnalyser();
                microphone = audioContext.createMediaStreamSource(stream);
                scriptProcessor = audioContext.createScriptProcessor(4096, 1, 1);
                
                analyser.fftSize = 8192;
                analyser.smoothingTimeConstant = 0.3;
                
                microphone.connect(analyser);
                analyser.connect(scriptProcessor);
                scriptProcessor.connect(audioContext.destination);
                
                scriptProcessor.onaudioprocess = processAudioFrame;
                
                console.log('실시간 분석 시작');
                
            } catch (error) {
                alert('마이크 접근 실패: ' + error.message);
            }
        }

        // 오디오 프레임 처리
        function processAudioFrame() {
            const bufferLength = analyser.frequencyBinCount;
            const dataArray = new Uint8Array(bufferLength);
            analyser.getByteFrequencyData(dataArray);
            
            // FFT 특징 추출
            const features = extractFeatures(dataArray, audioContext.sampleRate);
            
            // 유량 추정
            const result = estimateFlow(features);
            
            // UI 업데이트
            updateDisplay(result, features);
            
            // 통계 업데이트
            updateStatistics(result.flow);
        }

        // 특징 추출
        function extractFeatures(fftData, sampleRate) {
            const freqStep = sampleRate / (fftData.length * 2);
            
            let totalEnergy = 0;
            let weightedSum = 0;
            let maxMag = 0;
            let peakFreq = 0;
            let midEnergy = 0;
            let totalMag = 0;
            
            for (let i = 0; i < fftData.length; i++) {
                const freq = i * freqStep;
                const mag = fftData[i];
                
                // 100Hz 이하 무시 (노이즈)
                if (freq < 100) continue;
                
                totalEnergy += mag * mag;
                totalMag += mag;
                weightedSum += freq * mag;
                
                if (mag > maxMag) {
                    maxMag = mag;
                    peakFreq = freq;
                }
                
                // Mid-band (1000-2000 Hz)
                if (freq >= 1000 && freq < 2000) {
                    midEnergy += mag * mag;
                }
            }
            
            const spectralCentroid = totalMag > 0 ? weightedSum / totalMag : 0;
            
            return {
                spectralCentroid: spectralCentroid,
                peakFreq: peakFreq,
                totalEnergy: totalEnergy,
                midEnergy: midEnergy
            };
        }

        // 유량 추정
        function estimateFlow(features) {
            let minDistance = Infinity;
            let bestMatch = { angle: 0, flow: 0, distance: 0 };
            
            // 에너지가 너무 낮으면 물이 안 나오는 것으로 판단
            if (features.totalEnergy < 1000) {
                return bestMatch;
            }
            
            for (const [angle, ref] of Object.entries(angleFlowMap)) {
                const distance = Math.sqrt(
                    Math.pow((features.spectralCentroid - ref.centroid) / 1000, 2) +
                    Math.pow((features.peakFreq - ref.peakFreq) / 100, 2) +
                    Math.pow((features.totalEnergy - ref.energy) / 1e7, 2)
                );
                
                if (distance < minDistance) {
                    minDistance = distance;
                    bestMatch = {
                        angle: parseInt(angle),
                        flow: ref.flow,
                        distance: distance
                    };
                }
            }
            
            // 거리가 너무 멀면 0으로 처리
            if (minDistance > 50) {
                bestMatch.flow = 0;
            }
            
            return bestMatch;
        }

        // 화면 업데이트
        function updateDisplay(result, features) {
            const flowEl = document.getElementById('flowDisplay');
            flowEl.textContent = result.flow.toFixed(2) + ' ml/s';
            
            if (result.flow > 5) {
                flowEl.classList.add('flowing');
            } else {
                flowEl.classList.remove('flowing');
            }
            
            document.getElementById('angleDisplay').textContent = `각도: ${result.angle}°`;
            document.getElementById('currentFlow').textContent = result.flow.toFixed(1);
            
            document.getElementById('featureDisplay').innerHTML = 
                `Centroid: ${features.spectralCentroid.toFixed(0)} Hz | ` +
                `Peak: ${features.peakFreq.toFixed(0)} Hz<br>` +
                `Energy: ${features.totalEnergy.toExponential(2)} | ` +
                `Distance: ${result.distance.toFixed(4)}`;
        }

        // 통계 업데이트
        function updateStatistics(flow) {
            const now = Date.now();
            const deltaTime = (now - lastUpdateTime) / 1000; // 초
            lastUpdateTime = now;
            
            currentFlow = flow;
            
            if (flow > 1) {
                isFlowing = true;
                const volume = flow * deltaTime;
                totalVolume += volume;
                sessionTime += deltaTime;
                
                // 절약량 계산 (평균 60 ml/s 기준)
                if (flow < 60) {
                    savedAmount += (60 - flow) * deltaTime;
                }
            } else {
                isFlowing = false;
            }
            
            document.getElementById('totalVolume').textContent = Math.round(totalVolume);
            document.getElementById('savedAmount').textContent = Math.round(savedAmount);
            document.getElementById('sessionTime').textContent = Math.round(sessionTime);
        }

        // 데이터 로드
        function loadStoredData() {
            const stored = localStorage.getItem('naiad_data');
            if (stored) {
                const data = JSON.parse(stored);
                const today = new Date().toDateString();
                
                if (data.date === today) {
                    totalVolume = data.totalVolume || 0;
                    savedAmount = data.savedAmount || 0;
                    sessionTime = data.sessionTime || 0;
                }
            }
        }

        // 데이터 저장
        function saveData() {
            const today = new Date().toDateString();
            localStorage.setItem('naiad_data', JSON.stringify({
                date: today,
                totalVolume: totalVolume,
                savedAmount: savedAmount,
                sessionTime: sessionTime
            }));
        }

        // 자동 저장 (5초마다)
        setInterval(saveData, 5000);

        // 일일 초기화
        function resetDaily() {
            if (confirm('오늘의 데이터를 초기화하시겠습니까?')) {
                totalVolume = 0;
                savedAmount = 0;
                sessionTime = 0;
                saveData();
                
                document.getElementById('totalVolume').textContent = '0';
                document.getElementById('savedAmount').textContent = '0';
                document.getElementById('sessionTime').textContent = '0';
            }
        }

        // 페이지 언로드 시 저장
        window.addEventListener('beforeunload', saveData);
    </script>
</body>
</html>
