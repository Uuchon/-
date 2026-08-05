// ============================================
// メイン設定と状態管理
// ============================================
let audioProcessor;
let frequencyMapper;
let harmonicAnalyzer;
let fft;

let dominantFreq = 0;
let dominantAmplitude = 0;
let circleRadius = 50;
let currentColor = { r: 255, g: 0, b: 0 };
let harmonicRatio = 0.5;
let spectrumData = [];

// 音声解析用グローバル変数
let spectralCentroid = 0;
let spectralSpread = 0;

let micActive = false;
let noiseActive = false;

// 3秒環境テスト（キャリブレーション）用変数
let isCalibrating = false;
let calibrationStartTime = 0;
let calibrationDuration = 3000; // 3000ms = 3秒間
let tempCollectedFreqs = [];

let micButton, noiseButton, testButton;
let colorHistory = []; 

function setup() {
    createCanvas(windowWidth, windowHeight);
    
    audioProcessor = new AudioProcessor();
    frequencyMapper = new FrequencyMapper(); // 初期値: 80Hz 〜 3000Hz（日常音域）
    harmonicAnalyzer = new HarmonicAnalyzer();
    fft = new p5.FFT(0.8, 512);
    
    createUIButtons();
}

function draw() {
    background(250, 252, 255);
    
    // スイープテスト音動作中の処理
    if (noiseActive) {
        let sweepFreq = map(sin(frameCount * 0.03), -1, 1, frequencyMapper.minFreq, frequencyMapper.maxFreq);
        audioProcessor.setFreq(sweepFreq);
    }

    // 音声解析
    spectrumData = fft.analyze();
    analyzeFrequencyContent();
    
    // 3秒間テストの進行処理
    handleCalibration();
    
    // 時系列の蓄積
    pushColorHistory();
    
    // 1. 中央の動的オブジェクト描画
    push();
    translate(width / 2, height / 2 - 40);
    updateCircleProperties();
    drawDynamicCircle();
    pop();
    
    // 2. UIパネル・UI情報の描画
    drawStatusPanel();
    drawSpectrumBar();
    drawFrequencyGraph();
    drawInfoText();
    
    // 3. 最下部カラータイムライン
    drawColorHistoryTimeline();
}

// ============================================
// 音声解析 & カラーマッピング
// ============================================
function analyzeFrequencyContent() {
    let totalAmp = 0;
    let weightedFreq = 0;
    let maxAmplitude = 0;
    let maxFreqIndex = 0;
    const nyquist = sampleRate() / 2;

    for (let i = 1; i < spectrumData.length; i++) {
        const amp = spectrumData[i];
        if (amp > maxAmplitude) {
            maxAmplitude = amp;
            maxFreqIndex = i;
        }
    }
    dominantAmplitude = maxAmplitude / 255;
    dominantFreq = (maxFreqIndex * nyquist) / spectrumData.length;

    // スペクトル重心（Centroid）の計算
    for (let i = 1; i < spectrumData.length; i++) {
        const amp = spectrumData[i];
        if (amp < 5) continue;
        const freq = (i * nyquist) / spectrumData.length;
        if (freq < 20 || freq > 20000) continue;

        weightedFreq += freq * amp;
        totalAmp += amp;
    }
    spectralCentroid = totalAmp > 0 ? weightedFreq / totalAmp : 0;

    // スペクトル分散（Spread）の計算
    let variance = 0;
    for (let i = 1; i < spectrumData.length; i++) {
        const amp = spectrumData[i];
        if (amp < 5) continue;
        const freq = (i * nyquist) / spectrumData.length;
        if (freq < 20 || freq > 20000) continue;

        variance += amp * Math.pow(freq - spectralCentroid, 2);
    }
    spectralSpread = totalAmp > 0 ? Math.sqrt(variance / totalAmp) : 0;

    // 倍音構造解析
    harmonicRatio = harmonicAnalyzer.analyzeHarmonics(spectrumData, dominantFreq);

    // Dynamic Range（キャリブレーション値）に応じた色算出
    currentColor = calculateCentroidColor();
}

