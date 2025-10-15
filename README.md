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

        .control-btn.recording {
            background: #f44336;
            animation: pulse 1.5s infinite;
        }

        .control-btn.processing {
            background: #ff9800;
        }

        @keyframes pulse {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.7; }
        }

        .status-text {
            text-align: center;
            margin: 10px 0;
            font-size: 14px;
            opacity: 0.8;
        }

        .feature-display {
            background: rgba(0,0,0,0.2);
            border-radius: 10px;
            padding: 10px;
            margin-top: 10px;
            font-size: 12px;
            font-family: monospace;
        }

        .error-message {
            background: rgba(244, 67, 54, 0.3);
            padding: 10px;
            border-radius: 10px;
            margin: 10px 0;
            text-align: center;
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>💧 나이아드 물 절약 시스템</h1>
        <p>음향 기반 실시간 유량 측정</p>
    </div>

    <div class="main-container">
        <!-- 측정 결과 카드 -->
        <div class="measurement-card">
            <h3>현재 유량</h3>
            <div class="flow-display" id="flowDisplay">0.0 ml/s</div>
            <div class="angle-display" id="angleDisplay">각도: 0°</div>
            <div class="status-text" id="statusText">측정 대기 중</div>
            
            <div class="feature-display" id="featureDisplay">
                Centroid: - Hz<br>
                Peak Freq: - Hz<br>
                Energy: -
            </div>
        </div>

        <!-- 통계 -->
        <div class="stats-grid">
            <div class="stat-card">
                <div class="stat-value" id="sessionVolume">0</div>
                <div class="stat-label">이번 세션 (ml)</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="totalVolume">0</div>
                <div class="stat-label">오늘 총 사용량 (ml)</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="savedAmount">0</div>
                <div class="stat-label">절약량 (ml)</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="measureCount">0</div>
                <div class="stat-label">측정 횟수</div>
            </div>
        </div>

        <!-- 컨트롤 -->
        <button class="control-btn" id="recordBtn" onclick="toggleRecording()">
            🎤 녹음 시작
        </button>
        
        <button class="control-btn" onclick="resetSession()">
            🔄 세션 초기화
        </button>

        <div id="errorMessage" class="error-message" style="display:none;"></div>
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

        let audioContext;
        let analyser;
        let microphone;
        let isMonitoring = false;
        let currentFlow = 0;
        let sessionVolume = 0;
        let totalVolume = 0;
        let savedAmount = 0;
        let measureCount = 0;
        let lastUpdateTime = Date.now();

        // 연속 측정 시작
        async function toggleRecording() {
            if (!isMonitoring) {
                await startMonitoring();
            } else {
                stopMonitoring();
            }
        }

        async function startMonitoring() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                
                audioContext = new (window.AudioContext || window.webkitAudioContext)();
                analyser = audioContext.createAnalyser();
                microphone = audioContext.createMediaStreamSource(stream);
                
                analyser.fftSize = 8192;
                analyser.smoothingTimeConstant = 0.8;
                microphone.connect(analyser);
                
                isMonitoring = true;
                document.getElementById('recordBtn').textContent = '⏹️ 측정 중지';
                document.getElementById('recordBtn').classList.add('recording');
                document.getElementById('statusText').textContent = '실시간 측정 중...';
                
                // 연속 분석 시작
                continuousAnalysis();
                
            } catch (error) {
                showError('마이크 접근 실패: ' + error.message);
            }
        }

        function stopMonitoring() {
            isMonitoring = false;
            
            if (microphone) {
                microphone.disconnect();
                microphone.mediaStream.getTracks().forEach(track => track.stop());
            }
            if (audioContext) {
                audioContext.close();
            }
            
            currentFlow = 0;
            document.getElementById('recordBtn').textContent = '🎤 측정 시작';
            document.getElementById('recordBtn').classList.remove('recording');
            document.getElementById('statusText').textContent = '측정 대기 중';
            document.getElementById('flowDisplay').textContent = '0.0 ml/s';
        }

        // 연속 분석 루프
        function continuousAnalysis() {
            if (!isMonitoring) return;
            
            const bufferLength = analyser.frequencyBinCount;
            const dataArray = new Float32Array(bufferLength);
            analyser.getFloatFrequencyData(dataArray);
            
            // FFT 데이터 분석
            const features = analyzeFrequencyData(dataArray, audioContext.sampleRate);
            const result = estimateFlow(features);
            
            // 결과 업데이트
            currentFlow = result.flow;
            displayResults(result, features);
            
            // 누적 계산
            const now = Date.now();
            const deltaTime = (now - lastUpdateTime) / 1000; // 초
            
            if (currentFlow > 1.0) { // 1ml/s 이상일 때만 물 사용으로 간주
                const volume = currentFlow * deltaTime;
                sessionVolume += volume;
                totalVolume += volume;
                
                // 절약량 계산
                const avgFlow = 60;
                if (currentFlow < avgFlow) {
                    savedAmount += (avgFlow - currentFlow) * deltaTime;
                }
                
                updateDisplay();
            }
            
            lastUpdateTime = now;
            
            // 200ms마다 반복 (초당 5회)
            setTimeout(continuousAnalysis, 200);
        }

        // 주파수 데이터 분석
        function analyzeFrequencyData(freqData, sampleRate) {
            const bufferLength = freqData.length;
            const nyquist = sampleRate / 2;
            const freqStep = nyquist / bufferLength;
            
            let totalEnergy = 0;
            let weightedSum = 0;
            let maxMag = -Infinity;
            let peakFreq = 0;
            let midEnergy = 0;
            let totalMag = 0;
            
            for (let i = 0; i < bufferLength; i++) {
                const freq = i * freqStep;
                const db = freqData[i];
                const mag = Math.pow(10, db / 20); // dB를 선형으로 변환
                
                if (freq >= 100 && freq <= 8000) { // 관심 주파수 대역
                    totalEnergy += mag * mag;
                    weightedSum += freq * mag;
                    totalMag += mag;
                    
                    if (db > maxMag) {
                        maxMag = db;
                        peakFreq = freq;
                    }
                    
                    if (freq >= 1000 && freq < 2000) {
                        midEnergy += mag * mag;
                    }
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
            let bestMatch = null;
            
            for (const [angle, ref] of Object.entries(angleFlowMap)) {
                const distance = Math.sqrt(
                    Math.pow((features.spectralCentroid - ref.centroid) / 1000, 2) +
                    Math.pow((features.peakFreq - ref.peakFreq) / 100, 2)
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
            
            return bestMatch || { angle: 0, flow: 0, distance: 0 };
        }

        // 결과 표시
        function displayResults(result, features) {
            document.getElementById('flowDisplay').textContent = result.flow.toFixed(2) + ' ml/s';
            document.getElementById('angleDisplay').textContent = `각도: ${result.angle}°`;
            
            document.getElementById('featureDisplay').innerHTML = `
                Centroid: ${features.spectralCentroid.toFixed(1)} Hz<br>
                Peak Freq: ${features.peakFreq.toFixed(1)} Hz<br>
                Distance: ${result.distance.toFixed(3)}
            `;
        }

        // 통계 표시
        function updateDisplay() {
            document.getElementById('sessionVolume').textContent = Math.round(sessionVolume);
            document.getElementById('totalVolume').textContent = Math.round(totalVolume);
            document.getElementById('savedAmount').textContent = Math.round(savedAmount);
        }

        // 세션 초기화
        function resetSession() {
            sessionVolume = 0;
            document.getElementById('sessionVolume').textContent = '0';
        }

        // 에러 표시
        function showError(message) {
            const errorDiv = document.getElementById('errorMessage');
            errorDiv.textContent = message;
            errorDiv.style.display = 'block';
            setTimeout(() => errorDiv.style.display = 'none', 5000);
        }

        // 데이터 로드/저장
        window.addEventListener('load', () => {
            const stored = localStorage.getItem('naiad_data');
            if (stored) {
                const data = JSON.parse(stored);
                totalVolume = data.totalVolume || 0;
                savedAmount = data.savedAmount || 0;
                measureCount = data.measureCount || 0;
                document.getElementById('totalVolume').textContent = Math.round(totalVolume);
                document.getElementById('savedAmount').textContent = Math.round(savedAmount);
                document.getElementById('measureCount').textContent = measureCount;
            }
        });

        window.addEventListener('beforeunload', () => {
            localStorage.setItem('naiad_data', JSON.stringify({
                totalVolume: totalVolume,
                savedAmount: savedAmount,
                measureCount: measureCount
            }));
        });
    </script>
</body>
</html>
