<template>
  <div class="slide-root" @click="handleClick" tabindex="0" @keydown="handleKey">
    <div class="slide-title">
      AI Quote <span class="blue">Intake</span> — Pipeline Walk-through
    </div>

    <div class="pipeline-area">
      <!-- FORWARD PASS -->
      <div class="pass-header fwd-header">Forward extraction pass f(X)</div>

      <div class="lane">
        <!-- INPUT -->
        <PNode id="n-input" :state="nodeState('n-input')" title="PDF" sub="Input X" />
        <Arrow id="arr-0" :active="isActive('arr-0')" />

        <!-- OCR GROUP -->
        <div class="group" :class="{ 'group-fwd': anyActive(['n-1a','n-1b','n-2b']) }">
          <div class="group-label">OCR</div>
          <div class="group-inner">
            <PNode id="n-1a" :state="nodeState('n-1a')" title="1a. Digital" sub="PyPDF text" />
            <span class="or-label">or</span>
            <PNode id="n-1b" :state="nodeState('n-1b')" title="1b. Raster" sub="PyPDF→PNG" />
            <Arrow id="arr-1b" :active="isActive('arr-1b')" />
            <PNode id="n-2b" :state="nodeState('n-2b')" title="2b. OCR" sub="docTR" />
          </div>
        </div>

        <Arrow id="arr-2" :active="isActive('arr-2')" />
        <PNode id="n-3" :state="nodeState('n-3')" title="3. Line Recon" sub="Group & sort" :wide="true" />
        <Arrow id="arr-3" :active="isActive('arr-3')" />

        <!-- CORTEX GROUP -->
        <div class="group" :class="{ 'group-fwd': anyActive(['n-4a','n-4b','n-4c']) }">
          <div class="group-label">Cortex Extraction</div>
          <div class="group-inner">
            <PNode id="n-4a" :state="nodeState('n-4a')" title="4a. Prompt" sub="Rules & hints" />
            <Arrow id="arr-4a" :active="isActive('arr-4a')" small />
            <PNode id="n-4b" :state="nodeState('n-4b')" title="4b. Extract" sub="LLM batch" />
            <Arrow id="arr-4b" :active="isActive('arr-4b')" small />
            <PNode id="n-4c" :state="nodeState('n-4c')" title="4c. Locate" sub="Exact/fuzzy" />
          </div>
        </div>

        <Arrow id="arr-5" :active="isActive('arr-5')" />
        <PNode id="n-5" :state="nodeState('n-5')" title="5. Assemble" sub="Nested schema" :wide="true" />
        <Arrow id="arr-6" :active="isActive('arr-6')" />

        <!-- TEACHER GROUP -->
        <div class="group" :class="{ 'group-fwd': anyActive(['n-6a','n-6b','n-6c']) }">
          <div class="group-label">Teacher Pass</div>
          <div class="group-inner">
            <PNode id="n-6a" :state="nodeState('n-6a')" title="6a. Flag" sub="Low-conf" />
            <Arrow id="arr-6a" :active="isActive('arr-6a')" small />
            <PNode id="n-6b" :state="nodeState('n-6b')" title="6b. Verify" sub="LLM x-check" />
            <Arrow id="arr-6b" :active="isActive('arr-6b')" small />
            <PNode id="n-6c" :state="nodeState('n-6c')" title="6c. Retry" sub="Targeted fix" />
          </div>
        </div>

        <Arrow id="arr-out" :active="isActive('arr-out')" />
        <PNode id="n-output" :state="nodeState('n-output')" title="JSON" sub="Output Y" output />
      </div>

      <!-- BACKWARD PASS -->
      <div class="pass-header bwd-header" style="margin-top: 10px;">
        Backward generation pass f̃⁻¹(y)
      </div>

      <div class="lane lane-reverse">
        <PNode id="n-b-start" :state="nodeState('n-b-start')" title="Output Y" sub="seed profiles" bwd-end />
        <Arrow id="arr-b1" :active="isActive('arr-b1')" bwd reverse />
        <PNode id="n-b1" :state="nodeState('n-b1')" title="B1. Gen Data" sub="LLM profiles" :wide="true" />
        <Arrow id="arr-b2" :active="isActive('arr-b2')" bwd reverse />
        <PNode id="n-b2" :state="nodeState('n-b2')" title="B2. Layout" sub="Vector PDF" />
        <Arrow id="arr-b3" :active="isActive('arr-b3')" bwd reverse />
        <PNode id="n-b3" :state="nodeState('n-b3')" title="B3. Rasterise" sub="DPI drop" />
        <Arrow id="arr-b4" :active="isActive('arr-b4')" bwd reverse />
        <PNode id="n-b4" :state="nodeState('n-b4')" title="B4. Degrade" sub="Scan artifacts" />
        <Arrow id="arr-b5" :active="isActive('arr-b5')" bwd reverse />
        <PNode id="n-b-end" :state="nodeState('n-b-end')" title="Synth PDF" sub="→ feeds Input X" bwd-end />
      </div>
    </div>

    <!-- DATA DISPLAY -->
    <div class="data-box">
      <div class="data-label">Data at current stage</div>
      <transition name="fade">
        <div :key="currentStep" class="data-value">{{ currentData }}</div>
      </transition>
    </div>

    <!-- CONTROLS -->
    <div class="controls">
      <div class="step-dots">
        <div
          v-for="(s, i) in STEPS"
          :key="i"
          class="dot"
          :class="{
            'dot-fwd': i < currentStep && s.pass === 'fwd',
            'dot-bwd': i < currentStep && s.pass === 'bwd',
            'dot-cur': i === currentStep - 1,
            'dot-cur-bwd': i === currentStep - 1 && s.pass === 'bwd'
          }"
        />
      </div>
      <div class="step-counter">Step {{ currentStep }} / {{ STEPS.length }}</div>
    </div>
    <div class="hint">Press <b>Space</b> to advance · <b>←</b> to go back · click anywhere to advance</div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

