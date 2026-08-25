# Sprint
A game that makes learning fun. Paste the file with what you want to learn and play!
[README.md](https://github.com/user-attachments/files/31417262/README.md)
# SPRINT

Turn any notes into a race. Upload a `.txt`, `.md`, or `.pdf` file (or paste text), and SPRINT
splits it into **legs**. Each leg opens with the passage to read, then drops you into a
three-lane dodge-run. Obstacles you can dodge; **Splits** you can't — they're checkpoint
questions pulled from what you just read, and you have to answer one correctly to keep running.
Get it wrong and SPRINT shows you the right answer and the sentence it came from before you
move on. Clear every leg, then survive the **Photo Finish**, a mixed review of everything.

It's a single static HTML file. No build step, no server required to run it, no framework.

## What's included

- `index.html` — the whole game: UI, canvas gameplay, an offline question generator, and an
  optional AI hint system.
- `functions/api/hint.js` — an optional serverless proxy (Cloudflare Pages Functions) so AI
  hints can run in a real deployment without exposing an API key in the browser.
- `LICENSE` — MIT.

## Run it right now

Just open `index.html` in a browser. Typed or pasted text works completely offline. PDF
uploads need an internet connection the first time, just to load a PDF-reading library from a
CDN — your document's contents never leave the browser either way.

## Deploy it for real, with HTTPS

SPRINT is a static site, so any static host gives you a free, valid HTTPS certificate
automatically — that's what makes a browser treat it as secure. Three ways to do it, in order
of simplicity:

**GitHub Pages**
1. Create a new public repo and push these files to it (this is also what makes it "open
   source" in practice — a public repo with the MIT license attached).
2. Repo Settings → Pages → set the source to your main branch.
3. GitHub gives you `https://<you>.github.io/<repo>/` with HTTPS already on.

**Cloudflare Pages / Netlify / Vercel**
Drag the project folder into any of their dashboards, or connect the GitHub repo. All three
auto-provision HTTPS the same way. Cloudflare Pages is the one used below for AI hints,
because its **Pages Functions** feature deploys `functions/api/hint.js` as a serverless
endpoint automatically, on the same domain as your site.

## Turning on AI hints (optional)

The "Get a Hint" button works two ways:

- **Previewing inside Claude.ai** — it calls Anthropic's API directly through the proxy
  Claude.ai provides for artifacts. No setup, no key. This *only* works inside a Claude.ai
  preview, not once you deploy the file elsewhere.
- **A real deployment** — needs its own backend, because a static site can't hold a secret API
  key without exposing it to anyone who opens dev tools. `functions/api/hint.js` is that
  backend, ready to go:
  1. Deploy to Cloudflare Pages with the `functions/` folder intact.
  2. In the Pages dashboard: Settings → Environment variables → add `ANTHROPIC_API_KEY` as
     **encrypted**. Get a key from your Anthropic Console.
  3. In SPRINT, open Settings (gear icon) and set the hint proxy URL to
     `https://your-site.pages.dev/api/hint`.

If you'd rather use Netlify or Vercel, the same idea applies — adapt `hint.js` to their
function format and point the same settings field at your function's URL. If you skip this
entirely, the game is fully playable — hints just won't be available.

**The boundary, by design:** the hint system is instructed to never state or imply the correct
option, only to nudge conceptually, and to decline if asked directly for the answer. That's
enforced twice — once in the system prompt, and once again by a plain string check that
throws away the hint if it happens to contain an option's exact text. It only ever sees the
passage, the question, and the answer *options* — never which one is correct.

## How the offline question generator works

No AI required for the core game. SPRINT splits your text into legs (by Markdown headers if
present, otherwise by paragraph groups), then, for each leg, picks sentences with a strong
"key term" — a proper noun, a number, or (as a fallback) the longest distinctive word — and
blanks it out as a fill-in-the-blank question, with distractors drawn from other terms in your
document. It's a simple heuristic, not language understanding, so it works best on fact-dense
material (names, dates, defined terms) and can occasionally produce an awkward question on very
short or very abstract text. Connecting AI hints doesn't change question generation — only
v1's one AI touchpoint, hints, uses it.

## License

MIT — see `LICENSE`. Do whatever you'd like with it.



[index.html](https://github.com/user-attachments/files/31417300/index.html)
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>SPRINT — turn your notes into a race</title>
<meta name="description" content="Upload any notes and SPRINT turns them into a lane-dodging race. Answer checkpoints correctly to keep running.">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Anton&family=IBM+Plex+Mono:wght@400;500;600;700&family=IBM+Plex+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">
<style>
  :root{
    --navy:#14213D;
    --navy-deep:#0E1830;
    --chalk:#F5F3EE;
    --amber:#FFB627;
    --go:#2ED9A3;
    --stop:#E8493A;
    --steel:#3A5F8A;
    --display: 'Anton', 'Arial Narrow', sans-serif;
    --body: 'IBM Plex Sans', system-ui, -apple-system, sans-serif;
    --mono: 'IBM Plex Mono', 'Courier New', monospace;
    --radius: 10px;
  }
  *{box-sizing:border-box;}
  html,body{margin:0;padding:0;}
  body{
    background:var(--navy);
    color:var(--chalk);
    font-family:var(--body);
    min-height:100vh;
    display:flex;
    flex-direction:column;
    align-items:center;
  }
  body::before{
    content:"";
    position:fixed; inset:0;
    background:
      radial-gradient(circle at 15% 8%, rgba(255,182,39,0.08), transparent 40%),
      radial-gradient(circle at 85% 92%, rgba(46,217,163,0.07), transparent 40%);
    pointer-events:none;
    z-index:0;
  }
  h1,h2,h3{font-family:var(--display); font-weight:400; letter-spacing:0.02em; margin:0;}
  button{font-family:var(--body); cursor:pointer;}
  button:focus-visible, input:focus-visible, textarea:focus-visible, a:focus-visible{
    outline:3px solid var(--amber);
    outline-offset:2px;
  }
  .hidden{display:none !important;}
  .wrap{
    width:100%; max-width:880px;
    padding:0 16px 40px;
    position:relative; z-index:1;
  }

  /* ---------- signature progress strip ---------- */
  #progressStrip{
    width:100%;
    display:flex;
    justify-content:center;
    gap:6px;
    padding:14px 10px;
    background:var(--navy-deep);
    border-bottom:2px solid var(--steel);
    position:relative; z-index:2;
    flex-wrap:wrap;
  }
  .seg{
    font-family:var(--mono);
    font-size:12px;
    font-weight:600;
    color:var(--steel);
    border:2px solid var(--steel);
    border-radius:5px;
    padding:3px 8px;
    min-width:22px;
    text-align:center;
    transition:all .2s ease;
  }
  .seg-current{ color:var(--navy-deep); background:var(--amber); border-color:var(--amber); }
  .seg-done{ color:var(--navy-deep); background:var(--go); border-color:var(--go); }

  /* ---------- top bar ---------- */
  .topbar{
    width:100%; max-width:880px;
    display:flex; justify-content:space-between; align-items:center;
    padding:14px 16px 0;
  }
  .brand{
    font-family:var(--display);
    font-size:22px;
    letter-spacing:0.04em;
  }
  .brand .flag{color:var(--amber);}
  .iconbtn{
    background:transparent; border:2px solid var(--steel); color:var(--chalk);
    border-radius:8px; width:38px; height:38px; font-size:16px;
    display:flex; align-items:center; justify-content:center;
  }
  .iconbtn:hover{border-color:var(--amber);}

  /* ---------- generic screen card ---------- */
  .screen{ padding-top:20px; }
  .card{
    background:var(--navy-deep);
    border:2px solid var(--steel);
    border-radius:var(--radius);
    padding:24px;
  }
  .hero-title{
    font-size:clamp(48px,11vw,84px);
    line-height:0.95;
  }
  .tagline{
    font-size:clamp(15px,2.4vw,18px);
    color:var(--chalk);
    opacity:0.85;
    margin-top:6px;
    max-width:46ch;
  }
  .grid2{
    display:grid;
    grid-template-columns:1.1fr 1fr;
    gap:18px;
    margin-top:26px;
  }
  @media (max-width:680px){ .grid2{grid-template-columns:1fr;} }

  .tabbtns{ display:flex; gap:8px; margin-bottom:12px; }
  .tabbtn{
    flex:1; background:transparent; border:2px solid var(--steel); color:var(--chalk);
    border-radius:8px; padding:8px 10px; font-size:13px; font-weight:600;
  }
  .tabbtn.active{ background:var(--steel); }

  input[type=file]{
    width:100%; color:var(--chalk); font-family:var(--body); font-size:13px;
  }
  textarea{
    width:100%; min-height:140px; resize:vertical;
    background:var(--navy); color:var(--chalk);
    border:2px solid var(--steel); border-radius:8px;
    padding:10px; font-family:var(--body); font-size:14px;
  }
  .sample-link{
    background:none; border:none; color:var(--go); text-decoration:underline;
    font-size:13px; padding:0; margin-top:10px; display:inline-block;
  }

  .howto li{ margin-bottom:8px; font-size:14px; line-height:1.5; }
  .howto{ padding-left:20px; margin:10px 0 0; }

  .btn{
    font-family:var(--display);
    letter-spacing:0.03em;
    font-size:17px;
    background:var(--amber);
    color:var(--navy-deep);
    border:none;
    border-radius:8px;
    padding:14px 22px;
    width:100%;
    transition:transform .12s ease, box-shadow .12s ease;
  }
  .btn:hover:not(:disabled){ transform:translateY(-1px); box-shadow:0 6px 0 rgba(0,0,0,0.25); }
  .btn:active:not(:disabled){ transform:translateY(1px); box-shadow:none; }
  .btn:disabled{ opacity:0.4; cursor:not-allowed; }
  .btn-secondary{ background:var(--steel); color:var(--chalk); }
  .btn-ghost{
    background:transparent; border:2px solid var(--steel); color:var(--chalk);
    font-family:var(--body); font-weight:600; font-size:14px; border-radius:8px; padding:10px 16px;
  }
  .btn-row{ display:flex; gap:10px; margin-top:16px; }
  .btn-row .btn, .btn-row .btn-ghost{ width:auto; flex:1; }

  #startError{ color:var(--stop); font-size:13px; margin-top:10px; min-height:16px; }
  #startError.status{ color:var(--amber); }

  #resumeBanner{
    background:var(--steel); border-radius:8px; padding:12px 14px;
    display:flex; justify-content:space-between; align-items:center; gap:10px;
    margin-bottom:18px; flex-wrap:wrap;
  }
  #resumeBanner p{ margin:0; font-size:14px; }

  /* ---------- leg intro ---------- */
  .leg-label{ font-family:var(--mono); color:var(--amber); font-size:13px; letter-spacing:0.08em; }
  #legIntroTitle{ font-size:clamp(30px,6vw,44px); margin-top:4px; }
  #legIntroBody{
    margin-top:16px; line-height:1.7; font-size:15.5px;
    max-height:38vh; overflow-y:auto;
    background:var(--navy); border:1px solid var(--steel); border-radius:8px; padding:16px;
    white-space:pre-wrap;
  }
  .skip-link{
    background:none; border:none; color:var(--chalk); opacity:0.6;
    text-decoration:underline; font-size:12px; margin-top:12px; padding:4px;
  }

  /* ---------- game screen ---------- */
  .hud{
    display:flex; justify-content:space-between; align-items:center;
    font-family:var(--mono); font-size:13px; font-weight:600;
    background:var(--navy-deep); border:2px solid var(--steel); border-bottom:none;
    border-radius:var(--radius) var(--radius) 0 0;
    padding:10px 14px;
    flex-wrap:wrap; gap:8px;
  }
  .hud .dots{ display:inline-flex; gap:4px; margin-left:6px; vertical-align:middle; }
  .dot{ width:10px; height:10px; border-radius:50%; background:var(--steel); display:inline-block; }
  .dot.on{ background:var(--stop); }
  .canvas-wrap{
    position:relative;
    border:2px solid var(--steel);
    border-radius:0 0 var(--radius) var(--radius);
    overflow:hidden;
    background:var(--navy-deep);
  }
  .canvas-wrap.hitflash{ animation:hitflash .25s ease; }
  @keyframes hitflash{ 0%{background:var(--stop);} 100%{background:var(--navy-deep);} }
  canvas{ display:block; width:100%; height:auto; touch-action:none; }
  .lane-controls{
    position:absolute; right:10px; top:50%; transform:translateY(-50%);
    display:flex; flex-direction:column; gap:10px;
  }
  .lanebtn{
    width:44px; height:44px; border-radius:50%;
    background:rgba(20,33,61,0.75); border:2px solid var(--chalk); color:var(--chalk);
    font-size:18px; display:flex; align-items:center; justify-content:center;
  }
  #pausedOverlay{
    position:absolute; inset:0; background:rgba(14,24,48,0.9);
    display:flex; flex-direction:column; align-items:center; justify-content:center; gap:14px;
  }

  /* ---------- split (question) modal ---------- */
  #splitModal{
    position:fixed; inset:0; background:rgba(14,24,48,0.85);
    display:flex; align-items:center; justify-content:center;
    padding:16px; z-index:50;
  }
  .split-card{
    background:var(--navy-deep); border:3px solid var(--amber); border-radius:var(--radius);
    max-width:520px; width:100%; padding:22px;
    background-image:repeating-linear-gradient(45deg, rgba(255,182,39,0.06) 0 10px, transparent 10px 20px);
  }
  .split-eyebrow{ font-family:var(--mono); color:var(--amber); font-size:13px; letter-spacing:0.1em; }
  #splitPrompt{ font-size:18px; line-height:1.6; margin:10px 0 16px; }
  #splitPrompt .blank{ color:var(--amber); font-weight:700; }
  #splitOptions{ display:flex; flex-direction:column; gap:8px; }
  .opt-btn{
    text-align:left; background:var(--navy); border:2px solid var(--steel); color:var(--chalk);
    border-radius:8px; padding:10px 12px; font-size:14.5px; display:flex; gap:10px;
  }
  .opt-btn:hover:not(:disabled){ border-color:var(--amber); }
  .opt-btn .num{ font-family:var(--mono); color:var(--amber); font-weight:700; }
  .opt-btn:disabled{ opacity:0.35; text-decoration:line-through; }
  .opt-btn.correct{ border-color:var(--go); background:rgba(46,217,163,0.12); }
  .opt-btn.wrong{ border-color:var(--stop); background:rgba(232,73,58,0.12); }

  #splitFeedback{ margin-top:14px; font-size:14px; line-height:1.55; }
  #splitFeedback.ok{ color:var(--go); }
  #splitFeedback.bad{ color:var(--stop); }
  #splitFeedback .source{ display:block; margin-top:6px; color:var(--chalk); opacity:0.8; font-style:italic; }

  .hint-row{ margin-top:14px; border-top:1px dashed var(--steel); padding-top:12px; }
  #hintBtn{ background:none; border:2px solid var(--go); color:var(--go); border-radius:8px; padding:7px 12px; font-size:13px; font-weight:600; }
  #hintArea{ margin-top:8px; font-size:13.5px; color:var(--chalk); opacity:0.9; min-height:1em; }

  /* ---------- stat screens ---------- */
  .stat-grid{ display:grid; grid-template-columns:repeat(3,1fr); gap:10px; margin:18px 0; }
  .stat-box{ background:var(--navy); border:1px solid var(--steel); border-radius:8px; padding:12px; text-align:center; }
  .stat-box .num{ font-family:var(--mono); font-size:22px; color:var(--amber); }
  .stat-box .label{ font-size:11px; opacity:0.7; text-transform:uppercase; letter-spacing:0.05em; }

  /* ---------- settings modal ---------- */
  #settingsModal{
    position:fixed; inset:0; background:rgba(14,24,48,0.85);
    display:flex; align-items:center; justify-content:center; padding:16px; z-index:60;
  }
  .settings-card{ background:var(--navy-deep); border:2px solid var(--steel); border-radius:var(--radius); max-width:460px; width:100%; padding:22px; }
  .settings-card label{ font-size:13px; font-weight:600; display:block; margin-bottom:6px; }
  .settings-card input[type=text]{
    width:100%; background:var(--navy); border:2px solid var(--steel); border-radius:8px;
    color:var(--chalk); padding:9px 10px; font-family:var(--mono); font-size:13px; margin-bottom:6px;
  }
  .settings-help{ font-size:12.5px; opacity:0.75; line-height:1.5; margin-bottom:16px; }

  /* confetti */
  .confetti-piece{
    position:fixed; top:-10px; width:8px; height:14px; opacity:0.9;
    animation:fall linear forwards; z-index:100; pointer-events:none;
  }
  @keyframes fall{ to{ transform:translateY(100vh) rotate(360deg); opacity:0; } }

  @media (prefers-reduced-motion: reduce){
    .btn, .canvas-wrap.hitflash, .confetti-piece{ animation:none !important; transition:none !important; }
  }

  footer{ text-align:center; font-size:12px; opacity:0.5; padding:20px 0 10px; }
  footer a{ color:var(--chalk); }
