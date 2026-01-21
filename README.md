<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>うん！ぷよぷよしよ！</title>
    <style>
        :root { --bg: #fff; --text: #000; --warn: #f00; }
        body { background: var(--bg); color: var(--text); font-family: 'MS Gothic', sans-serif; margin: 0; overflow: hidden; height: 100vh; display: flex; flex-direction: column; }

        /* メインコンテナ：はみ出し防止のflex構造 */
        #os-root { display: flex; flex: 1; border: 15px solid #000; box-sizing: border-box; background: #ddd; height: 100%; }
        
        /* ステージエリア：画面に合わせて自動縮小 */
        #stage { flex-grow: 1; display: flex; justify-content: center; align-items: center; position: relative; background: #eee; overflow: hidden; }
        canvas { background: #fff; border: 8px solid #000; max-height: 90vh; max-width: 100%; object-fit: contain; }

        /* 右側操作パネル：固定幅だが高さは100% */
        #right-ui { width: 300px; background: #ccc; border-left: 8px solid #000; padding: 15px; box-sizing: border-box; display: flex; flex-direction: column; overflow-y: auto; }
        
        .lcd-display { background: #9db310; border: 4px inset #666; padding: 10px; font-family: monospace; color: #222; margin-bottom: 15px; }
        .insult-btn { background: #000; color: #fff; border: 4px outset #999; padding: 12px; margin: 4px 0; cursor: pointer; font-weight: bold; font-size: 14px; }
        .insult-btn:hover { background: #f00; color: #000; }

        /* ゲージ：3倍貯まりにくい設定 */
        .gauge-wrap { width: 100%; height: 25px; background: #333; border: 3px solid #000; position: relative; }
        #gauge-bar { width: 0%; height: 100%; background: linear-gradient(90deg, #f00, #ff0); transition: 0.5s; }

        /* ポータル画面 */
        #portal { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: #fff; z-index: 9999; display: flex; flex-direction: column; align-items: center; justify-content: center; }
        .win-box { border: 5px solid #000; padding: 30px; background: #eee; box-shadow: 10px 10px 0px #000; text-align: center; }

        /* 破壊演出用：地獄のアニメーション */
        @keyframes glitch-shake {
            0% { transform: translate(0); clip-path: inset(0% 0% 0% 0%); }
            10% { transform: translate(-10px, 5px); clip-path: inset(10% 0% 30% 0%); }
            20% { transform: translate(10px, -5px); clip-path: inset(30% 0% 10% 0%); filter: hue-rotate(90deg); }
            100% { transform: translate(0); }
        }
        .system-crashing { animation: glitch-shake 0.1s infinite; filter: invert(1) contrast(200%); }

        .zako-pop { position: absolute; background: #ff0; border: 2px solid #f00; color: #f00; padding: 5px; font-size: 12px; font-weight: bold; z-index: 1000; }
    </style>
</head>
<body id="main-body">

<div id="portal">
    <div class="win-box">
        <h1 style="margin:0;">ざこOS1.28バージョンでございま＾す！</h1>
        <p>うん！ぷよぷよしよw</p>
        <button class="insult-btn" style="width:100%" onclick="startGame()">孤独に開始（笑）</button>
        <button class="insult-btn" style="width:100%; font-size:12px;" onclick="fakeMulti()">オンライン煽りマッチ</button>
    </div>
</div>

<div style="background:#000; color:#fff; padding:3px; font-size:11px; text-align:center;">
    ERROR_LOG: [お前の指、3倍速にしてもまだ遅いぞｗ] | UI_MODE: フィット（逃げ場なし）
</div>

<div id="os-root">
    <div id="stage">
        <canvas id="gameCanvas" width="600" height="750"></canvas>
    </div>

    <div id="right-ui">
        <div class="lcd-display">
            <div>ZAKO_SCORE:</div>
            <div id="scoreDisplay" style="font-size:32px; text-align:right;">000000</div>
        </div>

        <p style="font-size:12px; font-weight:bold; margin:5px 0;">お助けゲージ：</p>
        <div class="gauge-wrap">
            <div id="gauge-bar"></div>
        </div>
        
        <button id="skill-btn" class="insult-btn" style="display:none; background:red;" onclick="useSkill()">★ お助けボタン！</button>

        <div style="margin-top:20px;">
            <button class="insult-btn" style="width:100%" onclick="say('諦めたら？ｗ')">肉声で煽る</button>
            <button class="insult-btn" style="width:100%; background: #333;" onclick="destroyPC()">ここ押すのおすすめだよ♡</button>
            <button class="insult-btn" style="width:100%; background: #666;" onclick="location.reload()">拗ねちゃうw</button>
        </div>

        <div id="log" style="font-size:10px; margin-top:auto; border:1px solid #999; padding:5px; height:80px; overflow-y:hidden; background:#fff;">
            - システム正常（お前以外）<br>
            - 解像度調整：完了<br>
            - 煽り待機中...
        </div>
    </div>
</div>

<script>
    const C=10, R=12, S=60, canvas=document.getElementById("gameCanvas"), ctx=canvas.getContext("2d");
    let board=[], p=null, sc=0, g=0, over=false, active=false, spd=400, timer=0, last=0;

    const BALLS = [
        { c1: "#4AADD6", c2: "#FFDE00" }, { c1: "#FFF", c2: "#BC002D" },
        { c1: "#006A4E", c2: "#F42A41" }, { c1: "#FFF", c2: "#d00c33" }, { c1: "#E8112D", c2: "#FFD700" }
    ];

    function say(t) {
        const u = new SpeechSynthesisUtterance(t);
        u.pitch = 0.4; u.rate = 1.2;
        speechSynthesis.speak(u);
    }

    function startGame() {
        document.getElementById("portal").style.display = "none";
        board = Array.from({length:R}, ()=>Array(C).fill(null));
        sc=0; g=0; over=false; active=true;
        say("お、キタキタきたーー");
        spawn(); update();
    }

    function fakeMulti() {
        say("接続中、ちょっと待ってろ");
        setTimeout(() => {
            alert("対戦相手 [ぷよ神] が入室しました");
            say("あ、相手が『お前みたいなはみ出しザコとはやりたくないｗ』って言ってるぞｗ笑");
            setTimeout(() => {
                alert("対戦相手が「時間の無駄ｗ」と言って切断しました。");
                say("はい、永遠のボッチ確定ｗ");
            }, 1500);
        }, 1000);
    }

    function spawn() {
        p = {x:4, y:0, b:[{ox:0, oy:0, t:rand()}, {ox:0, oy:-1, t:rand()}]};
        if(board[0][4]) { over=true; say("ゲームオーバーｗ 下手すぎて画面が泣いてるぞｗ"); }
    }
    function rand() { return Math.floor(Math.random()*5); }

    function drawBall(x, y, t) {
        const r = S*0.45;
        ctx.save();
        ctx.translate(x*S+S/2, y*S+S/2);
        ctx.beginPath(); ctx.arc(0,0,r,0,Math.PI*2); ctx.fillStyle=BALLS[t].c1; ctx.fill();
        ctx.strokeStyle="#000"; ctx.lineWidth=2; ctx.stroke();
        ctx.beginPath(); ctx.arc(-r*0.3,0,r*0.4,0,Math.PI*2); ctx.fillStyle=BALLS[t].c2; ctx.fill();
        // 煽り目
        ctx.beginPath(); ctx.moveTo(-10,-5); ctx.lineTo(-2,-5); ctx.moveTo(10,-5); ctx.lineTo(2,-5); ctx.stroke();
        ctx.restore();
    }

    function update(t=0) {
        if(!active) return;
        const dt = t - last; last = t;
        if(!over) {
            timer += dt;
            if(timer > spd) {
                timer=0;
                if(!coll(0,1)) p.y++; else lock();
            }
        }
        ctx.clearRect(0,0,600,750);
        board.forEach((row,y)=>row.forEach((v,x)=>{ if(v!==null) drawBall(x,y,v); }));
        if(p) p.b.forEach(b=>drawBall(p.x+b.ox, p.y+b.oy, b.t));
        if(over) {
            ctx.fillStyle="rgba(0,0,0,0.8)"; ctx.fillRect(0,0,600,750);
            ctx.fillStyle="red"; ctx.font="40px Arial"; ctx.textAlign="center";
            ctx.fillText("ザコ認定：完了ｗ", 300, 375);
        }
        requestAnimationFrame(update);
    }

    function coll(dx,dy,blks=p.b) {
        return blks.some(b=>{
            let nx=p.x+b.ox+dx, ny=p.y+b.oy+dy;
            return nx<0||nx>=C||ny>=R||(ny>=0&&board[ny][nx]!==null);
        });
    }

    function lock() {
        p.b.forEach(b=>{ if(p.y+b.oy>=0) board[p.y+b.oy][p.x+b.ox]=b.t; });
        check(); spawn();
    }

    function check() {
        let hit = false;
        for(let y=0; y<R; y++) for(let x=0; x<C; x++) {
            if(board[y][x]===null) continue;
            let target=board[y][x], group=[], visited=board.map(r=>r.map(()=>false)), stack=[{x,y}];
            visited[y][x]=true;
            while(stack.length){
                let c=stack.pop(); group.push(c);
                [[0,1],[0,-1],[1,0],[-1,0]].forEach(d=>{
                    let nx=c.x+d[0], ny=c.y+d[1];
                    if(nx>=0&&nx<C&&ny>=0&&ny<R&&!visited[ny][nx]&&board[ny][nx]===target){
                        visited[ny][nx]=true; stack.push({x:nx,y:ny});
                    }
                });
            }
            if(group.length >= 4) {
                group.forEach(g=>board[g.y][g.x]=null);
                hit=true; 
                sc+=group.length*100; 
                // ゲージ：従来の3倍貯まりにくく修正 (1%ずつ)
                g = Math.min(100, g + 5); 
                popMsg();
            }
        }
        if(hit) {
            gravity();
            document.getElementById("scoreDisplay").innerText = sc.toString().padStart(6, '0');
            document.getElementById("gauge-bar").style.width = g + "%";
            if(g>=100) document.getElementById("skill-btn").style.display="block";
            say("たまたま消せただけだろｗ");
            setTimeout(check, 250);
        }
    }

    function gravity() {
        for(let x=0; x<C; x++){
            let empty=R-1;
            for(let y=R-1; y>=0; y--) if(board[y][x]!==null){ let t=board[y][x]; board[y][x]=null; board[empty][x]=t; empty--; }
        }
    }

    function popMsg() {
        const p = document.createElement("div");
        p.className = "zako-pop";
        p.innerText = "ザコ草！";
        p.style.left = Math.random()*80+"%"; p.style.top = Math.random()*80+"%";
        document.getElementById("stage").appendChild(p);
        setTimeout(()=>p.remove(), 800);
    }

    function useSkill() {
        say("うん！、来ると思った！、実力じゃ無理だもんなｗ");
        board = Array.from({length:R}, ()=>Array(C).fill(null));
        g=0; document.getElementById("gauge-bar").style.width = "0%";
        document.getElementById("skill-btn").style.display="none";
    }

    /* --- PC破壊ボタン演出（強化版） --- */
    function destroyPC() {
        say("jaisjsおfsつfsかrieれ");
        const body = document.getElementById("main-body");
        body.classList.add("system-crashing");
        
        // 偽のブルースクリーン的な音と視覚効果
        let i = 0;
        const interval = setInterval(() => {
            const crashTxt = document.createElement("div");
            crashTxt.style.position = "fixed";
            crashTxt.style.color = "lime";
            crashTxt.style.background = "black";
            crashTxt.style.top = Math.random()*100+"vh";
            crashTxt.style.left = Math.random()*100+"vw";
            crashTxt.style.zIndex = "10000";
            crashTxt.innerText = "FATAL_ERROR: ZAKO_DETECTED";
            document.body.appendChild(crashTxt);
            if(i++ > 50) clearInterval(interval);
        }, 50);

        setTimeout(() => {
            body.classList.remove("system-crashing");
            alert("PCに煽り菌が発生しました。今すぐ再起動してください！");
            document.querySelectorAll("div[style*='position: fixed']").forEach(el => el.remove());
        }, 3000);
    }

    window.onkeydown = e => {
        if(!p || over) return;
        if(e.key==="ArrowLeft" && !coll(-1,0)) p.x--;
        if(e.key==="ArrowRight" && !coll(1,0)) p.x++;
        if(e.key==="ArrowDown") { if(!coll(0,1)) p.y++; else lock(); }
        if(e.key==="ArrowUp") {
            let r = p.b.map((b,i)=>i===0?b:{ox:-b.oy, oy:b.ox, t:b.t});
            if(!coll(0,0,r)) p.b=r;
        }
    };
</script>
</body>
</html>
