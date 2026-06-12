# Polish RAG Demo Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild the "Live Demo" section of `index.html` as a Polish-language, guided narrated stepper that teaches RAG honestly, with a 2-D vector map and similarity-driven retrieval.

**Architecture:** Single static file `index.html`, vanilla JS, no build step, no new runtime deps. Pure RAG logic (toy 2-D embeddings, cosine similarity, retrieval, answer assembly) is separated from DOM rendering inside one script block. The demo is a precompute-then-reveal state machine: on query submit the whole pipeline is computed, and the 5 stages progressively reveal it. The vector map is inline SVG.

**Tech Stack:** HTML, CSS (existing neobrutalist variables), vanilla JS, inline SVG, lucide icons (already loaded).

**Spec:** `docs/superpowers/specs/2026-06-12-rag-demo-polish-redesign-design.md`

---

## Verification methodology (read first)

There is no test runner. TDD is adapted as follows:

- **Logic tasks (Task 1):** write a temporary `console.assert` self-check block FIRST, load the page, confirm it fails (functions undefined), implement, confirm it prints `[DEMO TEST] PASS`. The block is removed in Task 8.
- **UI tasks:** verify by loading the page in Chrome and taking a screenshot / reading the console for errors.
- **Loading the page:** open `file:///home/andrzey/git-claude/hospital/index.html` in Chrome (claude-in-chrome `navigate`), then `read_console_messages` (expect no errors) and screenshot the `#demo` section.

After each task, commit.

---

## File structure

Only `index.html` is modified. Three regions:

- **CSS** (`<style>` … `</style>`, before line 411): add new classes for stepper, stage area, narrator, vector map, ranked list, prompt blocks, answer panel. Old demo-specific pipeline CSS (lines ~301–346) is removed in Task 8.
- **Markup** (`<section class="demo-section" id="demo">` … `</section>`, lines 476–588): replaced wholesale in Task 3. The `<section class="how">` (How-it-works) is translated to Polish in Task 7.
- **Script** (`<script>` … `</script>`, lines 630–919): the old demo script is deleted in Task 3; a new script holding both the pure logic (Task 1) and the controller (Tasks 4–6) replaces it.

To avoid `const` redeclaration collisions while the old script still exists (Tasks 1–2), the new code uses new identifiers: `DOCS`, `embedText`, `cosineSim`, `retrieve`, `assembleAnswer`, `buildPrompt`, `THRESHOLD`.

---

## Task 1: Pure RAG logic + Polish data model

**Files:**
- Modify: `index.html` — insert a NEW `<script>` block immediately before `</body>` (after the existing old script, which stays for now).

- [ ] **Step 1: Write the failing self-check block**

