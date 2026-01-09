<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <title>国旗ポーランドボールゲーム</title>
  <style>
    body {
      background: #222;
      color: #eee;
      font-family: system-ui, sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      margin: 0;
      padding: 10px;
    }
    h1 {
      margin: 5px 0;
      font-size: 20px;
    }
    canvas {
      background: #111;
      border: 2px solid #555;
    }
    #controls {
      margin-top: 10px;
    }
    button {
      padding: 6px 12px;
      margin: 0 5px;
      font-size: 14px;
      cursor: pointer;
    }
    #score {
      margin-top: 10px;
      font-size: 18px;
    }
  </style>
</head>
<body>
  <h1>ポーランドボいずみー　ぷよぷよしてて草</h1>

  <canvas id="game" width="850" height="850"></canvas>

  <div id="controls">
    <button id="startBtn">▶ 再生</button>
    <button id="pauseBtn">⏸ 一時停止</button>
  </div>

  <div id="score">Score: 0</div>
<div id="difficulty">
  <button id="easyBtn">簡単</button>
  <button id="normalBtn">普通</button>
  <button id="hardBtn">難しい</button>
</div>

 <script>
  const COLS = 12;
  const ROWS = 12;
  const CELL = 70;

function setDifficulty(level) {
  if (level === "easy") dropInterval = 900;   // ゆっくり
  if (level === "normal") dropInterval = 500; // 標準
  if (level === "hard") dropInterval = 250;   // 速い
}
  const canvas = document.getElementById("game");
  const ctx = canvas.getContext("2d");

  let score = 0;
  let paused = false;

  /* -------------------------
     国旗データ・描画
  ------------------------- */

 const POLANDBALLS = [
  { id: 0, name: "Palau", desc: "青い海と黄色い月" },
  { id: 1, name: "Japan", desc: "白地に赤い日の丸" },
  { id: 2, name: "Bangladesh", desc: "緑地に赤い太陽" },
  { id: 3, name: "Greenland", desc: "赤白の半円デザイン" },
  { id: 4, name: "Kyrgyzstan", desc: "赤地に太陽とユルト模様" },
];

  function drawCircleBase(x, y, r) {
    ctx.beginPath();
    ctx.arc(x, y, r, 0, Math.PI * 2);
    ctx.closePath();
  }

  function drawPalauFlag(x, y, r) {
    ctx.beginPath();
    ctx.arc(x, y, r, 0, Math.PI * 2);
    ctx.fillStyle = "#4AADD6";
    ctx.fill();

    ctx.beginPath();
    ctx.arc(x - r * 0.25, y, r * 0.45, 0, Math.PI * 2);
    ctx.fillStyle = "#FFDE00";
    ctx.fill();
  }

  function drawJapanFlag(x, y, r) {
    drawCircleBase(x, y, r);
    ctx.fillStyle = "#ffffff";
    ctx.fill();

    ctx.beginPath();
    ctx.arc(x, y, r * 0.45, 0, Math.PI * 2);
    ctx.fillStyle = "#bc002d";
    ctx.fill();
  }

  function drawBangladeshFlag(x, y, r) {
    ctx.beginPath();
    ctx.arc(x, y, r, 0, Math.PI * 2);
    ctx.fillStyle = "#006A4E";
    ctx.fill();

    ctx.beginPath();
    ctx.arc(x - r * 0.2, y, r * 0.45, 0, Math.PI * 2);
    ctx.fillStyle = "#F42A41";
    ctx.fill();
  }

  function drawGreenlandFlag(x, y, r) {
    ctx.beginPath();
    ctx.arc(x, y, r, Math.PI, 0);
    ctx.lineTo(x, y);
    ctx.closePath();
    ctx.fillStyle = "#ffffff";
    ctx.fill();

    ctx.beginPath();
    ctx.arc(x, y, r, 0, Math.PI);
    ctx.lineTo(x, y);
    ctx.closePath();
    ctx.fillStyle = "#d00c33";
    ctx.fill();

    ctx.beginPath();
    ctx.arc(x - r * 0.25, y, r * 0.45, Math.PI, 0);
    ctx.closePath();
    ctx.fillStyle = "#d00c33";
    ctx.fill();

    ctx.beginPath();
    ctx.arc(x - r * 0.25, y, r * 0.45, 0, Math.PI);
    ctx.closePath();
    ctx.fillStyle = "#ffffff";
    ctx.fill();
  }

  function drawKyrgyzstanFlag(x, y, r) {
    ctx.beginPath();
    ctx.arc(x, y, r, 0, Math.PI * 2);
    ctx.closePath();
    ctx.fillStyle = "#E8112D";
    ctx.fill();

    const sunR = r * 0.45;
    ctx.beginPath();
    ctx.arc(x, y, sunR, 0, Math.PI * 2);
    ctx.closePath();
    ctx.fillStyle = "#FFD700";
    ctx.fill();

    ctx.save();
    ctx.translate(x, y);
    ctx.rotate(Math.PI / 6);

    ctx.beginPath();
    ctx.ellipse(0, 0, sunR * 0.75, sunR * 0.35, 0, 0, Math.PI * 2);
    ctx.strokeStyle = "#E8112D";
    ctx.lineWidth = sunR * 0.18;
    ctx.stroke();

    ctx.rotate(Math.PI / 2);
    ctx.beginPath();
    ctx.ellipse(0, 0, sunR * 0.75, sunR * 0.35, 0, 0, Math.PI * 2);
    ctx.stroke();

    ctx.restore();
  }

  function drawPolandballSmileEyes(x, y, r) {
    const ex = r * 0.45;
    const ey = -r * 0.05;
    const w = r * 0.22;

    ctx.strokeStyle = "#000";
    ctx.lineWidth = r * 0.10;

    ctx.beginPath();
    ctx.arc(x - ex, y + ey, w, 0, Math.PI);
    ctx.stroke();

    ctx.beginPath();
    ctx.arc(x + ex, y + ey, w, 0, Math.PI);
    ctx.stroke();
  }

  function drawOutline(x, y, r) {
    ctx.beginPath();
    ctx.arc(x, y, r, 0, Math.PI * 2);
    ctx.lineWidth = r * 0.12;
    ctx.strokeStyle = "#000000";
    ctx.stroke();
  }

  function drawPolandball(col, row, type) {
    const pb = POLANDBALLS[type];
    const x = col * CELL + CELL / 2;
    const y = row * CELL + CELL / 2;
    const r = CELL * 0.45;

    switch (pb.name) {
      case "Palau":      drawPalauFlag(x, y, r); break;
      case "Japan":      drawJapanFlag(x, y, r); break;
      case "Bangladesh": drawBangladeshFlag(x, y, r); break;
      case "Greenland":  drawGreenlandFlag(x, y, r); break;
      case "Kyrgyzstan": drawKyrgyzstanFlag(x, y, r); break;
    }

    drawOutline(x, y, r);
    drawPolandballSmileEyes(x, y, r);
  }

  /* -------------------------
     ゲームロジック
  ------------------------- */

  let board = [];
  function resetBoard() {
    board = [];
    for (let r = 0; r < ROWS; r++) {
      const row = [];
      for (let c = 0; c < COLS; c++) row.push(null);
      board.push(row);
    }
  }

  let currentPair = null;
  let gameOver = false;
  let dropTimer = 0;
  let dropInterval = 500;

  function randomBallType() {
    return Math.floor(Math.random() * POLANDBALLS.length);
  }

  function spawnPair() {
    const cx = Math.floor(COLS / 2);
    const cy = 0;

    // ★ ここでゲームオーバー判定
    if (board[cy][cx] !== null) {
      gameOver = true;
      return;
    }

    const t1 = randomBallType();
    const t2 = randomBallType();

    currentPair = {
      cx,
      cy,
      blocks: [
        { ox: 0, oy: 0, type: t1 },
        { ox: 0, oy: -1, type: t2 }
      ],
    };
  }

  function collides(dx, dy, blocks) {
    for (const b of blocks) {
      const x = currentPair.cx + b.ox + dx;
      const y = currentPair.cy + b.oy + dy;
      if (x < 0 || x >= COLS || y >= ROWS) return true;
      if (y >= 0 && board[y][x] !== null) return true;
    }
    return false;
  }

  function rotatePair() {
    const rotated = currentPair.blocks.map((b, i) => {
      if (i === 0) return { ...b };
      return { ox: -b.oy, oy: b.ox, type: b.type };
    });
    const old = currentPair.blocks;
    currentPair.blocks = rotated;
    if (collides(0, 0, rotated)) currentPair.blocks = old;
  }

  function lockPair() {
    for (const b of currentPair.blocks) {
      const x = currentPair.cx + b.ox;
      const y = currentPair.cy + b.oy;
      if (y >= 0) board[y][x] = { type: b.type };
      else {
        // 画面外（上）に食い込んだら即ゲームオーバー
        gameOver = true;
      }
    }
    currentPair = null;
    clearMatches();
  }

  function clearMatches() {
    let chain = 0;

    while (true) {
      let visited = Array.from({ length: ROWS }, () => Array(COLS).fill(false));
      let cleared = false;

      for (let r = 0; r < ROWS; r++) {
        for (let c = 0; c < COLS; c++) {
          if (!board[r][c] || visited[r][c]) continue;

          const type = board[r][c].type;
          const stack = [{ r, c }];
          const group = [];
          visited[r][c] = true;

          while (stack.length) {
            const { r: cr, c: cc } = stack.pop();
            group.push({ r: cr, c: cc });

            const dirs = [
              { dr: -1, dc: 0 },
              { dr: 1, dc: 0 },
              { dr: 0, dc: -1 },
              { dr: 0, dc: 1 },
            ];

            for (const d of dirs) {
              const nr = cr + d.dr;
              const nc = cc + d.dc;
              if (
                nr >= 0 && nr < ROWS &&
                nc >= 0 && nc < COLS &&
                !visited[nr][nc] &&
                board[nr][nc] &&
                board[nr][nc].type === type
              ) {
                visited[nr][nc] = true;
                stack.push({ r: nr, c: nc });
              }
            }
          }

          if (group.length >= 4) {
            cleared = true;
            for (const g of group) board[g.r][g.c] = null;
          }
        }
      }

      if (!cleared) break;

      chain++;
      applyGravity();
    }

    if (chain > 0) {
      score += chain * 100;
      document.getElementById("score").textContent = "Score: " + score;
    }
  }

  function applyGravity() {
    for (let c = 0; c < COLS; c++) {
      let write = ROWS - 1;
      for (let r = ROWS - 1; r >= 0; r--) {
        if (board[r][c]) {
          if (write !== r) {
            board[write][c] = board[r][c];
            board[r][c] = null;
          }
          write--;
        }
      }
    }
  }
  function drawBoard() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    for (let r = 0; r < ROWS; r++) {
      for (let c = 0; c < COLS; c++) {
        if (board[r][c]) drawPolandball(c, r, board[r][c].type);
      }
    }

    if (currentPair) {
      for (const b of currentPair.blocks) {
        const x = currentPair.cx + b.ox;
        const y = currentPair.cy + b.oy;
        if (y >= 0) drawPolandball(x, y, b.type);
      }
    }
  }

function drawBallInfo() {
  const startX = canvas.width - 180; // ← キャンバス内に収める
  let y = 40;

  ctx.textAlign = "left";
  ctx.font = "22px Arial";

  for (const ball of POLANDBALLS) {
    // 名前
    ctx.fillStyle = "white";
    ctx.fillText(ball.name, startX, y);

    // 説明
    ctx.font = "16px Arial";
    ctx.fillStyle = "#cccccc";
    ctx.fillText(ball.desc, startX, y + 20);

    // 小アイコン
    drawPolandballIcon(startX - 30, y - 10, ball.id);

    y += 60;
    ctx.font = "22px Arial";
  }
}



function drawPolandballIcon(x, y, type) {
  const r = 15;
  const pb = POLANDBALLS[type];

  switch (pb.name) {
    case "Palau":      drawPalauFlag(x, y, r); break;
