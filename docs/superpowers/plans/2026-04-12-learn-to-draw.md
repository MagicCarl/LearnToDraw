# Learn To Draw Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a two-page magic trick app — spectator picks a subject on a public webpage, magician's phone instantly shows a hand-drawn sketch of that subject inside a fake drawing app UI.

**Architecture:** Two plain HTML files (`index.html` + `receiver.html`), no build system. Firebase Realtime Database (existing project) handles real-time sync via SSE — same pattern as AnySearch. Spectator writes a key string to `/learnToDraw/selection`; receiver listens and renders the matching sketch instantly.

**Tech Stack:** Vanilla HTML/CSS/JS, Firebase Realtime Database REST API + EventSource SSE, HTML5 Canvas (spectator drawing pad).

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `vercel.json` | Create | Deploy config, no-cache header |
| `index.html` | Create | Spectator page — 12 picture buttons, drawing canvas, Firebase write |
| `receiver.html` | Create | Magician's phone — fake drawing app UI, 12 hand-drawn SVGs, SSE listener, Erase |

---

## Task 1: vercel.json

**Files:**
- Create: `vercel.json`

- [ ] **Step 1: Create `vercel.json`**

```json
{
  "headers": [
    {
      "source": "/",
      "headers": [{ "key": "Cache-Control", "value": "no-cache, no-store, must-revalidate" }]
    }
  ]
}
```

- [ ] **Step 2: Commit**

```bash
cd /Users/carlandrews/Downloads/LearnToDraw
git add vercel.json
git commit -m "Add vercel.json deploy config"
```

---