// ── Sub-components ────────────────────────────────────────────────────────────

const PNode = {
  props: ['id','state','title','sub','wide','output','bwdEnd'],
  template: `
    <div
      class="pnode"
      :class="[
        state,
        wide  ? 'pnode-wide'  : '',
        output? 'pnode-output': '',
        bwdEnd? 'pnode-bwd-end':''
      ]"
    >
      <div class="pnode-title">{{ title }}</div>
      <div v-if="sub" class="pnode-sub">{{ sub }}</div>
    </div>
  `
}

const Arrow = {
  props: ['id','active','small','bwd','reverse'],
  template: `
    <div class="arr-wrap" :class="{ 'arr-reverse': reverse }">
      <svg :width="small ? 16 : 20" height="12" overflow="visible">
        <defs>
          <marker :id="'ah-'+id" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto">
            <path d="M1 1l4 2-4 2" fill="none" stroke-width="1.5"
              :stroke="active ? (bwd ? '#c0392b' : '#0107ba') : '#ccc'" />
          </marker>
        </defs>
        <line x1="1" y1="6" :x2="small ? 13 : 17" y2="6"
          :stroke="active ? (bwd ? '#c0392b' : '#0107ba') : '#ccc'"
          stroke-width="1.5"
          :marker-end="'url(#ah-'+id+')'"
        />
      </svg>
    </div>
  `
}

// ── Steps data ────────────────────────────────────────────────────────────────

