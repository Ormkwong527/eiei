<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1" />
  <title>拍照识物 + 英文读音</title>
  <style>
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial; margin:0; padding:0; background:#f7f7f8; color:#111; display:flex; flex-direction:column; align-items:center; }
    header { width:100%; padding:12px 16px; box-shadow:0 1px 0 rgba(0,0,0,0.06); background:white; display:flex; justify-content:space-between; align-items:center; }
    header h1 { margin:0; font-size:16px; }
    .container { max-width:900px; width:100%; padding:12px; box-sizing:border-box; }
    .video-wrap { position:relative; width:100%; padding-top:56%; background:#000; border-radius:10px; overflow:hidden; }
    video { position:absolute; top:0; left:0; width:100%; height:100%; object-fit:cover; }
    canvas { display:none; }
    .controls { margin-top:12px; display:flex; gap:8px; flex-wrap:wrap; }
    button { padding:10px 14px; border-radius:8px; border:0; background:#0066ff; color:white; font-weight:600; }
    button.secondary { background:#f1f3f5; color:#111; border:1px solid #ddd; }
    .results { margin-top:14px; background:white; padding:12px; border-radius:10px; box-shadow:0 4px 20px rgba(0,0,0,0.04); }
    .pred { display:flex; align-items:center; justify-content:space-between; padding:8px 0; border-bottom:1px dashed #eee; }
    .pred:last-child { border-bottom:0; }
    .label { font-size:16px; font-weight:700; }
    .prob { color:#666; }
    footer { margin:18px 0; color:#666; font-size:13px; }
    .small { font-size:13px; color:#666; }
  </style>
</head>
<body>
  <header>
    <h1>拍照识物 · 英文 + 发音</h1>
    <div class="small">Model: MobileNet · Client-side</div>
  </header>

  <div class="container">
    <div class="video-wrap" id="videoWrap">
      <video id="video" autoplay playsinline muted></video>
      <canvas id="snapshot"></canvas>
    </div>

    <div class="controls">
      <button id="btnCapture">📸 拍照并识别</button>
      <button id="btnToggleTorch" class="secondary">🔦 手电（若支持）</button>
      <button id="btnSpeak" class="secondary">🔊 发音</button>
      <div id="status" class="small" style="align-self:center;">初始化中…请允许相机权限</div>
    </div>

    <div class="results" id="results" aria-live="polite">
      <div class="small">识别结果（Top-3）</div>
      <div id="predList"></div>
    </div>

    <footer>
      Tip: 首次加载会下载模型文件，可能需几秒到数十秒，之后加载更快。支持现代手机浏览器（Chrome/Edge/Safari）。若要做原生 App，可把这个逻辑嵌到 WebView 或移植为 React Native + TensorFlow Lite。
    </footer>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.12.0/dist/tf.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/mobilenet@2.2.1/dist/mobilenet.min.js"></script>

  <script>
  (async () => {
    const video = document.getElementById('video');
    const canvas = document.getElementById('snapshot');
    const btnCapture = document.getElementById('btnCapture');
    const btnSpeak = document.getElementById('btnSpeak');
    const btnToggleTorch = document.getElementById('btnToggleTorch');
    const status = document.getElementById('status');
    const predList = document.getElementById('predList');

    let model = null;
    let stream = null;
    let track = null;
    let torchOn = false;
    let lastTopLabel = '';

    async function startCamera() {
      try {
        stream = await navigator.mediaDevices.getUserMedia({
          audio: false,
          video: { facingMode: { ideal: 'environment' }, width: { ideal: 1280 }, height: { ideal: 720 } }
        });
        video.srcObject = stream;
        track = stream.getVideoTracks()[0];
        status.textContent = '相机已开启，加载模型中…';
      } catch (err) {
        console.error('无法打开相机:', err);
        status.textContent = '无法打开相机：' + (err.message || err.name);
      }
    }

    async function toggleTorch() {
      if (!track) return;
      const capabilities = track.getCapabilities?.();
      if (!capabilities || !capabilities.torch) {
        alert('当前设备/浏览器不支持手电控制');
        return;
      }
      try {
        torchOn = !torchOn;
        await track.applyConstraints({ advanced: [{ torch: torchOn }] });
        btnToggleTorch.textContent = torchOn ? '🔦 关手电' : '🔦 手电（若支持）';
      } catch (e) {
        alert('切换手电失败：' + e.message);
      }
    }

    async function loadModel() {
      try {
        model = await mobilenet.load({ version: 2, alpha: 1.0 });
        status.textContent = '模型加载完成，准备就绪';
      } catch (err) {
        status.textContent = '加载模型失败：' + (err.message || err.name);
      }
    }

    async function captureAndClassify() {
      if (!model) { alert('模型尚未加载完成'); return; }
      const vw = video.videoWidth;
      const vh = video.videoHeight;
      if (!vw || !vh) { alert('视频尚未就绪，请稍候'); return; }
      canvas.width = vw;
      canvas.height = vh;
      const ctx = canvas.getContext('2d');
      ctx.drawImage(video, 0, 0, vw, vh);
      status.textContent = '识别中…';
      try {
        const predictions = await model.classify(canvas, 3);
        showPredictions(predictions);
        status.textContent = '识别完成';
        if (predictions.length > 0) {
          lastTopLabel = normalizeLabel(predictions[0].className);
          speakText(lastTopLabel);
        }
      } catch (err) {
        status.textContent = '识别失败：' + err.message;
      }
    }

    function showPredictions(predictions) {
      predList.innerHTML = '';
      if (!predictions || predictions.length === 0) {
        predList.innerHTML = '<div class="small">没有识别到结果</div>';
        return;
      }
      predictions.forEach(pred => {
        const div = document.createElement('div');
        div.className = 'pred';
        const labelDiv = document.createElement('div');
        labelDiv.className = 'label';
        labelDiv.textContent = normalizeLabel(pred.className);
        const probDiv = document.createElement('div');
        probDiv.className = 'prob';
        probDiv.textContent = (pred.probability*100).toFixed(1) + '%';
        div.appendChild(labelDiv);
        div.appendChild(probDiv);
        predList.appendChild(div);
      });
    }

    function normalizeLabel(raw) {
      if (!raw) return '';
      let main = raw.split(',')[0].split(' with ')[0].split(' and ')[0].trim();
      return main.split(' ').map(s => s.charAt(0).toUpperCase() + s.slice(1)).join(' ');
    }

    function speakText(text) {
      if (!text) return;
      if (!('speechSynthesis' in window)) {
        alert('当前浏览器不支持语音合成');
        return;
      }
      const utter = new SpeechSynthesisUtterance(text);
      utter.lang = 'en-US';
      const voices = window.speechSynthesis.getVoices();
      if (voices && voices.length) {
        const candidate = voices.find(v => /en-?us/i.test(v.lang)) || voices.find(v => /en-?gb/i.test(v.lang)) || voices[0];
        if (candidate) utter.voice = candidate;
      }
      utter.rate = 0.95;
      utter.pitch = 1.0;
      window.speechSynthesis.cancel();
      window.speechSynthesis.speak(utter);
    }

    btnSpeak.addEventListener('click', () => {
      if (!lastTopLabel) { alert('还没有识别结果，先拍张照吧'); return; }
      speakText(lastTopLabel);
    });

    btnCapture.addEventListener('click', captureAndClassify);
    btnToggleTorch.addEventListener('click', toggleTorch);

    await startCamera();
    await new Promise(resolve => { if (video.readyState >= 2) return resolve(); video.onloadedmetadata = () => resolve(); });
    window.speechSynthesis.getVoices();
    await loadModel();
  })();
  </script>
</body>
</html>