## Task 2: Spectator Page (index.html)

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create `index.html` with complete spectator page**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0">
  <title>Learn To Draw</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
    body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; background: #fff; min-height: 100vh; }

    .header { text-align: center; padding: 28px 20px 16px; }
    .header h1 { font-size: 26px; font-weight: 700; color: #222; margin-bottom: 4px; }
    .header p { font-size: 14px; color: #999; }

    .grid {
      display: grid;
      grid-template-columns: repeat(4, 1fr);
      gap: 10px;
      padding: 0 14px 20px;
      max-width: 480px;
      margin: 0 auto;
    }

    .subject-btn {
      display: flex; flex-direction: column; align-items: center; justify-content: center;
      background: #f7f7f7; border: 2.5px solid transparent; border-radius: 14px;
      padding: 10px 6px 8px; cursor: pointer; transition: border-color 0.15s, background 0.15s;
      aspect-ratio: 1; -webkit-appearance: none;
    }
    .subject-btn:active { transform: scale(0.94); }
    .subject-btn.selected { border-color: #f4a820; background: #fffbf0; }
    .subject-btn svg { width: 48px; height: 44px; margin-bottom: 5px; display: block; }
    .subject-btn .lbl { font-size: 10px; font-weight: 600; color: #555; text-align: center; line-height: 1.2; }

    #canvas-area { display: none; padding: 0 14px 40px; max-width: 480px; margin: 0 auto; }
    #canvas-area .cue { text-align: center; font-size: 14px; color: #999; margin-bottom: 10px; }
    #drawing-canvas {
      display: block; width: 100%;
      border: 2px dashed #ddd; border-radius: 12px;
      background: #fafafa; touch-action: none; cursor: crosshair;
    }
  </style>
</head>
<body>

<div class="header">
  <h1>What will you draw?</h1>
  <p>Tap a picture to choose your subject</p>
</div>

<div class="grid" id="grid"></div>

<div id="canvas-area">
  <p class="cue">Try to draw it here ✏️</p>
  <canvas id="drawing-canvas" width="600" height="420"></canvas>
</div>

<script>
const FIREBASE_URL = 'https://magictrick-5fc12-default-rtdb.firebaseio.com';

const SUBJECTS = [
  { key: 'house', label: 'House', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
    <polyline points="50,6 95,44 5,44"/>
    <rect x="12" y="44" width="76" height="40"/>
    <path d="M40,84 V65 Q50,57 60,65 V84"/>
    <rect x="16" y="51" width="19" height="15"/>
    <line x1="25.5" y1="51" x2="25.5" y2="66"/>
    <line x1="16" y1="58.5" x2="35" y2="58.5"/>
    <rect x="65" y="51" width="19" height="15"/>
    <line x1="74.5" y1="51" x2="74.5" y2="66"/>
    <line x1="65" y1="58.5" x2="84" y2="58.5"/>
  </svg>` },
  { key: 'flower', label: 'Flower', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round">
    <ellipse cx="50" cy="25" rx="9" ry="13"/>
    <ellipse cx="50" cy="53" rx="9" ry="13"/>
    <ellipse cx="31" cy="29" rx="9" ry="13" transform="rotate(-60 31 29)"/>
    <ellipse cx="69" cy="29" rx="9" ry="13" transform="rotate(60 69 29)"/>
    <ellipse cx="31" cy="49" rx="9" ry="13" transform="rotate(60 31 49)"/>
    <ellipse cx="69" cy="49" rx="9" ry="13" transform="rotate(-60 69 49)"/>
    <circle cx="50" cy="39" r="10" fill="#fff" stroke="#444" stroke-width="2.5"/>
    <path d="M50,49 Q51,64 50,80"/>
    <path d="M50,62 Q39,58 34,65"/>
    <path d="M50,62 Q61,58 66,65"/>
  </svg>` },
  { key: 'tree', label: 'Tree', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
    <path d="M50,6 Q80,8 83,30 Q88,52 68,57 Q60,65 50,59 Q40,65 32,57 Q12,52 17,30 Q20,8 50,6 Z"/>
    <rect x="43" y="56" width="14" height="28"/>
  </svg>` },
  { key: 'car', label: 'Car', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
    <path d="M8,54 L8,66 Q8,72 14,72 L86,72 Q92,72 92,66 L92,54"/>
    <path d="M14,54 L23,36 Q25,32 31,32 L69,32 Q75,32 77,36 L86,54 Z"/>
    <circle cx="26" cy="72" r="9"/>
    <circle cx="74" cy="72" r="9"/>
    <rect x="28" y="35" width="17" height="14" rx="2"/>
    <rect x="55" y="35" width="17" height="14" rx="2"/>
  </svg>` },
  { key: 'airplane', label: 'Airplane', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
    <path d="M8,44 Q32,34 56,42 L90,42 Q97,42 97,46 Q97,50 90,50 L56,50 Q32,56 8,46 Z"/>
    <path d="M22,42 L32,26 L44,26 L36,42"/>
    <path d="M22,50 L32,66 L44,66 L36,50"/>
    <path d="M72,42 L76,34 L84,34 L80,42"/>
    <circle cx="84" cy="46" r="4" fill="#444"/>
  </svg>` },
  { key: 'star', label: 'Star', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
    <polygon points="50,5 61,35 93,35 68,54 77,85 50,66 23,85 32,54 7,35 39,35"/>
  </svg>` },
  { key: 'sun', label: 'Sun', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round">
    <circle cx="50" cy="45" r="18"/>
    <line x1="50" y1="6" x2="50" y2="19"/>
    <line x1="50" y1="71" x2="50" y2="84"/>
    <line x1="11" y1="45" x2="24" y2="45"/>
    <line x1="76" y1="45" x2="89" y2="45"/>
    <line x1="22" y1="17" x2="31" y2="26"/>
    <line x1="69" y1="64" x2="78" y2="73"/>
    <line x1="78" y1="17" x2="69" y2="26"/>
    <line x1="31" y1="64" x2="22" y2="73"/>
  </svg>` },
  { key: 'heart', label: 'Heart', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
    <path d="M50,76 L12,42 Q4,18 23,13 Q38,8 50,28 Q62,8 77,13 Q96,18 88,42 Z"/>
  </svg>` },
  { key: 'smiley', label: 'Smiley Face', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round">
    <circle cx="50" cy="45" r="36"/>
    <circle cx="37" cy="38" r="3.5" fill="#444"/>
    <circle cx="63" cy="38" r="3.5" fill="#444"/>
    <path d="M31,56 Q50,72 69,56"/>
  </svg>` },
  { key: 'clock', label: 'Clock Face', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round">
    <circle cx="50" cy="45" r="36"/>
    <line x1="50" y1="45" x2="34" y2="22"/>
    <line x1="50" y1="45" x2="70" y2="34"/>
    <circle cx="50" cy="45" r="3" fill="#444"/>
    <line x1="50" y1="10" x2="50" y2="17"/>
    <line x1="50" y1="73" x2="50" y2="80"/>
    <line x1="14" y1="45" x2="21" y2="45"/>
    <line x1="79" y1="45" x2="86" y2="45"/>
  </svg>` },
  { key: 'stickman', label: 'Stickman', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round">
    <circle cx="50" cy="14" r="11"/>
    <line x1="50" y1="25" x2="50" y2="58"/>
    <line x1="50" y1="36" x2="28" y2="24"/>
    <line x1="50" y1="36" x2="72" y2="24"/>
    <line x1="50" y1="58" x2="33" y2="82"/>
    <line x1="50" y1="58" x2="67" y2="82"/>
  </svg>` },
  { key: 'rocket', label: 'Rocket', svg: `<svg viewBox="0 0 100 90" fill="none" stroke="#444" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
    <path d="M50,4 Q66,14 66,40 L66,62 Q58,68 50,68 Q42,68 34,62 L34,40 Q34,14 50,4 Z"/>
    <path d="M34,56 L20,70 L20,62 L34,62"/>
    <path d="M66,56 L80,70 L80,62 L66,62"/>
    <path d="M42,68 Q50,83 58,68"/>
    <circle cx="50" cy="30" r="7"/>
  </svg>` },
];

// ── Build grid ────────────────────────────────────────────────────
const grid = document.getElementById('grid');
SUBJECTS.forEach(s => {
  const btn = document.createElement('button');
  btn.className = 'subject-btn';
  btn.dataset.key = s.key;
  btn.innerHTML = s.svg + `<span class="lbl">${s.label}</span>`;
  btn.addEventListener('click', () => selectSubject(s.key, btn));
  grid.appendChild(btn);
});

// ── Selection + Firebase ──────────────────────────────────────────
let selectedBtn = null;

function selectSubject(key, btn) {
  if (selectedBtn) selectedBtn.classList.remove('selected');
  btn.classList.add('selected');
  selectedBtn = btn;

  fetch(`${FIREBASE_URL}/learnToDraw.json`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ selection: key })
  }).catch(() => {});

  const area = document.getElementById('canvas-area');
  area.style.display = 'block';
  clearCanvas();
  area.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
}

// ── Drawing canvas ────────────────────────────────────────────────
const canvas = document.getElementById('drawing-canvas');
const ctx = canvas.getContext('2d');
let drawing = false;

ctx.strokeStyle = '#222';
ctx.lineWidth = 4;
ctx.lineCap = 'round';
ctx.lineJoin = 'round';

function clearCanvas() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
}

function getPos(e) {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  const src = e.touches ? e.touches[0] : e;
  return { x: (src.clientX - rect.left) * scaleX, y: (src.clientY - rect.top) * scaleY };
}

canvas.addEventListener('mousedown', e => { drawing = true; ctx.beginPath(); const p = getPos(e); ctx.moveTo(p.x, p.y); });
canvas.addEventListener('mousemove', e => { if (!drawing) return; const p = getPos(e); ctx.lineTo(p.x, p.y); ctx.stroke(); });
canvas.addEventListener('mouseup', () => { drawing = false; });
canvas.addEventListener('mouseleave', () => { drawing = false; });
canvas.addEventListener('touchstart', e => { e.preventDefault(); drawing = true; ctx.beginPath(); const p = getPos(e); ctx.moveTo(p.x, p.y); }, { passive: false });
canvas.addEventListener('touchmove', e => { e.preventDefault(); if (!drawing) return; const p = getPos(e); ctx.lineTo(p.x, p.y); ctx.stroke(); }, { passive: false });
canvas.addEventListener('touchend', () => { drawing = false; });
</script>
</body>
</html>
```

- [ ] **Step 2: Open `index.html` directly in browser to verify**

```bash
open /Users/carlandrews/Downloads/LearnToDraw/index.html
```

Expected: white page, "What will you draw?" heading, 4×3 grid of 12 picture buttons with recognizable SVG icons and labels. Tap/click a button — it gets a gold border, drawing canvas appears below. Can draw on canvas with finger or mouse. Tapping a different button moves the gold border and clears the canvas.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add spectator page with 12 picture buttons, Firebase write, drawing canvas"
```

---

## Task 3: Receiver Page (receiver.html)

**Files:**
- Create: `receiver.html`

- [ ] **Step 1: Create `receiver.html` with complete phone app**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black">
  <title>SketchPad</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
    html, body { height: 100%; background: #1c1c1e; overflow: hidden; }

    #app {
      display: flex; flex-direction: column; height: 100vh;
      max-width: 480px; margin: 0 auto;
    }

    /* Top toolbar */
    #toolbar-top {
      background: #2c2c2e;
      padding: 10px 14px;
      display: flex; align-items: center; justify-content: space-between;
      border-bottom: 1px solid #3a3a3c;
      flex-shrink: 0;
    }
    .app-title { color: #fff; font-family: -apple-system, sans-serif; font-size: 16px; font-weight: 600; }
    .tool-icons { display: flex; gap: 8px; }
    .tool-icon {
      width: 34px; height: 34px; background: #3a3a3c; border-radius: 8px;
      display: flex; align-items: center; justify-content: center;
      font-size: 16px; color: #ccc; cursor: default;
    }

    /* Canvas area */
    #canvas-area {
      flex: 1; background: #f5f0e8;
      display: flex; align-items: center; justify-content: center;
      overflow: hidden; position: relative;
    }
    #sketch-wrap {
      width: min(90vw, 380px);
      height: min(60vh, 380px);
      display: flex; align-items: center; justify-content: center;
    }
    #sketch-wrap svg { width: 100%; height: 100%; }

    /* Bottom toolbar */
    #toolbar-bottom {
      background: #2c2c2e;
      padding: 10px 14px;
      display: flex; align-items: center; justify-content: space-between;
      border-top: 1px solid #3a3a3c;
      flex-shrink: 0;
    }
    .swatches { display: flex; gap: 8px; align-items: center; }
    .swatch { width: 22px; height: 22px; border-radius: 50%; border: 2px solid #555; }
    #erase-btn {
      background: #c0392b; color: #fff;
      border: none; border-radius: 10px;
      padding: 8px 18px; font-size: 14px; font-weight: 700;
      cursor: pointer; font-family: -apple-system, sans-serif;
      transition: opacity 0.15s;
    }
    #erase-btn:active { opacity: 0.75; }
  </style>
</head>
<body>
<div id="app">
  <div id="toolbar-top">
    <span class="app-title">✏️ SketchPad</span>
    <div class="tool-icons">
      <div class="tool-icon">🖊</div>
      <div class="tool-icon">🖌</div>
      <div class="tool-icon">◯</div>
    </div>
  </div>

  <div id="canvas-area">
    <div id="sketch-wrap"></div>
  </div>

  <div id="toolbar-bottom">
    <div class="swatches">
      <div class="swatch" style="background:#2c2c2c"></div>
      <div class="swatch" style="background:#6b4f3a"></div>
      <div class="swatch" style="background:#888"></div>
    </div>
    <button id="erase-btn">🗑 Erase</button>
  </div>
</div>

<script>
const FIREBASE_URL = 'https://magictrick-5fc12-default-rtdb.firebaseio.com';

// ── Hand-drawn SVG sketches ───────────────────────────────────────
// Each uses wobbly bezier curves + ghost duplicate strokes for pencil feel.
// ViewBox 0 0 160 150. stroke-linecap/join round throughout.

const SKETCHES = {

house: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round" stroke-linejoin="round">
    <path d="M80,16 Q52,44 21,73" stroke-width="3"/>
    <path d="M80,16 Q108,44 139,73" stroke-width="3"/>
    <path d="M21,73 Q21,106 22,138 Q80,139 138,137 Q139,104 139,73" stroke-width="3"/>
    <path d="M80,16 Q52,44 20,72" stroke-width="1.2" opacity="0.25"/>
    <path d="M80,16 Q109,45 140,72" stroke-width="1.2" opacity="0.25"/>
    <path d="M20,72 L20,139 L140,138 L140,72" stroke-width="1" opacity="0.2"/>
    <path d="M62,138 Q61,112 62,97 Q71,87 80,92 Q89,87 98,97 Q99,114 98,138" stroke-width="2.5"/>
    <path d="M27,89 Q27,89 28,102 Q36,103 49,102 Q50,95 49,89 Q37,88 27,89" stroke-width="2"/>
    <line x1="38" y1="89" x2="38.5" y2="102" stroke-width="1.8"/>
    <line x1="27" y1="95.5" x2="49" y2="96" stroke-width="1.8"/>
    <path d="M111,89 Q111,89 112,102 Q120,103 133,102 Q134,95 133,89 Q121,88 111,89" stroke-width="2"/>
    <line x1="122" y1="89" x2="122.5" y2="102" stroke-width="1.8"/>
    <line x1="111" y1="95.5" x2="133" y2="96" stroke-width="1.8"/>
  </g>
</svg>`,

flower: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round">
    <ellipse cx="80" cy="38" rx="14" ry="20" stroke-width="2.5"/>
    <ellipse cx="80" cy="38" rx="14" ry="20" stroke-width="1" opacity="0.2" transform="translate(1.5,1)"/>
    <ellipse cx="80" cy="86" rx="14" ry="20" stroke-width="2.5"/>
    <ellipse cx="80" cy="86" rx="14" ry="20" stroke-width="1" opacity="0.2" transform="translate(-1,1)"/>
    <ellipse cx="50" cy="45" rx="14" ry="20" transform="rotate(-60 50 45)" stroke-width="2.5"/>
    <ellipse cx="110" cy="45" rx="14" ry="20" transform="rotate(60 110 45)" stroke-width="2.5"/>
    <ellipse cx="50" cy="77" rx="14" ry="20" transform="rotate(60 50 77)" stroke-width="2.5"/>
    <ellipse cx="110" cy="77" rx="14" ry="20" transform="rotate(-60 110 77)" stroke-width="2.5"/>
    <circle cx="80" cy="62" r="16" fill="#f5f0e8" stroke="#2c2c2c" stroke-width="2.8"/>
    <circle cx="80" cy="62" r="16" stroke-width="1" opacity="0.2" transform="translate(1,0.5)"/>
    <path d="M80,78 Q81,102 80,128" stroke-width="2.5"/>
    <path d="M80,102 Q64,96 58,106" stroke-width="2"/>
    <path d="M80,102 Q96,96 102,106" stroke-width="2"/>
  </g>
</svg>`,

tree: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round" stroke-linejoin="round">
    <path d="M80,10 Q116,14 122,44 Q130,76 108,88 Q96,100 80,94 Q64,100 52,88 Q30,76 38,44 Q44,14 80,10 Z" stroke-width="3"/>
    <path d="M80,10 Q118,16 124,46 Q132,78 110,90 Q96,101 80,95" stroke-width="1" opacity="0.2"/>
    <path d="M71,91 Q70,112 71,136 L89,136 Q90,112 89,91" stroke-width="2.8"/>
    <path d="M70,90 Q69,112 70,137 L90,137" stroke-width="1" opacity="0.2"/>
  </g>
</svg>`,

car: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round" stroke-linejoin="round">
    <path d="M12,88 Q12,88 13,106 Q13,116 22,116 L138,115 Q147,115 147,105 L148,88" stroke-width="3"/>
    <path d="M12,87 Q13,107 14,117 L138,116" stroke-width="1" opacity="0.2"/>
    <path d="M22,88 L34,56 Q37,50 46,50 L114,50 Q123,50 126,56 L138,88 Z" stroke-width="3"/>
    <path d="M22,88 L35,57 Q38,51 47,51 L113,51" stroke-width="1" opacity="0.2"/>
    <circle cx="40" cy="116" r="16" stroke-width="2.8"/>
    <circle cx="40" cy="116" r="16" stroke-width="1" opacity="0.2" transform="translate(1,0.5)"/>
    <circle cx="120" cy="116" r="16" stroke-width="2.8"/>
    <circle cx="120" cy="116" r="16" stroke-width="1" opacity="0.2" transform="translate(-0.5,1)"/>
    <path d="M44,54 Q44,54 45,68 Q58,69 72,68 Q73,61 72,54 Q58,53 44,54" stroke-width="2.2"/>
    <path d="M88,54 Q88,54 89,68 Q102,69 116,68 Q117,61 116,54 Q102,53 88,54" stroke-width="2.2"/>
  </g>
</svg>`,

airplane: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round" stroke-linejoin="round">
    <path d="M10,72 Q46,56 86,66 L142,66 Q154,66 154,74 Q154,82 142,82 L86,82 Q46,90 10,74 Z" stroke-width="3"/>
    <path d="M10,72 Q46,56 86,66 L142,66" stroke-width="1" opacity="0.2"/>
    <path d="M32,66 L46,40 L64,40 L54,66" stroke-width="2.8"/>
    <path d="M32,67 L47,41 L65,41" stroke-width="1" opacity="0.2"/>
    <path d="M32,82 L46,108 L64,108 L54,82" stroke-width="2.8"/>
    <path d="M112,66 L118,52 L132,52 L126,66" stroke-width="2.5"/>
    <circle cx="136" cy="74" r="7" fill="#2c2c2c"/>
    <circle cx="136" cy="74" r="7" stroke-width="1.5" opacity="0.25"/>
  </g>
</svg>`,

star: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round" stroke-linejoin="round">
    <path d="M80,10 Q85,38 93,52 L148,54 Q126,72 112,84 Q120,114 124,136 Q102,120 80,108 Q58,120 36,136 Q40,114 48,84 Q34,72 12,54 L67,52 Q75,38 80,10 Z" stroke-width="3"/>
    <path d="M80,10 Q85,40 94,53 L150,55" stroke-width="1" opacity="0.2"/>
    <path d="M12,54 L68,53 Q75,39 80,10" stroke-width="1" opacity="0.18"/>
  </g>
</svg>`,

sun: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round">
    <circle cx="80" cy="74" r="30" stroke-width="3"/>
    <circle cx="80" cy="74" r="30" stroke-width="1" opacity="0.2" transform="translate(1,0.5)"/>
    <line x1="80" y1="10" x2="80.5" y2="28" stroke-width="2.8"/>
    <line x1="80" y1="120" x2="79.5" y2="138" stroke-width="2.8"/>
    <line x1="16" y1="74" x2="34" y2="73.5" stroke-width="2.8"/>
    <line x1="126" y1="74" x2="144" y2="74.5" stroke-width="2.8"/>
    <line x1="34" y1="28" x2="47" y2="41" stroke-width="2.8"/>
    <line x1="113" y1="107" x2="126" y2="120" stroke-width="2.8"/>
    <line x1="126" y1="28" x2="113" y2="41" stroke-width="2.8"/>
    <line x1="47" y1="107" x2="34" y2="120" stroke-width="2.8"/>
  </g>
</svg>`,

heart: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round" stroke-linejoin="round">
    <path d="M80,124 Q46,96 20,66 Q6,28 36,20 Q56,14 80,42 Q104,14 124,20 Q154,28 140,66 Q114,96 80,124 Z" stroke-width="3"/>
    <path d="M80,124 Q44,95 18,64 Q4,26 36,18" stroke-width="1" opacity="0.2"/>
    <path d="M80,42 Q104,14 126,20 Q156,30 140,68" stroke-width="1" opacity="0.2"/>
  </g>
</svg>`,

smiley: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round">
    <circle cx="80" cy="74" r="58" stroke-width="3"/>
    <circle cx="80" cy="74" r="58" stroke-width="1" opacity="0.2" transform="translate(1.5,1)"/>
    <circle cx="58" cy="60" r="6" fill="#2c2c2c"/>
    <circle cx="102" cy="60" r="6" fill="#2c2c2c"/>
    <path d="M46,90 Q80,118 114,90" stroke-width="3"/>
    <path d="M46,90 Q80,120 114,90" stroke-width="1" opacity="0.2"/>
  </g>
</svg>`,

clock: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round">
    <circle cx="80" cy="74" r="58" stroke-width="3"/>
    <circle cx="80" cy="74" r="58" stroke-width="1" opacity="0.2" transform="translate(1,0.8)"/>
    <line x1="80" y1="74" x2="54" y2="38" stroke-width="3.5"/>
    <line x1="80" y1="74" x2="112" y2="56" stroke-width="2.8"/>
    <circle cx="80" cy="74" r="5" fill="#2c2c2c"/>
    <line x1="80" y1="17" x2="80.5" y2="26" stroke-width="2.5"/>
    <line x1="80" y1="122" x2="79.5" y2="131" stroke-width="2.5"/>
    <line x1="23" y1="74" x2="32" y2="74.5" stroke-width="2.5"/>
    <line x1="128" y1="74" x2="137" y2="73.5" stroke-width="2.5"/>
    <line x1="38" y1="30" x2="44" y2="36" stroke-width="2"/>
    <line x1="122" y1="30" x2="116" y2="36" stroke-width="2"/>
    <line x1="38" y1="118" x2="44" y2="112" stroke-width="2"/>
    <line x1="122" y1="118" x2="116" y2="112" stroke-width="2"/>
  </g>
</svg>`,

stickman: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round">
    <circle cx="80" cy="22" r="18" stroke-width="3"/>
    <circle cx="80" cy="22" r="18" stroke-width="1" opacity="0.2" transform="translate(1,0.5)"/>
    <line x1="80" y1="40" x2="80.5" y2="94" stroke-width="3"/>
    <line x1="79" y1="40" x2="79" y2="94" stroke-width="1" opacity="0.2"/>
    <line x1="80" y1="58" x2="44" y2="38" stroke-width="3"/>
    <line x1="80" y1="58" x2="116" y2="38" stroke-width="3"/>
    <line x1="80" y1="94" x2="52" y2="132" stroke-width="3"/>
    <line x1="80" y1="94" x2="108" y2="132" stroke-width="3"/>
    <line x1="81" y1="95" x2="53" y2="133" stroke-width="1" opacity="0.2"/>
  </g>
</svg>`,

rocket: `<svg viewBox="0 0 160 150" fill="none" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#2c2c2c" stroke-linecap="round" stroke-linejoin="round">
    <path d="M80,6 Q106,20 106,62 L106,100 Q94,110 80,110 Q66,110 54,100 L54,62 Q54,20 80,6 Z" stroke-width="3"/>
    <path d="M80,6 Q108,22 108,64 L108,102" stroke-width="1" opacity="0.2"/>
    <path d="M54,90 L34,114 L34,102 L54,100" stroke-width="2.8"/>
    <path d="M106,90 L126,114 L126,102 L106,100" stroke-width="2.8"/>
    <path d="M66,110 Q80,132 94,110" stroke-width="2.8"/>
    <path d="M66,110 Q80,134 94,110" stroke-width="1" opacity="0.2"/>
    <circle cx="80" cy="48" r="12" stroke-width="2.5"/>
    <circle cx="80" cy="48" r="12" stroke-width="1" opacity="0.2" transform="translate(0.5,0.5)"/>
  </g>
</svg>`,
};

// ── Firebase SSE ──────────────────────────────────────────────────
const wrap = document.getElementById('sketch-wrap');
let currentKey = null;
let es = null;

function showSketch(key) {
  currentKey = key;
  wrap.innerHTML = key && SKETCHES[key] ? SKETCHES[key] : '';
}

function startListening() {
  if (es) es.close();
  es = new EventSource(`${FIREBASE_URL}/learnToDraw/selection.json`);

  es.addEventListener('put', e => {
    try {
      const val = JSON.parse(e.data).data;
      showSketch(val || null);
    } catch {}
  });

  es.addEventListener('patch', e => {
    try {
      const val = JSON.parse(e.data).data;
      showSketch(val || null);
    } catch {}
  });

  es.onerror = () => {
    es.close();
    setTimeout(startListening, 3000);
  };
}

// ── Erase ─────────────────────────────────────────────────────────
document.getElementById('erase-btn').addEventListener('click', () => {
  fetch(`${FIREBASE_URL}/learnToDraw.json`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ selection: null })
  }).catch(() => {});
  showSketch(null);
});

// ── Reset on tab show (re-read Firebase state) ────────────────────
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') startListening();
});