</style>
</head>
<body>

<div id="progressStrip" class="hidden"></div>

<div class="topbar">
  <div class="brand"><span class="flag">▸</span> SPRINT</div>
  <div style="display:flex; gap:8px;">
    <button id="exitBtn" class="iconbtn hidden" title="Exit to start" aria-label="Exit to start" style="width:auto;padding:0 12px;font-family:var(--mono);font-size:12px;font-weight:700;">EXIT</button>
    <button id="settingsBtn" class="iconbtn" title="Settings" aria-label="Settings">⚙</button>
  </div>
</div>

<div class="wrap">

  <!-- START SCREEN -->
  <section id="screen-start" class="screen">
    <div id="resumeBanner" class="hidden">
      <p>Pick up your last run at <strong class="resume-leg">Leg 1</strong>?</p>
      <div style="display:flex;gap:8px;">
        <button class="btn-ghost" id="resumeBtn">Resume</button>
        <button class="btn-ghost" id="dismissResumeBtn">Start over</button>
      </div>
    </div>

    <h1 class="hero-title">SPRINT</h1>
    <p class="tagline">Turn any notes into a race. Answer right to keep running — get it wrong and you'll see exactly what you missed before you go on.</p>

    <div class="grid2">
      <div class="card">
        <div class="tabbtns">
          <button class="tabbtn active" id="tabUpload">Upload a file</button>
          <button class="tabbtn" id="tabPaste">Paste text</button>
        </div>
        <div id="uploadPane">
          <input type="file" id="fileInput" accept=".txt,.md,.pdf,text/plain,text/markdown,application/pdf">
          <p style="font-size:12px;opacity:0.65;margin:8px 0 0;">Accepts .txt, .md, or .pdf</p>
        </div>
        <div id="pastePane" class="hidden">
          <textarea id="pasteArea" placeholder="Paste a chapter, article, or your own notes here…"></textarea>
        </div>
        <button class="sample-link" id="sampleBtn">Or try a sample passage →</button>
        <div id="startError"></div>
      </div>
      <div class="card">
        <h3 style="font-size:15px; font-family:var(--body); font-weight:700;">HOW IT WORKS</h3>
        <ol class="howto">
          <li>Load any material — a study guide, an article, your class notes.</li>
          <li>Read each leg, then run it: dodge obstacles across three lanes.</li>
          <li>Hit a <strong>Split</strong> and you must answer correctly to keep going. Wrong answers show you the right one before you move on.</li>
          <li>Clear every leg, then survive the <strong>Photo Finish</strong> — a mixed review of everything.</li>
        </ol>
      </div>
    </div>

    <button class="btn" id="startBtn" style="margin-top:20px;" disabled>ON YOUR MARK →</button>
  </section>

  <!-- LEG INTRO -->
  <section id="screen-legIntro" class="screen hidden">
    <div class="card">
      <div class="leg-label" id="legLabel">LEG 1 OF 4</div>
      <h2 id="legIntroTitle">Title</h2>
      <div id="legIntroBody"></div>
      <div class="btn-row">
        <button class="btn" id="readyBtn">I'M READY →</button>
      </div>
      <button class="skip-link" id="skipReadBtn">skip reading, just run</button>
    </div>
  </section>

  <!-- GAME SCREEN -->
  <section id="screen-game" class="screen hidden">
    <div class="hud">
      <span id="hudLeg">LEG 1/4</span>
      <span id="hudScore">SCORE 0</span>
      <span>FALSE STARTS <span class="dots" id="hudLives"></span></span>
      <button class="btn-ghost" id="pauseBtn" style="padding:4px 10px;">II</button>
    </div>
    <div class="canvas-wrap" id="canvasWrap">
      <canvas id="gameCanvas" width="800" height="380"></canvas>
      <div class="lane-controls">
        <button class="lanebtn" id="laneUpBtn" aria-label="Move up a lane">▲</button>
        <button class="lanebtn" id="laneDownBtn" aria-label="Move down a lane">▼</button>
      </div>
      <div id="pausedOverlay" class="hidden">
        <h2 style="font-size:32px;">PAUSED</h2>
        <button class="btn" id="resumeFromPauseBtn" style="width:auto;padding:12px 24px;">RESUME</button>
        <button class="btn-ghost" id="quitBtn">Quit to start</button>
      </div>
    </div>
  </section>

  <!-- LEG COMPLETE -->
  <section id="screen-legComplete" class="screen hidden">
    <div class="card">
      <div class="leg-label">LEG CLEAR</div>
      <h2 id="legCompleteTitle">Leg 1 clear</h2>
      <div class="stat-grid" id="legCompleteStats"></div>
      <button class="btn" id="nextLegBtn">NEXT LEG →</button>
    </div>
  </section>

  <!-- LEG FAILED -->
  <section id="screen-legFailed" class="screen hidden">
    <div class="card">
      <div class="leg-label" style="color:var(--stop);">OUT OF FALSE STARTS</div>
      <h2>Catch your breath.</h2>
      <p id="legFailedMsg" style="opacity:0.85;">Take this leg again — same material, fresh start.</p>
      <button class="btn" id="retryLegBtn">RETRY LEG</button>
    </div>
  </section>

  <!-- GAME COMPLETE -->
  <section id="screen-complete" class="screen hidden">
    <div class="card">
      <div class="leg-label" style="color:var(--go);">RACE COMPLETE</div>
      <h2>You finished the race.</h2>
      <div class="stat-grid" id="completeStats"></div>
      <div class="btn-row">
        <button class="btn" id="playAgainBtn">RUN IT AGAIN</button>
        <button class="btn btn-secondary" id="newMaterialBtn">LOAD NEW MATERIAL</button>
      </div>
    </div>
  </section>

