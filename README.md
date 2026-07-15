<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>施設見学・体験 予約カレンダー</title>
<style>
body{font-family:'Helvetica Neue',Arial,sans-serif;background-color:#f7f9fa;color:#333;margin:0;padding:20px;}
.container{max-width:600px;margin:0 auto;background:white;padding:30px;border-radius:12px;box-shadow:0 4px 12px rgba(0,0,0,0.05);}
/* ★見出しの文字サイズをスマホ向けに最適化しました */
h1{text-align:center;font-size:20px;color:#2c3e50;margin-bottom:25px;line-height:1.4;}
@media (min-width: 480px) { h1{font-size:24px;} } /* パソコン等では元の大きさに */
label{font-weight:bold;display:block;margin-top:20px;margin-bottom:8px;font-size:16px;}
.type-selector{display:flex;gap:15px;margin-bottom:20px;}
.type-btn{flex:1;padding:15px;font-size:18px;font-weight:bold;border:2px solid #cbd5e1;border-radius:8px;background:white;cursor:pointer;transition:all 0.2s;}
.type-btn.selected{background:#3498db;color:white;border-color:#3498db;}
input[type="text"],input[type="tel"],input[type="date"],select,textarea{width:100%;padding:12px;font-size:16px;border:2px solid #cbd5e1;border-radius:8px;box-sizing:border-box;font-family:inherit;}
input[type="date"].has-value{background-color:#e8f4fd;border-color:#3498db;}
select.has-value{background-color:#e8f4fd;border-color:#3498db;}
textarea{resize:vertical;min-height:80px;}
option:disabled{color:#94a3b8;background-color:#f1f5f9;}
.submit-btn{width:100%;background:#2ecc71;color:white;border:none;padding:15px;font-size:20px;font-weight:bold;border-radius:8px;cursor:pointer;margin-top:30px;}
.submit-btn:disabled{background:#bdc3c7;cursor:not-allowed;}
.note{font-size:14px;color:#7f8c8d;margin-top:5px;}
#loadingText{color:#e67e22;font-weight:bold;display:none;margin-top:5px;}
#errorLog{background-color:#fde8e8;color:#e74c3c;padding:10px;border-radius:6px;margin-top:10px;font-size:14px;display:none;border:1px solid #f5c6cb;}
.trial-only-fields{display:none;}
</style>
</head>
<body>
<div class="title-group">
<h1>就労継続支援A型事業所sabot</h1>
   <p class="subtitle">施設見学・予約</p>
</div>
<label>1. どちらを希望しますか？</label>
<div class="type-selector">
<button type="button" class="type-btn selected" id="btn-visit" onclick="selectType('見学',1)">見学 (1時間)</button>
<button type="button" class="type-btn" id="btn-trial" onclick="selectType('体験',2)">体験 (2時間)</button>
</div>
<form id="reserveForm">
   <input type="hidden" name="shopType" value="A"> 
<input type="hidden" id="reserveType" name="reserveType" value="見学">
<input type="hidden" id="duration" name="duration" value="1">
<label for="reserveDate">2. 日にちを選んでください</label>
<input type="date" id="reserveDate" name="reserveDate" required>
<p class="note">※明日から30日先まで選べます</p>
<p id="loadingText">空いている時間を調べています。もう少々お待ちください...</p>
<div id="errorLog"></div>
<label for="reserveTime">3. 時間を選んでください</label>
<select id="reserveTime" name="reserveTime" required disabled onchange="handleTimeChange(this)">
<option value="">-- 日にちを先に選んでください --</option>
</select>
<label for="userName">4. お名前</label>
<input type="text" id="userName" name="userName" placeholder="お名前をご記入ください" required>
<label for="userPhone">5. 電話番号</label>
<input type="tel" id="userPhone" name="userPhone" placeholder="（例）09012345678" required>
<label for="illnessName">6. 障害名/病名</label>
<input type="text" id="illnessName" name="illnessName" placeholder="ご記入ください">
<label for="companionName">7. 同伴者 (同行される場合のみ記入)</label>
<input type="text" id="companionName" name="companionName" placeholder="お名前をご記入ください">
<div id="trialFields" class="trial-only-fields">
<label for="desiredTask">8. 体験したい作業 (体験の場合のみ記入)</label>
<select id="desiredTask" name="desiredTask" onchange="handleTimeChange(this)">
<option value="">-- 作業内容を選んでください --</option>
<option value="動画編集">動画編集</option>
<option value="PC作業（事務/デザイン）">PC作業（事務/デザイン）</option>
<option value="ネイリスト（有資格者のみ）">ネイリスト（有資格者のみ）</option>
<option value="清掃">清掃</option> 
</select>
</div>
<label for="otherComment">9. その他</label>
<textarea id="otherComment" name="otherComment" placeholder="質問があれば記載してください"></textarea>
<button type="submit" class="submit-btn" id="submitBtn">この内容で予約する</button>
</form>
</div>
<script>
const GAS_URL="https://script.google.com/macros/s/AKfycbzW6klhzwQ-3jb4g7HeUfiSc5nbC6OsmSeQ39tIa2l7ehNQ_H6oWa5IiOJ2H1Q9LMQ/exec";
const dateInput=document.getElementById('reserveDate');
const today=new Date();

// 最小値（予約できる最初の日）を「明日」に設定
const minDate=new Date();
minDate.setDate(today.getDate() + 1);

// 最大値（予約できる最後の日）を「31日後」に設定
const maxDate=new Date();
maxDate.setDate(today.getDate() + 31);

// カレンダーの選択可能範囲を設定
dateInput.min=`${minDate.getFullYear()}-${String(minDate.getMonth()+1).padStart(2,'0')}-${String(minDate.getDate()).padStart(2,'0')}`;
dateInput.max=`${maxDate.getFullYear()}-${String(maxDate.getMonth()+1).padStart(2,'0')}-${String(maxDate.getDate()).padStart(2,'0')}`;

function selectType(type,hours){
document.getElementById('reserveType').value=type;
document.getElementById('duration').value=hours;
document.getElementById('btn-visit').classList.toggle('selected',type==='見学');
document.getElementById('btn-trial').classList.toggle('selected',type==='体験');
const tf=document.getElementById('trialFields');
if(type==='体験'){tf.style.display='block';}else{tf.style.display='none';const s=document.getElementById('desiredTask');s.value='';s.classList.remove('has-value');}
if(dateInput.value)fetchAvailableSlots(dateInput.value);
}
// ★ここから追加
dateInput.value = dateInput.min;
dateInput.classList.add('has-value');
fetchAvailableSlots(dateInput.value);
// ★ここまで追加


function handleTimeChange(select){if(select.value){select.classList.add('has-value');}else{select.classList.remove('has-value');}}
function fetchAvailableSlots(dateStr){
const ts=document.getElementById('reserveTime');
const lt=document.getElementById('loadingText');
const el=document.getElementById('errorLog');
const type=document.getElementById('reserveType').value;
lt.style.display="block";el.style.display="none";ts.disabled=true;ts.innerHTML='<option value="">調べる中...</option>';
fetch(`${GAS_URL}?action=check&date=${dateStr}&type=${encodeURIComponent(type)}`,{redirect:"follow"})
.then(res=>{if(!res.ok)throw new Error("サーバー通信エラー");return res.json();})
.then(slots=>{
if(slots.status==="error")throw new Error(slots.message);
ts.innerHTML='<option value="">-- 時間枠を選んでください --</option>';
slots.forEach(slot=>{
const op=document.createElement('option');op.value=slot.start;
if(slot.available){op.textContent=`${slot.start} ～ ${slot.end}`;}else{op.textContent=`× ${slot.start} ～ ${slot.end} (予定あり)`;op.disabled=true;}
ts.appendChild(op);
});
ts.disabled=false;
})
.catch(err=>{ts.innerHTML='<option value="">エラー発生</option>';el.innerText="【エラー】: "+err.message;el.style.display="block";alert("取得失敗:\n"+err.message);})
.finally(()=>{lt.style.display="none";});
}
// iPhoneの発火バグと時差・範囲無視を完璧に防ぐチェック処理
let isChanging = false;
dateInput.addEventListener('change', () => { isChanging = true; });

dateInput.addEventListener('blur', function() {
    if (!isChanging) return; 
    isChanging = false;

    if (!this.value) {
        this.classList.remove('has-value');
        return;
    }

    const now = new Date();
    const tYear = now.getFullYear();
    const tMonth = now.getMonth();
    const tDate = now.getDate();

    const minCheck = new Date(tYear, tMonth, tDate + 1).getTime();
    const maxCheck = new Date(tYear, tMonth, tDate + 30).getTime();

    const parts = this.value.split('-');
    const selectedCheck = new Date(parseInt(parts[0], 10), parseInt(parts[1], 10) - 1, parseInt(parts[2], 10)).getTime();
    // ★【追加】土日（曜日）のチェックを追加
    const selectedDay = new Date(parseInt(parts, 10), parseInt(parts, 10) - 1, parseInt(parts, 10)).getDay(); // 0:日, 6:土
    if (selectedDay === 0 || selectedDay === 6) {
        alert("土曜日・日曜日は定休日です。\n平日の日付を選択してください。");
        this.value = "";
        this.classList.remove('has-value');
        
        const ts = document.getElementById('reserveTime');
        ts.innerHTML = '<option value="">-- 日にちを先に選んでください --</option>';
        ts.disabled = true;
        return;
    }

    // 範囲外（明日より前、または31日後より先）を完全に弾く
    if (selectedCheck < minCheck || selectedCheck > maxCheck) {
        alert("予約は【明日から31日先まで】の間で選択してください。");
        this.value = ""; 
        this.classList.remove('has-value');
        
        const ts = document.getElementById('reserveTime');
        ts.innerHTML = '<option value="">-- 日にちを先に選んでください --</option>';
        ts.disabled = true;
        return;
    }

    this.classList.add('has-value');
    fetchAvailableSlots(this.value);
});

document.getElementById('reserveForm').addEventListener('submit',function(e){
e.preventDefault();
const btn=document.getElementById('submitBtn');
const el=document.getElementById('errorLog');
btn.disabled=true;btn.textContent="送信中...";el.style.display="none";
fetch(GAS_URL,{method:"POST",body:new URLSearchParams(Object.fromEntries(new FormData(this).entries())),redirect:"follow"})
.then(res=>{if(!res.ok)throw new Error("送信失敗");return res.json();})
.then(result=>{
if(result.status==="success"){
alert("予約が完了しました！");this.reset();dateInput.classList.remove('has-value');
const ts=document.getElementById('reserveTime');ts.classList.remove('has-value');
ts.innerHTML='<option value="">-- 日にちを先に選んでください --</option>';ts.disabled=true;selectType('見学',1);
}else{throw new Error(result.message||"予約処理エラー");}
})
.catch(err=>{el.innerText="【送信エラー】: "+err.message;el.style.display="block";alert("予約失敗:\n"+err.message);})
.finally(()=>{btn.disabled=false;btn.textContent="この内容で予約する";});
});
</script>
</body>
</html>