const STEPS = [
  {
    pass: 'fwd', node: 'n-input', arrows: [],
    data: '📄  Binary PDF file (INPUT X)\n\nRaw bytes — not yet parsed. Could be a scanned document or a native digital PDF.'
  },
  {
    pass: 'fwd', node: 'n-1a', arrows: ['arr-0'], extra: null,
    data: '📝  Raw text string (digital path)\n\n"Policy Number: 12345\\nInsured: Acme Corp\\nPremium: $10,450\\nEffective: 01-Jan-2025 …"\n\nPyPDF extracts embedded glyphs directly.'
  },
  {
    pass: 'fwd', node: 'n-1b', arrows: ['arr-0'], extra: 'n-2b',
    data: '🖼️  Raster → OCR (scanned path)\n\nPNG at 300 DPI → docTR → raw text string with noise: "Po l i cy Num ber: 1 2345 …"'
  },
  {
    pass: 'fwd', node: 'n-3', arrows: ['arr-0','arr-1b','arr-2'],
    data: '📋  Structured line list\n\n[ { "line": "Policy Number: 12345", "page": 1, "y": 142 },\n  { "line": "Insured: Acme Corp",   "page": 1, "y": 168 },\n  { "line": "Premium: $10,450",     "page": 1, "y": 194 }, … ]\n\nGrouped by reading order and bounding-box Y.'
  },
  {
    pass: 'fwd', node: 'n-4a', arrows: ['arr-0','arr-1b','arr-2','arr-3'],
    data: '📐  Extraction prompt (rules + hints)\n\nSystem: "Extract: policy_number (int), insured_name (str), premium (float USD), effective_date (ISO 8601). Flag confidence < 0.8. Return JSON only."\n\nHints injected from document-type classifier.'
  },
  {
    pass: 'fwd', node: 'n-4b', arrows: ['arr-0','arr-1b','arr-2','arr-3','arr-4a'],
    data: '🤖  LLM extraction answers (raw JSON)\n\n{ "policy_number": { "value": "12345", "conf": 0.98 },\n  "insured_name":  { "value": "Acme Corp", "conf": 0.97 },\n  "premium":       { "value": 10450.00,  "conf": 0.95 },\n  "effective_date":{ "value": "2025-01-01","conf": 0.72 } }\n\nNote: effective_date conf < 0.8 → will be flagged.'
  },
  {
    pass: 'fwd', node: 'n-4c', arrows: ['arr-0','arr-1b','arr-2','arr-3','arr-4a','arr-4b'],
    data: '📍  Located source spans\n\n• policy_number  → page 1, chars 16-20, exact match\n• insured_name   → page 1, chars 57-64, exact match\n• premium        → page 1, chars 80-85, exact match\n• effective_date → page 1, chars 100-109, fuzzy (date format varies)'
  },
  {
    pass: 'fwd', node: 'n-5', arrows: ['arr-0','arr-1b','arr-2','arr-3','arr-4a','arr-4b','arr-5'],
    data: '🗂️  Assembled nested schema\n\n{ "quote": {\n    "policy_number": "12345",\n    "insured": { "name": "Acme Corp" },\n    "financials": { "premium": 10450.00, "currency": "USD" },\n    "dates": { "effective": "2025-01-01" }\n  },\n  "_meta": { "low_conf_fields": ["effective_date"] } }'
  },
  {
    pass: 'fwd', node: 'n-6a', arrows: ['arr-0','arr-1b','arr-2','arr-3','arr-4a','arr-4b','arr-5','arr-6'],
    data: '🚩  Flagged low-confidence fields\n\nFields where conf < 0.8:\n• effective_date: 0.72 → sent to LLM cross-check\n\nAll other fields pass directly to output.'
  },
  {
    pass: 'fwd', node: 'n-6b', arrows: ['arr-0','arr-1b','arr-2','arr-3','arr-4a','arr-4b','arr-5','arr-6','arr-6a'],
    data: '✅  LLM cross-check result\n\nVerification prompt: "Does \'2025-01-01\' match the span \'01/01/2025\' on page 1?"\n\nResponse: { "match": true, "corrected_value": "2025-01-01", "conf": 0.94 }\n\nField accepted — confidence restored.'
  },
  {
    pass: 'fwd', node: 'n-6c', arrows: ['arr-0','arr-1b','arr-2','arr-3','arr-4a','arr-4b','arr-5','arr-6','arr-6a','arr-6b'],
    data: '🔄  Targeted retry (if still failing)\n\nIf verify still fails → re-extract with tighter prompt + wider context window around flagged span.\n\nIn this run: not needed. Confidence restored ✔'
  },
  {
    pass: 'fwd', node: 'n-output', arrows: ['arr-0','arr-1b','arr-2','arr-3','arr-4a','arr-4b','arr-5','arr-6','arr-6a','arr-6b','arr-out'],
    data: '🎯  Final OUTPUT Y (JSON)\n\n{ "policy_number": "12345",\n  "insured_name": "Acme Corp",\n  "premium": 10450.00,\n  "effective_date": "2025-01-01",\n  "_verified": true,\n  "_processing_path": "digital" }\n\n→ Feeds downstream systems (CRM, rating engine, …)'
  },
  // BACKWARD PASS
  {
    pass: 'bwd', node: 'n-b-start', arrows: [],
    data: '🌱  Backward pass — starts from Output Y\n\nLLM generates synthetic insurance profiles:\n{ "insured": "Bright Futures LLC", "premium": 8200, "policy_type": "GL", … }\n\nNo real customer data used.'
  },
  {
    pass: 'bwd', node: 'n-b1', arrows: ['arr-b1'],
    data: '📐  B2. Vector PDF layout\n\nSynthetic data injected into a LaTeX/HTML template — correct fonts, tables, headings matching real doc structure.\n\nOutput: clean machine-perfect vector PDF.'
  },
  {
    pass: 'bwd', node: 'n-b2', arrows: ['arr-b1','arr-b2'],
    data: '🖨️  B3. Rasterised at low DPI\n\nVector PDF → PNG at 72–150 DPI.\nSimulates scanner output: pixels, aliasing, slight blur.\n\nReady for scan-artifact injection.'
  },
  {
    pass: 'bwd', node: 'n-b3', arrows: ['arr-b1','arr-b2','arr-b3'],
    data: '🌫️  B4. Scan artifacts applied\n\n• Random rotation ±2°\n• Salt-and-pepper noise\n• JPEG compression (quality 60–80)\n• Streaks, skew, smudge\n\nOutput looks like a real field-office scan.'
  },
  {
    pass: 'bwd', node: 'n-b4', arrows: ['arr-b1','arr-b2','arr-b3','arr-b4'],
    data: '🌫️  B4. Degrade applied\n\nFinal degradation pass: contrast drop, edge blur, random ink bleed.\n\nThe synthetic doc is now visually indistinguishable from a real scanned policy.'
  },
  {
    pass: 'bwd', node: 'n-b-end', arrows: ['arr-b1','arr-b2','arr-b3','arr-b4','arr-b5'],
    data: '🔁  Synthetic PDF → feeds back into Input X\n\nDegraded synthetic PDF passed through forward pass f(X).\nGround-truth labels known from B1 profiles → enables supervised training & eval without real data.\n\nLoop closes. Pipeline is self-supervising.'
  }
]