Insert immediately before `</body>` (the new script will hold the demo's permanent logic; the IIFE at the bottom is the temporary test, removed in Task 8):

```html
<script>
/* ===== MedVector — silnik demonstracyjny RAG (uproszczony, dydaktyczny) ===== */
const THRESHOLD = 0.5;

/* Każdy dokument ma ręcznie przypisany wektor 2-D (jednostkowy) — do celów
   dydaktycznych. Prawdziwe modele używają ~1500 wymiarów. */
const DOCS = [
  { id:1, title:"Wytyczne ESC — Ostry zawał serca (STEMI)", specialty:"kardiologia", category:"stan nagły", vec:[0.940,0.342],
    text:"W ostrym zawale serca z uniesieniem odcinka ST (STEMI) należy: 1) podać 150–300 mg kwasu acetylosalicylowego, 2) wykonać EKG w ciągu 10 minut od przyjęcia, 3) rozpocząć podwójną terapię przeciwpłytkową (ASA + inhibitor P2Y12), 4) rozważyć leczenie przeciwkrzepliwe (heparyna), 5) ocenić wskazania do reperfuzji: pierwotna PCI jest preferowana, jeśli możliwa w ciągu 120 minut, w przeciwnym razie fibrynoliza." },
  { id:2, title:"Pozaszpitalne zapalenie płuc — antybiotykoterapia", specialty:"choroby zakaźne", category:"leczenie", vec:[-0.643,0.766],
    text:"W leczeniu ambulatoryjnym pozaszpitalnego zapalenia płuc u wcześniej zdrowych dorosłych zaleca się makrolid (azytromycyna, klarytromycyna) lub doksycyklinę. U chorych z chorobami współistniejącymi preferuje się fluorochinolon oddechowy (lewofloksacyna, moksyfloksacyna) lub beta-laktam z makrolidem. Pacjenci hospitalizowani powinni otrzymać fluorochinolon oddechowy lub beta-laktam z makrolidem." },
  { id:3, title:"Sepsa i wstrząs septyczny — pakiet 1-godzinny", specialty:"intensywna terapia", category:"stan nagły", vec:[-0.259,0.966],
    text:"W sepsie i wstrząsie septycznym w ciągu pierwszej godziny należy: 1) oznaczyć stężenie mleczanu, 2) pobrać posiewy krwi przed antybiotykiem, 3) podać antybiotyk o szerokim spektrum, 4) rozpocząć szybki wlew krystaloidów 30 ml/kg przy hipotensji lub mleczanie ≥ 4 mmol/l, 5) zastosować leki obkurczające naczynia, jeśli hipotensja utrzymuje się mimo płynoterapii." },
  { id:4, title:"Docelowe ciśnienie tętnicze w cukrzycy", specialty:"endokrynologia", category:"choroba przewlekła", vec:[-0.940,0.342],
    text:"U większości chorych na cukrzycę zaleca się docelowe ciśnienie tętnicze < 140/90 mmHg. U osób z wysokim ryzykiem sercowo-naczyniowym można rozważyć cel < 130/80 mmHg, jeśli jest bezpiecznie osiągalny. Lekiem pierwszego wyboru jest inhibitor ACE lub sartan (ARB), zwłaszcza przy albuminurii. Często konieczne jest leczenie skojarzone kilkoma lekami." },
  { id:5, title:"Ostry udar niedokrwienny mózgu", specialty:"neurologia", category:"stan nagły", vec:[0.574,0.819],
    text:"W ostrym udarze niedokrwiennym mózgu zaleca się dożylne podanie alteplazy (tPA) w ciągu 3–4,5 godziny od wystąpienia objawów u kwalifikujących się chorych. Trombektomię mechaniczną należy rozważyć w niedrożności dużego naczynia do 24 godzin od ostatniego momentu dobrostanu. Po udarze stosuje się leczenie przeciwpłytkowe, statyny, kontrolę ciśnienia oraz rehabilitację." },
  { id:6, title:"Wytyczne GOLD — POChP", specialty:"pulmonologia", category:"choroba przewlekła", vec:[0.174,0.985],
    text:"W przewlekłej obturacyjnej chorobie płuc (POChP) leczenie obejmuje zaprzestanie palenia, szczepienia oraz farmakoterapię zależną od nasilenia objawów i ryzyka zaostrzeń. Krótko działające leki rozszerzające oskrzela stosuje się doraźnie, a długo działające (LABA, LAMA) jako leczenie podtrzymujące. Wziewne glikokortykosteroidy dodaje się u chorych z zaostrzeniami i eozynofilią. Zaleca się rehabilitację oddechową." },
];

/* Reguła „słowo kluczowe → kierunek w przestrzeni 2-D". Sumujemy kierunki
   dopasowanych słów i normalizujemy. Brak dopasowania → wektor [0,-1] (poza bazą). */
const SPECIALTY_DIRS = [
  { dir:[0.940,0.342],  keys:['zawał','serce','serca','sercow','wieńc','stemi'] },
  { dir:[0.574,0.819],  keys:['udar','mózg','niedokrwien','alteplaz','tpa','trombektom'] },
  { dir:[0.174,0.985],  keys:['pochp','obturacyjn','oskrzel','duszność','wziewn','lama','laba'] },
  { dir:[-0.259,0.966], keys:['sepsa','sepsie','wstrząs','posocznic','mleczan'] },
  { dir:[-0.643,0.766], keys:['zapaleni','antybiotyk','makrolid','fluorochinolon','doksycyklin'] },
  { dir:[-0.940,0.342], keys:['cukrzyc','ciśnieni','nadciśnien','glikemi','insulin'] },
];

const SPECIALTY_COLORS = {
  'kardiologia':'#FF6B6B', 'choroby zakaźne':'#6BCB77', 'intensywna terapia':'#C77DFF',
  'endokrynologia':'#FFD93D', 'neurologia':'#4D96FF', 'pulmonologia':'#6366f1'
};

function embedText(text){
  const t = (text||'').toLowerCase();
  let x=0,y=0;
  SPECIALTY_DIRS.forEach(s=>{ if(s.keys.some(k=>t.includes(k))){ x+=s.dir[0]; y+=s.dir[1]; } });
  if(x===0 && y===0) return [0,-1];
  const m=Math.hypot(x,y); return [x/m,y/m];
}

function cosineSim(a,b){
  const dot=a[0]*b[0]+a[1]*b[1];
  const ma=Math.hypot(a[0],a[1]), mb=Math.hypot(b[0],b[1]);
  return (ma===0||mb===0) ? 0 : dot/(ma*mb);
}

function retrieve(qVec, k=3){
  return DOCS.map(d=>({ ...d, sim:cosineSim(qVec,d.vec) }))
             .sort((a,b)=>b.sim-a.sim).slice(0,k);
}

function assembleAnswer(retrieved){
  if(!retrieved.length || retrieved[0].sim < THRESHOLD){
    return "Brak wystarczająco podobnych dokumentów w bazie wiedzy. Zadaj pytanie kliniczne dotyczące dostępnych wytycznych.";
  }
  let out = "Na podstawie odnalezionych wytycznych:";
  retrieved.filter(d=>d.sim>=THRESHOLD).forEach((d,i)=>{ out += `\n\n[${i+1}] ${d.title}\n${d.text}`; });
  out += "\n\nUwaga: To demonstracja edukacyjna — nie jest poradą medyczną.";
  return out;
}

/* ── TEMPORARY SELF-CHECK (removed in Task 8) ── */
(function(){
  const log=[];
  const okMag = DOCS.every(d=>Math.abs(Math.hypot(d.vec[0],d.vec[1])-1)<0.01);
  log.push(['wektory dokumentów są jednostkowe', okMag]);
  const qHeart = embedText("Jaki jest standardowy protokół leczenia ostrego zawału serca?");
  log.push(['zapytanie o zawał → dok. id 1 najbliżej', retrieve(qHeart)[0].id===1]);
  const qWeather = embedText("Jaka jest dzisiaj pogoda?");
  log.push(['pytanie poza tematem → max podobieństwo < próg', retrieve(qWeather)[0].sim < THRESHOLD]);
  log.push(['odpowiedź poza tematem zawiera „Brak"', assembleAnswer(retrieve(qWeather)).includes('Brak')]);
  log.push(['odpowiedź o zawale cytuje tytuł dok. 1', assembleAnswer(retrieve(qHeart)).includes('STEMI')]);
  const allPass = log.every(r=>r[1]);
  console.log('[DEMO TEST] '+(allPass?'PASS':'FAIL'));
  log.forEach(r=>console.log('  '+(r[1]?'✓':'✗')+' '+r[0]));
})();
</script>
```

- [ ] **Step 2: Run to verify it fails**

Before pasting, temporarily verify the red state by loading the page with ONLY the `(function(){…})()` IIFE present and the function definitions commented out — OR simply trust that on first paste a typo would surface. Practically: load `file:///…/index.html`, `read_console_messages`. Expected once implemented correctly: `[DEMO TEST] PASS` with five ✓ lines. If any ✗, fix the data/logic.

- [ ] **Step 3: Load page and confirm PASS**

Run: navigate to `file:///home/andrzey/git-claude/hospital/index.html`, `read_console_messages`
Expected: `[DEMO TEST] PASS` and no JS errors. (The old demo above still renders unchanged.)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add Polish RAG logic and 2-D vector data model"
```

---

## Task 2: CSS for stepper, stage area, narrator, vector map, panels

**Files:**
- Modify: `index.html` — insert the block below immediately before `</style>` (currently line 411).

- [ ] **Step 1: Add the new CSS**

```css
/* ── REDESIGNED DEMO ── */
.stepper{display:flex;gap:8px;flex-wrap:wrap;margin-bottom:16px}
.step-pill{
  display:flex;align-items:center;gap:8px;
  background:#fff;border:3px solid var(--dark);border-radius:99px;
  padding:8px 14px;font-size:13px;font-weight:800;color:#999;
  box-shadow:2px 2px 0 var(--dark);flex:1 1 auto;justify-content:center;
  transition:background .2s,color .2s,transform .2s;
}
.step-pill-num{
  width:22px;height:22px;border-radius:50%;border:2.5px solid var(--dark);
  background:#eee;color:var(--dark);
  display:flex;align-items:center;justify-content:center;font-size:12px;font-weight:900;flex-shrink:0;
}
.step-pill.done{color:var(--dark)}
.step-pill.done .step-pill-num{background:var(--green)}
.step-pill.active{background:var(--dark);color:#fff;transform:translateY(-2px)}
.step-pill.active .step-pill-num{background:var(--yellow)}

.stage-area{
  background:#fff;border:3px solid var(--dark);border-radius:18px;
  box-shadow:4px 4px 0 var(--dark);padding:20px;min-height:340px;margin-bottom:16px;
}
.stage-heading{font-size:1rem;font-weight:900;margin-bottom:4px}
.stage-caption{font-size:12px;font-weight:700;color:#888;margin-bottom:14px}

.narrator{
  background:var(--dark);color:#fff;
  border:3px solid var(--dark);border-radius:16px;padding:16px 18px;
  display:flex;flex-direction:column;gap:12px;
}
.narrator-text{font-size:.95rem;font-weight:700;line-height:1.5}
.narrator-controls{display:flex;gap:8px;flex-wrap:wrap}

/* Vector map */
.vmap-wrap{display:flex;gap:18px;flex-wrap:wrap;align-items:flex-start}
.vmap{flex:1 1 320px;min-width:280px}
.vmap svg{width:100%;height:auto;display:block;background:var(--cream);border:2.5px solid var(--dark);border-radius:12px}
.vmap-caption{font-size:11px;font-weight:700;color:#888;margin-top:6px;text-align:center}
.rank-list{flex:1 1 240px;min-width:220px;display:flex;flex-direction:column;gap:8px}
.rank-item{border:2.5px solid var(--dark);border-radius:10px;padding:8px 10px;box-shadow:2px 2px 0 var(--dark);font-size:12px;font-weight:800}
.rank-item.dim{opacity:.5}
.rank-bar{height:8px;border-radius:99px;background:#eee;border:2px solid var(--dark);margin-top:6px;overflow:hidden}
.rank-bar-fill{height:100%;background:var(--green)}

/* Prompt blocks */
.prompt-block{border:2.5px solid var(--dark);border-radius:10px;padding:10px 12px;margin-bottom:10px;box-shadow:2px 2px 0 var(--dark);background:#fff}
.prompt-tag{display:inline-block;color:#fff;padding:2px 10px;border-radius:6px;font-size:11px;font-weight:900;margin-bottom:6px}
.prompt-body{font-size:13px;font-weight:600;line-height:1.5;white-space:pre-wrap;color:#222}

/* Answer panel */
.answer-panel{font-size:14px;font-weight:600;line-height:1.65;white-space:pre-wrap;color:#1a1a2e}
.answer-cite{background:var(--yellow);border:2px solid var(--dark);border-radius:6px;padding:0 5px;font-weight:900;font-size:12px}
.answer-disclaimer{margin-top:16px;padding:12px 14px;background:var(--red);color:#fff;border:2.5px solid var(--dark);border-radius:12px;box-shadow:2px 2px 0 var(--dark);font-size:13px;font-weight:800}

@media(max-width:760px){
  .stepper{flex-direction:column}
  .vmap-wrap{flex-direction:column}
}
```

- [ ] **Step 2: Verify no visual regression**

Run: reload the page, screenshot. Expected: page looks identical (new classes are unused so far); no console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add CSS for redesigned demo (stepper, vector map, panels)"
```

---

## Task 3: Replace demo markup and remove the old demo script

**Files:**
- Modify: `index.html` — replace the entire `<section class="demo-section" id="demo"> … </section>` block (lines 476–588).
- Modify: `index.html` — delete the OLD `<script>` block that begins with `let currentStep = 0;` (lines 630–919). Keep the new script from Task 1.

- [ ] **Step 1: Replace the demo section markup**

Replace the whole `<section class="demo-section" id="demo">…</section>` with:

```html
<!-- DEMO -->
<section class="demo-section" id="demo">
  <div style="text-align:center;margin-bottom:48px">
    <div class="section-label label-blue">Demo na żywo</div>
    <h2 class="section-title">Zobacz, jak to działa</h2>
  </div>

  <div class="demo-outer">
    <div class="demo-label"><span class="pulse-dot"></span> Interaktywne demo — wypróbuj sam</div>

    <div class="demo-inner">
      <!-- Pytanie -->
      <div class="app-card">
        <div class="app-card-label">
          <span class="app-icon" style="background:#dbeafe;color:var(--blue)"><i data-lucide="search" style="width:16px;height:16px"></i></span>
          Pytanie kliniczne
        </div>
        <div style="display:flex;gap:8px;flex-wrap:wrap">
          <input id="queryInput" type="text" class="app-input" placeholder="Wpisz pytanie kliniczne..." value="Jaki jest standardowy protokół leczenia ostrego zawału serca?">
        </div>
        <div style="display:flex;flex-wrap:wrap;gap:6px;margin-top:10px;align-items:center">
          <span style="font-size:12px;font-weight:800;color:#888">Przykłady:</span>
          <button class="sampleQuery sample-chip" data-query="Jaki jest standardowy protokół leczenia ostrego zawału serca?">Zawał serca</button>
          <button class="sampleQuery sample-chip" data-query="Jakie są antybiotyki pierwszego rzutu w pozaszpitalnym zapaleniu płuc?">Antybiotyki w zapaleniu płuc</button>
          <button class="sampleQuery sample-chip" data-query="Jakie jest docelowe ciśnienie tętnicze u chorych na cukrzycę?">Ciśnienie w cukrzycy</button>
          <button class="sampleQuery sample-chip sample-chip-gray" data-query="Jaka jest dzisiaj pogoda?">Pogoda (poza tematem)</button>
        </div>
      </div>

      <!-- Stepper -->
      <div class="stepper" id="stepper">
        <div class="step-pill" data-stage="1"><span class="step-pill-num">1</span> Indeksowanie</div>
        <div class="step-pill" data-stage="2"><span class="step-pill-num">2</span> Osadzanie zapytania</div>
        <div class="step-pill" data-stage="3"><span class="step-pill-num">3</span> Wyszukiwanie</div>
        <div class="step-pill" data-stage="4"><span class="step-pill-num">4</span> Budowa promptu</div>
        <div class="step-pill" data-stage="5"><span class="step-pill-num">5</span> Odpowiedź</div>
      </div>

      <!-- Scena -->
      <div class="stage-area" id="stageArea"></div>

      <!-- Narrator + sterowanie -->
      <div class="narrator">
        <div class="narrator-text" id="narratorText">Naciśnij „Następny krok" albo „Odtwórz całość", aby rozpocząć.</div>
        <div class="narrator-controls">
          <button id="nextStepButton" class="app-btn app-btn-green"><i data-lucide="arrow-right" style="width:14px;height:14px"></i> Następny krok</button>
          <button id="runSimulationButton" class="app-btn app-btn-blue"><i data-lucide="play" style="width:14px;height:14px"></i> Odtwórz całość</button>
          <button id="resetButton" class="app-btn app-btn-red"><i data-lucide="rotate-ccw" style="width:14px;height:14px"></i> Reset</button>
        </div>
      </div>
    </div>

    <div class="demo-note">To demo działa w całości w Twojej przeglądarce. Żadne dane nie są wysyłane na serwer.</div>
  </div>
</section>
```

- [ ] **Step 2: Delete the old demo script**

Delete the entire `<script> … </script>` block that starts with `let currentStep = 0;` (it ends just before the new Task-1 `<script>`). The new Task-1 script remains and becomes the only demo script.

- [ ] **Step 3: Verify page loads clean**

Run: reload page, `read_console_messages`, screenshot.
Expected: `[DEMO TEST] PASS` still prints; the new demo markup is visible (query box with Polish chips, empty stepper pills, empty stage area, narrator with three Polish buttons); no JS errors. Buttons do nothing yet (controller not wired) — that is expected.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Replace demo markup with Polish stepper layout; remove old script"
```

---

## Task 4: Vector map SVG renderer

**Files:**
- Modify: `index.html` — add the functions below to the new `<script>` (above the temporary self-check IIFE).

- [ ] **Step 1: Add the map renderer**

Coordinates: model `(x,y)` with `x,y ∈ [-1,1]` map to SVG via `px = x*100`, `py = -y*100` (y flipped so positive y is up). `viewBox="-120 -120 240 240"`.

```js
function specialtyIcon(spec){
  const m={'kardiologia':'heart-pulse','choroby zakaźne':'bug','intensywna terapia':'activity',
           'endokrynologia':'flask-conical','neurologia':'brain','pulmonologia':'wind'};
  return m[spec]||'clipboard-list';
}

/* opts: { showQuery:bool, showLines:bool, topIds:[ids] } */
function renderVectorMap(opts){
  const o = Object.assign({showQuery:false, showLines:false, topIds:[]}, opts||{});
  const P = v => [ (v[0]*100).toFixed(1), (-v[1]*100).toFixed(1) ];
  let svg = '<svg viewBox="-120 -120 240 240" role="img" aria-label="Mapa wektorowa">';
  // osie
  svg += '<line x1="-115" y1="0" x2="115" y2="0" stroke="#ddd" stroke-width="1.5"/>';
  svg += '<line x1="0" y1="-115" x2="0" y2="115" stroke="#ddd" stroke-width="1.5"/>';
  svg += '<circle cx="0" cy="0" r="100" fill="none" stroke="#e6ddd0" stroke-width="1.5" stroke-dasharray="4 4"/>';
  // linie podobieństwa (rysowane pod punktami)
  if(o.showLines && state.qVec){
    const [qx,qy]=P(state.qVec);
    state.retrieved.forEach(()=>{});
    DOCS.forEach(d=>{
      const sim=cosineSim(state.qVec,d.vec);
      const [dx,dy]=P(d.vec);
      const w=Math.max(0.4, sim*5).toFixed(2);
      const op=Math.max(0.08, Math.min(1, sim)).toFixed(2);
      const hot=o.topIds.includes(d.id);
      svg += `<line x1="${qx}" y1="${qy}" x2="${dx}" y2="${dy}" stroke="${hot?'#1a1a2e':'#4D96FF'}" stroke-width="${hot?Math.max(1.5,w):w}" stroke-opacity="${op}"/>`;
    });
  }
  // punkty dokumentów
  DOCS.forEach(d=>{
    const [dx,dy]=P(d.vec);
    const color=SPECIALTY_COLORS[d.specialty]||'#999';
    const hot=o.topIds.includes(d.id);
    svg += `<circle cx="${dx}" cy="${dy}" r="${hot?9:7}" fill="${color}" stroke="#1a1a2e" stroke-width="${hot?3:2}"/>`;
    svg += `<text x="${dx}" y="${dy-12}" font-size="9" font-weight="800" text-anchor="middle" fill="#1a1a2e">${d.id}</text>`;
  });
  // gwiazdka zapytania
  if(o.showQuery && state.qVec){
    const [qx,qy]=P(state.qVec);
    svg += `<polygon points="${qx},${qy-10} ${(+qx+3).toFixed(1)},${(+qy-3).toFixed(1)} ${(+qx+10).toFixed(1)},${(+qy-3).toFixed(1)} ${(+qx+4).toFixed(1)},${(+qy+2).toFixed(1)} ${(+qx+6).toFixed(1)},${(+qy+9).toFixed(1)} ${qx},${(+qy+4).toFixed(1)} ${(+qx-6).toFixed(1)},${(+qy+9).toFixed(1)} ${(+qx-4).toFixed(1)},${(+qy+2).toFixed(1)} ${(+qx-10).toFixed(1)},${(+qy-3).toFixed(1)} ${(+qx-3).toFixed(1)},${(+qy-3).toFixed(1)}" fill="#FFD93D" stroke="#1a1a2e" stroke-width="2"/>`;
  }
  svg += '</svg>';
  return svg;
}
```

Note: `state` is defined in Task 5; this task's verification calls `renderVectorMap` with a stubbed global. Add this temporary stub line directly above `renderVectorMap` for now (removed at start of Task 5):

```js
var state = { qVec:[0.940,0.342], retrieved:[] };
```

- [ ] **Step 2: Verify the map renders**

Temporarily append to the self-check IIFE: `document.getElementById('stageArea').innerHTML = '<div class="vmap">'+renderVectorMap({showQuery:true,showLines:true,topIds:[1]})+'</div>';`
Run: reload page, screenshot the stage area.
Expected: an SVG with 6 coloured numbered dots arranged around a dashed circle, a yellow star near dot 1, and blue lines from the star to all dots (darkest to dot 1). Remove this temporary append line after confirming.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add inline-SVG vector map renderer"
```

---

## Task 5: State machine, stepper, narrator, and controls

**Files:**
- Modify: `index.html` — replace the temporary `var state = …` stub with the real state object; add the controller functions; wire event listeners.

- [ ] **Step 1: Add state, query handling, `buildPrompt`, and stage 1–3 rendering**

Replace the temporary `var state = {…}` stub line (from Task 4) with the following. (`buildPrompt` is defined here because `setQuery` calls it at init; Task 6 only consumes it.)

```js
const state = { query:'', qVec:null, retrieved:[], promptBlocks:[], answer:'', stage:0, running:false };

function buildPrompt(retrieved, query){
  const ctx = retrieved.filter(d=>d.sim>=THRESHOLD);
  return [
    {tag:'SYSTEM', color:'#1a1a2e', body:'Jesteś asystentem wspomagającym decyzje kliniczne. Odpowiadaj wyłącznie na podstawie dostarczonych dowodów.'},
    {tag:'KONTEKST', color:'#4D96FF', body: ctx.length ? ctx.map((d,i)=>'['+(i+1)+'] '+d.title+' ('+(d.sim*100).toFixed(0)+'%) — '+d.text.slice(0,90)+'…').join('\n') : '(brak pasujących dokumentów)'},
    {tag:'PYTANIE', color:'#6366f1', body: query},
    {tag:'INSTRUKCJA', color:'#6BCB77', body:'Udziel odpowiedzi opartej na dowodach, powołując się na odnalezione wytyczne.'}
  ];
}

const NARRATION = {
  0:'Naciśnij „Następny krok" albo „Odtwórz całość", aby rozpocząć.',
  1:'Każdy dokument zamieniamy na wektor — listę liczb opisującą jego znaczenie. Podobne tematy trafiają blisko siebie na mapie.',
  2:'To samo robimy z pytaniem — zamieniamy je na wektor w tej samej przestrzeni, żeby móc je porównać z dokumentami.',
  3:'Mierzymy podobieństwo (kąt między wektorami). Im bliżej, tym bardziej dokument pasuje. Wybieramy 3 najlepsze.',
  4:'Z odnalezionych dowodów składamy prompt: instrukcje + kontekst + pytanie. To wszystko, co model dostaje na wejściu.',
  5:'Model generuje odpowiedź wyłącznie na podstawie dostarczonego kontekstu — dlatego jest oparta na wytycznych, a nie zmyślona.'
};

function setQuery(q){
  state.query = q;
  state.qVec = embedText(q);
  state.retrieved = retrieve(state.qVec);
  state.promptBlocks = buildPrompt(state.retrieved, q);
  state.answer = assembleAnswer(state.retrieved);
  state.stage = 0;
  render();
}

function topIds(){
  return state.retrieved.filter(d=>d.sim>=THRESHOLD).map(d=>d.id);
}

function renderStage(){
  const area = document.getElementById('stageArea');
  const cap = '<div class="stage-caption">Uproszczony wektor 2-D do celów dydaktycznych. Prawdziwe modele używają ~1500 wymiarów.</div>';
  if(state.stage===0){
    area.innerHTML = '<div class="stage-heading">Gotowe do startu</div>'+
      '<div class="stage-caption">Wybierz przykład powyżej lub wpisz własne pytanie, a następnie przejdź przez kolejne kroki.</div>'+
      '<div class="vmap-wrap"><div class="vmap">'+renderVectorMap({})+'<div class="vmap-caption">Baza wiedzy w przestrzeni wektorowej (jeszcze bez zapytania)</div></div></div>';
  } else if(state.stage===1){
    area.innerHTML = '<div class="stage-heading">Krok 1 — Indeksowanie</div>'+cap+
      '<div class="vmap-wrap"><div class="vmap">'+renderVectorMap({})+'<div class="vmap-caption">6 dokumentów jako wektory — kolor oznacza specjalizację</div></div></div>';
  } else if(state.stage===2){
    area.innerHTML = '<div class="stage-heading">Krok 2 — Osadzanie zapytania</div>'+
      '<div class="stage-caption">Wektor zapytania: ['+state.qVec.map(v=>v.toFixed(3)).join(', ')+']</div>'+
      '<div class="vmap-wrap"><div class="vmap">'+renderVectorMap({showQuery:true})+'<div class="vmap-caption">⭐ zapytanie trafia do tej samej przestrzeni</div></div></div>';
  } else if(state.stage===3){
    area.innerHTML = '<div class="stage-heading">Krok 3 — Wyszukiwanie</div>'+
      '<div class="vmap-wrap"><div class="vmap">'+renderVectorMap({showQuery:true,showLines:true,topIds:topIds()})+
      '<div class="vmap-caption">Grubość linii = podobieństwo. Wyróżnione = 3 najlepsze</div></div>'+
      '<div class="rank-list">'+renderRankList()+'</div></div>';
  }
  if(typeof lucide!=='undefined') lucide.createIcons();
}

function renderRankList(){
  if(!state.retrieved.length) return '';
  const any = topIds().length>0;
  let html = '';
  if(!any) html += '<div class="rank-item dim">Brak dokumentów powyżej progu podobieństwa ('+(THRESHOLD*100)+'%).</div>';
  state.retrieved.forEach(d=>{
    const pct=Math.max(0,d.sim*100);
    const dim = d.sim<THRESHOLD ? ' dim':'';
    html += '<div class="rank-item'+dim+'"><div>Dok '+d.id+' — '+d.title+'</div>'+
      '<div style="font-size:11px;color:#666">podobieństwo: '+pct.toFixed(0)+'%</div>'+
      '<div class="rank-bar"><div class="rank-bar-fill" style="width:'+Math.max(0,Math.min(100,pct)).toFixed(0)+'%"></div></div></div>';
  });
  return html;
}
```

- [ ] **Step 2: Add stepper/narrator render, controls, and listeners**

```js
function renderStepper(){
  document.querySelectorAll('.step-pill').forEach(p=>{
    const s=+p.getAttribute('data-stage');
    p.classList.toggle('active', s===state.stage);
    p.classList.toggle('done', s<state.stage);
  });
}

function render(){
  renderStepper();
  renderStage();              // stages 4–5 added in Task 6
  document.getElementById('narratorText').textContent = NARRATION[state.stage];
  document.getElementById('nextStepButton').disabled = state.running || state.stage>=5;
  document.getElementById('runSimulationButton').disabled = state.running;
  if(typeof lucide!=='undefined') lucide.createIcons();
}

function nextStep(){ if(state.running) return; if(state.stage<5){ state.stage++; render(); } }
function reset(){ if(state.running) return; setQuery(document.getElementById('queryInput').value); }

async function runAll(){
  if(state.running) return;
  state.running=true; state.stage=0; render();
  for(let s=1;s<=5;s++){
    await new Promise(r=>setTimeout(r, s===1?400:900));
    state.stage=s; render();
  }
  state.running=false; render();
}

document.getElementById('nextStepButton').addEventListener('click', nextStep);
document.getElementById('runSimulationButton').addEventListener('click', runAll);
document.getElementById('resetButton').addEventListener('click', reset);
document.getElementById('queryInput').addEventListener('input', e=>setQuery(e.target.value));
document.querySelectorAll('.sampleQuery').forEach(b=>b.addEventListener('click', ()=>{
  document.getElementById('queryInput').value = b.getAttribute('data-query');
  setQuery(b.getAttribute('data-query'));
}));

setQuery(document.getElementById('queryInput').value);   // inicjalizacja
```

- [ ] **Step 3: Verify stages 1–3 work**

Run: reload page. Click „Następny krok" three times; screenshot after each. Then click a different chip and step again.
Expected: stepper pills fill in; stage 1 shows the map; stage 2 adds the star + query-vector text; stage 3 adds similarity lines + ranked list with bars; narrator text changes each step in Polish. Selecting the „Pogoda" chip and reaching stage 3 shows faint lines and a „Brak dokumentów powyżej progu" note. No console errors. (Stages 4–5 still render blank — added next task.)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add demo state machine, stepper, narrator, and controls (stages 1-3)"
```

---

## Task 6: Stages 4 (prompt) and 5 (answer)

**Files:**
- Modify: `index.html` — add `buildPrompt`, extend `renderStage()` for stages 4–5, add answer streaming + citation highlighting.

- [ ] **Step 1: Extend `renderStage()` for stages 4 and 5**

(`buildPrompt` is already defined in Task 5; `state.promptBlocks` is already populated.)

Add these two branches at the end of the `if/else` chain in `renderStage()` (before the `lucide.createIcons()` line):

```js
  else if(state.stage===4){
    let blocks='';
    state.promptBlocks.forEach(b=>{
      blocks += '<div class="prompt-block"><span class="prompt-tag" style="background:'+b.color+'">'+b.tag+'</span><div class="prompt-body">'+escapeHtml(b.body)+'</div></div>';
    });
    area.innerHTML = '<div class="stage-heading">Krok 4 — Budowa promptu</div>'+
      '<div class="stage-caption">Tak wygląda wejście przekazywane do modelu językowego.</div>'+blocks;
  }
  else if(state.stage===5){
    area.innerHTML = '<div class="stage-heading">Krok 5 — Odpowiedź</div>'+
      '<div class="stage-caption">W prawdziwym systemie ten krok wykonuje model językowy. Tutaj składamy odpowiedź z fragmentów dokumentów, aby pokazać zasadę.</div>'+
      '<div class="answer-panel" id="answerPanel"></div>';
    streamAnswer();
  }
```

- [ ] **Step 2: Add helpers `escapeHtml`, `highlightCitations`, `streamAnswer`**

```js
function escapeHtml(s){
  return s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
}
function highlightCitations(s){
  return s.replace(/\[(\d+)\]/g, '<span class="answer-cite">[$1]</span>');
}
function streamAnswer(){
  const el=document.getElementById('answerPanel');
  if(!el) return;
  const full=state.answer;
  const hasDisc = full.includes('nie jest poradą medyczną');
  const main = hasDisc ? full.split('\n\nUwaga:')[0] : full;
  const words = main.split(' ');
  let i=0;
  el.innerHTML='';
  (function tick(){
    if(state.stage!==5) return;            // przerwij, jeśli zmieniono krok
    i++;
    el.innerHTML = highlightCitations(escapeHtml(words.slice(0,i).join(' ')));
    if(i<words.length){ setTimeout(tick, 35); }
    else if(hasDisc){ el.innerHTML += '<div class="answer-disclaimer">Demonstracja edukacyjna — nie jest poradą medyczną.</div>'; }
  })();
}
```

- [ ] **Step 3: Verify stages 4–5**

Run: reload page. Step to stage 4 then 5 for the „Zawał serca" query; screenshot each. Repeat „Odtwórz całość". Then run the „Pogoda" query to stage 5.
Expected: stage 4 shows four colour-tagged Polish prompt blocks (SYSTEM/KONTEKST/PYTANIE/INSTRUKCJA) at readable size; stage 5 streams the answer word by word with highlighted `[1]`/`[2]` citations and a red disclaimer box. The „Pogoda" query at stage 5 shows the „Brak wystarczająco podobnych dokumentów" message with no citation box. No console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add prompt-building and answer stages to demo"
```

---

## Task 7: Translate the How-it-works section to Polish

**Files:**
- Modify: `index.html` — the `<section class="how" id="how-it-works"> … </section>` block (currently starts line 590).

- [ ] **Step 1: Translate headings and step cards**

Replace the section's text content (keep structure/classes) with:

- Section label: `Jak to działa`
- Title: `Indeksuj · Osadzaj · Wyszukuj · Buduj prompt · Odpowiadaj`
- Step 1 `Indeksowanie dokumentów` — `Literatura medyczna i wytyczne kliniczne są zamieniane na wektory przez model osadzeń i przechowywane w bazie wektorowej.`
- Step 2 `Zrozumienie zapytania` — `Pytanie klinicysty jest zamieniane na wektor, który oddaje znaczenie kliniczne, a nie tylko słowa kluczowe.`
- Step 3 `Wyszukiwanie dowodów` — `Wyszukiwanie po podobieństwie wektorów znajduje najbardziej pasujące dokumenty medyczne z bazy wiedzy.`
- Step 4 `Budowa promptu` — `Odnalezione dowody są składane w uporządkowany prompt: instrukcje systemowe, dokumenty kontekstu i oryginalne pytanie.`
- Step 5 `Generowanie odpowiedzi` — `Model przetwarza zbudowany prompt, łącząc odnalezione dowody z pytaniem, aby wygenerować odpowiedź opartą na faktach.`
- Note box: `<strong>Uwaga:</strong> Jeśli pytanie wykracza poza bazę wiedzy (np. pytanie niemedyczne), system poinformuje, że nie ma odpowiednich dokumentów, aby udzielić odpowiedzi.`

- [ ] **Step 2: Verify**

Run: reload page, screenshot the How-it-works section. Expected: all five cards and the note in Polish; layout unchanged.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Translate How-it-works section to Polish"
```

---

## Task 8: Remove dead CSS and the temporary self-check; final verification

**Files:**
- Modify: `index.html` — delete the temporary self-check IIFE (the `(function(){ … })();` block ending with `[DEMO TEST]`) and the now-unused old demo CSS.

- [ ] **Step 1: Remove the temporary self-check IIFE**

Delete the entire `/* ── TEMPORARY SELF-CHECK … */ (function(){ … })();` block from the script.

- [ ] **Step 2: Remove unused old demo CSS**

Delete the now-unused rules: `/* Pipeline boxes */` through the end of `.doc-tag{…}` (the `.pipe-*`, `.doc-grid`, `.doc-card-app`, `.doc-tag` rules, originally lines ~301–346). Verify by searching that none of these class names remain in the markup:

Run: `grep -n 'pipe-\|doc-card-app\|doc-grid\|doc-tag\|documentStoreBox\|customerQueryBox' index.html`
Expected: no matches.

- [ ] **Step 3: Full end-to-end verification**

Run: reload `file:///home/andrzey/git-claude/hospital/index.html`, `read_console_messages`, and walk all four sample chips plus one custom Polish query, using both „Następny krok" and „Odtwórz całość", then „Reset".
Expected, for each:
- All five stages render; narrator text is Polish and matches the stage.
- Map shows dots (stage 1), star (stage 2), similarity lines + ranked bars (stage 3).
- Prompt blocks readable (stage 4); answer streams with citations + disclaimer (stage 5).
- „Pogoda" query reaches „Brak wystarczająco podobnych dokumentów" via low similarity (no citation box).
- „Reset" returns to stage 0.
- No console errors; no `[DEMO TEST]` line; no English in the demo or How-it-works sections.
- Resize to ~375px wide: stepper stacks, map and rank list stack, nothing overflows horizontally.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Remove temporary self-check and dead demo CSS; finalize redesign"
```

---

## Self-review against spec

- **Polish-only (spec goal 2):** Tasks 1, 3, 5, 6, 7 translate data, UI, narration, prompt, answer, and How-it-works. ✓
- **Single guided flow / remove redundant buttons (spec):** Task 3 drops `Generate Vectors`, `Toggle View`, and the morphing button; Task 5 gives one stepper + three clear controls. ✓
- **Payoff gets space (spec goal 3):** Task 6 renders prompt blocks and a full-width answer panel instead of 11px scroll boxes. ✓
- **2-D vector map (spec goal 4):** Task 4 renderer; Task 5 wires it into stages 1–3. ✓
- **Honest model (spec section):** 2-D unit vectors plotted directly (Task 1/4); retrieval + no-match purely cosine-driven, hardcoded `weather` branch gone (Task 1); answer assembled from retrieved text (Task 1); keyword shortcut named in narration (Task 5 NARRATION[1]). ✓
- **"Pre-indexed" contradiction removed (spec):** old markup deleted in Task 3; stage 1 shows indexing happening. ✓
- **Not medical advice / simplified framing (spec):** disclaimer in answer (Task 6), 2-D caption (Task 5), "real LLM" note (Task 6). ✓
- **Constraints (single file, no build, no deps, mobile, neobrutalist):** inline SVG, existing CSS variables, responsive rules in Task 2. ✓
- **Verification (spec testing section):** Task 8 step 3 covers all six listed manual checks. ✓

No spec requirement is left without a task.
```