</div>

<footer>Open source under the MIT license. No data leaves your browser unless you turn on AI hints.</footer>

<!-- SPLIT (question) MODAL -->
<div id="splitModal" class="hidden">
  <div class="split-card">
    <div class="split-eyebrow" id="splitTitle">SPLIT 1 OF 3</div>
    <div id="splitPrompt">Prompt</div>
    <div id="splitOptions"></div>
    <div id="splitFeedback"></div>
    <div class="hint-row">
      <button id="hintBtn">GET A HINT</button>
      <div id="hintArea"></div>
    </div>
    <div class="btn-row">
      <button class="btn hidden" id="splitContinueBtn">KEEP RUNNING →</button>
    </div>
  </div>
</div>

<!-- SETTINGS MODAL -->
<div id="settingsModal" class="hidden">
  <div class="settings-card">
    <h2 style="font-size:22px;">Settings</h2>
    <p class="settings-help">
      SPRINT works fully offline for typed or pasted text (PDF uploads need a connection just to load a PDF
      reader library — your document itself never leaves the browser). Turning on AI hints is optional — see
      the README for a ready-made, secure serverless proxy you can deploy so hints never expose an API key.
    </p>
    <label for="proxyUrlInput">Hint proxy URL (optional)</label>
    <input type="text" id="proxyUrlInput" placeholder="https://your-site.pages.dev/api/hint">
    <p class="settings-help">Leave blank to use the built-in demo endpoint, which only works while previewing inside Claude.ai.</p>
    <div class="btn-row">
      <button class="btn" id="saveSettingsBtn">Save</button>
      <button class="btn-ghost" id="closeSettingsBtn">Close</button>
    </div>
  </div>
