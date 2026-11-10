<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'
import { QUESTIONS } from './questions.js'

document.documentElement.classList.add('js')

const MAX_ERR = 5
const screen = ref('landing') // landing | play | won | lost
const deck = ref([]), idx = ref(0), sel = ref([]), answered = ref(false), lastOk = ref(false)
const correct = ref(0), errors = ref(0), streak = ref(0), maxStreak = ref(0)
const best = ref(Number(localStorage.getItem('quiz-best') || 0))
const muted = ref(false), missed = ref([])
const cur = computed(() => deck.value[idx.value])
const isMulti = computed(() => cur.value && cur.value.a.length > 1)
const pct = computed(() => (deck.value.length ? Math.round((idx.value / deck.value.length) * 100) : 0))

let ac
function tone(f, d, type = 'sine', delay = 0, v = 0.15) {
  if (muted.value) return
  ac ??= new (window.AudioContext || window.webkitAudioContext)()
  const o = ac.createOscillator(), g = ac.createGain(), t = ac.currentTime + delay
  o.type = type; o.frequency.value = f; o.connect(g); g.connect(ac.destination)
  g.gain.setValueAtTime(v, t); g.gain.exponentialRampToValueAtTime(0.001, t + d)
  o.start(t); o.stop(t + d)
}
const sfx = {
  ok: () => { tone(660, 0.12); tone(880, 0.18, 'sine', 0.1) },
  streak: () => [523, 659, 784, 1047].forEach((f, i) => tone(f, 0.15, 'triangle', 0.18 + i * 0.08)),
  bad: () => { tone(220, 0.2, 'sawtooth', 0, 0.1); tone(150, 0.3, 'sawtooth', 0.15, 0.1) },
  win: () => [523, 659, 784, 1047, 1319].forEach((f, i) => tone(f, 0.25, 'triangle', i * 0.12)),
  lose: () => [392, 330, 262, 196].forEach((f, i) => tone(f, 0.3, 'sawtooth', i * 0.18, 0.1)),
  pick: () => tone(440, 0.05, 'square', 0, 0.05),
}

function shuffle(a) {
  a = [...a]
  for (let i = a.length - 1; i > 0; i--) { const j = Math.floor(Math.random() * (i + 1)); [a[i], a[j]] = [a[j], a[i]] }
  return a
}
function prepare(x) {
  const items = x.o.map((t, i) => ({ t, ok: x.a.includes(i) }))
  const isFix = o => /las anteriores/i.test(o.t)
  const all = [...shuffle(items.filter(o => !isFix(o))), ...items.filter(isFix)]
  return { ...x, o: all.map(o => o.t), a: all.flatMap((o, i) => (o.ok ? [i] : [])) }
}
function start() {
  deck.value = shuffle(QUESTIONS).map(prepare); idx.value = 0; sel.value = []; answered.value = false
  correct.value = errors.value = streak.value = maxStreak.value = 0; missed.value = []
  screen.value = 'play'
}
function pick(n) {
  if (answered.value) return
  sfx.pick()
  if (isMulti.value) sel.value = sel.value.includes(n) ? sel.value.filter(x => x !== n) : [...sel.value, n]
  else sel.value = [n]
}
function submit() {
  if (!sel.value.length || answered.value) return
  const a = [...cur.value.a].sort().join(), s = [...sel.value].sort().join()
  lastOk.value = a === s; answered.value = true
  if (lastOk.value) {
    correct.value++; streak.value++
    if (streak.value > maxStreak.value) maxStreak.value = streak.value
    if (streak.value > best.value) { best.value = streak.value; localStorage.setItem('quiz-best', best.value) }
    streak.value % 3 === 0 ? sfx.streak() : sfx.ok()
  } else {
    errors.value++; streak.value = 0; missed.value.push(cur.value); sfx.bad()
  }
}
function next() {
  if (errors.value >= MAX_ERR) { screen.value = 'lost'; sfx.lose(); return }
  if (idx.value + 1 >= deck.value.length) { screen.value = 'won'; sfx.win(); return }
  idx.value++; sel.value = []; answered.value = false
}
function optClass(i) {
  const picked = sel.value.includes(i), right = cur.value.a.includes(i)
  if (!answered.value) return { picked }
  return { right, wrong: picked && !right, missedRight: right && !picked }
}
function onKey(e) {
  if (e.ctrlKey || e.metaKey || e.altKey) return
  const k = e.key.toLowerCase()
  if (k === 'm') { muted.value = !muted.value; return }
  if (k === 'enter' || k === ' ') {
    e.preventDefault()
    if (screen.value !== 'play') start()
    else answered.value ? next() : submit()
    return
  }
  if (screen.value !== 'play' || k.length !== 1) return
  const n = '1234'.includes(k) ? '1234'.indexOf(k) : 'abcd'.indexOf(k)
  if (n >= 0 && n < cur.value.o.length) pick(n)
}
let ob
onMounted(() => {
  window.addEventListener('keydown', onKey)
  ob = new IntersectionObserver((entries) => {
    entries.forEach((e) => { if (e.isIntersecting) { e.target.classList.add('in'); ob.unobserve(e.target); } })
  }, { threshold: 0.12 })
  document.querySelectorAll('.reveal').forEach((el) => ob.observe(el))
  document.querySelectorAll('.navbar, .sentence, .support, .statement, .thumb').forEach((el, i) => {
    el.classList.add('load-in')
    el.style.animationDelay = (i * 90) + 'ms'
  })
  const video = document.querySelector('.motion-video')
  if (video && window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
    video.pause()
    video.removeAttribute('autoplay')
    video.removeAttribute('loop')
  }
})
onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKey)
  if (ob) ob.disconnect()
})
</script>

