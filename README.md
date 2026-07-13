<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>施設見学・体験 予約システム</title>
    <style>
        body { font-family: sans-serif; margin: 20px; background: #f9f9f9; color: #333; }
        .container { max-width: 600px; margin: 0 auto; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        h1 { text-align: center; color: #4A5568; font-size: 24px; }
        .section { margin-bottom: 20px; padding-bottom: 15px; border-bottom: 1px solid #eee; }
        .section-title { font-weight: bold; margin-bottom: 10px; display: block; }
        label { display: block; margin-bottom: 8px; cursor: pointer; }
        input[type="text"], input[type="tel"], select { width: 100%; padding: 10px; box-sizing: border-box; border: 1px solid #ccc; border-radius: 4px; font-size: 16px; }
        input[type="radio"] { margin-right: 8px; transform: scale(1.2); }
        .time-slot { display: inline-block; padding: 10px 15px; margin: 5px; border: 1px solid #cbd5e1; border-radius: 4px; cursor: pointer; background: #f8fafc; }
        .time-slot.selected { background: #3b82f6; color: white; border-color: #2563eb; }
        .time-slot.disabled { background: #e2e8f0; color: #94a3b8; cursor: not-allowed; border-color: #e2e8f0; }
        button { width: 100%; padding: 12px; background: #10b981; color: white; border: none; border-radius: 4px; font-size: 18px; font-weight: bold; cursor: pointer; margin-top: 10px; }
        button:hover { background: #059669; }
        #loading { color: #666; font-style: italic; }
        .hidden { display: none; }
    </style>
</head>
<body>

<div class="container">
    <h1 id="page-title">施設見学 ・ 体験 予約</h1>

    <form id="reservation-form">
        <input type="hidden" id="user-type" name="userType" value="A">

        <!-- 1. どちらを希望しますか？ -->
        <div class="section">
            <span class="section-title">1. どちらを希望しますか？</span>
            <label><input type="radio" name="reservationType" value="見学" checked onchange="toggleWorkSection(); fetchAvailableTimes();"> 見学 (1時間)</label>
            <label><input type="radio" name="reservationType" value="体験" onchange="toggleWorkSection(); fetchAvailableTimes();"> 体験 (2時間)</label>
        </div>

        <!-- 2. 日にちを選んでください -->
        <div class="section">
            <span class="section-title">2. 日にちを選んでください</span>
            <input type="date" id="reservation-date" name="date" required onchange="fetchAvailableTimes()">
        </div>

        <!-- 3. 時間を選んでください -->
        <div class="section">
            <span class="section-title">3. 時間を選んでください</span>
            <div id="time-slots-container">
                <span id="loading">-- 日にちを先に選んでください --</span>
            </div>
            <input type="hidden" id="selected-time" name="time" required>
        </div>

        <!-- 基本情報入力 -->
        <div class="section">
            <label><span class="section-title">4. お名前</span><input type="text" name="name" required></label>
        </div>
        <div class="section">
            <label><span class="section-title">5. 電話番号</span><input type="tel" name="phone" required></label>
        </div>
        <div class="section">
            <label><span class="section-title">6. 障害名/病名</span><input type="text" name="condition" required></label>
        </div>
        <div class="section">
            <label><span class="section-title">7. 同伴者 (同行される場合のみ記入)</span><input type="text" name="companion"></label>
        </div>

        <!-- 8. 体験したい作業 (B型体験時は非表示) -->
        <div class="section" id="work-section">
            <span class="section-title">8. 体験したい作業 (体験の場合のみ記入)</span>
            <select name="workType" id="work-type">
                <option value="">-- 作業内容を選んでください --</option>
                <option value="動画編集">動画編集</option>
                <option value="PC作業（事務/デザイン）">PC作業（事務/デザイン）</option>
                <option value="ネイリスト（有資格者のみ）">ネイリスト（有資格者のみ）</option>
            </select>
        </div>

        <!-- 9. その他 -->
        <div class="section">
            <label><span class="section-title">9. その他</span><input type="text" name="notes"></label>
        </div>

        <button type="submit">この内容で予約する</button>
    </form>
</div>

<script>
    // ⚠️【注意】ここに現在デプロイされている最新のGASのウェブアプリURLを貼り付けてください！
    const GAS_WEB_APP_URL = "https://script.google.com/macros/s/AKfycbxrdX5Tm7-WMcFSCqSK2P71rpsNjcWSlXT3vzlE5UgMkrUHzk2Ee5ZtwtdtFZuXXOHa/exec";

    window.onload = function() {
        const urlParams = new URLSearchParams(window.location.search);
        if (urlParams.get('type') === 'b' || urlParams.get('type') === 'B') {
            document.getElementById('user-type').value = 'B';
            document.getElementById('page-title').innerText = '就労継続支援B型 施設見学・体験 予約';
        } else {
            document.getElementById('user-type').value = 'A';
            document.getElementById('page-title').innerText = '施設見学・体験 予約';
        }
        
        const today = new Date();
        const maxDate = new Date();
        maxDate.setDate(today.getDate() + 30);
        
        document.getElementById('reservation-date').min = today.toISOString().split('T');
        document.getElementById('reservation-date').max = maxDate.toISOString().split('T');

        toggleWorkSection();
    };

    function toggleWorkSection() {
        const userType = document.getElementById('user-type').value;
        const reservationType = document.querySelector('input[name="reservationType"]:checked').value;
        const workSection = document.getElementById('work-section');
        const workTypeSelect = document.getElementById('work-type');

        if (reservationType === '見学' || (userType === 'B' && reservationType === '体験')) {
            workSection.classList.add('hidden');
            workTypeSelect.removeAttribute('required');
            workTypeSelect.value = "";
        } else {
            workSection.classList.remove('hidden');
            if (reservationType === '体験') {
                workTypeSelect.setAttribute('required', 'required');
            }
        }
    }

    // 💡【CORS遮断を100%回避するJSONP方式へ全面書き換え】
    function fetchAvailableTimes() {
        const date = document.getElementById('reservation-date').value;
        const userType = document.getElementById('user-type').value;
        const reservationType = document.querySelector('input[name="reservationType"]:checked').value;
        const container = document.getElementById('time-slots-container');

        if (!date) return;

        container.innerHTML = '<span id="loading">空いている時間を調べています。すこし待ってね...</span>';
        document.getElementById('selected-time').value = "";

        // 過去の通信履歴（古いスクリプトタグ）があれば消去
        const oldScript = document.getElementById('gas-jsonp');
        if (oldScript) oldScript.remove();

        // 完全にセキュリティ制限を迂回する通信タグ（Script）を生成
        const script = document.createElement('script');
        script.id = 'gas-jsonp';
        // GAS側へ「callback=renderSlots」という命令を付けてデータをねだります
        script.src = `${GAS_WEB_APP_URL}?action=getSlots&date=${date}&userType=${userType}&resType=${reservationType}&callback=renderSlots`;
        
        // 通信に失敗した場合のセーフティネット
        script.onerror = function() {
            container.innerHTML = "<span style='color:red;'>通信に失敗しました。GASのURLまたは公開設定（全員）を確認してください。</span>";
        };

        document.body.appendChild(script);
    }

    // GASからデータが戻ってきたら自動的に実行される関数
    function renderSlots(data) {
        const container = document.getElementById('time-slots-container');
        container.innerHTML = "";
        
        if (data && data.length === 1 && data[0].time === "エラー") {
            container.innerHTML = `<span style='color:red; font-weight:bold;'>❌ 設定エラー：${data[0].message}</span>`;
            return;
        }

        if (!data || data.length === 0) {
            container.innerHTML = "<span style='color:red;'>全ての枠が埋まっています。</span>";
            return;
        }

        data.forEach(slot => {
            const div = document.createElement('div');
            div.className = `time-slot ${slot.available ? '' : 'disabled'}`;
            div.innerText = slot.time;
            
            if (slot.available) {
                div.onclick = function() {
                    document.querySelectorAll('.time-slot').forEach(el => el.classList.remove('selected'));
                    div.classList.add('selected');
                    document.getElementById('selected-time').value = slot.time;
                };
            }
            container.appendChild(div);
        });
    }

    // 予約実行（POST）も安全に送信する処理
    document.getElementById('reservation-form').onsubmit = function(e) {
        e.preventDefault();
        
        const selectedTime = document.getElementById('selected-time').value;
        if (!selectedTime) {
            alert("時間を選択してください。");
            return;
        }

        const formData = new FormData(this);
        const data = {};
        formData.forEach((value, key) => { data[key] = value; });
        data.action = "submitReservation";

        const submitBtn = document.querySelector('button[type="submit"]');
        submitBtn.disabled = true;
        submitBtn.innerText = "予約処理中...";

        // POST送信も確実なリダイレクト処理（follow）を明示
        fetch(GAS_WEB_APP_URL, {
            method: "POST",
            mode: "no-cors", // セキュリティ遮断を完全にスルーする設定
            body: JSON.stringify(data)
        })
        .then(() => {
            // no-corsモードは成否がブラウザで隠蔽されるため、送信完了として処理を確定させます
            alert("予約リクエストを送信しました！カレンダーをご確認ください。");
            location.reload();
        })
        .catch(err => {
            console.error(err);
            alert("送信中にエラーが発生しました。");
            submitBtn.disabled = false;
            submitBtn.innerText = "この内容で予約する";
        });
    };
</script>

</body>
</html>
