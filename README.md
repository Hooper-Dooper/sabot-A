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
    <!-- A型・B型のタイトルをJavaScriptで自動切替 -->
    <h1 id="page-title">施設見学 ・ 体験 予約</h1>

    <form id="reservation-form">
        <!-- ユーザータイプを隠しパラメータで保持 -->
        <input type="hidden" id="user-type" name="userType" value="A">

        <!-- 1. どちらを希望しますか？ -->
        <div class="section">
            <span class="section-title">1. どちらを希望しますか？</span>
            <label><input type="radio" name="reservationType" value="見学" checked onchange="toggleWorkSection()"> 見学 (1時間)</label>
            <label><input type="radio" name="reservationType" value="体験" onchange="toggleWorkSection()"> 体験 (2時間)</label>
        </div>

        <!-- 2. 日にちを選んでください -->
        <div class="section">
            <span class="section-title">2. 日にちを選んでください</span>
            <p style="font-size: 12px; color: #666; margin-top: -5px;">※今日から30日先まで選べます</p>
            <input type="date" id="reservation-date" name="date" required onchange="fetchAvailableTimes()">
        </div>

        <!-- 3. 時間を選んでください -->
        <div class="section">
            <span class="section-title">3. 時間を選んでください</span>
            <div id="time-slots-container">
                <span id="loading">-- 日にちを先に選んでください --</span>
            </div>
            <!-- 選択された時間を格納する隠しフィールド -->
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
    // ⚠️ここに【GASでデプロイしたウェブアプリのURL】を貼り付けてください
    const GAS_WEB_APP_URL = "https://script.google.com/macros/s/AKfycbxXllSlIy6M2Kb3r71nFrl5geq6Dh9_JQidE0CmsrHhawYtiAjPo79iBLYmZONzQrGu/exec";

    // ページ読み込み時の初期設定 (A型・B型の判定)
    window.onload = function() {
        const urlParams = new URLSearchParams(window.location.search);
        // URLが ?type=b ならB型ページにする
        if (urlParams.get('type') === 'b' || urlParams.get('type') === 'B') {
            document.getElementById('user-type').value = 'B';
            document.getElementById('page-title').innerText = '就労継続支援B型 施設見学・体験 予約';
        } else {
            document.getElementById('user-type').value = 'A';
            document.getElementById('page-title').innerText = '就労継続支援A型 施設見学・体験 予約';
        }
        
        // 日付の選択範囲を今日から30日先までに制限
        const today = new Date();
        const maxDate = new Date();
        maxDate.setDate(today.getDate() + 30);
        
        document.getElementById('reservation-date').min = today.toISOString().split('T')[0];
        document.getElementById('reservation-date').max = maxDate.toISOString().split('T')[0];

        toggleWorkSection();
    };

    // B型の体験選択時、または「見学」選択時に作業選択欄を非表示にする制御
    function toggleWorkSection() {
        const userType = document.getElementById('user-type').value;
        const reservationType = document.querySelector('input[name="reservationType"]:checked').value;
        const workSection = document.getElementById('work-section');
        const workTypeSelect = document.getElementById('work-type');

        if (reservationType === '見学' || (userType === 'B' && reservationType === '体験')) {
            workSection.classList.add('hidden');
            workTypeSelect.removeAttribute('required');
            workTypeSelect.value = ""; // 値をクリア
        } else {
            workSection.classList.remove('hidden');
            if (reservationType === '体験') {
                workTypeSelect.setAttribute('required', 'required');
            }
        }
    }

    // 空き時間をGASに問い合わせる処理
    function fetchAvailableTimes() {
        const date = document.getElementById('reservation-date').value;
        const userType = document.getElementById('user-type').value;
        const reservationType = document.querySelector('input[name="reservationType"]:checked').value;
        const container = document.getElementById('time-slots-container');

        if (!date) return;

        container.innerHTML = '<span id="loading">空いている時間を調べています。すこし待ってね...</span>';
        document.getElementById('selected-time').value = ""; // 選択をリセット

        // GASへ送るリクエストURLを作成
        const url = `${GAS_WEB_APP_URL}?action=getSlots&date=${date}&userType=${userType}&resType=${reservationType}`;

        fetch(url)
            .then(response => response.json())
            .then(data => {
                container.innerHTML = "";
                if (data.length === 0) {
                    container.innerHTML = "<span style='color:red;'>申し訳ありません、この日は全ての枠が埋まっています。</span>";
                    return;
                }

                // 取得した時間枠をボタンとして生成
                data.forEach(slot => {
                    const div = document.createElement('div');
                    div.className = `time-slot ${slot.available ? '' : 'disabled'}`;
                    div.innerText = slot.time;
                    
                    if (slot.available) {
                        div.onclick = function() {
                            // 他の選択を解除してこれを選択状態にする
                            document.querySelectorAll('.time-slot').forEach(el => el.classList.remove('selected'));
                            div.classList.add('selected');
                            document.getElementById('selected-time').value = slot.time;
                        };
                    }
                    container.appendChild(div);
                });
            })
            .catch(err => {
                console.error(err);
                container.innerHTML = "<span style='color:red;'>時間の取得に失敗しました。もう一度お試しください。</span>";
            });
    }

    // 予約フォーム送信時の処理
    document.getElementById('reservation-form').onsubmit = function(e) {
        e.preventDefault();
        
        const selectedTime = document.getElementById('selected-time').value;
        if (!selectedTime) {
            alert("時間を選択してください。");
            return;
        }

        const formData = new FormData(this);
        // フォームデータをオブジェクトに変換
        const data = {};
        formData.forEach((value, key) => { data[key] = value; });
        data.action = "submitReservation"; // 処理の識別子

        // 送信ボタンを無効化
        const submitBtn = document.querySelector('button[type="submit"]');
        submitBtn.disabled = true;
        submitBtn.innerText = "予約処理中...";

        // GASにPOST送信
        fetch(GAS_WEB_APP_URL, {
            method: "POST",
            body: JSON.stringify(data)
        })
        .then(response => response.json())
        .then(res => {
            if (res.status === "success") {
                alert("予約が完了しました！カレンダーをご確認ください。");
                location.reload(); // ページをリフレッシュ
            } else {
                alert("エラー: " + res.message);
                submitBtn.disabled = false;
                submitBtn.innerText = "この内容で予約する";
            }
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
# sabot-agata
