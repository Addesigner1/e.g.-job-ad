<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Find Work Now — Code Video Ad (Recordable)</title>
<style>
  :root { --w:1280; --h:720; }
  * { box-sizing: border-box; }
  html, body { height:100%; margin:0; background:#0b0f1a; color:#fff; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; }
  .wrap { display:grid; place-items:center; min-height:100%; padding:16px; }
  .shell { width:min(100%, 1000px); }
  .stage { position:relative; aspect-ratio:16/9; border-radius:18px; overflow:hidden; border:1px solid rgba(255,255,255,.12); box-shadow: 0 30px 80px rgba(0,0,0,.6); background:#0b0f1a; }
  canvas { width:100%; height:100%; display:block; }
  .controls { display:flex; flex-wrap:wrap; gap:8px; margin-top:12px; }
  .btn { appearance:none; border:1px solid rgba(255,255,255,.2); background:rgba(255,255,255,.06); color:#fff; padding:.75em 1em; border-radius:12px; cursor:pointer; }
  .btn.primary { border:none; background: linear-gradient(180deg,#00ffc8,#00d6a6); color:#062b24; font-weight:800; }
  .row { display:flex; align-items:center; gap:10px; flex-wrap:wrap; }
  .note { font-size:12px; opacity:.8; margin-top:6px; }
  .link { color:#00ffc8; text-decoration:none; }
  .chip { font-size:12px; padding:.25em .6em; border:1px solid rgba(255,255,255,.2); border-radius:999px; opacity:.8; }
</style>
</head>
<body>
  <div class="wrap">
    <div class="shell">
      <div class="stage" aria-label="Recordable promotional animation">
        <canvas id="c" width="1280" height="720"></canvas>
      </div>

      <div class="controls" aria-label="Playback & Recording">
        <button class="btn" id="play">⏵ Play</button>
        <button class="btn" id="pause">⏸ Pause</button>
        <button class="btn" id="replay">↺ Replay</button>
        <span class="chip" id="time">00.0s</span>

        <button class="btn primary" id="rec">● Start Recording</button>
        <button class="btn" id="stop" disabled>■ Stop & Save</button>
        <label class="row"><input type="checkbox" id="useMic" checked /> Use microphone in recording</label>
        <button class="btn" id="voice">🔊 Voiceover</button>
        <a class="btn" id="cta" href="https://example.com/jobs" target="_blank" rel="noopener">Visit: Find Work Now</a>
      </div>

      <div class="note">
        • This file draws everything on a <code>&lt;canvas&gt;</code> and records it with <code>MediaRecorder</code>.
        • If you enable the mic, your narration is captured. Built-in TTS is for preview and usually won’t go into the recording.
      </div>
      <div id="out" class="note"></div>
    </div>
  </div>

<script>
/* ============================
   Config (edit text & styling)
   ============================ */
const CONFIG = {
  width: 1280,
  height: 720,
  fps: 30,
  scenes: [
    { dur: 2200, kicker: "No Experience? No Problem.", title: "Find Work Fast", accent: true, sub: "Flexible gigs • Full-time • Remote & local" },
    { dur: 2200, kicker: "Step 1", title: "Tell us what you’re good at", sub: "Pick skills—construction, driving, retail, admin, more." },
    { dur: 2200, kicker: "Step 2", title: "Match with real openings", sub: "Roles paying today, near you or remote." },
    { dur: 2400, kicker: "Step 3", title: "Apply in minutes", sub: "One simple profile. Interview faster." },
    { dur: 2600, kicker: "Ready?", title: "Start earning this week", sub: "Tap “Find Work Now” — jobs are waiting." },
  ],
  ctaText: "Find Work Now",
  brandless: true
};

/* ============================
   Helpers
   ============================ */
const $ = sel => document.querySelector(sel);
const lerp = (a,b,t)=> a+(b-a)*t;
const clamp = (v,lo,hi)=> Math.max(lo, Math.min(hi, v));
const ease = t => t<.5 ? 4*t*t*t : 1-Math.pow(-2*t+2,3)/2; // cubic in/out
const fmt = ms => (ms/1000).toFixed(1).padStart(4,' ');

/* ============================
   Canvas & Timeline
   ============================ */
const canvas = $('#c');
const ctx = canvas.getContext('2d');
const DPR = window.devicePixelRatio || 1;
function resizeCanvas() {
  const {width:w, height:h} = CONFIG;
  canvas.width = w * DPR;
  canvas.height = h * DPR;
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
}
resizeCanvas(); window.addEventListener('resize', resizeCanvas);

const durations = CONFIG.scenes.map(s=>s.dur);
const total = durations.reduce((a,b)=>a+b,0);

let playing = false;
let t = 0;           // elapsed ms
let last = 0;        // last raf time
let rafId = 0;

function sceneIndexAt(timeMs) {
  let acc = 0;
  for (let i=0;i<durations.length;i++){
    acc += durations[i];
    if (timeMs < acc) return i;
  }
  return durations.length-1;
}

function timeIntoScene(timeMs){
  let acc = 0;
  for (let i=0;i<durations.length;i++){
    const d = durations[i];
    if (timeMs < acc + d) return {i, t: timeMs - acc, d};
    acc += d;
  }
  const i = durations.length-1;
  const d = durations[i];
  return {i, t:d, d};
}

/* ============================
   Visuals (draw everything)
   ============================ */
function drawBackground(ms){
  const {width:w, height:h} = CONFIG;
  // Animated hue-shifted gradient
  const hue = (ms*0.02)%360;
  const g1 = ctx.createLinearGradient(0,0,w,h);
  g1.addColorStop(0, `hsl(${(hue+200)%360},60%,12%)`);
  g1.addColorStop(1, `hsl(${(hue+240)%360},60%,18%)`);
  ctx.fillStyle = g1;
  ctx.fillRect(0,0,w,h);

  // Soft orbs
  function orb(x,y,r,c1,c2,alpha=0.6){
    const g = ctx.createRadialGradient(x,y, r*0.1, x,y,r);
    g.addColorStop(0, c1);
    g.addColorStop(1, c2);
    ctx.globalAlpha = alpha;
    ctx.fillStyle = g;
    ctx.beginPath();
    ctx.arc(x,y,r,0,Math.PI*2);
    ctx.fill();
    ctx.globalAlpha = 1;
  }
  const ox = Math.sin(ms*0.0005)*40;
  const oy = Math.cos(ms*0.0007)*30;
  orb(220+ox, 180+oy, 190, 'rgba(110,163,255,.8)', 'rgba(110,163,255,0)');
  orb(w-180-ox, h-140-oy, 160, 'rgba(0,255,200,.8)', 'rgba(0,255,200,0)');
  // faint scanlines
  ctx.globalAlpha = .08;
  for (let y=0;y<h;y+=3){ ctx.fillStyle='white'; ctx.fillRect(0,y,w,1); }
  ctx.globalAlpha = 1;
}

function drawText(kicker, title, sub, alpha, accent=false){
  const {width:w, height:h} = CONFIG;
  ctx.save();
  ctx.globalAlpha = alpha;

  // Kicker
  ctx.font = "600 18px system-ui, -apple-system, Segoe UI, Roboto, Arial";
  ctx.textAlign = 'center';
  ctx.fillStyle = 'rgba(255,255,255,.9)';
  ctx.fillText(kicker, w/2, h*0.36);

  // Title
  ctx.font = "900 64px system-ui, -apple-system, Segoe UI, Roboto, Arial";
  ctx.textBaseline = 'top';
  const titleY = h*0.36 + 20;
  if (accent){
    // Split to color the word "Fast"
    const parts = title.split(' ');
    const fastAt = parts.findIndex(p=>/fast/i.test(p));
    const tBefore = parts.slice(0,fastAt).join(' ') + (fastAt>0?' ':'');
    const fast = fastAt>=0 ? parts[fastAt] : '';
    const tAfter = parts.slice(fastAt+1).join(' ');
    const measure = txt => ctx.measureText(txt).width;

    const x = w/2;
    const widthBefore = measure(tBefore);
    const widthFast = measure(fast + (tAfter?' ':''));
    // Draw before
    ctx.fillStyle = '#ffffff';
    ctx.fillText(tBefore, x - (widthBefore + widthFast)/2, titleY);
    // Accent "Fast"
    const grad = ctx.createLinearGradient(x, titleY, x, titleY+64);
    grad.addColorStop(0,'#00ffc8'); grad.addColorStop(1,'#00d6a6');
    ctx.fillStyle = grad;
    ctx.fillText(fast + (tAfter?' ':''), x - (widthFast)/2 + (widthBefore - widthFast)/2, titleY);
    // After
    ctx.fillStyle = '#ffffff';
    ctx.fillText(tAfter, x + (widthFast)/2 + (tAfter?0:0), titleY);
  } else {
    ctx.fillStyle = '#ffffff';
    ctx.fillText(title, w/2, titleY);
  }

  // Sub
  ctx.font = "500 24px system-ui, -apple-system, Segoe UI, Roboto, Arial";
  ctx.fillStyle = 'rgba(255,255,255,.9)';
  ctx.fillText(sub, w/2, titleY + 80);

  // CTA badge (visual only, part of recording)
  const btnW = 320, btnH = 56, rx = 14;
  const bx = w/2 - btnW/2, by = titleY + 130;
  const grad = ctx.createLinearGradient(bx, by, bx, by+btnH);
  grad.addColorStop(0,'#00ffc8'); grad.addColorStop(1,'#00d6a6');
  ctx.fillStyle = grad;
  roundRect(ctx, bx, by, btnW, btnH, rx, true, false);
  ctx.fillStyle = '#062b24';
  ctx.font = "800 22px system-ui, -apple-system, Segoe UI, Roboto, Arial";
  ctx.fillText(CONFIG.ctaText + "  ➜", w/2, by + btnH/2 - 12);
  ctx.restore();
}

function roundRect(ctx,x,y,w,h,r,fill,stroke){
  if (r> w/2) r=w/2; if (r>h/2) r=h/2;
  ctx.beginPath();
  ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  ctx.closePath();
  if (fill) ctx.fill();
  if (stroke) ctx.stroke();
}

function drawProgress(time){
  const {width:w, height:h} = CONFIG;
  const p = clamp(time/total,0,1);
  const barW = w*0.7, barH = 8;
  const x = (w-barW)/2, y = h-40;
  ctx.globalAlpha = .9;
  ctx.fillStyle = 'rgba(255,255,255,.18)';
  roundRect(ctx, x, y, barW, barH, 999, true, false);
  const g = ctx.createLinearGradient(x,y,x+barW,y);
  g.addColorStop(0,'#6aa3ff'); g.addColorStop(1,'#00ffc8');
  ctx.fillStyle = g;
  roundRect(ctx, x, y, barW*p, barH, 999, true, false);
  ctx.globalAlpha = 1;
}

function render(now){
  if (!last) last = now;
  const dt = now - last; last = now;
  if (playing) t += dt;

  // Loop
  if (t > total) { playing = false; }

  // Draw
  drawBackground(t);
  const {i, t:ts, d} = timeIntoScene(t);
  const sc = CONFIG.scenes[i];
  // fade in/out 350ms
  const fade = 350;
  const aIn = clamp(ts/fade, 0, 1);
  const aOut = clamp((d - ts)/fade, 0, 1);
  const alpha = Math.min(aIn, aOut);
  drawText(sc.kicker, sc.title, sc.sub, alpha, !!sc.accent);
  drawProgress(t);

  // overlay counter (tiny, top-left)
  ctx.globalAlpha = .8;
  ctx.font = "600 14px system-ui,-apple-system,Segoe UI,Roboto,Arial";
  ctx.fillStyle = "rgba(255,255,255,.9)";
  ctx.fillText(`Time: ${fmt(t)} / ${(total/1000).toFixed(1)}s`, 16, 22);
  ctx.globalAlpha = 1;

  $('#time').textContent = `${(t/1000).toFixed(1)}s`;

  rafId = requestAnimationFrame(render);
}

/* ============================
   Controls
   ============================ */
$('#play').onclick = ()=> { playing = true; if (!rafId) rafId = requestAnimationFrame(render); };
$('#pause').onclick = ()=> { playing = false; };
$('#replay').onclick = ()=> { t = 0; playing = true; if (!rafId) rafId = requestAnimationFrame(render); };

/* ============================
   Voiceover (preview only)
   ============================ */
const VO_LINES = [
  "Find work fast. No experience? No problem.",
  "Step one: tell us what you're good at.",
  "Step two: match with real openings near you, or remote.",
  "Step three: apply in minutes.",
  "Start earning this week. Tap Find Work Now."
];
$('#voice').onclick = () => {
  if (!('speechSynthesis' in window)) { alert('Voiceover not supported.'); return; }
  speechSynthesis.cancel();
  const pickVoice = () => {
    const voices = speechSynthesis.getVoices()||[];
    const maleHints = [/Male/i,/David/i,/Daniel/i,/George/i,/Alex\b/i,/Guy\b/i];
    return voices.find(v=>/en/i.test(v.lang) && maleHints.some(r=>r.test(v.name)))
        || voices.find(v=>/en/i.test(v.lang));
  };
  const v = pickVoice();
  VO_LINES.forEach((line, idx)=>{
    const u = new SpeechSynthesisUtterance(line);
    if (v) u.voice = v;
    u.rate = 1.02; u.pitch = 0.92; u.volume = 1;
    setTimeout(()=>speechSynthesis.speak(u), idx*60);
  });
};

/* ============================
   Recording (MediaRecorder)
   ============================ */
let recorder = null;
let chunks = [];
let micStream = null;

async function startRecording(){
  // Canvas frames
  const canvasStream = canvas.captureStream(CONFIG.fps);

  // Optional mic (for your narration)
  let mixedStream = canvasStream;
  if ($('#useMic').checked && navigator.mediaDevices && navigator.mediaDevices.getUserMedia){
    try {
      micStream = await navigator.mediaDevices.getUserMedia({audio:true});
      mixedStream = new MediaStream([
        ...canvasStream.getVideoTracks(),
        ...micStream.getAudioTracks()
      ]);
    } catch (e) {
      console.warn('Mic access denied or failed:', e);
    }
  }

  // Pick codec
  let mime = '';
  if (MediaRecorder.isTypeSupported('video/webm;codecs=vp9,opus')) {
    mime = 'video/webm;codecs=vp9,opus';
  } else if (MediaRecorder.isTypeSupported('video/webm;codecs=vp8,opus')) {
    mime = 'video/webm;codecs=vp8,opus';
  } else {
    mime = 'video/webm';
  }

  chunks = [];
  recorder = new MediaRecorder(mixedStream, { mimeType: mime, videoBitsPerSecond: 5_000_000 });
  recorder.ondataavailable = e => { if (e.data && e.data.size) chunks.push(e.data); };
  recorder.onstop = () => {
    if (micStream) micStream.getTracks().forEach(t=>t.stop());
    const blob = new Blob(chunks, { type: recorder.mimeType || 'video/webm' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = 'find-work-now.webm';
    a.textContent = '⬇️ Download .webm';
    a.className = 'link';
    const open = document.createElement('a');
    open.href = url; open.target = '_blank'; open.rel='noopener';
    open.textContent = 'Open in new tab'; open.className='link'; open.style.marginLeft='12px';

    $('#out').innerHTML = 'Saved recording: ';
    $('#out').appendChild(a);
    $('#out').appendChild(open);
  };
  recorder.start();
  $('#rec').disabled = true; $('#stop').disabled = false;
}

function stopRecording(){
  if (recorder && recorder.state !== 'inactive') recorder.stop();
  $('#rec').disabled = false; $('#stop').disabled = true;
}

$('#rec').onclick = startRecording;
$('#stop').onclick = stopRecording;

// Autostart render loop for first paint
rafId = requestAnimationFrame(render);
</script>
</body>
</html>