<template>
  <main>
    <button class="mute" @click="muted = !muted" :title="muted ? 'Sonido apagado (M)' : 'Sonido encendido (M)'" :aria-label="muted ? 'Activar sonido' : 'Silenciar'" :aria-pressed="muted">
      <svg v-if="!muted" viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
        <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5" fill="currentColor" stroke="none"/>
        <path d="M15.5 8.5a5 5 0 0 1 0 7"/>
        <path d="M18.5 5.5a9 9 0 0 1 0 13"/>
      </svg>
      <svg v-else viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
        <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5" fill="currentColor" stroke="none"/>
        <line x1="16" y1="9" x2="22" y2="15"/>
        <line x1="22" y1="9" x2="16" y2="15"/>
      </svg>
    </button>

    <!-- LANDING ---------------------------------------------------------- -->
    <section v-if="screen === 'landing'" class="landing">
      <nav class="navbar">
        <div class="wordmark">Quiz de estudio</div>
        <button class="primary" @click="start">Iniciar test</button>
      </nav>
      <div class="hero">
        <div class="hero-wrap"><img class="hero-img" src="../assets/extrapolated-global-average-temperature-1970-2100.png"
          width="1728" height="863" alt="Pronóstico de temperatura global 1970-2100, una línea de tendencia que sube hacia la derecha"></div>
        <div class="scrim"></div>
        <h1 class="sentence">
          La <span class="slot"><img class="slot-img" src="../assets/sanz-baeza-estadistica-navarra-1858.jpg"
            width="1220" height="1689" alt="incertidumbre"></span> se decide con datos
        </h1>
        <p class="support">60 preguntas de Naive Bayes, Laplace, densidad y métricas. Cinco vidas, una racha.</p>
      </div>
      <section class="intro reveal">
        <p class="statement">Naive Bayes asume independencia; Laplace evita los ceros; densidad mide lo continuo; F1 mide el acierto.</p>
        <img class="thumb" src="../assets/forecasting-process.png"
          width="1725" height="227" alt="Diagrama del proceso de forecasting">
        <img class="thumb" src="../assets/ocde-atur-previst-fins-2022.jpg"
          width="1722" height="1159" alt="Pronóstico de paro anual de la OCDE hasta 2022">
        <img class="band" src="../assets/edificio-del-i-n-e-madrid-01.jpg"
          width="1920" height="1466" alt="Sede del Instituto Nacional de Estadística, Paseo de la Castellana, Madrid">
        <figure class="motion" aria-label="Pronóstico en movimiento">
          <video class="motion-video" autoplay muted loop playsinline preload="metadata"
            poster="../assets/global-smoke-forecast-visualization-daily-updated-svs5624-poster.jpg"
            width="854" height="428" aria-hidden="true">
            <source src="../assets/global-smoke-forecast-visualization-daily-updated-svs5624.webm" type="video/webm">
          </video>
          <figcaption>Pronóstico global de humo (difusión de aerosoles) · NASA SVS, dominio público · bucle silencioso.</figcaption>
        </figure>
      </section>
    </section>

    <!-- PLAY ------------------------------------------------------------- -->
    <section v-else-if="screen === 'play'" class="panel card">
      <header>
        <span class="ok">OK {{ correct }}</span>
        <span :class="{ hot: streak >= 3 }">Racha {{ streak }}</span>
        <span class="lives">{{ errors }} / {{ MAX_ERR }} fallos</span>
      </header>
      <div class="bar"><div :style="{ width: pct + '%' }"></div></div>
      <p class="small">Pregunta {{ idx + 1 }} de {{ deck.length }}</p>
      <h2>{{ cur.t }}</h2>
      <p v-if="isMulti" class="tag">Selecciona TODAS las correctas</p>
      <button v-for="(o, i) in cur.o" :key="i" class="opt" :class="optClass(i)" @click="pick(i)">
        <b>{{ 'ABCD'[i] }}</b><span class="box" v-if="isMulti">{{ sel.includes(i) ? 'x' : '' }}</span>{{ o }}
      </button>
      <div v-if="answered" class="fb" :class="lastOk ? 'good' : 'bad'">
        <strong>{{ lastOk ? '¡Correcto!' : 'Incorrecto' }}</strong> {{ cur.e }}
      </div>
      <button v-if="!answered" class="primary" :disabled="!sel.length" @click="submit">Confirmar (Enter)</button>
      <button v-else class="primary" @click="next">Siguiente (Enter)</button>
    </section>

    <!-- WON / LOST ------------------------------------------------------- -->
    <section v-else class="panel card center">
      <h1>{{ screen === 'won' ? '¡Terminaste!' : 'Te quedaste sin vidas' }}</h1>
      <p>Aciertos: <b>{{ correct }}</b> · Errores: <b>{{ errors }}</b> · Mejor racha: <b>{{ maxStreak }}</b></p>
      <p v-if="missed.length" class="small left"><b>Repasa estas:</b></p>
      <ul v-if="missed.length" class="left small"><li v-for="m in missed" :key="m.t">{{ m.t }} → <i>{{ m.e }}</i></li></ul>
      <button class="primary" @click="start">Jugar de nuevo (Enter)</button>
    </section>

    <!-- FOOTER ----------------------------------------------------------- -->
    <footer>
      <div class="foot-name">Quiz de estudio</div>
      <div class="foot-credits">
        Imágenes: «Extrapolated global average temperature 1970-2100» por Ldecola, CC0, vía Wikimedia Commons ·
        «Forecasting Process» por Renoy.zachariah, CC BY-SA 4.0, vía Wikimedia Commons ·
        «OCDE % atur previst fins 2022» por OECD, CC BY-SA 4.0, vía Wikimedia Commons ·
        «Sanz Baeza. Estadística Navarra (1858)» por Florencio Sanz Baeza, CC BY-SA 4.0, vía Wikimedia Commons ·
        «Edificio del I.N.E. (Madrid) 01» por Luis García (Zaqarbal), CC BY-SA 2.5, vía Wikimedia Commons ·
        «Global Smoke Forecast Visualization (Daily Updated) (SVS5624)» por NASA SVS, dominio público, vía Wikimedia Commons.
      </div>
    </footer>
  </main>
