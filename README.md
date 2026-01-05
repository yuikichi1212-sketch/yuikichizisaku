<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>ゆいきちナビ</title>
    
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <link rel="stylesheet" href="https://unpkg.com/leaflet-routing-machine@3.2.12/dist/leaflet-routing-machine.css" />

    <style>
        /* --- デザイン定義 (モノクロ・シンプル) --- */
        body { margin: 0; padding: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; overflow: hidden; background: #333; }
        
        /* 地図本体 */
        #map { position: absolute; top: 0; bottom: 0; width: 100%; z-index: 1; background: #ddd; transition: transform 0.5s linear; }

        /* ヘッダー (シンプル) */
        .header {
            position: absolute; top: 0; left: 0; right: 0; 
            background: rgba(255, 255, 255, 0.95); padding: 10px; z-index: 1000;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1); display: flex; flex-direction: column; gap: 8px;
        }
        .app-title { font-size: 16px; font-weight: bold; color: #333; text-align: center; margin-bottom: 5px; }

        /* 検索フォーム */
        .search-box { display: flex; gap: 5px; }
        input { flex: 1; padding: 8px; border: 1px solid #ccc; border-radius: 4px; font-size: 14px; }
        button { padding: 8px 15px; background: #333; color: #fff; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; }
        button:active { background: #555; }

        /* ナビ情報パネル (下部・コンパクト) */
        .nav-info {
            position: absolute; bottom: 20px; left: 10px; right: 10px;
            background: rgba(255, 255, 255, 0.95); padding: 15px; border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.2); z-index: 1000;
            display: none; /* 最初は非表示 */
        }
        .instruction { font-size: 18px; font-weight: bold; color: #000; margin-bottom: 5px; }
        .details { font-size: 14px; color: #666; display: flex; justify-content: space-between; }
        
        /* 自車アイコン */
        .user-marker {
            width: 20px; height: 20px; background: #007bff; border: 2px solid #fff;
            border-radius: 50%; box-shadow: 0 0 5px rgba(0,0,0,0.5);
        }

        /* Leaflet Routing Machineのデフォルト表示を消す */
        .leaflet-routing-container { display: none !important; }

        /* コンパスボタン */
        .compass-btn {
            position: absolute; bottom: 100px; right: 10px; width: 40px; height: 40px;
            background: white; border-radius: 50%; z-index: 1000;
            display: flex; align-items: center; justify-content: center;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2); font-size: 20px; cursor: pointer;
        }
    </style>
</head>
<body>

    <div class="header">
        <div class="app-title">ゆいきちナビ</div>
        <div class="search-box">
            <input type="text" id="start-input" placeholder="出発地 (例: 東京駅)" />
            <input type="text" id="end-input" placeholder="目的地 (例: スカイツリー)" />
            <button onclick="searchAndRoute()">検索</button>
        </div>
    </div>

    <div id="map"></div>

    <div class="compass-btn" onclick="resetMapRotation()">N</div>

    <div id="nav-panel" class="nav-info">
        <div id="instruction-text" class="instruction">ルートを検索してください</div>
        <div class="details">
            <span id="distance-text">-- km</span>
            <span id="time-text">-- 分</span>
        </div>
    </div>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script src="https://unpkg.com/leaflet-routing-machine@3.2.12/dist/leaflet-routing-machine.js"></script>

    <script>
        // --- 変数定義 ---
        let map;
        let routingControl = null;
        let userMarker = null;
        let watchId = null;
        let currentHeading = 0;

        // 1. 地図の初期化
        map = L.map('map', { zoomControl: false }).setView([35.6812, 139.7671], 15);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            attribution: '&copy; OpenStreetMap contributors'
        }).addTo(map);

        // 2. 検索 & ルート計算 (Nominatim + OSRM)
        async function searchAndRoute() {
            const startText = document.getElementById('start-input').value;
            const endText = document.getElementById('end-input').value;

            if(!startText || !endText) {
                alert("出発地と目的地を入力してください");
                return;
            }

            try {
                // 住所から座標へ変換 (Geocoding)
                const startCoords = await getCoords(startText);
                const endCoords = await getCoords(endText);

                if(!startCoords || !endCoords) {
                    alert("場所が見つかりませんでした");
                    return;
                }

                // 既存のルートがあれば消す
                if(routingControl) {
                    map.removeControl(routingControl);
                }

                // ルート計算エンジンの起動 (道路沿いのルート)
                routingControl = L.Routing.control({
                    waypoints: [
                        L.latLng(startCoords.lat, startCoords.lon),
                        L.latLng(endCoords.lat, endCoords.lon)
                    ],
                    routeWhileDragging: false,
                    language: 'ja', // 日本語案内
                    show: false,    // デフォルトのパネルは隠す
                    createMarker: function() { return null; }, // デフォルトマーカーを消す
                    lineOptions: {
                        styles: [{color: '#007bff', opacity: 0.8, weight: 6}]
                    }
                }).addTo(map);

                // ルートが見つかった時の処理
                routingControl.on('routesfound', function(e) {
                    const routes = e.routes;
                    const summary = routes[0].summary;
                    
                    // パネル表示
                    document.getElementById('nav-panel').style.display = 'block';
                    document.getElementById('distance-text').innerText = (summary.totalDistance / 1000).toFixed(1) + " km";
                    document.getElementById('time-text').innerText = Math.round(summary.totalTime / 60) + " 分";
                    
                    // 最初の指示を表示
                    if(routes[0].instructions.length > 0) {
                        document.getElementById('instruction-text').innerText = routes[0].instructions[0].text;
                    }

                    // GPSナビ開始
                    startGPSNavigation();
                });

            } catch (err) {
                alert("エラーが発生しました: " + err);
            }
        }

        // 住所検索関数 (Nominatim API)
        async function getCoords(query) {
            const url = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(query)}`;
            const res = await fetch(url);
            const data = await res.json();
            return data[0];
        }

        // 3. GPS追跡と地図回転
        function startGPSNavigation() {
            if (!navigator.geolocation) return;

            watchId = navigator.geolocation.watchPosition(position => {
                const lat = position.coords.latitude;
                const lng = position.coords.longitude;
                const heading = position.coords.heading; // 進行方向 (0-360)

                const currentLatLng = [lat, lng];

                // 自車マーカー更新
                if (!userMarker) {
                    userMarker = L.marker(currentLatLng, {
                        icon: L.divIcon({ className: 'user-marker', iconSize: [20,20] })
                    }).addTo(map);
                } else {
                    userMarker.setLatLng(currentLatLng);
                }

                // 地図の中心を現在地に
                map.setView(currentLatLng);

                // ヘディングアップ (地図を回転)
                if (heading !== null && !isNaN(heading)) {
                    rotateMap(heading);
                }

            }, err => console.error(err), {
                enableHighAccuracy: true,
                maximumAge: 0
            });
        }

        // 地図回転処理 (CSS Transform)
        function rotateMap(heading) {
            // 地図のコンテナ自体を回転させる
            // ※注: Leafletで地図を回転させると文字も逆さになりますが、APIなしでの実装限界です
            const mapContainer = document.getElementById('map');
            mapContainer.style.transform = `rotate(${-heading}deg)`;
            currentHeading = heading;
        }

        // 回転リセット (北を上にする)
        function resetMapRotation() {
            const mapContainer = document.getElementById('map');
            mapContainer.style.transform = `rotate(0deg)`;
            currentHeading = 0;
        }

    </script>
</body>
</html>