function calculateCentroidColor() {
    if (dominantAmplitude < 0.02 || spectralCentroid === 0) {
        return { r: 230, g: 235, b: 240 }; // 無音時
    }

    // ① キャリブレーションされた範囲に基づき ナノメートル (nm) 算出
    let nm = frequencyMapper.frequencyToNanometers(spectralCentroid);
    
    // ② 780nm(低音:赤) -> 380nm(高音:紫) を Hue: 0° -> 270° にマッピング
    let hueValue = map(nm, 780, 380, 0, 270, true);

    // ③ スペクトル分散 (Hz) -> 彩度 (Sat)
    let spreadLog = Math.log(spectralSpread + 1);
    let sat = map(spreadLog, 0, 9, 100, 20, true);

    // ④ 振幅 -> 明度 (Bri)
    let bri = map(dominantAmplitude, 0.02, 1, 30, 100, true);

    // ⑤ HSBで生成しRGBに即時変換
    colorMode(HSB, 360, 100, 100);
    let c = color(hueValue, sat, bri);
    
    let result = { r: red(c), g: green(c), b: blue(c) };
    colorMode(RGB, 255);
    return result;
}

// ============================================
// ⏱️ 3秒環境テスト（キャリブレーション）処理
// ============================================
function startCalibration() {
    if (!micActive && !noiseActive) {
        alert("マイクを開始するか、テスト音を起動してからテストしてください！");
        return;
    }
    isCalibrating = true;
    calibrationStartTime = millis();
    tempCollectedFreqs = [];
}

function handleCalibration() {
    if (!isCalibrating) return;
    
    let elapsed = millis() - calibrationStartTime;
    let progress = Math.min(100, Math.floor((elapsed / calibrationDuration) * 100));
    
    testButton.html(`⏳ 計測中 ${progress}%`);
    testButton.style('background-color', '#ff9f43');

    if (elapsed < calibrationDuration) {
        // テスト中：音が検知されている間、スペクトル重心（または主要周波数）を記録
        if (spectralCentroid > 0 && dominantAmplitude > 0.03) {
            tempCollectedFreqs.push(spectralCentroid);
        }
    } else {
        // 3秒経過：範囲決定
        isCalibrating = false;
        testButton.html('⏱️ 3秒環境テスト');
        testButton.style('background-color', '#10ac84');
        
        if (tempCollectedFreqs.length > 5) {
            let detectedMin = Math.min(...tempCollectedFreqs);
            let detectedMax = Math.max(...tempCollectedFreqs);
            
            // 最低限の帯域幅（200Hz）を確保（高音・低音の差が小さすぎる場合対策）
            if (detectedMax - detectedMin < 200) {
                detectedMin = Math.max(20, detectedMin - 100);
                detectedMax = Math.min(20000, detectedMax + 100);
            }
            
            // 周波数マッパーの範囲を更新！
            frequencyMapper.updateRange(detectedMin, detectedMax);
        } else {
            alert("十分な音量が検知できませんでした。声を出しながらもう一度お試しください。");
        }
    }
}

// ============================================
// UI & グラフィック描画
// ============================================
function pushColorHistory() {
    colorHistory.push(currentColor);
    if (colorHistory.length > width) {
        colorHistory.shift();
    }
}

function drawColorHistoryTimeline() {
    const timelineHeight = 40;
    const timelineY = height - timelineHeight;
    
    noStroke();
    for (let i = 0; i < colorHistory.length; i++) {
        let x = map(i, 0, colorHistory.length, 0, width);
        let w = (width / colorHistory.length) + 1;
        let col = colorHistory[i];
        fill(col.r, col.g, col.b);
        rect(x, timelineY, w, timelineHeight);
    }
    
    stroke(200);
    strokeWeight(1);
    line(0, timelineY, width, timelineY);
    
    fill(100);
    noStroke();
    textSize(11);
    textAlign(LEFT, BOTTOM);
    text('⏱️ 音色の時系列ヒストリー（右端が現在）', 20, timelineY - 6);
}

function updateCircleProperties() {
    let targetRadius = 60 + dominantAmplitude * 250;
    circleRadius = lerp(circleRadius, targetRadius, 0.15);
}

function drawDynamicCircle() {
    const segments = 128;
    const baseRadius = circleRadius;
    const deformation = (1 - harmonicRatio) * 0.4;
    
    noStroke();
    fill(currentColor.r, currentColor.g, currentColor.b, 220); 
    
    beginShape();
    for (let i = 0; i < segments; i++) {
        const angle = TWO_PI * i / segments;
        const noiseAmount = deformation * sin(angle * 5 + frameCount * 0.05) * baseRadius * dominantAmplitude;
        const r = baseRadius + noiseAmount;
        vertex(r * cos(angle), r * sin(angle));
    }
    endShape(CLOSE);
    
    stroke(currentColor.r, currentColor.g, currentColor.b);
    strokeWeight(3);
    noFill();
    
    beginShape();
    for (let i = 0; i < segments; i++) {
        const angle = TWO_PI * i / segments;
        const noiseAmount = deformation * sin(angle * 5 + frameCount * 0.05) * baseRadius * dominantAmplitude;
        const r = baseRadius + noiseAmount;
        vertex(r * cos(angle), r * sin(angle));
    }
    endShape(CLOSE);
}