</template>

<style>
:root {
  --bg: #18191b;
  --surface: #222427;
  --ink: #eff0f1;
  --muted: #9c9ea3;
  --accent: #ff4a1f;
  --accent-ink: #18191b;
  --line: #3a3d43;
  --ok: #2ecc71;
  --bad: #ff5c6c;
  --font: 'Schibsted Grotesk', system-ui, -apple-system, sans-serif;
}
* { box-sizing: border-box }
html, body { margin: 0; background: var(--bg); color: var(--ink); font-family: var(--font); font-size: 1rem; }
body { min-height: 100svh; }

main { min-height: 100svh; display: grid; grid-template-rows: auto 1fr auto; }

/* ---------- shared ---------- */
.card { width: 100%; max-width: 720px; background: var(--surface); border-radius: 16px; padding: 24px; margin: 0 auto 40px; }
.center { text-align: center } .left { text-align: left }
.small { font-size: 0.85rem; color: var(--muted) }
.mute { position: fixed; bottom: 16px; right: 16px; background: #2c2f36; color: var(--ink); border-radius: 50%; width: 44px; height: 44px; cursor: pointer; border: 0; display: grid; place-items: center; z-index: 10 }
.mute svg { display: block }
.primary { margin-top: 10px; padding: 12px 22px; font-size: 1rem; font-weight: 600; color: var(--accent-ink); background: var(--accent); border: 0; border-radius: 0; cursor: pointer; font-family: inherit }
.primary:hover { opacity: 0.9 }
.primary:disabled { opacity: 0.4; cursor: not-allowed }
kbd { background: var(--line); border-radius: 4px; padding: 1px 6px }

/* ---------- landing ---------- */
.landing { display: grid; grid-template-rows: auto 1fr auto; min-height: 100svh; }
.navbar { display: flex; justify-content: space-between; align-items: center; padding: 24px 32px; }
.wordmark { font-weight: 800; font-size: 1.25rem; letter-spacing: -0.02em; }
.hero { position: relative; height: 78vh; min-height: 560px; overflow: hidden; display: grid; place-items: center; }
.hero-img { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; object-position: center bottom; z-index: 0; display: block; }
.scrim { position: absolute; inset: 0; background: rgba(24,25,27,0.45); z-index: 1; }
.sentence { position: relative; z-index: 2; font-weight: 800; font-size: clamp(7vw, 9vw, 13vw); line-height: 0.88; letter-spacing: -0.05em; text-align: center; color: var(--ink); max-width: 90vw; margin: 0 auto; hyphens: auto; }
.sentence .slot { display: inline-block; height: 1em; width: 2.4em; vertical-align: -0.1em; margin: 0 0.12em; overflow: hidden; border-radius: 0.14em; }
.sentence .slot-img { width: 100%; height: 100%; object-fit: cover; display: block; }
.support { position: relative; z-index: 2; font-size: 1rem; color: var(--ink); text-align: center; max-width: 42ch; margin: 14px auto 0; padding: 0 16px; }
.intro { padding: 56px 32px 40px; background: var(--surface); }
.statement { font-size: clamp(20px, 3vw, 28px); line-height: 1.12; font-weight: 600; text-align: center; color: var(--ink); max-width: 60ch; margin: 0 auto 24px; }
.thumb { width: 100%; max-width: 640px; height: auto; display: block; margin: 0 auto; border-radius: 0; }
.band { width: 100%; height: auto; display: block; margin: 24px 0; }
.motion { margin: 32px 0; text-align: center; }
.motion-video { width: 100%; max-width: 720px; height: auto; display: block; margin: 0 auto; border-radius: 0; }
.motion figcaption { font-size: 0.875rem; color: var(--muted); margin-top: 8px; }

/* ---------- play / end ---------- */
.panel { max-width: 720px; margin: 40px auto 0; }
header { display: flex; justify-content: space-between; font-weight: 600; margin-bottom: 10px; gap: 8px }
.hot { color: #ffb347 }
.bar { height: 6px; background: var(--line); border-radius: 6px; overflow: hidden; margin: 10px 0 6px }
.bar div { height: 100%; background: var(--accent); transition: width 0.3s }
h2 { line-height: 1.35; font-size: 1.5rem }
.tag { display: inline-block; background: #3b2f6b; padding: 3px 10px; border-radius: 20px; font-size: 0.9rem; margin: 8px 0 }
.opt { display: flex; gap: 10px; align-items: center; width: 100%; text-align: left; margin: 8px 0; padding: 12px; color: inherit; background: #2c2f36; border: 2px solid var(--line); border-radius: 0; font-size: 1rem; cursor: pointer }
.opt:hover { border-color: var(--accent) }
.opt b { background: var(--line); border-radius: 0; padding: 2px 8px }
.opt.picked { border-color: var(--accent); background: #2c2f36 }
.opt.right { border-color: var(--ok); background: #173a2a }
.opt.wrong { border-color: var(--bad); background: #41202a }
.opt.missedRight { border-color: var(--ok); border-style: dashed }
.fb { margin: 12px 0; padding: 12px; border-radius: 10px }
.fb.good { background: #173a2a } .fb.bad { background: #41202a }

/* ---------- footer ---------- */
footer { padding: 48px 32px 32px; background: var(--surface); color: var(--ink); font-size: 1rem; line-height: 1.5; margin-top: 40px; max-width: 60ch; margin-left: auto; margin-right: auto; }
.foot-name { font-weight: 800; color: var(--ink); font-size: 1.1rem; margin-bottom: 6px; }

/* ---------- motion ---------- */
.js .reveal > * { opacity: 0; transform: translateY(22px); transition: opacity 0.5s ease, transform 0.5s ease; }
.js .reveal > *:nth-child(1) { transition-delay: 0ms; }
.js .reveal > *:nth-child(2) { transition-delay: 70ms; }
.js .reveal > *:nth-child(3) { transition-delay: 140ms; }
.js .reveal > *:nth-child(4) { transition-delay: 210ms; }
.js .reveal.in > * { opacity: 1; transform: translateY(0); }
.hero-wrap { position: absolute; inset: 0; overflow: hidden; animation: drift 10s ease-in-out infinite alternate; }
@keyframes drift { from { transform: translate(-1.5%, -1.5%) scale(1); } to { transform: translate(1.5%, 1.5%) scale(1.02); } }
.hero-img { animation: shutter 1.1s ease forwards; transform-origin: center; display: block; }
@keyframes shutter { from { clip-path: inset(0 100% 0 0); transform: scale(1.12); } to { clip-path: inset(0); transform: scale(1); } }
@keyframes fadeup { from { opacity: 0; transform: translateY(18px); } to { opacity: 1; transform: none; } }
.load-in { animation: fadeup 0.6s ease both; }
button:focus-visible { outline: 3px solid var(--accent); outline-offset: 2px; }
button:active, .opt:active { transform: scale(0.97); }
@media (prefers-reduced-motion: reduce) {
  .reveal > * { opacity: 1; transform: none; transition: none; }
  .load-in { animation: none; opacity: 1; transform: none; }
  .hero-img { animation: none; clip-path: inset(0); transform: scale(1); }
  .hero-wrap { animation: none; transform: none; }
  button:active, .opt:active { transform: none; }
}
</style>