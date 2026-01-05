<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>ゆいきちナビ</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <link href="https://fonts.googleapis.com/css2?family=M+PLUS+Rounded+1c:wght@700&display=swap" rel="stylesheet">
    <style>
        /* UIデザイン - スマホ最適化 */
        body { margin: 0; padding: 0; font-family: 'M PLUS Rounded 1c', sans-serif; overflow: hidden; background: #eee; }
        
        /* ヘッダー */
        .app-header {
            position: absolute; top: 0; left: 0; right: 0; height: 60px;
            background: linear-gradient(135deg, #00b09b, #96c93d);
            color: white; display: flex; align-items: center; justify-content: center;
            font-size: 24px; box-shadow: 0 2px 10px rgba(0,0,0,0.2); z-index: 2000;
        }

        #map { position: absolute; top: 60px; bottom: 0; width: 100%; z-index: 1; }

        /* ナビゲーションパネル（下部） */
        .nav-panel {
            position: absolute; bottom: 20px; left: 15px; right: 15px;
            background: rgba(255, 255, 255, 0.95); padding: 20px; border-radius: 20px;
            box-shadow: 0 -5px 20px rgba(0,0,0,0.15); z-index: 1000;
            transition: transform 0.3s ease;
        }

        /* 情報表示エリア */
        .info-grid {
            display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 15px;
        }
        .info-box { text-align: center; background: #f0f2f5; padding: 10px; border-radius: 12px; }
        .info-value { font-size: 22px; color: #333; }
        .info-label { font-size: 11px; color: #777; }
        .large-text { color: #00b09b; }

        /* モード切替ボタン */
        .mode-selector {
            display: flex; justify-content: space-around; margin-bottom: 15px;
            background: #e9ecef; border-radius: 30px; padding: 5px;
        }
        .mode-btn {
            flex: 1; text-align: center; padding: 8px; border-radius: 25px; cursor: pointer; font-size: 14px;
            transition: all 0.2s; color: #666;
        }
        .mode-btn.active { background: #fff; color: #00b09b; font-weight: bold; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }

        /* アクションボタン */
        .action-btn {
            width: 100%; padding: 15px; border: none; border-radius: 30px;
            font-size: 18px; font-weight: bold; color: white; cursor: pointer;
            box-shadow: 0 4px 10px rgba(0,0,0,0.2);
        }
        .btn-start { background: #007bff; }
        .btn-stop { background: #dc3545; display: none; }

        /* カスタムマーカー（自分） */
        .my-location {
            width: 0; height: 0;
            border-left: 12px solid transparent;
            border-right: 12px solid transparent;
            border-bottom: 24px solid #007bff;
            filter: drop-shadow(0 0 5px white);
            transition: transform 0.5s ease;
        }
    </style>
</head>
<body>

    <div class="app-header">ゆいきちナビ </div>

    <div id="map"></div>

    <div class="nav-panel">
        <div class="mode-selector">
            <div class="mode-btn active" onclick="setMode('car')"> 車</div>
            <div class="mode-btn" onclick="setMode('bike')"> 自転車</div>
            <div class="mode-btn" onclick="setMode('walk')"> 徒歩</div>
        </div>

        <div class="info-grid">
            <div class="info-box">
                <div id="dist-val" class="info-value large-text">--</div>
                <div class="info-label">残り距離 (km)</div>
            </div>
            <div class="info-box">
                <div id="time-val" class="info-value">--</div>
                <div class="info-label">到着予想 (分)</div>
            </div>
            <div class="info-box">
                <div id="speed-val" class="info-value">0</div>
                <div class="info-label">速度 (km/h)</div>
            </div>
            <div class="info-box">
                <div id="status-text" class="info-value" style="font-size:14px; line-height:33px;">待機中</div>
                <div class="info-label">ステータス</div>
            </div>
        </div>

        <button id="start-btn" class="action-btn btn-start" onclick="startNav()">ナビを開始する</button>
        <button id="stop-btn" class="action-btn btn-stop" onclick="stopNav()">ナビを終了する</button>
    </div>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script>
        // --- 設定値 ---
        const speeds = { car: 40, bike: 15, walk: 4.8 };
        let currentMode = 'car';
        let rerouteThreshold = 0.05; // 50m離れたらリルート判定

        // --- 変数 ---
        let map, userMarker, destMarker, routeLine, traceLine;
        let watchId = null;
        let currentPos = null;      // 現在地 [lat, lng]
        let destinationPos = null;  // 目的地 [lat, lng]
        let startPosForRoute = null; // ルート計算の基準点（リルート判定用）
        let traceCoords = [];       // 軌跡用配列

        // 1. マップ初期化
        map = L.map('map', { zoomControl: false }).setView([35.6812, 139.7671], 15);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

        // 自分アイコン
        const myIcon = L.divIcon({
            className: 'my-location',
            iconSize: [24, 24],
            iconAnchor: [12, 18]
        });

        // 2. 目的地設定（タップ）
        map.on('click', (e) => {
            if (watchId) return; // ナビ中は変更不可
            
            destinationPos = e.latlng;
            
            if (destMarker) map.removeLayer(destMarker);
            destMarker = L.marker(destinationPos).addTo(map).bindPopup("🏁 目的地").openPopup();
            
            updateInfoDisplay(0); // 表示更新
            document.getElementById('status-text').innerText = "準備OK";
            speak("目的地をセットしました。");
        });

        // 3. モード切替
        function setMode(mode) {
            currentMode = mode;
            document.querySelectorAll('.mode-btn').forEach(btn => btn.classList.remove('active'));
            event.target.classList.add('active');
            
            // 再計算
            if (destinationPos && currentPos) {
                const dist = map.distance(currentPos, destinationPos) / 1000;
                updateInfoDisplay(dist);
            }
            speak(mode === 'car' ? "車モード" : mode === 'bike' ? "自転車モード" : "徒歩モード");
        }

        // 4. ナビ開始
        function startNav() {
            if (!destinationPos) {
                alert("地図をタップして目的地を決めてください！");
                return;
            }
            if (!navigator.geolocation) {
                alert("GPSが使えません");
                return;
            }

            // UI変更
            document.getElementById('start-btn').style.display = 'none';
            document.getElementById('stop-btn').style.display = 'block';
            document.getElementById('status-text').innerText = "ナビ中...";

            speak("ゆいきちナビ、スタートです！安全運転で行きましょう。");

            // GPS追跡開始
            watchId = navigator.geolocation.watchPosition(
                onLocationUpdate, 
                err => console.error(err), 
                { enableHighAccuracy: true, maximumAge: 0, timeout: 5000 }
            );
        }

        // 5. ナビ終了
        function stopNav() {
            if (watchId) navigator.geolocation.clearWatch(watchId);
            watchId = null;
            document.getElementById('start-btn').style.display = 'block';
            document.getElementById('stop-btn').style.display = 'none';
            document.getElementById('status-text').innerText = "終了";
            traceCoords = []; // 軌跡リセット
            speak("ナビを終了します。お疲れ様でした。");
        }

        // 6. メインロジック（位置更新時に呼ばれる）
        function onLocationUpdate(position) {
            const { latitude, longitude, heading, speed } = position.coords;
            currentPos = L.latLng(latitude, longitude);

            // 初回、またはリルート後の基準点設定
            if (!startPosForRoute) startPosForRoute = currentPos;

            // --- A. マーカー更新 ---
            if (!userMarker) {
                userMarker = L.marker(currentPos, {icon: myIcon}).addTo(map);
            } else {
                userMarker.setLatLng(currentPos);
                // ヘディング回転（対応端末のみ）
                if (heading) {
                    document.querySelector('.my-location').style.transform = `rotate(${heading}deg)`;
                }
            }

            // --- B. 軌跡（足跡）描画 ---
            traceCoords.push(currentPos);
            if (traceLine) map.removeLayer(traceLine);
            traceLine = L.polyline(traceCoords, { color: '#00b09b', weight: 8, opacity: 0.6 }).addTo(map);

            // --- C. ルートラインとリルート判定 ---
            // 擬似的なリルートロジック：
            // 現在地から目的地への直線を描くが、もし「本来のライン」から大きく外れたら「リルート」演出を入れる。
            // ※APIがないため、今回は「常に目的地への直線を更新し続ける」ことで自動補正します。
            
            // 前回のルートラインまでの距離を計算（簡易版：スタート地点からの距離が離れたらリルートとみなす演出）
            // ここではシンプルに「常に最新のルート（青点線）を引き直す」処理にします。
            if (routeLine) map.removeLayer(routeLine);
            routeLine = L.polyline([currentPos, destinationPos], { color: '#007bff', weight: 5, dashArray: '10, 15' }).addTo(map);

            // リルート演出判定（あくまで演出です）
            // もし「目的地までの距離」が、直前の計算より極端に増えた場合などを検知できますが、
            // 今回はシンプルに「30秒おき」や「特定の条件」でアナウンスを入れる代わりに、
            // 常に最短（直線）を示す仕様としています。

            // --- D. 情報更新 ---
            const distMeters = map.distance(currentPos, destinationPos);
            const distKm = distMeters / 1000;
            
            // 速度表示 (m/s -> km/h)
            const speedKmh = speed ? (speed * 3.6).toFixed(0) : 0;
            document.getElementById('speed-val').innerText = speedKmh;

            updateInfoDisplay(distKm);

            map.panTo(currentPos); // カメラ追従

            // --- E. 到着判定 ---
            if (distMeters < 30) {
                speak("まもなく目的地です。案内を終了します。");
                stopNav();
            }
        }

        // 表示更新用関数
        function updateInfoDisplay(distKm) {
            document.getElementById('dist-val').innerText = distKm.toFixed(1);
            
            const speed = speeds[currentMode];
            const timeMin = Math.round((distKm / speed) * 60);
            document.getElementById('time-val').innerText = timeMin;
        }

        // リルート（シミュレーション用関数：本来はここでルート再計算APIを叩く）
        function checkReroute(current, start, end) {
            // 直線からの乖離距離を計算するのは複雑なため、
            // 今回は「目的地の方角」が大きく変わった場合にトリガーするなどが考えられます。
            // このサンプルではシンプルさを優先し、実装を省略しています。
        }

        // 音声合成
        function speak(text) {
            if (!window.speechSynthesis) return;
            // 読み上げ中の場合はキャンセル
            window.speechSynthesis.cancel();
            const uttr = new SpeechSynthesisUtterance(text);
            uttr.lang = 'ja-JP';
            uttr.rate = 1.0;
            window.speechSynthesis.speak(uttr);
        }
    </script>
</body>
</html>
