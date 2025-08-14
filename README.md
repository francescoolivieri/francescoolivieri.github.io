<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Underwater TL Match – Guess the Sediment Speed</title>
<style>
  :root {
    --deep: #001a33;
    --mid: #003d66;
    --light: #4da6ff;
    --accent: #00d4ff;
  }
  * { box-sizing: border-box; }
  body {
    margin: 0;
    font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, sans-serif;
    color: white;
    min-height: 100vh;
    /* Underwater gradient */
    background: radial-gradient(ellipse at top, rgba(77,166,255,0.35), transparent 60%),
                linear-gradient(to bottom, var(--light), var(--mid) 40%, var(--deep) 90%);
    position: relative;
    overflow-x: hidden;
  }
  /* Light rays overlay */
  .rays {
    position: fixed;
    inset: 0;
    pointer-events: none;
    background: repeating-linear-gradient(120deg, rgba(255,255,255,0.06) 0 2px, transparent 2px 12px);
    mix-blend-mode: screen;
    filter: blur(1px) opacity(0.45);
  }

  header {
    text-align: center;
    padding: 24px 16px 8px;
  }
  h1 {
    margin: 0 0 6px 0;
    font-size: clamp(24px, 5vw, 44px);
    letter-spacing: 0.5px;
  }
  p.sub {
    margin: 0;
    opacity: 0.9;
  }

  .container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 16px;
    display: grid;
    grid-template-columns: 1fr 320px;
    gap: 18px;
  }
  .card {
    background: rgba(0, 0, 0, 0.25);
    border: 1px solid rgba(255,255,255,0.12);
    border-radius: 16px;
    padding: 16px;
    backdrop-filter: blur(6px);
    box-shadow: 0 10px 30px rgba(0,0,0,0.25);
  }
  .game {
    display: grid;
    grid-template-rows: auto auto auto;
    gap: 14px;
  }
  .preview {
    display: grid;
    grid-template-columns: 1fr;
    gap: 10px;
    place-items: center;
  }
  .preview img {
    width: 100%;
    max-width: 900px;
    border-radius: 12px;
    border: 2px solid rgba(255,255,255,0.2);
    background: #001a33;
  }
  .controls {
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 10px;
    align-items: center;
  }
  .sliderwrap {
    padding: 10px 12px;
    background: rgba(0,0,0,0.25);
    border: 1px solid rgba(255,255,255,0.12);
    border-radius: 12px;
  }
  input[type="range"] {
    width: 100%;
  }
  .value {
    min-width: 140px;
    text-align: right;
    font-weight: 700;
    font-variant-numeric: tabular-nums;
  }
  .actions {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
  }
  button {
    background: linear-gradient(135deg, var(--accent), #2ecfff);
    color: #002233;
    border: none;
    padding: 10px 14px;
    border-radius: 10px;
    font-weight: 700;
    cursor: pointer;
    box-shadow: 0 6px 16px rgba(0, 212, 255, 0.3);
  }
  button.secondary {
    background: transparent;
    color: white;
    border: 1px solid rgba(255,255,255,0.3);
  }

  /* Sidebar */
  .sidebar h2 {
    margin-top: 0;
    font-size: 22px;
  }
  table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 8px;
    font-size: 14px;
  }
  th, td {
    text-align: left;
    padding: 8px 6px;
    border-bottom: 1px solid rgba(255,255,255,0.12);
  }
  th {
    opacity: 0.9;
  }
  .tiny {
    opacity: 0.9;
    font-size: 12px;
  }
  .footer {
    text-align: center;
    padding: 10px;
    opacity: 0.8;
    font-size: 12px;
  }
</style>
</head>
<body>
<div class="rays"></div>
<header>
  <h1>Underwater TL Match</h1>
  <p class="sub">Adjust the sediment sound speed to match the hidden scenario. Lock your guess and climb the leaderboard!</p>
</header>