function drawStatusPanel() {
    fill(40);
    noStroke();
    textSize(13);
    textAlign(LEFT, TOP);
    
    let statusText = '待機中';
    if (isCalibrating) statusText = '⚠️ 3秒間音域計測中...';
    else if (micActive) statusText = 'マイク入力中';
    else if (noiseActive) statusText = 'スイープテスト音生成中';
    
    const nmValue = frequencyMapper.frequencyToNanometers(spectralCentroid);
    
    text(`状態: ${statusText}`, 20, 80);
    text(`主要周波数: ${dominantFreq.toFixed(0)} Hz`, 20, 105);
    text(`スペクトル重心: ${spectralCentroid.toFixed(0)} Hz`, 20, 130);
    text(`推定波長: ${spectralCentroid > 0 ? nmValue.toFixed(0) : "0"} nm`, 20, 155);
    
    // 現在適合している周波数範囲の表示
    fill(10, 120, 200);
    text(`適合音域: ${frequencyMapper.minFreq.toFixed(0)} Hz 〜 ${frequencyMapper.maxFreq.toFixed(0)} Hz`, 20, 180);
}

function drawSpectrumBar() {
    const barX = 20;
    const barY = height - 110;
    const barWidth = min(width - 40, 700);
    const barHeight = 25;

    // 赤（左: 0°）から 紫（右: 270°）へのグラデーション
    colorMode(HSB, 360, 100, 100);
    for (let i = 0; i < barWidth; i++) {
        let hueVal = map(i, 0, barWidth, 0, 270);
        stroke(hueVal, 90, 90);
        line(barX + i, barY, barX + i, barY + barHeight);
    }
    colorMode(RGB, 255);

    noFill();
    stroke(180);
    rect(barX, barY, barWidth, barHeight, 4);

    // 重心マーカーの表示
    if (spectralCentroid > 0 && dominantAmplitude > 0.02) {
        const currentNm = frequencyMapper.frequencyToNanometers(spectralCentroid);
        // 780nm(赤)=barX, 380nm(紫)=barX+barWidth
        const markerX = map(currentNm, 780, 380, barX, barX + barWidth);

        fill(30);
        noStroke();
        triangle(markerX, barY - 2, markerX - 6, barY - 10, markerX + 6, barY - 10);
    }

    fill(80);
    noStroke();
    textSize(11);
    textAlign(LEFT, TOP);
    text(`低音 赤 (${frequencyMapper.minFreq.toFixed(0)}Hz)`, barX, barY + barHeight + 5);
    textAlign(RIGHT, TOP);
    text(`高音 紫 (${frequencyMapper.maxFreq.toFixed(0)}Hz)`, barX + barWidth, barY + barHeight + 5);
}

function drawFrequencyGraph() {
    const panelWidth = 280;
    const panelHeight = 120;
    const panelX = width - panelWidth - 20;
    const panelY = height - panelHeight - 110;
    
    fill(245, 247, 250, 220);
    stroke(220);
    strokeWeight(1);
    rect(panelX, panelY, panelWidth, panelHeight, 8);
    
    noFill();
    stroke(currentColor.r, currentColor.g, currentColor.b);
    strokeWeight(2);
    beginShape();
    for (let i = 0; i < spectrumData.length; i++) {
        const x = map(i, 0, spectrumData.length, panelX + 10, panelX + panelWidth - 10);
        const y = map(spectrumData[i], 0, 255, panelY + panelHeight - 10, panelY + 10);
        vertex(x, y);
    }
    endShape();
    
    fill(100);
    noStroke();
    textSize(11);
    textAlign(LEFT, TOP);
    text('リアルタイムFFTスペクトル', panelX + 10, panelY + 8);
}

function drawInfoText() {
    const panelX = width - 240;
    const panelY = 20;
    
    fill(40);
    noStroke();
    textSize(14);
    textAlign(LEFT, TOP);
    text('リアルタイム音声・音色可視化', panelX, panelY);
    
    fill(100);
    textSize(11);
    text('1. 「🎤マイク開始」を押す\n2. 低い声〜高い声を出しながら\n   「⏱️3秒環境テスト」を押す\n3. あなたの声域に合わせて\n   赤〜紫の色変化が最適化されます！', panelX, panelY + 25);
}