// ── State ─────────────────────────────────────────────────────────────────────

const currentStep = ref(0)

const currentData = computed(() => {
  if (currentStep.value === 0) return 'Press Space or click to begin the walk-through.'
  return STEPS[currentStep.value - 1].data
})

function nodeState(id) {
  if (currentStep.value === 0) return ''
  const cur = STEPS[currentStep.value - 1]
  // active
  if (cur.node === id || cur.extra === id) {
    return cur.pass === 'fwd' ? 'node-active-fwd' : 'node-active-bwd'
  }
  // done
  for (let i = 0; i < currentStep.value - 1; i++) {
    const s = STEPS[i]
    if (s.node === id || s.extra === id) {
      return s.pass === 'fwd' ? 'node-done-fwd' : 'node-done-bwd'
    }
  }
  return ''
}

function isActive(arrowId) {
  if (currentStep.value === 0) return false
  const cur = STEPS[currentStep.value - 1]
  return (cur.arrows || []).includes(arrowId)
}

function anyActive(ids) {
  return ids.some(id => {
    const s = nodeState(id)
    return s === 'node-active-fwd' || s === 'node-active-bwd'
  })
}

function nextStep() {
  if (currentStep.value < STEPS.length) currentStep.value++
}
function prevStep() {
  if (currentStep.value > 0) currentStep.value--
}

function handleKey(e) {
  if (e.code === 'Space' || e.code === 'ArrowRight') { e.preventDefault(); nextStep() }
  if (e.code === 'ArrowLeft') { e.preventDefault(); prevStep() }
}

function handleClick() { nextStep() }
</script>

<style scoped>
.slide-root {
  width: 100%;
  height: 100%;
  background: white;
  color: black;
  font-family: 'Arial', sans-serif;
  padding: 24px 28px 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  outline: none;
  cursor: pointer;
}