startListening();
</script>
</body>
</html>
```

- [ ] **Step 2: Open `receiver.html` directly in browser to verify the UI**

```bash
open /Users/carlandrews/Downloads/LearnToDraw/receiver.html
```

Expected: dark top toolbar with "✏️ SketchPad" title and fake tool icons, warm paper-coloured canvas area (blank), dark bottom toolbar with 3 colour swatches and red Erase button. Nothing should crash.

- [ ] **Step 3: Live test — open both pages and verify real-time sync**

Open two browser tabs:
- Tab 1: `file:///Users/carlandrews/Downloads/LearnToDraw/index.html`
- Tab 2: `file:///Users/carlandrews/Downloads/LearnToDraw/receiver.html`

In Tab 1, tap **House**. Expected: receiver instantly shows a hand-drawn house sketch on the paper canvas.
Tap **Rocket**. Expected: receiver instantly swaps to rocket sketch.
In Tab 2, tap **Erase**. Expected: canvas goes blank in both tabs (Firebase is cleared).

- [ ] **Step 4: Commit**

```bash
git add receiver.html
git commit -m "Add receiver page with fake drawing app UI, 12 hand-drawn sketches, SSE listener"
```

---

## Task 4: GitHub Repo + Vercel Deploy + Smoke Test

- [ ] **Step 1: Create GitHub repo and push**