</div>

<script type="module">
  import * as pdfjsLib from 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/6.1.200/pdf.min.mjs';
  pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/6.1.200/pdf.worker.min.mjs';

  // Exposed for the classic script below. Runs entirely in the browser —
  // the PDF's contents never leave the page, only this library's code
  // is fetched from the CDN.
  window.extractPdfText = async function(arrayBuffer){
    const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise;
    let text = '';
    for (let i=1; i<=pdf.numPages; i++){
      const page = await pdf.getPage(i);
      const content = await page.getTextContent();
      text += content.items.map(it=>it.str).join(' ') + '\n\n';
    }
    return text.trim();
  };
</script>

<script>
(function(){
"use strict";

/* ============================= CONFIG ============================= */
const LANE_COUNT = 3;
const CANVAS_W = 800, CANVAS_H = 380;
const TRACK_TOP = 40, LANE_H = 100;
const PLAYER_X = 110, SPAWN_X = 820;
const BASE_SPEED = 230, SPEED_STEP = 16;
const OBST_W = 30, OBST_H = 60, PLAYER_SIZE = 34;
const GRACE_MS = 900;
const SPLIT_GAP_SECONDS = 7.5;
const COLORS = { chalk:'#F5F3EE', amber:'#FFB627', go:'#2ED9A3', stop:'#E8493A', steel:'#3A5F8A' };
const CLAUDE_ARTIFACT_ENDPOINT = 'https://api.anthropic.com/v1/messages';
const STOPWORDS = new Set(['the','a','an','and','or','but','of','to','in','on','at','for','with','is','are','was','were','be','been','being','this','that','these','those','it','its','as','by','from','into','their','his','her','they','them','which','who','what','when','where','how','why','not','no','so','if','than','then','also','such','can','will','would','should','could','may','might','has','have','had','you','your','our','we']);

const SAMPLE_TEXT = "Around 1440, a goldsmith named Johannes Gutenberg began developing a printing system in the German city of Mainz. His central invention was movable metal type: individual letters cast from a metal alloy that could be arranged, inked, pressed onto paper, and rearranged for the next page. Earlier printing methods, like carved wooden blocks, had to be remade entirely for every new page, but movable type let printers reuse the same letters over and over.\n\nGutenberg combined his type with a modified screw press, borrowed from wine and olive presses used across Europe at the time. This let him apply even pressure across a full page in a single motion. By 1455, his workshop had completed copies of a Latin Bible, now known as the Gutenberg Bible. Historians estimate around 180 copies were printed, and roughly 49 survive today in libraries and museums around the world.\n\nThe technology spread quickly. Within fifty years, printing presses were operating in more than 200 European cities. Books that once took months to copy by hand could now be produced in days. This drop in cost and time helped ideas travel faster than ever before, feeding movements like the Renaissance and, later, the Reformation led by figures such as Martin Luther, whose writings were printed and distributed widely across Germany.\n\nGutenberg himself did not profit from his invention. He borrowed heavily to fund his workshop, and in 1455 a business partner named Johann Fust sued him and took control of the press and much of the equipment. Gutenberg died around 1468, largely without recognition. It was only in later centuries that his role in transforming communication was fully appreciated, and today he is widely regarded as one of the most influential inventors in history.";

/* ============================= STATE ============================= */
const S = {
  screen:'start',
  levels:[], legIndex:0, onFinal:false, finalLeg:null,
  score:0, falseStarts:3,
  totalCorrectFirstTry:0, totalSplits:0,
  player:{lane:1},
  items:[], legTimer:0, nextObstacleAt:1, nextSplitAt:0,
  splitsSpawned:0, splitsToSpawn:0, splitsCleared:0,
  currentQuestionIndex:0, splitAttempt:0, splitResolved:false, disabledOptions:new Set(),
  distance:0, speed:BASE_SPEED, lastTime:0, graceUntil:0, paused:false,
  sourceText:'',
  settings:{ proxyUrl:'' },
  _resumeSnap:null,
  loopStarted:false
};

/* ============================= STORAGE (dual-mode) ============================= */
const storage = {
  async get(key){
    try{
      if (window.storage){ const r = await window.storage.get(key); return r ? JSON.parse(r.value) : null; }
      if (window.localStorage){ const v = localStorage.getItem(key); return v ? JSON.parse(v) : null; }
    }catch(e){ return null; }
    return null;
  },
  async set(key, value){
    try{
      if (window.storage){ await window.storage.set(key, JSON.stringify(value)); return true; }
      if (window.localStorage){ localStorage.setItem(key, JSON.stringify(value)); return true; }
    }catch(e){ return false; }
    return false;
  }
};

/* ============================= CONTENT PARSING ============================= */
function splitIntoLevels(text){
  text = text.trim().replace(/\r\n/g,'\n');
  const headerRe = /^#{1,3}\s+.+$/gm;
  const headers = [...text.matchAll(headerRe)];
  let chunks = [];
  if (headers.length>=2 && headers.length<=8){
    for (let i=0;i<headers.length;i++){
      const start = headers[i].index;
      const end = i+1<headers.length ? headers[i+1].index : text.length;
      const title = headers[i][0].replace(/^#{1,3}\s+/,'').trim();
      const body = text.slice(start,end).replace(headerRe,'').trim();
      if (body.length>20) chunks.push({title,body});
    }
  }
  if (chunks.length===0){
    const paras = text.split(/\n\s*\n/).map(p=>p.trim()).filter(p=>p.length>10);
    const target = 130;
    const raw = [];
    let cur='', curWords=0;
    for (const p of paras){
      const w = p.split(/\s+/).length;
      if (curWords>0 && curWords+w>target*1.5){
        raw.push(cur.trim()); cur=p; curWords=w;
      } else {
        cur += (cur?'\n\n':'')+p; curWords += w;
      }
    }
    if (cur) raw.push(cur.trim());
    while (raw.length>6){
      let idx=0, minLen=Infinity;
      for (let i=0;i<raw.length-1;i++){
        const len = raw[i].length+raw[i+1].length;
        if (len<minLen){ minLen=len; idx=i; }
      }
      raw[idx] = raw[idx]+'\n\n'+raw[idx+1];
      raw.splice(idx+1,1);
    }
    chunks = raw.map((body,i)=>({title:'Part '+(i+1), body}));
  }
  return chunks;
}

function extractTerms(text){
  const terms = new Set();
  const properRe = /\b[A-Z][a-zA-Z]*(?:\s+[A-Z][a-zA-Z]*){0,2}\b/g;
  let m;
  while ((m = properRe.exec(text)) !== null){
    const t = m[0].trim();
    const words = t.split(/\s+/);
    if (words.some(w=>!STOPWORDS.has(w.toLowerCase())) && t.length>2) terms.add(t);
  }
  const numRe = /\b\d[\d,.]*%?\b/g;
  while ((m = numRe.exec(text)) !== null) terms.add(m[0]);
  return [...terms];
}

function longestWordFallback(sentence){
  const words = sentence.split(/\s+/).map(w=>w.replace(/[^a-zA-Z']/g,''));
  const candidates = words.filter(w=>w.length>=5 && !STOPWORDS.has(w.toLowerCase()));
  if (!candidates.length) return null;
  return candidates.sort((a,b)=>b.length-a.length)[0];
}

function shuffle(arr){
  const a = arr.slice();
  for (let i=a.length-1;i>0;i--){
    const j = Math.floor(Math.random()*(i+1));
    [a[i],a[j]] = [a[j],a[i]];
  }
  return a;
}

function buildQuestionsForLeg(body, globalTerms){
  const sentences = body.match(/[^.!?]+[.!?]+/g) || [body];
  const candidates = [];
  for (const raw of sentences){
    const s = raw.trim();
    if (s.length<25 || s.length>220) continue;
    const localTerms = extractTerms(s);
    let answer = localTerms.sort((a,b)=>b.length-a.length)[0];
    if (!answer) answer = longestWordFallback(s);
    if (!answer) continue;
    if (!s.toLowerCase().includes(answer.toLowerCase())) continue;
    candidates.push({sentence:s, answer});
  }
  const seen = new Set(), picked = [];
  for (const c of candidates){
    const key = c.answer.toLowerCase();
    if (seen.has(key)) continue;
    seen.add(key); picked.push(c);
    if (picked.length>=3) break;
  }
  return picked.map(c=>{
    const idx = c.sentence.toLowerCase().indexOf(c.answer.toLowerCase());
    const blanked = c.sentence.slice(0,idx) + '_____' + c.sentence.slice(idx+c.answer.length);
    const pool = globalTerms.filter(t=>t.toLowerCase()!==c.answer.toLowerCase());
    const distractors = shuffle(pool).slice(0,3);
    let guard=0;
    while (distractors.length<3 && distractors.length<pool.length && guard<20){
      guard++;
      const extra = pool[Math.floor(Math.random()*pool.length)];
      if (!distractors.some(d=>d.toLowerCase()===extra.toLowerCase())) distractors.push(extra);
    }
    if (distractors.length===0) return null;
    const options = shuffle([c.answer, ...distractors]);
    return {
      prompt: blanked,
      options,
      correctIndex: options.indexOf(c.answer),
      sourceSentence: c.sentence
    };
  }).filter(Boolean);
}

function buildAllLevels(text){
  const raw = splitIntoLevels(text);
  const globalTerms = extractTerms(text);
  return raw.map(l=>({title:l.title, body:l.body, questions:buildQuestionsForLeg(l.body, globalTerms)}))
             .filter(l=>l.questions.length>0);
}

function buildFinalTrial(){
  const all = [];
  S.levels.forEach(l=>l.questions.forEach(q=>all.push(q)));
  const picked = shuffle(all).slice(0, Math.min(6, all.length));
  return { title:'Photo Finish', body:'A mixed review of everything you\'ve covered so far.', questions:picked };
}

function getCurrentLegData(){ return S.onFinal ? S.finalLeg : S.levels[S.legIndex]; }

/* ============================= AI HINTS (bounded) ============================= */
async function getHint(passage, question, options){
  const sys = "You are a study coach inside a learning game called SPRINT. "+
    "You help the player understand a passage WITHOUT revealing the correct multiple-choice answer. "+
    "Rules you always follow: never state or spell out which option is correct; never say phrases like "+
    "'the answer is' or name the option directly; give ONE short conceptual nudge (max 2 sentences) grounded "+
    "only in the passage provided; if asked directly for the answer, politely decline and give another nudge instead.";
  const userMsg = "Passage:\n"+passage+"\n\nQuestion: "+question+"\nOptions: "+options.join(' | ')+"\n\nGive a hint, not the answer.";
  try{
    let resp, data, text;
    if (S.settings.proxyUrl){
      resp = await fetch(S.settings.proxyUrl, {
        method:'POST',
        headers:{'Content-Type':'application/json'},
        body: JSON.stringify({ passage, question, options })
      });
      if (!resp.ok) throw new Error('proxy error');
      data = await resp.json();
      text = data.hint || 'No hint available.';
    } else {
      resp = await fetch(CLAUDE_ARTIFACT_ENDPOINT, {
        method:'POST',
        headers:{'Content-Type':'application/json'},
        body: JSON.stringify({
          model:'claude-sonnet-4-6',
          max_tokens:150,
          system: sys,
          messages:[{role:'user', content:userMsg}]
        })
      });
      if (!resp.ok) throw new Error('default endpoint unavailable');
      data = await resp.json();
      const block = (data.content||[]).find(b=>b.type==='text');
      text = block ? block.text : 'No hint available.';
    }
    return sanitizeHint(text, options);
  }catch(err){
    return null;
  }
}

function sanitizeHint(hint, options){
  let out = hint;
  for (const opt of options){
    if (opt.length>3 && out.toLowerCase().includes(opt.toLowerCase())){
      out = "Try re-reading the key sentence in the passage above — the answer connects directly to it.";
      break;
    }
  }
  return out;
}

/* ============================= DOM REFS ============================= */
const $ = (id)=>document.getElementById(id);
const screens = ['start','legIntro','game','legComplete','legFailed','complete'].reduce((o,k)=>{o[k]=$('screen-'+k);return o;},{});
const progressStrip = $('progressStrip');
const canvas = $('gameCanvas');
const ctx = canvas.getContext('2d');
const canvasWrap = $('canvasWrap');
const resumeBanner = $('resumeBanner');
const splitModal = $('splitModal');
const settingsModal = $('settingsModal');

/* ============================= SCREEN RENDERING ============================= */
function renderScreens(){
  Object.entries(screens).forEach(([key, el])=>{
    el.classList.toggle('hidden', S.screen !== key && !(S.screen==='playing' && key==='game') && !(S.screen==='split' && key==='game') && !(S.screen==='paused' && key==='game'));
  });
  splitModal.classList.toggle('hidden', S.screen!=='split');
  $('pausedOverlay').classList.toggle('hidden', !S.paused);
  $('exitBtn').classList.toggle('hidden', S.screen==='start');
  renderProgressStrip();
  renderHUD();
}

function renderProgressStrip(){
  if (!S.levels.length){ progressStrip.classList.add('hidden'); progressStrip.innerHTML=''; return; }
  progressStrip.classList.remove('hidden');
  const total = S.levels.length+1;
  const curIndex = S.onFinal ? S.levels.length : S.legIndex;
  let html='';
  for (let i=0;i<total;i++){
    const isFinal = i===S.levels.length;
    const label = isFinal ? 'FIN' : String(i+1);
    let cls='seg';
    if (S.screen==='complete' || i<curIndex) cls+=' seg-done';
    else if (i===curIndex) cls+=' seg-current';
    html += '<span class="'+cls+'">'+label+'</span>';
  }
  progressStrip.innerHTML = html;
}

function renderHUD(){
  if (!S.levels.length) return;
  const total = S.levels.length+1;
  const cur = (S.onFinal ? S.levels.length : S.legIndex) + 1;
  $('hudLeg').textContent = 'LEG '+cur+'/'+total;
  $('hudScore').textContent = 'SCORE '+S.score;
  const dots = $('hudLives');
  dots.innerHTML = '';
  for (let i=0;i<3;i++){
    const d = document.createElement('span');
    d.className = 'dot' + (i<S.falseStarts ? ' on' : '');
    dots.appendChild(d);
  }
}

/* ============================= FLOW CONTROL ============================= */
function showStartError(msg, isStatus){
  const el = $('startError');
  el.textContent = msg || '';
  el.classList.toggle('status', !!isStatus);
}

function initFromText(text){
  const levels = buildAllLevels(text);
  if (levels.length===0){
    showStartError("That text is a little short or plain for SPRINT to build a run from. Try a longer passage — a few solid paragraphs works best.");
    return;
  }
  S.levels = levels;
  S.legIndex = 0;
  S.onFinal = false;
  S.finalLeg = null;
  S.score = 0;
  S.falseStarts = 3;
  S.totalCorrectFirstTry = 0;
  S.totalSplits = 0;
  S.sourceText = text;
  showStartError('');
  saveProgressSnapshot();
  showLegIntro();
}

function showLegIntro(){
  const legData = getCurrentLegData();
  S.screen = 'legIntro';
  const total = S.levels.length+1;
  const cur = (S.onFinal ? S.levels.length : S.legIndex)+1;
  $('legLabel').textContent = 'LEG '+cur+' OF '+total;
  $('legIntroTitle').textContent = legData.title;
  $('legIntroBody').textContent = legData.body;
  renderScreens();
}

function beginLegRun(){
  const legData = getCurrentLegData();
  S.items = [];
  S.legTimer = 0;
  S.splitsSpawned = 0;
  S.splitsCleared = 0;
  S.splitsToSpawn = legData.questions.length;
  S.nextObstacleAt = 1.0;
  S.nextSplitAt = SPLIT_GAP_SECONDS;
  S.distance = 0;
  const levelStep = S.onFinal ? S.levels.length : S.legIndex;
  S.speed = BASE_SPEED + Math.min(levelStep,6)*SPEED_STEP;
  S.player.lane = 1;
  S.graceUntil = performance.now() + GRACE_MS;
  S.paused = false;
  S.screen = 'playing';
  renderScreens();
  if (!S.loopStarted){ S.loopStarted = true; requestAnimationFrame(gameLoop); }
}

function completeLeg(){
  S.screen = 'legComplete';
  const legData = getCurrentLegData();
  const accuracy = legData.questions.length ? Math.round((S.splitsCleared/legData.questions.length)*100) : 100;
  $('legCompleteTitle').textContent = legData.title + ' clear';
  $('legCompleteStats').innerHTML =
    statBox(S.score,'Score') + statBox(S.falseStarts,'False starts left') + statBox(accuracy+'%','Splits this leg');
  saveProgressSnapshot();
  renderScreens();
}

function statBox(num,label){
  return '<div class="stat-box"><div class="num">'+num+'</div><div class="label">'+label+'</div></div>';
}

function nextLeg(){
  if (!S.onFinal && S.legIndex+1 < S.levels.length){
    S.legIndex++;
    showLegIntro();
  } else if (!S.onFinal){
    S.onFinal = true;
    S.finalLeg = buildFinalTrial();
    showLegIntro();
  }
  saveProgressSnapshot();
}

function failLeg(){
  S.screen = 'legFailed';
  renderScreens();
}

function exitToStart(){
  if (S.levels.length){ saveProgressSnapshot(); }
  S.paused = false;
  S.screen = 'start';
  S.levels = [];
  S.onFinal = false;
  splitModal.classList.add('hidden');
  renderScreens();
  checkResume();
}

function retryLeg(){
  S.falseStarts = 3;
  beginLegRun();
}

function finishGame(){
  S.screen = 'complete';
  const totalQ = S.totalSplits || 1;
  const acc = Math.round((S.totalCorrectFirstTry/totalQ)*100);
  $('completeStats').innerHTML =
    statBox(S.score,'Final score') + statBox(acc+'%','First-try accuracy') + statBox(S.levels.length+1,'Legs run');
  storage.set('sprint:progress', null);
  fireConfetti();
  renderScreens();
}

/* ============================= GAME LOOP ============================= */
function gameLoop(ts){
  if (!S.lastTime) S.lastTime = ts;
  const dt = Math.min(0.05, (ts - S.lastTime)/1000);
  S.lastTime = ts;
  if (!S.paused) update(dt);
  render();
  requestAnimationFrame(gameLoop);
}

function update(dt){
  if (S.screen !== 'playing') return;
  const now = performance.now();
  S.legTimer += dt;
  S.distance += S.speed*dt;

  for (const it of S.items) it.x -= S.speed*dt;

  if (now >= S.graceUntil && S.legTimer >= S.nextObstacleAt){
    spawnObstacle();
    const levelStep = S.onFinal ? S.levels.length : S.legIndex;
    const minGap = Math.max(0.55, 1.15 - levelStep*0.05);
    S.nextObstacleAt = S.legTimer + minGap + Math.random()*0.5;
  }
  if (S.splitsSpawned < S.splitsToSpawn && S.legTimer >= S.nextSplitAt){
    spawnSplit(S.splitsSpawned);
    S.splitsSpawned++;
    S.nextSplitAt = S.legTimer + SPLIT_GAP_SECONDS;
  }

  for (let i=S.items.length-1;i>=0;i--){
    const it = S.items[i];
    const reach = it.type==='split' ? (it.x <= PLAYER_X) : (Math.abs(it.x-PLAYER_X) < (OBST_W/2+PLAYER_SIZE/2));
    if (!reach) continue;
    if (it.type==='split'){
      S.items.splice(i,1);
      triggerSplit(it.questionIndex);
      return;
    }
    S.items.splice(i,1);
    if (it.lane === S.player.lane){
      S.falseStarts--;
      flashHit();
      renderHUD();
      if (S.falseStarts<=0){ failLeg(); return; }
    } else {
      S.score += 5;
    }
  }
  S.items = S.items.filter(it=>it.x > -60);
}

function flashHit(){
  canvasWrap.classList.remove('hitflash');
  void canvasWrap.offsetWidth;
  canvasWrap.classList.add('hitflash');
}

function spawnObstacle(){
  const lane = Math.floor(Math.random()*LANE_COUNT);
  S.items.push({type:'obstacle', x:SPAWN_X, lane, resolved:false});
}
function spawnSplit(qIdx){
  S.items.push({type:'split', x:SPAWN_X, questionIndex:qIdx, resolved:false});
}

function laneY(lane){ return TRACK_TOP + LANE_H*lane + LANE_H/2; }

function roundRect(c,x,y,w,h,r){
  c.beginPath();
  c.moveTo(x+r,y);
  c.arcTo(x+w,y,x+w,y+h,r);
  c.arcTo(x+w,y+h,x,y+h,r);
  c.arcTo(x,y+h,x,y,r);
  c.arcTo(x,y,x+w,y,r);
  c.closePath();
}

function render(){
  ctx.clearRect(0,0,CANVAS_W,CANVAS_H);
  ctx.fillStyle = 'rgba(245,243,238,0.05)';
  ctx.fillRect(0,TRACK_TOP,CANVAS_W,LANE_H*LANE_COUNT);
  ctx.strokeStyle = 'rgba(245,243,238,0.25)';
  ctx.lineWidth = 2;
  ctx.setLineDash([16,14]);
  ctx.lineDashOffset = -(S.distance*0.6)%30;
  for (let i=1;i<LANE_COUNT;i++){
    const y = TRACK_TOP+LANE_H*i;
    ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(CANVAS_W,y); ctx.stroke();
  }
  ctx.setLineDash([]);
  ctx.strokeStyle = 'rgba(245,243,238,0.5)';
  ctx.lineWidth = 3;
  ctx.strokeRect(1,TRACK_TOP,CANVAS_W-2,LANE_H*LANE_COUNT);

  for (const it of S.items){
    if (it.type==='split') drawSplitMarker(it); else drawObstacle(it);
  }
  drawPlayer();
}

function drawObstacle(o){
  const y = laneY(o.lane);
  ctx.fillStyle = COLORS.stop;
  roundRect(ctx, o.x-OBST_W/2, y-OBST_H/2, OBST_W, OBST_H, 6);
  ctx.fill();
}

function drawSplitMarker(it){
  ctx.save();
  ctx.strokeStyle = COLORS.amber;
  ctx.lineWidth = 6;
  ctx.setLineDash([10,8]);
  ctx.beginPath();
  ctx.moveTo(it.x, TRACK_TOP-4);
  ctx.lineTo(it.x, TRACK_TOP+LANE_H*LANE_COUNT+4);
  ctx.stroke();
  ctx.restore();
  ctx.fillStyle = COLORS.amber;
  ctx.font = '700 12px "IBM Plex Mono", monospace';
  ctx.textAlign = 'center';
  ctx.fillText('SPLIT', it.x, TRACK_TOP-10);
}

function drawPlayer(){
  const y = laneY(S.player.lane);
  ctx.globalAlpha = 0.25;
  ctx.fillStyle = COLORS.go;
  for (let i=1;i<=3;i++){
    ctx.beginPath();
    ctx.moveTo(PLAYER_X-10-i*14, y);
    ctx.lineTo(PLAYER_X-2-i*14, y-8);
    ctx.lineTo(PLAYER_X-2-i*14, y+8);
    ctx.closePath();
    ctx.fill();
  }
  ctx.globalAlpha = 1;
  ctx.fillStyle = COLORS.chalk;
  roundRect(ctx, PLAYER_X-PLAYER_SIZE/2, y-PLAYER_SIZE/2, PLAYER_SIZE, PLAYER_SIZE, 8);
  ctx.fill();
  ctx.strokeStyle = COLORS.amber;
  ctx.lineWidth = 3;
  roundRect(ctx, PLAYER_X-PLAYER_SIZE/2, y-PLAYER_SIZE/2, PLAYER_SIZE, PLAYER_SIZE, 8);
  ctx.stroke();
}

/* ============================= LANES / INPUT ============================= */
function setLane(l){
  if (S.screen!=='playing' || S.paused) return;
  S.player.lane = Math.max(0, Math.min(LANE_COUNT-1, l));
}
function laneUp(){ setLane(S.player.lane-1); }
function laneDown(){ setLane(S.player.lane+1); }

function togglePause(){
  if (S.screen!=='playing') return;
  S.paused = !S.paused;
  renderScreens();
}

/* ============================= SPLIT (question) LOGIC ============================= */
function triggerSplit(qIdx){
  S.screen = 'split';
  S.currentQuestionIndex = qIdx;
  S.splitAttempt = 0;
  S.splitResolved = false;
  S.disabledOptions = new Set();
  renderScreens();
  populateSplitModal();
}

function populateSplitModal(){
  const legData = getCurrentLegData();
  const q = legData.questions[S.currentQuestionIndex];
  $('splitTitle').textContent = 'SPLIT '+(S.currentQuestionIndex+1)+' OF '+legData.questions.length;
  const promptEl = $('splitPrompt');
  promptEl.innerHTML = 'Complete the passage: “' + escapeHtml(q.prompt).replace('_____','<span class="blank">_____</span>') + '”';
  const optsEl = $('splitOptions');
  optsEl.innerHTML = '';
  q.options.forEach((opt,i)=>{
    const b = document.createElement('button');
    b.className = 'opt-btn';
    b.innerHTML = '<span class="num">'+(i+1)+'</span><span>'+escapeHtml(opt)+'</span>';
    b.addEventListener('click', ()=>selectOption(i));
    optsEl.appendChild(b);
  });
  $('splitFeedback').textContent = '';
  $('splitFeedback').className = '';
  $('hintArea').textContent = '';
  $('splitContinueBtn').classList.add('hidden');
}

function escapeHtml(str){
  return str.replace(/[&<>"']/g, m=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[m]));
}

function selectOption(optIdx){
  if (S.splitResolved) return;
  const legData = getCurrentLegData();
  const q = legData.questions[S.currentQuestionIndex];
  const buttons = document.querySelectorAll('#splitOptions .opt-btn');

  if (optIdx === q.correctIndex){
    const gained = [100,60,30][S.splitAttempt] ?? 20;
    S.score += gained;
    if (S.splitAttempt===0) S.totalCorrectFirstTry++;
    S.splitResolved = true;
    S.splitsCleared++;
    S.totalSplits++;
    buttons[optIdx].classList.add('correct');
    buttons.forEach(b=>b.disabled=true);
    showSplitFeedback(true, q, gained, false);
  } else {
    S.disabledOptions.add(optIdx);
    S.splitAttempt++;
    S.falseStarts--;
    flashHit();
    buttons[optIdx].classList.add('wrong');
    buttons[optIdx].disabled = true;
    renderHUD();
    const outOfTries = S.splitAttempt >= 3;
    const outOfLives = S.falseStarts <= 0;
    if (outOfTries || outOfLives){
      S.splitResolved = true;
      S.splitsCleared++;
      S.totalSplits++;
      buttons[q.correctIndex].classList.add('correct');
      buttons.forEach(b=>b.disabled=true);
      showSplitFeedback(false, q, 0, true);
    } else {
      showSplitFeedback(false, q, 0, false);
    }
  }
}

function showSplitFeedback(correct, q, gained, revealed){
  const el = $('splitFeedback');
  if (correct){
    el.className = 'ok';
    el.innerHTML = 'Clean split. +'+gained+' points.';
  } else if (revealed){
    el.className = 'bad';
    el.innerHTML = 'The right answer was <strong>'+escapeHtml(q.options[q.correctIndex])+'</strong>.<span class="source">'+escapeHtml(q.sourceSentence)+'</span>';
  } else {
    const triesLeft = 3 - S.splitAttempt;
    el.className = 'bad';
    el.innerHTML = 'Not quite — '+triesLeft+' '+(triesLeft===1?'try':'tries')+' left.';
  }
  $('splitContinueBtn').classList.remove('hidden');
}

function afterSplitContinue(){
  splitModal.classList.add('hidden');
  if (S.falseStarts<=0){ failLeg(); return; }
  const legData = getCurrentLegData();
  if (S.splitsCleared >= legData.questions.length){
    if (S.onFinal) finishGame(); else completeLeg();
  } else {
    S.screen = 'playing';
    S.graceUntil = performance.now() + GRACE_MS;
    renderScreens();
  }
}

async function onGetHintClick(){
  const legData = getCurrentLegData();
  const q = legData.questions[S.currentQuestionIndex];
  const hintBtn = $('hintBtn');
  const hintArea = $('hintArea');
  hintArea.textContent = 'Thinking…';
  hintBtn.disabled = true;
  const hint = await getHint(legData.body, q.prompt, q.options);
  hintBtn.disabled = false;
  hintArea.textContent = hint || "Hints aren't set up for this deployment yet — add a hint proxy URL in Settings (see the README).";
}

/* ============================= PERSISTENCE ============================= */
async function saveProgressSnapshot(){
  await storage.set('sprint:progress', {
    sourceText: S.sourceText, legIndex: S.legIndex, onFinal: S.onFinal,
    score: S.score, totalCorrectFirstTry: S.totalCorrectFirstTry, totalSplits: S.totalSplits,
    ts: Date.now()
  });
}

async function checkResume(){
  const snap = await storage.get('sprint:progress');
  if (snap && snap.sourceText && typeof snap.legIndex === 'number'){
    S._resumeSnap = snap;
    resumeBanner.classList.remove('hidden');
    resumeBanner.querySelector('.resume-leg').textContent = snap.onFinal ? 'the Photo Finish' : ('Leg '+(snap.legIndex+1));
  }
}

function resumeRun(){
  const snap = S._resumeSnap;
  if (!snap) return;
  const levels = buildAllLevels(snap.sourceText);
  if (levels.length===0){ dismissResume(); return; }
  S.levels = levels;
  S.sourceText = snap.sourceText;
  S.legIndex = Math.min(snap.legIndex, levels.length-1);
  S.onFinal = !!snap.onFinal;
  S.finalLeg = S.onFinal ? buildFinalTrial() : null;
  S.score = snap.score||0;
  S.falseStarts = 3;
  S.totalCorrectFirstTry = snap.totalCorrectFirstTry||0;
  S.totalSplits = snap.totalSplits||0;
  dismissResume();
  showLegIntro();
}

function dismissResume(){ resumeBanner.classList.add('hidden'); storage.set('sprint:progress', null); }

/* ============================= CONFETTI ============================= */
function fireConfetti(){
  if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;
  const colors = [COLORS.amber, COLORS.go, COLORS.stop, COLORS.chalk];
  for (let i=0;i<26;i++){
    const p = document.createElement('div');
    p.className = 'confetti-piece';
    p.style.left = Math.random()*100+'vw';
    p.style.background = colors[i%colors.length];
    p.style.animationDuration = (1.6+Math.random()*1.2)+'s';
    p.style.animationDelay = (Math.random()*0.4)+'s';
    document.body.appendChild(p);
    setTimeout(()=>p.remove(), 3200);
  }
}

/* ============================= UI WIRING ============================= */
function validateStartInput(){
  const activeText = $('uploadPane').classList.contains('hidden') ? $('pasteArea').value : S._uploadedText;
  const ok = activeText && activeText.trim().length>150;
  $('startBtn').disabled = !ok;
  return activeText;
}

function bindEvents(){
  $('tabUpload').addEventListener('click', ()=>{
    $('tabUpload').classList.add('active'); $('tabPaste').classList.remove('active');
    $('uploadPane').classList.remove('hidden'); $('pastePane').classList.add('hidden');
    validateStartInput();
  });
  $('tabPaste').addEventListener('click', ()=>{
    $('tabPaste').classList.add('active'); $('tabUpload').classList.remove('active');
    $('pastePane').classList.remove('hidden'); $('uploadPane').classList.add('hidden');
    validateStartInput();
  });
  $('fileInput').addEventListener('change', (e)=>{
    const file = e.target.files[0];
    if (!file) return;
    S._uploadedText = '';
    validateStartInput();
    const isPdf = file.type === 'application/pdf' || /\.pdf$/i.test(file.name);
    if (isPdf){
      if (typeof window.extractPdfText !== 'function'){
        showStartError("PDF support didn't load — check your connection, or paste the text instead.");
        return;
      }
      showStartError('Reading PDF…', true);
      const reader = new FileReader();
      reader.onload = async ()=>{
        try{
          const text = await window.extractPdfText(reader.result);
          if (!text || text.trim().length < 30){
            showStartError("That PDF didn't have readable text — it may be scanned pages/images. Try pasting the text instead.");
          } else {
            S._uploadedText = text;
            showStartError('');
          }
        }catch(err){
          showStartError("Couldn't read that PDF — it may be password protected. Try pasting the text instead.");
        }
        validateStartInput();
      };
      reader.onerror = ()=>{ showStartError('Could not read that file — try pasting the text instead.'); validateStartInput(); };
      reader.readAsArrayBuffer(file);
    } else {
      const reader = new FileReader();
      reader.onload = ()=>{ S._uploadedText = String(reader.result||''); showStartError(''); validateStartInput(); };
      reader.onerror = ()=>{ showStartError('Could not read that file — try pasting the text instead.'); validateStartInput(); };
      reader.readAsText(file);
    }
  });
  $('pasteArea').addEventListener('input', validateStartInput);
  $('sampleBtn').addEventListener('click', ()=>{
    $('tabPaste').click();
    $('pasteArea').value = SAMPLE_TEXT;
    validateStartInput();
  });
  $('startBtn').addEventListener('click', ()=>{
    const text = $('uploadPane').classList.contains('hidden') ? $('pasteArea').value : S._uploadedText;
    initFromText((text||'').trim());
  });

  $('resumeBtn').addEventListener('click', resumeRun);
  $('dismissResumeBtn').addEventListener('click', dismissResume);

  $('readyBtn').addEventListener('click', beginLegRun);
  $('skipReadBtn').addEventListener('click', beginLegRun);

  $('laneUpBtn').addEventListener('click', laneUp);
  $('laneDownBtn').addEventListener('click', laneDown);
  $('pauseBtn').addEventListener('click', togglePause);
  $('resumeFromPauseBtn').addEventListener('click', togglePause);
  $('quitBtn').addEventListener('click', exitToStart);
  $('exitBtn').addEventListener('click', exitToStart);

  canvas.addEventListener('pointerdown', (e)=>{
    if (S.screen!=='playing' || S.paused) return;
    const rect = canvas.getBoundingClientRect();
    const relY = (e.clientY-rect.top)/rect.height;
    setLane(Math.min(LANE_COUNT-1, Math.floor(relY*LANE_COUNT)));
  });

  window.addEventListener('keydown', (e)=>{
    if (S.screen==='playing' && !S.paused){
      if (e.key==='ArrowUp' || e.key==='w' || e.key==='W'){ laneUp(); e.preventDefault(); }
      else if (e.key==='ArrowDown' || e.key==='s' || e.key==='S'){ laneDown(); e.preventDefault(); }
      else if (['1','2','3'].includes(e.key)){ setLane(parseInt(e.key,10)-1); }
      else if (e.key===' '){ togglePause(); e.preventDefault(); }
    } else if (S.screen==='playing' && S.paused && e.key===' '){
      togglePause(); e.preventDefault();
    } else if (S.screen==='split'){
      if (['1','2','3','4'].includes(e.key)){
        const idx = parseInt(e.key,10)-1;
        const btn = document.querySelectorAll('#splitOptions .opt-btn')[idx];
        if (btn && !btn.disabled) btn.click();
      }
    }
  });

  $('hintBtn').addEventListener('click', onGetHintClick);
  $('splitContinueBtn').addEventListener('click', afterSplitContinue);

  $('nextLegBtn').addEventListener('click', nextLeg);
  $('retryLegBtn').addEventListener('click', retryLeg);
  $('playAgainBtn').addEventListener('click', ()=>{
    S.legIndex=0; S.onFinal=false; S.finalLeg=null;
    S.score=0; S.falseStarts=3; S.totalCorrectFirstTry=0; S.totalSplits=0;
    showLegIntro();
  });
  $('newMaterialBtn').addEventListener('click', ()=>{
    S.levels=[]; S.screen='start';
    $('pasteArea').value=''; $('fileInput').value=''; S._uploadedText='';
    validateStartInput();
    renderScreens();
  });

  $('settingsBtn').addEventListener('click', ()=>settingsModal.classList.remove('hidden'));
  $('closeSettingsBtn').addEventListener('click', ()=>settingsModal.classList.add('hidden'));
  $('saveSettingsBtn').addEventListener('click', async ()=>{
    S.settings.proxyUrl = $('proxyUrlInput').value.trim();
    await storage.set('sprint:settings', S.settings);
    settingsModal.classList.add('hidden');
  });
}

/* ============================= INIT ============================= */
async function init(){
  bindEvents();
  renderScreens();
  const savedSettings = await storage.get('sprint:settings');
  if (savedSettings && savedSettings.proxyUrl){
    S.settings.proxyUrl = savedSettings.proxyUrl;
    $('proxyUrlInput').value = savedSettings.proxyUrl;
  }
  checkResume();
}

document.addEventListener('DOMContentLoaded', init);
})();
</script>
</body>
</html>