.slide-title {
  text-align: center;
  font-size: 20px;
  font-weight: 700;
  letter-spacing: 0.4px;
}
.blue { color: #0107ba; }

/* ── PASS HEADERS ── */
.pass-header {
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.6px;
  padding: 2px 9px;
  border-radius: 5px;
  display: inline-block;
  margin-bottom: 3px;
}
.fwd-header { color: #0107ba; background: #eef0ff; border: 1px solid #0107ba; }
.bwd-header { color: #c0392b; background: #fff0ee; border: 1px solid #c0392b; }

/* ── LANES ── */
.lane {
  display: flex;
  align-items: center;
  flex-wrap: nowrap;
  gap: 0;
}
.lane-reverse {
  flex-direction: row-reverse;
}

/* ── GROUPS ── */
.group {
  border: 1.5px dashed #bbb;
  border-radius: 8px;
  padding: 5px 7px 5px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  transition: border-color 0.3s;
}
.group-fwd { border-color: #0107ba; }
.group-label {
  font-size: 8px;
  font-weight: 700;
  letter-spacing: 0.8px;
  text-transform: uppercase;
  opacity: 0.45;
}
.group-inner {
  display: flex;
  align-items: center;
  gap: 0;
}
.or-label {
  font-size: 9px;
  color: #999;
  padding: 0 3px;
}

/* ── NODES ── */
.pnode {
  border-radius: 6px;
  border: 1.5px solid #ccc;
  padding: 6px 8px;
  text-align: center;
  min-width: 64px;
  transition: border-color 0.3s, background 0.3s, color 0.3s, box-shadow 0.3s;
}
.pnode-wide { min-width: 82px; }

.pnode-title {
  font-size: 10.5px;
  font-weight: 700;
  white-space: nowrap;
}
.pnode-sub {
  font-size: 9px;
  font-weight: 400;
  opacity: 0.65;
  margin-top: 1px;
  white-space: nowrap;
}

/* Node states */
.node-active-fwd {
  border-color: #0107ba !important;
  background: #eef0ff;
  color: #0107ba;
  box-shadow: 0 0 0 3px rgba(1, 7, 186, 0.15);
}
.node-active-bwd {
  border-color: #c0392b !important;
  background: #fff0ee;
  color: #c0392b;
  box-shadow: 0 0 0 3px rgba(192, 57, 43, 0.15);
}
.node-done-fwd { border-color: #0107ba; background: #f4f5ff; color: #0107ba; }
.node-done-bwd { border-color: #c0392b; background: #fff5f4; color: #c0392b; }

/* Special nodes */
.pnode-output { border-color: #2e7d32; }
.pnode-output .pnode-title { color: #2e7d32; }
.pnode-bwd-end { border-color: #c0392b; }
.pnode-bwd-end .pnode-title { color: #c0392b; }

/* ── ARROWS ── */
.arr-wrap {
  display: flex;
  align-items: center;
  flex-shrink: 0;
}
.arr-reverse { transform: rotate(180deg); }

/* ── DATA BOX ── */
.data-box {
  border: 1.5px solid #e5e5e5;
  border-radius: 8px;
  padding: 9px 14px;
  background: #f9f9f9;
  min-height: 72px;
  flex-shrink: 0;
}
.data-label {
  font-size: 8.5px;
  font-weight: 700;
  letter-spacing: 1px;
  text-transform: uppercase;
  opacity: 0.45;
  margin-bottom: 4px;
}
.data-value {
  font-family: monospace;
  font-size: 10.5px;
  color: #000;
  white-space: pre-wrap;
  line-height: 1.5;
}

/* ── CONTROLS ── */
.controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 2px;
}
.step-dots {
  display: flex;
  gap: 4px;
  align-items: center;
  flex-wrap: wrap;
  max-width: 220px;
}
.dot {
  width: 7px; height: 7px;
  border-radius: 50%;
  background: #ddd;
  transition: background 0.3s;
  flex-shrink: 0;
}
.dot-fwd { background: #0107ba; }
.dot-bwd { background: #c0392b; }
.dot-cur { box-shadow: 0 0 0 2px #fff, 0 0 0 3.5px #0107ba; }
.dot-cur-bwd { box-shadow: 0 0 0 2px #fff, 0 0 0 3.5px #c0392b; }

.step-counter {
  font-size: 10px;
  opacity: 0.45;
}

.hint {
  text-align: center;
  font-size: 9px;
  opacity: 0.35;
  margin-top: 2px;
}

/* ── TRANSITIONS ── */
.fade-enter-active, .fade-leave-active { transition: opacity 0.25s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