```bash
cd /Users/carlandrews/Downloads/LearnToDraw
gh repo create LearnToDraw --public --source=. --remote=origin --push
```

Expected: repo created at `https://github.com/MagicCarl/LearnToDraw`, all commits pushed.

- [ ] **Step 2: Link to Vercel and deploy**

```bash
npx vercel link --yes --scope carl-andrews-projects-2878b08a --project learn-to-draw
npx vercel --prod --yes --scope carl-andrews-projects-2878b08a
```

Expected: deployment completes. Note the production URL — it will be `https://learn-to-draw.vercel.app`.

- [ ] **Step 3: Smoke test — spectator URL**

Open `https://learn-to-draw.vercel.app` on any device. Expected: "What will you draw?" heading, 12 picture buttons, drawing canvas appears on tap.

- [ ] **Step 4: Smoke test — receiver URL on your phone**

Open `https://learn-to-draw.vercel.app/receiver` on your phone. Bookmark it. Expected: fake drawing app UI loads, canvas is blank.

- [ ] **Step 5: End-to-end test**

With receiver open on your phone and spectator URL open on another device:
- Tap **Star** on spectator → star sketch appears on phone instantly
- Tap **Heart** → heart sketch replaces it instantly
- Tap **Erase** on phone → canvas blanks, ready for next performance

- [ ] **Step 6: Push Vercel config**

```bash
git add .vercel/ 2>/dev/null
git commit -m "Add Vercel project config" 2>/dev/null || true
git push origin main
```
