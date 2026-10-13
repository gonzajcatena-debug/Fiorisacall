<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Fiorisacall</title>
<style>
:root{--bg:#f6f7fb;--card:#fff;--text:#14171f;--muted:#6b7280;--accent:#4f46e5;--rec:#e11d48;--ok:#059669;--line:#e5e7eb;
box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px)}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]){--bg:#0f1117;--card:#1a1d27;--text:#f3f4f6;--muted:#9ca3af;--accent:#818cf8;--rec:#fb7185;--ok:#34d399;--line:#2a2e3b;color-scheme:dark}}
:root[data-theme="dark"]{--bg:#0f1117;--card:#1a1d27;--text:#f3f4f6;--muted:#9ca3af;--accent:#818cf8;--rec:#fb7185;--ok:#34d399;--line:#2a2e3b;color-scheme:dark}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--text);font-family:system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;min-height:100vh;display:flex;justify-content:center}
main{width:100%;max-width:440px;padding:28px 20px;display:flex;flex-direction:column;gap:18px}
h1{margin:0;font-size:26px;letter-spacing:-.5px}
h1 span{color:var(--accent)}
.sub{color:var(--muted);font-size:14px;margin-top:4px}
.card{background:var(--card);border:1px solid var(--line);border-radius:16px;padding:16px}
.label{font-size:12px;text-transform:uppercase;letter-spacing:.8px;color:var(--muted);margin-bottom:6px}
.txt{min-height:48px;font-size:17px;line-height:1.4;word-break:break-word}
.txt.empty{color:var(--muted);font-size:15px}
#main{width:100%;border:0;border-radius:16px;padding:20px;font-size:19px;font-weight:600;color:#fff;background:var(--accent);cursor:pointer}
#main.rec{background:var(--rec);animation:pulse 1.2s infinite}
#main.play{background:var(--ok)}
#main:disabled{opacity:.6;cursor:default}
@keyframes pulse{50%{opacity:.75}}
#again{background:none;border:1px solid var(--line);color:var(--text);border-radius:12px;padding:12px;font-size:15px;cursor:pointer;display:none}
#status{font-size:14px;color:var(--muted);text-align:center;min-height:20px}
#status.err{color:var(--rec)}
</style>
</head>
<body>
<main>
  <div>
    <h1>Fiori<span>sacall</span></h1>
    <div class="sub">Hablá en español, se escucha en inglés.</div>
  </div>

  <div class="card">
    <div class="label">Lo que dijiste</div>
    <div id="es" class="txt empty">Tu voz transcripta aparecerá acá…</div>
  </div>

  <div class="card">
    <div class="label">En inglés</div>
    <div id="en" class="txt empty">La traducción aparecerá acá…</div>
  </div>

  <button id="main">Iniciar grabación</button>
  <button id="again">Grabar de nuevo</button>
  <div id="status"></div>
</main>

<script>
(function(){
  var SR = window.SpeechRecognition || window.webkitSpeechRecognition;
  var btn = document.getElementById('main'), again = document.getElementById('again');
  var esEl = document.getElementById('es'), enEl = document.getElementById('en'), st = document.getElementById('status');
  var lastErr = '', state = 'idle', rec = null, lastInterim = '', finalText = '', englishText = '', voices = [];

  function setStatus(m, err){ st.textContent = m || ''; st.className = err ? 'err' : ''; }
  function show(el, t, ph){ el.textContent = t || ph; el.className = 'txt' + (t ? '' : ' empty'); }

  function loadVoices(){ try { voices = speechSynthesis.getVoices(); } catch(e){} }
  try { loadVoices(); speechSynthesis.onvoiceschanged = loadVoices; } catch(e){}

  if (!SR) {
    btn.disabled = true;
    setStatus('Este navegador no soporta reconocimiento de voz. Probalo en Chrome o Safari.', true);
  }

  function setUI(s){
    state = s;
    btn.disabled = (s === 'working');
    btn.className = s === 'recording' ? 'rec' : (s === 'ready' ? 'play' : '');
    btn.id = 'main';
    btn.textContent = { idle:'Iniciar grabación', recording:'Terminar', working:'Traduciendo…', ready:'▶ Reproducir' }[s];
    again.style.display = s === 'ready' ? 'block' : 'none';
  }

  function friendly(c){
    var m = {
      'not-allowed':'El navegador bloqueó el micrófono dentro de esta página.',
      'service-not-allowed':'El navegador no permite el reconocimiento de voz en esta página.',
      'audio-capture':'No se detecta ningún micrófono.',
      'network':'El reconocimiento de voz no pudo conectarse a internet.',
      'language-not-supported':'El idioma no está soportado en este navegador.'
    };
    return (m[c] || 'Error de reconocimiento.') + ' (código: ' + c + ')';
  }

  function start(){
    finalText = ''; lastInterim = ''; englishText = ''; lastErr = '';
    show(esEl, '', 'Escuchando…'); show(enEl, '', 'La traducción aparecerá acá…');
    rec = new SR();
    rec.lang = 'es-AR'; rec.continuous = true; rec.interimResults = true;
    rec.onresult = function(e){
      var interim = '';
      for (var i = e.resultIndex; i < e.results.length; i++) {
        var t = e.results[i][0].transcript;
        if (e.results[i].isFinal) finalText += t + ' '; else interim += t;
      }
      lastInterim = interim;
      show(esEl, (finalText + interim).trim(), 'Escuchando…');
    };
    rec.onaudiostart = function(){ setStatus('Micrófono activo. Hablá ahora…'); };
    rec.onspeechstart = function(){ setStatus('Te escucho…'); };
    rec.onerror = function(e){
      if (e.error === 'no-speech' || e.error === 'aborted') return;
      lastErr = e.error;
      setStatus(friendly(e.error), true);
    };
    rec.onend = function(){ if (state === 'recording' || state === 'working') finish(); };
    try { rec.start(); setUI('recording'); setStatus('Grabando… hablá en español.'); }
    catch(e){ setStatus('No se pudo iniciar el micrófono.', true); }
  }

  function stop(){
    setUI('working'); setStatus('');
    try { rec.stop(); } catch(e){ finish(); return; }
    setTimeout(function(){ if (state === 'working') finish(); }, 2500);
  }

  var finishing = false;
  function finish(){
    if (finishing) return; finishing = true;
    setUI('working');
    var text = (finalText + ' ' + lastInterim).trim();
    if (!text) { finishing = false; setUI('idle'); setStatus(lastErr ? friendly(lastErr) : 'No llegó audio del micrófono. Revisá el permiso del micrófono y probá de nuevo.', true); return; }
    show(esEl, text, '');
    translate(text).then(function(en){
      englishText = en; show(enEl, en, '');
      finishing = false; setUI('ready'); setStatus('Listo. Tocá reproducir.');
    }).catch(function(err){
      finishing = false; setUI('idle');
      setStatus('No se pudo traducir' + (err && err.message ? ': ' + err.message : '.'), true);
    });
  }

  function translate(text){
    var url = 'https://api.mymemory.translated.net/get?langpair=es|en&q=' + encodeURIComponent(text);
    return fetch(url).then(function(r){
      if (!r.ok) throw new Error('el servicio de traducción respondió ' + r.status);
      return r.json();
    }).then(function(d){
      var t = d && d.responseData && d.responseData.translatedText;
      if (!t) throw new Error('respuesta vacía del traductor');
      return t.trim();
    });
  }

  function speak(){
    if (!englishText) return;
    try {
      speechSynthesis.cancel();
      var u = new SpeechSynthesisUtterance(englishText);
      u.lang = 'en-US';
      var v = voices.find(function(x){ return /^en[-_]US/i.test(x.lang); }) || voices.find(function(x){ return /^en/i.test(x.lang); });
      if (v) u.voice = v;
      u.rate = 1;
      u.onstart = function(){ setStatus('Reproduciendo…'); };
      u.onend = function(){ setStatus('Listo.'); };
      speechSynthesis.speak(u);
    } catch(e){ setStatus('Este navegador no puede reproducir voz.', true); }
  }

  btn.addEventListener('click', function(){
    if (state === 'idle') start();
    else if (state === 'recording') stop();
    else if (state === 'ready') speak();
  });
  again.addEventListener('click', function(){ try{speechSynthesis.cancel();}catch(e){} show(esEl,'','Tu voz transcripta aparecerá acá…'); show(enEl,'','La traducción aparecerá acá…'); setUI('idle'); setStatus(''); });

  setUI('idle');
})();
</script>
</body>
</html>