// ============================================
// UIコントロール
// ============================================
function createUIButtons() {
    micButton = createButton('🎤 マイク開始');
    micButton.position(20, 20);
    styleButton(micButton, '#4a69bd');
    micButton.mousePressed(async () => {
        await userStartAudio();
        if (!micActive) {
            if (noiseActive) toggleNoise();
            audioProcessor.startMicrophone();
            micActive = true;
            micButton.html('🎤 マイク停止');
        } else {
            audioProcessor.stopMicrophone();
            micActive = false;
            micButton.html('🎤 マイク開始');
        }
    });

    noiseButton = createButton('🔊 スイープ音テスト');
    noiseButton.position(170, 20);
    styleButton(noiseButton, '#4a69bd');
    noiseButton.mousePressed(async () => {
        await userStartAudio();
        toggleNoise();
    });

    testButton = createButton('⏱️ 3秒環境テスト');
    testButton.position(320, 20);
    styleButton(testButton, '#10ac84');
    testButton.mousePressed(startCalibration);
}

function toggleNoise() {
    if (!noiseActive) {
        if (micActive) {
            audioProcessor.stopMicrophone();
            micActive = false;
            micButton.html('🎤 マイク開始');
        }
        audioProcessor.startOscillator();
        noiseActive = true;
        noiseButton.html('🔊 テスト音停止');
    } else {
        audioProcessor.stopOscillator();
        noiseActive = false;
        noiseButton.html('🔊 スイープ音テスト');
    }
}

function styleButton(btn, bgColor) {
    btn.style('background-color', bgColor);
    btn.style('color', 'white');
    btn.style('border', 'none');
    btn.style('padding', '8px 12px');
    btn.style('border-radius', '20px');
    btn.style('cursor', 'pointer');
    btn.style('font-weight', 'bold');
    btn.style('font-size', '12px');
    btn.style('width', '135px');
    btn.style('box-shadow', '0 2px 5px rgba(0,0,0,0.1)');
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}

// ============================================
// 音声マッピングクラス
// ============================================
class FrequencyMapper {
    constructor() {
        // デフォルト範囲（80Hz〜3000Hz: 人の声に合わせた初期値）
        this.minFreq = 80;
        this.maxFreq = 3000;
    }

    updateRange(minF, maxF) {
        this.minFreq = minF;
        this.maxFreq = maxF;
    }

    frequencyToNanometers(freq) {
        const logMin = Math.log10(this.minFreq);
        const logMax = Math.log10(this.maxFreq);
        const logFreq = Math.log10(constrain(freq, this.minFreq, this.maxFreq));
        
        // 低音(minFreq) -> 780nm (赤)
        // 高音(maxFreq) -> 380nm (紫)
        return 780 - (780 - 380) * ((logFreq - logMin) / (logMax - logMin));
    }
}

class HarmonicAnalyzer {
    analyzeHarmonics(spectrum, fundamentalFreq) {
        if (fundamentalFreq < 20 || fundamentalFreq > 20000) return 0.5;
        let harmonicStrength = 0, totalStrength = 0;
        const nyquist = sampleRate() / 2;

        for (let harmonic = 1; harmonic <= 8; harmonic++) {
            const harmonicFreq = fundamentalFreq * harmonic;
            if (harmonicFreq > 20000) break;

            const index = Math.round((harmonicFreq / nyquist) * spectrum.length);
            if (index >= 0 && index < spectrum.length) {
                totalStrength += spectrum[index];
                harmonicStrength += spectrum[index] * (1 - (harmonic - 1) * 0.1);
            }
        }
        return totalStrength > 0 ? constrain(harmonicStrength / totalStrength, 0.1, 1.0) : 0.5;
    }
}

class AudioProcessor {
    constructor() {
        this.mic = new p5.AudioIn();
        this.osc = new p5.Oscillator('sine');
        this.osc.amp(0.2);
    }

    startMicrophone() {
        this.mic.start();
        fft.setInput(this.mic);
    }

    stopMicrophone() {
        this.mic.stop();
    }

    startOscillator() {
        this.osc.start();
        fft.setInput(this.osc);
    }

    stopOscillator() {
        this.osc.stop();
    }

    setFreq(freq) {
        this.osc.freq(freq);
    }
}
