<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>ゆいきちナビ>ゆいきちな>ゆ
<style>
  body { font-family: sans-serif; }
  canvas { border: 1px solid #333; }
</style>
</head>
<body>

<h2>ゆいきちナビ>ゆいき>

出発：
<select id="start"></select>
目的地：
<select id="goal"></select>
<button onclick="navigate()">ナビ開始</button>

<p id="result"></p>

<canvas id="map" width="600" height="400"></canvas>

<script>
const canvas = document.getElementById("map");
const ctx = canvas.getContext("2d");

// ===== 地点データ =====
const nodes = {
  A: { x: 100, y: 200 },
  B: { x: 300, y: 100 },
  C: { x: 500, y: 200 },
  D: { x: 300, y: 300 }
};

// ===== 道データ =====
const edges = {
  A: [{to:"B", cost:2}, {to:"D", cost:4}],
  B: [{to:"A", cost:2}, {to:"C", cost:3}],
  C: [{to:"B", cost:3}, {to:"D", cost:1}],
  D: [{to:"A", cost:4}, {to:"C", cost:1}]
};

// セレクト初期化
for (let k in nodes) {
  start.innerHTML += `<option>${k}</option>`;
  goal.innerHTML += `<option>${k}</option>`;
}

// ===== 地図描画 =====
function draw(path=[]) {
  ctx.clearRect(0,0,600,400);

  // 道
  for (let from in edges) {
    edges[from].forEach(e=>{
      ctx.beginPath();
      ctx.moveTo(nodes[from].x, nodes[from].y);
      ctx.lineTo(nodes[e.to].x, nodes[e.to].y);
      ctx.strokeStyle = "#aaa";
      ctx.stroke();
    });
  }

  // ルート
  if (path.length > 0) {
    ctx.beginPath();
    ctx.moveTo(nodes[path[0]].x, nodes[path[0]].y);
    path.forEach(p=>{
      ctx.lineTo(nodes[p].x, nodes[p].y);
    });
    ctx.strokeStyle = "red";
    ctx.lineWidth = 4;
    ctx.stroke();
    ctx.lineWidth = 1;
  }

  // 地点
  for (let k in nodes) {
    ctx.beginPath();
    ctx.arc(nodes[k].x, nodes[k].y, 8, 0, Math.PI*2);
    ctx.fill();
    ctx.fillText(k, nodes[k].x+10, nodes[k].y);
  }
}

draw();

// ===== ダイクストラ =====
function dijkstra(start, goal) {
  const dist = {};
  const prev = {};
  const q = new Set(Object.keys(nodes));

  for (let k in nodes) dist[k] = Infinity;
  dist[start] = 0;

  while (q.size) {
    let u = [...q].reduce((a,b)=>dist[a]<dist[b]?a:b);
    q.delete(u);

    if (u === goal) break;

    edges[u].forEach(e=>{
      let alt = dist[u] + e.cost;
      if (alt < dist[e.to]) {
        dist[e.to] = alt;
        prev[e.to] = u;
      }
    });
  }

  let path = [];
  let u = goal;
  while (u) {
    path.unshift(u);
    u = prev[u];
  }
  return path;
}

// ===== ナビ開始 =====
function navigate() {
  const s = start.value;
  const g = goal.value;
  const path = dijkstra(s, g);
  draw(path);
  result.textContent = "ルート：" + path.join(" → ");
}
</script>

</body>
</html>
