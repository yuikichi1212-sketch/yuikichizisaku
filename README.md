<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>【極】ザコ専用PBぷよぷよ：最終審判</title>
    <style>
        :root { --bg: #fff; --text: #000; --warn: #f00; }
        body { background: var(--bg); color: var(--text); font-family: 'MS Gothic', 'Courier New', monospace; margin: 0; overflow: hidden; }

        /* メインコンテナ */
        #os-root { display: flex; flex-direction: column; width: 100vw; height: 100vh; border: 20px solid #000; box-sizing: border-box; }
        
        /* 煽りポータル（スタート画面） */
        #portal { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: #fff; z-index: 9999; display: flex; flex-direction: column; align-items: center; justify-content: center; }
        .win-95 { border: 4px solid #000; padding: 60px; background: #ccc; box-shadow: 10px 10px 0px #000; text-align: center; }

        /* ゲームエリア */
        #stage { flex-grow: 1; display: flex; justify-content: center; align-items: center; position: relative; background: #eee; }
        canvas { background: #fff; border: 10px solid #000; box-shadow: 0 0 100px rgba(255,0,0,0.1); }

        /* サイドパネル（右側） */
        #right-ui { width: 350px; background: #ddd; border-left: 10px solid #000; padding: 20px; box-sizing: border-box; display: flex; flex-direction: column; }
        .lcd-display { background: #9db310; border: 4px inset #666; padding: 10px; font-family: 'Digital-7', monospace; color: #222; margin-bottom: 20px; }
        
        /* 煽りボタン群 */
        .insult-btn { background: #000; color: #fff; border: 4px outset #999; padding: 15px; margin: 5px 0; cursor: pointer; font-weight: bold; font-size: 16px; }
        .insult-btn:hover { background: #f00; color: #000; transform: scale(1.05); }
        .insult-btn:active { border-style: inset; }

        /* 警告ポップアップ（動的生成用） */
        .zako-popup { position: absolute; width: 200px; height: 100px; background: #ff0; border: 3px solid #f00; color: #f00; font-weight: bold; z-index: 1000; padding: 10px; pointer-events: none; animation: pop 0.5s ease-out; }
        @keyframes pop { from { transform: scale(0); rotate: 720deg; } to { transform: scale(1); rotate: 0deg; } }

        /* 画面振動 */
        .vibrate { animation: v 0.05s infinite; }
        @keyframes v { 0% { translate: 2px 2px; } 50% { translate: -2px -2px; } 100% { translate: 0 0; } }

        /* 難易度・通信ボタン */
        .sub-btn { font-size: 12px; padding: 5px; margin-top: 5px; background: #666; color: #fff; border: none; cursor: pointer; }
    </style>
</head>
<body id="b">

<div id="os-root">
    <div id="portal">
        <div class="win-95">
            <h1 style="font-size: 60px; margin: 0;">ZAKO-OS v10.0</h1>
            <p style="font-size: 20px; color: red;">※お前の低スペックな人生を初期化中ｗｗｗ</p>
            <hr border="4">
            <button class="insult-btn" style="width: 100%;" onclick="start(500)">ソロプレイ開始（ボッチ乙ｗｗ）</button>
            <button class="insult-btn" style="width: 100%; font-size: 14px;" onclick="fakeMulti()">超・煽り通信対戦（笑）</button>
            <div style="margin-top: 20px;">
                <p>難易度設定（どうせどれもできないだろｗ）</p>
                <button class="sub-btn" onclick="spd=1000; say('赤ちゃん用ねｗ')">赤ちゃん</button>
                <button class="sub-btn" onclick="spd=400; say('普通？お前には無理ｗ')">普通</button>
                <button class="sub-btn" onclick="spd=80; say('死ぬぞｗ')">神レベル</button>
            </div>
        </div>
    </div>

    <div style="background: #000; color: #0f0; padding: 5px; font-size: 14px; text-align: center;">
        SYSTEM_STATUS: [お前の指が遅すぎてCPUがあくびしてます] | 接続：ぼっち回線(速度1bps)
    </div>
    
    <div style="display: flex; flex: 1;">
        <div id="stage">
            <canvas id="cvs" width="750" height="850"></canvas>
            </div>

        <div id="right-ui">
            <div class="lcd-display">
                <div style="font-size: 14px;">Z雑魚スコア:</div>
                <div id="score" style="font-size: 40px; text-align: right;">000000</div>
            </div>

            <p style="font-weight: bold;">お助けゲージ：</p>
            <div style="width: 100%; height: 30px; background: #333; border: 4px solid #000;">
                <div id="gauge" style="width: 0%; height: 100%; background: linear-gradient(90deg, #f00, #ff0);"></div>
            </div>
            
            <button id="skill" class="insult-btn" style="display:none; background: #f00;" onclick="skill()">🔥 革命（お前の罪を消す）</button>

            <div style="margin-top: 50px;">
                <p>煽りアクション：</p>
                <button class="insult-btn" style="width: 100%; font-size: 12px;" onclick="say('おい、画面見ろよｗ')">直接煽る（声）</button>
                <button class="insult-btn" style="width: 100%; font-size: 12px;" onclick="location.reload()">敗走（逃げる）</button>
                <button class="insult-btn" style="width: 100%; font-size: 12px;" onclick="hackEffect()">PC破壊（偽）</button>
            </div>
            
            <div style="flex-grow: 1; border: 2px dashed #666; margin-top: 20px; font-size: 11px; padding: 5px;">
                履歴:<br>
                - 接続成功（ぼっち）<br>
                - ザコ検知: OK<br>
                - 煽りモジュール: 正常
            </div>
        </div>
    </div>
</div>

<script>
    const C=10, R=12, S=70, cvs=document.getElementById("cvs"), ctx=cvs.getContext("2d");
    let board=[], p=null, sc=0, g=0, over=false, active=false, spd=400, timer=0, last=0;

    const BALLS = [
        { c1: "#4AADD6", c2: "#FFDE00" }, // Palau
        { c1: "#FFFFFF", c2: "#BC002D" }, // Japan
        { c1: "#006A4E", c2: "#F42A41" }, // Bangladesh
        { c1: "#FFFFFF", c2: "#d00c33" }, // Greenland
        { c1: "#E8112D", c2: "#FFD700" }  // Kyrgyzstan
    ];

    const INSULTS = ["下手くそワロタ", "それ置くの？ｗ", "赤ちゃんかな？", "ゆびいたい？♡", "指ついてる？", "画面見てる？", "運ゲー乙ｗ"];

    function say(t) {
        const u = new SpeechSynthesisUtterance(t);
        u.pitch = Math.random() * 2; u.rate = 1.3;
        speechSynthesis.speak(u);
    }

    function createPopup() {
        const pop = document.createElement("div");
        pop.className = "zako-popup";
        pop.innerText = INSULTS[Math.floor(Math.random()*INSULTS.length)];
        pop.style.left = Math.random() * 500 + "px";
        pop.style.top = Math.random() * 600 + "px";
        document.getElementById("stage").appendChild(pop);
        setTimeout(() => pop.remove(), 1000);
    }

    function start() {
        document.getElementById("portal").style.display = "none";
        board = Array.from({length:R}, ()=>Array(C).fill(null));
        sc=0; g=0; over=false; active=true;
        say("ザコ専用OS、ブート開始。お前の実力を笑いに来たぞｗ");
        spawn(); update();
    }

    function fakeMulti() {
        say("対戦相手を検索中... あ、一人見つかりましたよ（笑）");
        setTimeout(() => {
            alert("対戦相手: [伝説のぷよ師] が入室しました。");
            say("相手が『え、人？？ザコすぎて時間の無駄だわ（笑）お前とやる価値ない』と言って切断しました。");
            setTimeout(() => {
                alert("対戦相手が「お前とはやる価値がないｗ」と言って切断しました。");
                say("はい、ぼっち確定。一人で虚しくやってな（笑）");
            }, 2000);
        }, 1500);
    }

    function spawn() {
        p = {x:4, y:0, b:[{ox:0, oy:0, t:rand()}, {ox:0, oy:-1, t:rand()}]};
        if(board[0][4]) {
            over=true;
            say("はい、ゲームオーバー。ザコ。本当にザコ。こっからどうするの？ｗにげちゃうかな？（笑）");
        }
    }
    function rand() { return Math.floor(Math.random()*5); }

    function drawBall(x, y, t) {
        const r = S*0.45;
        ctx.save();
        ctx.translate(x*S+S/2, y*S+S/2);
        ctx.beginPath(); ctx.arc(0,0,r,0,Math.PI*2); ctx.fillStyle=BALLS[t].c1; ctx.fill();
        ctx.strokeStyle="#000"; ctx.lineWidth=3; ctx.stroke();
        ctx.beginPath(); ctx.arc(-r*0.3,0,r*0.4,0,Math.PI*2); ctx.fillStyle=BALLS[t].c2; ctx.fill();
        // 煽り目
        ctx.strokeStyle="#000"; ctx.lineWidth=2;
        ctx.beginPath(); ctx.moveTo(-10,-5); ctx.lineTo(-2,-5); ctx.stroke();
        ctx.beginPath(); ctx.moveTo(10,-5); ctx.lineTo(2,-5); ctx.stroke();
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
        ctx.clearRect(0,0,750,850);
        board.forEach((row,y)=>row.forEach((v,x)=>{ if(v!==null) drawBall(x,y,v); }));
        if(p) p.b.forEach(b=>drawBall(p.x+b.ox, p.y+b.oy, b.t));
        if(over) {
            ctx.fillStyle="rgba(255,0,0,0.8)"; ctx.fillRect(0,0,750,850);
            ctx.fillStyle="#fff"; ctx.font="60px bold serif"; ctx.textAlign="center";
            ctx.fillText("ザコすぎて話にならなくてくうさ！", 375, 425);
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
                hit=true; sc+=group.length*100; g=Math.min(100, g+20);
                createPopup();
            }
        }
        if(hit) {
            gravity();
            document.getElementById("score").innerText = sc.toString().padStart(6, '0');
            document.getElementById("gauge").style.width = g + "%";
            if(g>=100) document.getElementById("skill").style.display="block";
            say(INSULTS[Math.floor(Math.random()*INSULTS.length)]);
            document.getElementById("b").classList.add("vibrate");
            setTimeout(()=>document.getElementById("b").classList.remove("vibrate"), 200);
            setTimeout(check, 250);
        }
    }

    function gravity() {
        for(let x=0; x<C; x++){
            let empty=R-1;
            for(let y=R-1; y>=0; y--) if(board[y][x]!==null){ let t=board[y][x]; board[y][x]=null; board[empty][x]=t; empty--; }
        }
    }

    function skill() {
        say("はい、実力じゃ勝てないから神頼みね（笑） 情けなくて草");
        board = Array.from({length:R}, ()=>Array(C).fill(null));
        g=0; document.getElementById("gauge").style.width = "0%";
        document.getElementById("skill").style.display="none";
        document.getElementById("stage").style.filter = "invert(1)";
        setTimeout(()=>document.getElementById("stage").style.filter = "none", 1000);
    }

    function hackEffect() {
        say("あーあ、変なボタン押しちゃった。お前のPC、もうザコすぎて耐えられないってさｗ");
        document.body.style.transform = "rotate(5deg) scale(0.9)";
        setTimeout(()=>document.body.style.transform = "none", 2000);
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
