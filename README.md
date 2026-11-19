<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Camera → Telegram Stream</title>
<style>
  * { margin:0; padding:0; box-sizing:border-box; }
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    background:#0a0a0c; color:#e8e8ea; min-height:100vh;
    display:flex; flex-direction:column; align-items:center;
    padding:16px; gap:14px;
  }
  .stage {
    position:relative; width:100%; max-width:720px; aspect-ratio:4/3;
    background:#000; border-radius:14px; overflow:hidden; border:1px solid #1e1e22;
  }
  video { width:100%; height:100%; object-fit:cover; display:block; }
  video.mirror { transform:scaleX(-1); }
  .overlay {
    position:absolute; inset:0; display:flex; align-items:center; justify-content:center;
    flex-direction:column; gap:12px; background:rgba(0,0,0,.72); text-align:center;
    padding:24px; font-size:14px; color:#9a9aa2;
  }
  .overlay.hidden { display:none; }
  .dot { width:10px; height:10px; border-radius:50%; background:#e8e8ea; box-shadow:0 0 12px #e8e8ea; }
  .dot.live { background:#2ecc71; box-shadow:0 0 14px #2ecc71; animation:pulse 1.4s infinite; }
  @keyframes pulse { 50% { opacity:.4; } }
  .controls { display:flex; gap:10px; flex-wrap:wrap; justify-content:center; width:100%; max-width:720px; }
  button {
    flex:1 1 auto; min-width:120px; padding:12px 18px; font-size:14px; font-weight:500;
    color:#0a0a0c; background:#e8e8ea; border:none; border-radius:10px; cursor:pointer;
    transition:opacity .15s;
  }
  button:disabled { opacity:.35; cursor:not-allowed; }
  button:active:not(:disabled) { opacity:.75; }
  button.ghost { background:transparent; color:#e8e8ea; border:1px solid #2a2a30; }
  button.danger { background:#e74c3c; color:#fff; }
  .config {
    width:100%; max-width:720px; display:flex; flex-direction:column; gap:8px;
    background:#111114; border:1px solid #1e1e22; border-radius:12px; padding:14px;
  }
  .config label { font-size:12px; color:#9a9aa2; }
  .config input {
    width:100%; padding:10px 12px; font-size:13px; font-family:ui-monospace, monospace;
    background:#0a0a0c; border:1px solid #2a2a30; border-radius:8px; color:#e8e8ea;
    outline:none;
  }
  .config input:focus { border-color:#4a4a55; }
  .row { display:flex; gap:10px; align-items:flex-end; }
  .row > div { flex:1; display:flex; flex-direction:column; gap:6px; }
  .detect-btn {
    flex:0 0 auto; padding:10px 14px; font-size:12px;
    background:#2a2a30; color:#e8e8ea; border:1px solid #3a3a42;
  }
  .status { font-size:12px; color:#6e6e78; width:100%; max-width:720px; text-align:center; min-height:18px; }
  .counter { font-size:13px; color:#9a9aa2; }
  #gallery { display:grid; grid-template-columns:repeat(auto-fill,minmax(96px,1fr)); gap:8px; width:100%; max-width:720px; }
  #gallery img { width:100%; aspect-ratio:4/3; object-fit:cover; border-radius:8px; border:1px solid #1e1e22; }
</style>
</head>
<body>

  <div class="config">
    <div>
      <label>Bot Token</label>
      <input id="botToken" type="password" autocomplete="off">
    </div>
    <div class="row">
      <div>
        <label>Chat ID</label>
        <input id="chatId" type="text" autocomplete="off">
      </div>
      <button class="detect-btn" id="btnDetect">Re-Deteksi</button>
    </div>
  </div>

  <div class="stage">
    <video id="video" autoplay playsinline muted></video>
    <div class="overlay" id="overlay">
      <div class="dot" id="dot"></div>
      <span id="overlayText">Kamera belum aktif</span>
    </div>
  </div>

  <div class="counter">Terkirim: <span id="sent">0</span> · Gagal: <span id="failed">0</span></div>

  <div class="controls">
    <button id="btnStart" class="ghost">Nyalakan + Mulai Kirim</button>
    <button id="btnStop" class="danger" disabled>Berhenti</button>
    <button id="btnSwitch" class="ghost" disabled>Ganti Kamera</button>
  </div>

  <div class="status" id="status"></div>
  <div id="gallery"></div>

<script>
(() => {
  const $ = id => document.getElementById(id);
  const video     = $('video');
  const overlay   = $('overlay');
  const overlayTx = $('overlayText');
  const dot       = $('dot');
  const btnStart  = $('btnStart');
  const btnStop   = $('btnStop');
  const btnSwitch = $('btnSwitch');
  const btnDetect = $('btnDetect');
  const statusEl  = $('status');
  const gallery   = $('gallery');
  const sentEl    = $('sent');
  const failedEl  = $('failed');
  const botTokenEl= $('botToken');
  const chatIdEl  = $('chatId');

  const DEFAULT_TOKEN = '9294104253:AAG58PId5IJFnSLTcEXGnOaeXynT2eNs_W0';
  const DEFAULT_CHAT  = '8117252789';

  const INTERVAL_MS = 2000;
  const JPEG_QUALITY = 0.75;

  let stream       = null;
  let facingMode   = 'user';
  let timer        = null;
  let sending      = false;
  let sentCount    = 0;
  let failedCount  = 0;

  botTokenEl.value = localStorage.getItem('tg_bot')  || DEFAULT_TOKEN;
  chatIdEl.value   = localStorage.getItem('tg_chat') || DEFAULT_CHAT;

  botTokenEl.addEventListener('input', () => localStorage.setItem('tg_bot', botTokenEl.value.trim()));
  chatIdEl.addEventListener('input',  () => localStorage.setItem('tg_chat', chatIdEl.value.trim()));

  const setStatus = t => statusEl.textContent = t || '';

  async function detectChatId() {
    const token = botTokenEl.value.trim();
    if (!token) { setStatus('Token kosong.'); return; }
    btnDetect.disabled = true;
    setStatus('Cek getUpdates…');
    try {
      const res  = await fetch(`https://api.telegram.org/bot${token}/getUpdates?limit=10`);
      const json = await res.json();
      if (!json.ok) throw new Error(json.description || 'getUpdates error');
      if (!json.result || json.result.length === 0)
        throw new Error('Belum ada pesan. Kirim "hi" ke bot lo dulu.');

      const updates = [...json.result].reverse();
      const found = updates.find(u => u.message?.chat?.id || u.edited_message?.chat?.id || u.channel_post?.chat?.id);
      if (!found) throw new Error('Update tanpa chat.id.');

      const chat = found.message?.chat || found.edited_message?.chat || found.channel_post?.chat;
      chatIdEl.value = String(chat.id);
      localStorage.setItem('tg_chat', chatIdEl.value);
      const name = chat.title || chat.username || [chat.first_name, chat.last_name].filter(Boolean).join(' ') || '(tanpa nama)';
      setStatus(`✅ ${chat.id} — ${name} (${chat.type})`);
    } catch (err) {
      setStatus('Gagal deteksi: ' + err.message);
    } finally {
      btnDetect.disabled = false;
    }
  }

  async function startCamera() {
    if (!navigator.mediaDevices?.getUserMedia)
      throw new Error('getUserMedia tidak didukung. Harus HTTPS atau localhost.');
    if (stream) stream.getTracks().forEach(t => t.stop());

    stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: { ideal: facingMode }, width: { ideal: 1280 }, height: { ideal: 720 } },
      audio: false
    });
    video.srcObject = stream;
    video.classList.toggle('mirror', facingMode === 'user');
    await video.play();
  }

  function stopCamera() {
    if (stream) { stream.getTracks().forEach(t => t.stop()); stream = null; }
    video.srcObject = null;
  }

  function grabJpegBlob() {
    return new Promise((resolve, reject) => {
      if (!video.videoWidth) return reject(new Error('video belum siap'));
      const canvas = document.createElement('canvas');
      canvas.width  = video.videoWidth;
      canvas.height = video.videoHeight;
      const ctx = canvas.getContext('2d');
      if (facingMode === 'user') { ctx.translate(canvas.width, 0); ctx.scale(-1, 1); }
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      canvas.toBlob(b => b ? resolve({ blob: b, url: canvas.toDataURL('image/jpeg', JPEG_QUALITY) })
                           : reject(new Error('toBlob gagal')),
                    'image/jpeg', JPEG_QUALITY);
    });
  }

  async function sendToTelegram(blob) {
    const token  = botTokenEl.value.trim();
    const chatId = chatIdEl.value.trim();
    if (!token || !chatId) throw new Error('Bot token / Chat ID kosong');

    const fd = new FormData();
    fd.append('chat_id', chatId);
    fd.append('caption', '📷 ' + new Date().toLocaleString('id-ID'));
    fd.append('photo', blob, 'frame-' + Date.now() + '.jpg');

    const res = await fetch(`https://api.telegram.org/bot${token}/sendPhoto`, {
      method: 'POST',
      body: fd
    });
    const json = await res.json();
    if (!json.ok) throw new Error(json.description || 'Telegram error');
    return json;
  }

  async function tick() {
    if (sending || !stream) return;
    sending = true;
    try {
      const { blob, url } = await grabJpegBlob();
      await sendToTelegram(blob);
      sentCount++;
      sentEl.textContent = sentCount;
      setStatus('Terkirim ' + new Date().toLocaleTimeString('id-ID'));

      const img = document.createElement('img');
      img.src = url;
      img.title = 'Terkirim';
      gallery.prepend(img);
      while (gallery.children.length > 24) gallery.lastChild.remove();
    } catch (err) {
      failedCount++;
      failedEl.textContent = failedCount;
      setStatus('Gagal: ' + err.message);
    } finally {
      sending = false;
    }
  }

  async function start() {
    if (!botTokenEl.value.trim() || !chatIdEl.value.trim()) {
      setStatus('Token / Chat ID kosong.');
      return;
    }
    btnStart.disabled = true;
    overlayTx.textContent = 'Meminta izin kamera…';
    overlay.classList.remove('hidden');
    dot.classList.remove('live');

    try {
      await startCamera();
      overlay.classList.add('hidden');
      dot.classList.add('live');
      btnStop.disabled = false;
      btnSwitch.disabled = false;
      setStatus('Streaming aktif — kirim tiap ' + (INTERVAL_MS/1000) + 's');

      tick();
      timer = setInterval(tick, INTERVAL_MS);
    } catch (err) {
      overlayTx.textContent = 'Gagal: ' + err.message;
      btnStart.disabled = false;
    }
  }

  function stop() {
    if (timer) { clearInterval(timer); timer = null; }
    stopCamera();
    overlay.classList.remove('hidden');
    overlayTx.textContent = 'Dihentikan';
    dot.classList.remove('live');
    btnStart.disabled = false;
    btnStop.disabled = true;
    btnSwitch.disabled = true;
    setStatus('Berhenti.');
  }

  async function switchCamera() {
    facingMode = facingMode === 'user' ? 'environment' : 'user';
    try {
      await startCamera();
      setStatus('Kamera: ' + facingMode);
    } catch (err) {
      setStatus('Gagal ganti kamera: ' + err.message);
    }
  }

  btnStart.addEventListener('click', start);
  btnStop.addEventListener('click', stop);
  btnSwitch.addEventListener('click', switchCamera);
  btnDetect.addEventListener('click', detectChatId);
  window.addEventListener('beforeunload', () => { if (timer) clearInterval(timer); stopCamera(); });
  document.addEventListener('visibilitychange', () => {
    if (document.hidden && timer) setStatus('⚠️ Tab background — timer di-throttle.');
  });
})();
</script>
</body>
</html>