<div class="container">
  <section class="card game">
    <div class="preview">
      <img id="tlImg" alt="Transmission Loss Map" src="" />
    </div>

    <div class="controls">
      <div class="sliderwrap">
        <label for="speed"><strong>Sediment Sound Speed</strong> (m/s)</label>
        <input id="speed" type="range" min="1450" max="1800" step="1" value="1644">
      </div>
      <div class="value">
        <span id="speedVal">1644</span> m/s
      </div>
    </div>

    <div class="actions">
      <button id="submitBtn">Submit Guess</button>
      <button class="secondary" id="newRoundBtn">New Round</button>
      <span id="roundMsg" class="tiny"></span>
    </div>
  </section>

  <aside class="card sidebar">
    <h2>Leaderboard (Local)</h2>
    <p class="tiny">Sorted by smallest error (m/s). Stored in your browser.</p>
    <table id="board">
      <thead>
        <tr><th>#</th><th>Name</th><th>Error (m/s)</th><th>Guess</th><th>Target</th></tr>
      </thead>
      <tbody></tbody>
    </table>
  </aside>
</div>

<div class="footer tiny">Tip: Replace images in <code>assets/</code> with your Bellhop3D TL maps named like <code>tl_speed_XXXX.png</code>.</div>

<script>
  const speeds = [1450, 1488, 1527, 1566, 1605, 1644, 1683, 1722, 1761, 1800];
  const imgForSpeed = (s) => `assets/tl_speed_${s}.png`;

  const tlImg = document.getElementById('tlImg');
  const slider = document.getElementById('speed');
  const speedVal = document.getElementById('speedVal');
  const submitBtn = document.getElementById('submitBtn');
  const newRoundBtn = document.getElementById('newRoundBtn');
  const roundMsg = document.getElementById('roundMsg');

  // Leaderboard (local)
  const BOARD_KEY = 'ua_game_leaderboard_v1';
  function loadBoard() {
    try { return JSON.parse(localStorage.getItem(BOARD_KEY)) || []; }
    catch (e) { return []; }
  }
  function saveBoard(rows) { localStorage.setItem(BOARD_KEY, JSON.stringify(rows)); }
  function renderBoard() {
    const rows = loadBoard().sort((a,b) => a.error - b.error).slice(0, 10);
    const tbody = document.querySelector('#board tbody');
    tbody.innerHTML = '';
    rows.forEach((r, i) => {
      const tr = document.createElement('tr');
      tr.innerHTML = `<td>${i+1}</td><td>${r.name}</td><td>${r.error.toFixed(0)}</td><td>${r.guess}</td><td>${r.target}</td>`;
      tbody.appendChild(tr);
    });
  }

  // Snap to nearest available speed
  function nearestSpeed(v) {
    let best = speeds[0];
    let bestd = Math.abs(v - best);
    for (const s of speeds) {
      const d = Math.abs(v - s);
      if (d < bestd) { best = s; bestd = d; }
    }
    return best;
  }

  // Game state
  let target = speeds[Math.floor(Math.random()*speeds.length)];
  let current = nearestSpeed(parseInt(slider.value, 10));
  function updateImage() {
    current = nearestSpeed(parseInt(slider.value, 10));
    speedVal.textContent = current;
    tlImg.src = imgForSpeed(current);
  }

  slider.addEventListener('input', updateImage);

  submitBtn.addEventListener('click', () => {
    const guess = current;
    const error = Math.abs(guess - target);

    let name = prompt('Enter your name or nickname:');
    if (!name) name = 'Anon';

    const row = { name, error, guess, target, ts: Date.now() };
    const rows = loadBoard();
    rows.push(row);
    saveBoard(rows);
    renderBoard();
    roundMsg.textContent = `Your error: ${error.toFixed(0)} m/s (guess ${guess} vs target ${target}).`;
  });

  newRoundBtn.addEventListener('click', () => {
    target = speeds[Math.floor(Math.random()*speeds.length)];
    roundMsg.textContent = 'New round started!';
  });

  // Init
  target = speeds[Math.floor(Math.random()*speeds.length)];
  renderBoard();
  updateImage();
</script>
</body>
</html>
