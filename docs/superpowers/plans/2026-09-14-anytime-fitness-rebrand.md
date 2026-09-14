# gym-tracker v2.0（エニタイムフィットネス美園店版）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** GOLD'S GYM 向け PWA を、エニタイムフィットネス美園店のマシン構成・ブランド色・公式ロゴに合わせて作り替え、記録を新規スタートにする。

**Architecture:** 自己完結の `index.html`（CSS/JS 同居・localStorage 永続）を直接編集する。色は CSS 変数の値差し替え、ロゴはインラインSVG、メニューは `SEED.menus` の定義差し替え、新スタートは保存キー変更＋目標未設定時の初回カードで実現する。アイコンPNGは公式SVGを headless Edge で描画して生成する。

**Tech Stack:** HTML/CSS/Vanilla JS（PWA・Service Worker）、Python 3.14 + Pillow（アイコン縮小）、Microsoft Edge headless（PNG生成・画面確認）、Node（構文チェック）。

## Global Constraints

- 主色（旧 `--gold`）: `#6244bb`、暗色（旧 `--gold-dim`）: `#3f2c80`、B系（旧 `--blue`）: `#00acce`。黄 `#FFEC00` と赤 `#ff4438` はファイル内から消す。
- 保存キー `anytime_v1`、`APP_VER = "v2.0"`、sw.js `CACHE = "gymtracker-v4"`。
- 回数10回固定・全セット✔で `+inc`（脚系5／その他2.5）・未達は据え置き、A/B 1回ごと交互 のロジックは触らない。
- 公式ロゴSVGのパスデータは `scratchpad/icon_anytime_reg.svg`（取得済み）。再取得URL: `https://www.anytimefitness.co.jp/assets/img/icon_anytime_reg.svg`
- 作業ディレクトリ: `C:\cloaudecode(test)\gym-tracker`（独立した git リポジトリ）。コミットは各タスク末で行い、push は最終タスクのみ。
- Edge 実行ファイル: `C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe`
- scratchpad: `C:/Users/NAKAHA~1/AppData/Local/Temp/claude/c--cloaudecode-test-/915c1827-d3b8-4243-84f0-1b2f44007f6b/scratchpad`

### 共通の検証コマンド

構文チェック（`<script>` 内を抜き出して node に食わせる）:
```bash
cd "C:/cloaudecode(test)/gym-tracker" && python -c "
import re;h=open('index.html',encoding='utf-8').read()
open('_check.js','w',encoding='utf-8').write(re.search(r'<script>(.*)</script>',h,re.S).group(1))" && node --check _check.js && rm _check.js && echo SYNTAX_OK
```
Expected: `SYNTAX_OK`

画面スクリーンショット（`#today|#cal|#weight|#progress|#settings` でタブ指定・Task 4 でハッシュ対応を入れる）:
```bash
E="C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe"; S=<scratchpad>
"$E" --headless=new --disable-gpu --hide-scrollbars --window-size=430,1300 --virtual-time-budget=2000 \
  --screenshot="$S/shot_today.png" "file:///C:/cloaudecode(test)/gym-tracker/index.html#today"
```
その後 Read ツールで PNG を開いて目視する（[[feedback_ai_visual_verification]]：見た目は自動テストで検証できない）。

---

### Task 1: ブランド色・ヘッダーロゴ・manifest

**Files:**
- Modify: `index.html:14-19`（CSS変数）、`index.html:136-146`（header）、`index.html:615-616, 663`（チャート色の直書き）、`index.html:357`（初回コツカードの border）
- Modify: `manifest.json`

**Interfaces:**
- Produces: CSS 変数名は変えない（`--gold`/`--gold-dim`/`--blue` のまま値だけ差替え）。後続タスクは `var(--gold)` 等をそのまま使う。

- [ ] **Step 1: CSS 変数を差し替える**

`index.html` の `:root` を次に置換:
```css
:root{
  /* ANYTIME FITNESS ブランド: パープル #6244bb（公式ロゴ色）× ブラック。差し色シアン #00acce */
  --bg:#0c0c0d; --card:#17171a; --card2:#212124; --line:#2e2e32;
  --text:#f2f2f0; --sub:#9c9c96; --gold:#6244bb; --gold-dim:#3f2c80;
  --green:#4cc38a; --red:#e5534b; --blue:#00acce;
}
```
`.badge{... color:#111 ...}`、`.cal-badge.A{... color:#151515}`、`.seg button.on{... color:#151515}`、`.fab`系（line 114 `color:#151515`）、`button.primary{color:#0c0c0d}` は紫地に黒文字だと読めないので、これら5箇所の文字色を `#fff` に変える。`.wk.today{... background:#26240a}` は `background:#1f1836` に変える。

- [ ] **Step 2: チャート直書き色を変える**

- line 615 `{pts:ma, color:"#ff4438", width:2}` → `color:"#00acce"`
- line 616 `{pts:actual, color:"#FFEC00", width:2, dots:true}` → `color:"#8f75e0"`（紫は黒地で沈むので明るい紫）
- line 663 `lineChart([{pts, color:"#FFEC00", ...` → `color:"#8f75e0"`
- 体重タブ凡例 `<span style="color:var(--gold)">━ 実績</span>` → `color:#8f75e0`

- [ ] **Step 3: ヘッダーを公式ロゴのインラインSVGに差し替える**

`<header>` 内の `<img class="brand-logo" src="data:image/png;base64,...">`（1行・巨大）を削除し、次を入れる:
```html
    <svg class="brand-logo" viewBox="-14 -10 121 121" aria-label="Anytime Fitness">
      <circle cx="46.4" cy="50.5" r="60.5" fill="#6244bb"/>
      <g fill="#fff">
        <path d="M75.4,2.25C72.67-.48,68-.87,65.44,1.9c0,0-16.59,21.6-20.93,25.9A90.6,90.6,0,0,1,24.48,43,2,2,0,0,1,22,42.78l-10-9.92C8.21,29,3.73,29.31,1.48,31.56c-2,2-2.35,5.61,1.3,9.26L17.34,55.3a7.35,7.35,0,0,0,9.17,1l.36-.24c4.34-3.14,11.34-11.25,16-17.24a2.44,2.44,0,0,1,3.3-.53C59.88,47.86,65.07,51.76,67,53.33s2.66,2.5,2.64,3.31c-.05,1.61-3.37,4.21-7,6.78a106.18,106.18,0,0,0-9.21,7.17c-2.73,2.6-3.77,3.92-4.4,6a7.28,7.28,0,0,0,.08,3.9,10,10,0,0,0,4.45,5L74.53,97.69c6.21,3.36,9.33,1.33,11.12-1.37,1.67-2.51,2.43-7.22-3.32-10.71l-15-9.45c-1.64-1-1-2.09-.69-2.46.76-.82,4-3.83,4.79-4.61,5-5,7.52-9.32,7.47-13.38s-2.74-7.26-5.06-9.49-4.3-4.17-7.46-7.28c-2.52-2.47-5.69-5.59-10.11-9.89A2.74,2.74,0,0,1,55.37,27a2.38,2.38,0,0,1,.87-1.73c1.74-1.43,4-3.42,6.15-5.34s4.32-3.83,5.79-5a1.89,1.89,0,0,1,2.5.11l10,9.67c2.42,2.42,5.43,3.42,8.06,2.74a5.94,5.94,0,0,0,2.2-1.11,5.06,5.06,0,0,0,1.65-2.57c.35-1.31.48-3.95-2.59-7Z"/>
        <circle cx="33.36" cy="15.97" r="10.81"/>
      </g>
    </svg>
    <div>
      <div class="brand-name">ANYTIME FITNESS</div>
      <div class="brand-sub">MISONO ・ WORKOUT TRACKER</div>
    </div>
```
`.brand-name` の `font-style:italic` は外し、`color:#fff` にする（公式ロゴタイプは白/紫の直立体）。`.brand-logo{... border-radius:50%}` はそのまま。

- [ ] **Step 4: manifest.json を書き換える**
```json
{
  "name": "Anytime Fitness ジム記録",
  "short_name": "ジム記録",
  "description": "エニタイムフィットネス美園店 マシントレーニング＆減量トラッカー",
  "start_url": "./",
  "scope": "./",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#0c0c0d",
  "theme_color": "#0c0c0d",
  "lang": "ja",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```
`index.html` の `<title>ジム記録 - Gym Tracker</title>` は `<title>ジム記録 - Anytime Fitness</title>` に。

- [ ] **Step 5: 旧色・旧名の残存をゼロにする**

Run: `grep -n -i "FFEC00\|ff4438\|gold's\|GOLD'S\|ゴールドジム" index.html manifest.json | grep -v "^index.html:15:"`
Expected: 出力なし（15行目のコメントも「ANYTIME」に書き換えているので実際は完全に空）。

- [ ] **Step 6: 構文チェック・スクリーンショット**

共通の構文チェック → `SYNTAX_OK`。`#today` のスクリーンショットを撮って Read で目視: ヘッダーに紫丸の走る人・「ANYTIME FITNESS」白、ボタン/バッジが紫で文字が白く読めること。

- [ ] **Step 7: Commit**
```bash
git add index.html manifest.json
git commit -m "feat: エニタイムフィットネスのブランド色と公式ロゴに差し替え

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 2: アプリアイコン（icon-512 / icon-192）

**Files:**
- Create: `<scratchpad>/icon_src.html`（生成用・リポジトリ外）
- Overwrite: `icon-512.png`, `icon-192.png`

- [ ] **Step 1: 生成用HTMLを作る**（Write ツールで scratchpad に）
```html
<html><body style="margin:0;width:512px;height:512px;background:#000">
<div style="width:512px;height:512px;border-radius:52px;
  background:linear-gradient(135deg,#6e38d5 0%,#6a33d4 36%,#6127d1 74%,#581acf 100%);
  display:flex;align-items:center;justify-content:center">
  <img src="icon_anytime_reg.svg" style="width:340px;filter:brightness(0) invert(1)">
</div></body></html>
```
（iOS はホーム画面で自動的に角丸マスクをかけるので、背景黒＋自前角丸でも問題ない。Android は maskable 未指定のためそのまま表示される。）

- [ ] **Step 2: Edge で 512px に描画し、Pillow で 192px を作る**
```bash
S=<scratchpad>; E="C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe"
"$E" --headless=new --disable-gpu --hide-scrollbars --window-size=512,512 --screenshot="$S/icon-512.png" "file:///$S/icon_src.html"
python -c "
from PIL import Image; im=Image.open(r'$S/icon-512.png').convert('RGB'); print(im.size)
assert im.size==(512,512); im.save(r'C:/cloaudecode(test)/gym-tracker/icon-512.png')
im.resize((192,192),Image.LANCZOS).save(r'C:/cloaudecode(test)/gym-tracker/icon-192.png'); print('ok')"
```
Expected: `(512, 512)` `ok`

- [ ] **Step 3: 目視**

Read で `icon-512.png` を開く: 紫グラデの角丸に白の走る人が中央・欠けなし。

- [ ] **Step 4: Commit**
```bash
git add icon-512.png icon-192.png
git commit -m "feat: アプリアイコンをエニタイム公式ロゴ（紫グラデ）に差し替え

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 3: メニュー A/B を美園店のマシン構成に差し替え

**Files:**
- Modify: `index.html:175-191`（`SEED.menus`）、`index.html:412`（有酸素の説明文）、`index.html:353`（ウォームアップ文言）

**Interfaces:**
- Produces: 種目オブジェクトの形 `{name, part, sets, rest, inc, machine, note}` は不変。

- [ ] **Step 1: SEED.menus を置換**
```js
  menus: {
    A: [
      {name:"レッグプレス", part:"脚・お尻", sets:3, rest:"90秒", inc:5,
       machine:"シートに深く座り、正面の大きな踏み板に両足を肩幅で置いて脚全体で押す大型マシン。Life Fitness の Leg Press。マシンエリアで一番大きい。安全ロックのレバーを外してから始め、終わったら戻す。",
       note:"膝とつま先を同じ向き。腰を浮かせない。膝を伸ばし切ってロックしない。"},
      {name:"チェストプレス", part:"胸・肩・腕", sets:3, rest:"75〜90秒", inc:2.5,
       machine:"背もたれ付きシートに座ると胸の高さに左右のハンドル。正面にまっすぐ押し出して戻す。ベンチプレスのマシン版。ショルダープレスとはハンドルの高さ（胸の高さ＝チェスト）で見分ける。",
       note:"肩をすくめず、胸を張って押す。肩甲骨は背もたれに預ける。"},
      {name:"ラットプルダウン", part:"背中", sets:3, rest:"75〜90秒", inc:2.5,
       machine:"マルチジャングル（ケーブルが何本も下がっている大型の複合ステーション）のプルダウン側。頭上の長いバーを握り、太ももをパッドで固定して胸に向かって引き下ろす。懸垂の代わり。プルダウンは2か所あるので空いている方でOK。",
       note:"胸に向かって引く。首の後ろに引かない。肘を脇腹に近づけるイメージ。"},
      {name:"シーテッドレッグカール", part:"もも裏", sets:2, rest:"60秒", inc:2.5,
       machine:"座って脚を前に伸ばし、ふくらはぎの上に乗るパッドを膝を曲げてイスの下へ巻き込むように押し下げる。もも裏専用。レッグエクステンションと形が似ているので「Seated Leg Curl」の表記を確認。",
       note:"反動を使わず、戻しをゆっくり。腰を丸めない。"},
      {name:"ショルダープレス", part:"肩", sets:2, rest:"60〜75秒", inc:2.5,
       machine:"背もたれ付きシートに座ると肩〜耳の高さに左右のハンドル。頭の上に押し上げて戻す。チェストプレスと似ているがハンドルの位置が高い。",
       note:"腰を反らせすぎない。肘は少し前。耳の横まで下ろせば十分。"},
      {name:"ヒップアブダクション/アダクション", part:"お尻・内もも", sets:2, rest:"60秒", inc:2.5,
       machine:"座って両ももの外側（または内側）にパッドを当て、脚を開く（アブダクション＝お尻の横）・閉じる（アダクション＝内もも）マシン。1台で切替式。まず「開く」を10回、レバーを切り替えて「閉じる」を10回で1セット。",
       note:"上体を倒さず骨盤を立てる。可動域は痛みのない範囲で。"},
      {name:"アブドミナルクランチ", part:"腹筋", sets:2, rest:"45〜60秒", inc:2.5,
       machine:"座って胸の前のパッド（またはハンドル）を抱え、お腹を丸めて上体を前に倒すマシン。腹筋運動を座ったままやるイメージ。トーソローテーション（ひねる方）の隣にあることが多い。",
       note:"首で引かず、みぞおちを丸める。戻すときも力を抜かない。"}
    ],
    B: [
      {name:"レッグエクステンション", part:"もも前", sets:3, rest:"90秒", inc:2.5,
       machine:"座って足首の前にパッドが当たる位置にセットし、膝を伸ばして脚を蹴り上げる。もも前専用。シーテッドレッグカールと形が似ているので「Leg Extension」の表記を確認。",
       note:"膝をロックし切らず、上で一瞬止める。背中は背もたれに付ける。"},
      {name:"シーテッドロー", part:"背中", sets:3, rest:"75〜90秒", inc:2.5,
       machine:"座って胸を前のパッドに当て、正面のハンドルを自分のお腹に向かって引き寄せる。ボートを漕ぐ動き。Life Fitness の Seated Row。マルチジャングルのロープーリーでも代用可。",
       note:"肩甲骨を寄せる。腰を反らせて引かない。引き切ったところで1秒止める。"},
      {name:"フライ", part:"胸", sets:2, rest:"60〜75秒", inc:2.5,
       machine:"「フライ/リアデルトイド」と書かれた兼用マシン。前向きに座り、両腕を横に開いた位置から胸の前で閉じる（抱きしめる動き）。ハンドルの位置は肩の高さに合わせる。逆向きに座ると同じマシンでリアデルトになる。",
       note:"肘は軽く曲げたまま。閉じたところで胸を絞る。開くときはゆっくり。"},
      {name:"ライイングレッグカール", part:"もも裏", sets:2, rest:"60秒", inc:2.5,
       machine:"うつ伏せに寝て、かかとの上に乗ったパッドを膝を曲げてお尻に向けて巻き上げる。シーテッドレッグカールの寝る版で、もも裏の効く場所が少し変わる。「Lying Leg Curl」表記。",
       note:"腰を反らせず、お腹をパッドに付けたまま。上げ切ったら1秒止めてゆっくり戻す。"},
      {name:"ラテラルレイズ", part:"肩（横）", sets:2, rest:"60秒", inc:2.5,
       machine:"座って両ひじ（または前腕）の外側をパッドに当て、腕を横に持ち上げるマシン。肩の横（中部）専用。軽い重さでも効くので無理に増やさない。",
       note:"肩をすくめず、ひじで持ち上げる。肩の高さまでで十分。"},
      {name:"リアデルト", part:"肩後部・背中上部", sets:2, rest:"60秒", inc:2.5,
       machine:"フライと同じ「フライ/リアデルトイド」マシンに逆向き（胸をパッドに当てて）に座り、両腕を前から後ろへ大きく開く。ハンドルの位置切替レバーを「Rear Delt」側にする。",
       note:"肩をすくめず、ひじで開く。背中上部を寄せる意識。"},
      {name:"トーソローテーション", part:"体幹", sets:2, rest:"45〜60秒", inc:2.5,
       machine:"座って胸か脚をパッドで固定し、上半身を左右にゆっくりひねるマシン。左右それぞれ10回で1セット。脇腹（腹斜筋）に効く。アブドミナルクランチの隣にあることが多い。",
       note:"勢いを使わず、可動域は無理しない。息を吐きながらひねる。"}
    ]
  },
```

- [ ] **Step 2: 有酸素・ウォームアップの文言**

- line 412: `"傾斜ウォーキング（傾斜5〜10%、会話できる強度）"` → `"トレッドミル傾斜ウォーキング（傾斜5〜10%・会話できる強度）※10台あるので空きに困らない"`、`"バイク or クロストレーナー（息が上がりすぎない強度）"` → `"クロストレーナー or リカンベントバイク（息が上がりすぎない強度）"`
- line 353: `ウォームアップ: トレッドミル or バイク 5〜8分（軽く汗ばむ程度）` → `ウォームアップ: トレッドミル or アップライトバイク 5〜8分（軽く汗ばむ程度）`

- [ ] **Step 3: 構文チェック・種目数の確認**

共通の構文チェック → `SYNTAX_OK`。
Run: `grep -c '{name:"' index.html` → Expected: `14`

- [ ] **Step 4: Commit**
```bash
git add index.html
git commit -m "feat: メニューA/Bを美園店のマシン構成（各7種目）に再構築

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 4: 新スタート（保存キー・目標未設定の初回カード・ガード・ハッシュでタブ指定）

**Files:**
- Modify: `index.html`（`APP_VER`/`KEY`、`SEED.settings`、`load()`、`renderToday()`、`paceTargetAt()`/`renderWeight()`/`weightChart()`、タブ初期化）
- Modify: `sw.js:1`

**Interfaces:**
- Produces: `goalIsSet()` → boolean、`saveGoalFirst()`（初回カードの保存）。`db.settings.startWeight/goalWeight/goalDate` は未設定時 `null`。

- [ ] **Step 1: 定数と SEED.settings**
```js
const APP_VER = "v2.0";
const KEY = "anytime_v1";   // v1.x は gymtracker_v1（GOLD'S GYM 時代）。移行せず新規スタート

const SEED = {
  settings: { startDate:null, startWeight:null, goalWeight:null, goalDate:null },
```

- [ ] **Step 2: load() で startDate の既定を今日にする**

`load()` の末尾 `return JSON.parse(JSON.stringify(SEED));` と、既存データ返却 `return d;` の直前にそれぞれ次を挟む（関数を1つ足して両方から呼ぶ）:
```js
function withDefaults(d){
  if(!d.settings) d.settings = {};
  if(!d.settings.startDate) d.settings.startDate = todayStr();
  return d;
}
```
`return d;` → `return withDefaults(d);`、`return JSON.parse(JSON.stringify(SEED));` → `return withDefaults(JSON.parse(JSON.stringify(SEED)));`。
`todayStr` は関数宣言なので巻き上げられ、`load()` から呼べる。

- [ ] **Step 3: goalIsSet() と初回カード**

`/* ================= 提案ロジック ================= */` の直前に追加:
```js
function goalIsSet(){
  const st = db.settings;
  return st.startWeight!=null && st.goalWeight!=null && !!st.goalDate;
}
function saveGoalFirst(){
  const st = db.settings;
  const sw = parseFloat(document.getElementById("g-start-w").value);
  const gw = parseFloat(document.getElementById("g-goal-w").value);
  const sd = document.getElementById("g-start").value;
  const gd = document.getElementById("g-goal-date").value;
  if(isNaN(sw) || isNaN(gw)){ toast("体重を数字で入れてください"); return; }
  if(!gd){ toast("目標日を選んでください"); return; }
  if(gd <= (sd||st.startDate)){ toast("目標日は開始日より後にしてください"); return; }
  st.startWeight = sw; st.goalWeight = gw; st.goalDate = gd;
  if(sd) st.startDate = sd;
  if(!db.weights.some(w=>w.date===st.startDate)){
    db.weights.push({date:st.startDate, kg:sw});
    db.weights.sort((a,b)=>a.date<b.date?-1:1);
  }
  save(); renderAll(); toast("目標を設定しました");
}
```
（開始体重は「開始日の体重記録」としても1件入れる。これで体重タブの実績線が初日から出る。）

`renderToday()` の `let html = \`` 直前に:
```js
  const goalCard = goalIsSet() ? "" : `
  <div class="card" style="border-color:var(--gold)">
    <h3>🎯 はじめに目標を設定</h3>
    <div class="muted small" style="margin-bottom:8px">エニタイムでの新スタートです。現在の体重と目標を入れてください（あとで設定タブから変更できます）。</div>
    <div class="cardio-grid" style="margin-bottom:8px">
      <div><label class="field-label">開始日</label><input type="date" id="g-start" value="${db.settings.startDate}"></div>
      <div><label class="field-label">目標日</label><input type="date" id="g-goal-date" value=""></div>
      <div><label class="field-label">現在の体重(kg)</label><input type="text" inputmode="decimal" id="g-start-w" placeholder="例: 96.5"></div>
      <div><label class="field-label">目標体重(kg)</label><input type="text" inputmode="decimal" id="g-goal-w" placeholder="例: 89"></div>
    </div>
    <button class="primary" onclick="saveGoalFirst()">目標を保存する</button>
  </div>`;
```
そして `let html = \`` の直後（`<div class="seg">` の前）に `${goalCard}` を入れる。

- [ ] **Step 4: 体重タブ・設定タブのガード**

`paceTargetAt`:
```js
function paceTargetAt(dateStr){
  const st = db.settings;
  if(!goalIsSet()) return null;
  const total = dayDiff(st.startDate, st.goalDate);
  if(total <= 0) return st.goalWeight;
  const cur = Math.min(Math.max(dayDiff(st.startDate, dateStr),0), total);
  return st.startWeight + (st.goalWeight - st.startWeight) * cur / total;
}
```
`renderWeight()`:
- `const target = paceTargetAt(t);` の直後の stat 表示 `${target.toFixed(1)}` → `${target!=null?target.toFixed(1):"-"}`
- `if(latest){ const d = latest.kg - paceTargetAt(latest.date); ...}` → `if(latest && goalIsSet()){ ... }`
- 凡例 `┅ 目標ペース(${st.startWeight}→${st.goalWeight}kg)` → `┅ 目標ペース(${goalIsSet()?st.startWeight+"→"+st.goalWeight+"kg":"未設定"})`
- 履歴テーブルの `const d = w.kg - paceTargetAt(w.date);` → `const pt = paceTargetAt(w.date); const d = pt==null ? null : w.kg - pt;` とし、セル出力を `${d==null?"-":(d>0?"+":"")+d.toFixed(1)+"kg"}`、色は `d==null||d<=0 ? "var(--green)" : "var(--red)"` に。
- `weightChart()` 内で目標ペース線を作っている箇所（`paceTargetAt` を呼んで点列を作る部分）を `if(goalIsSet()){ ... }` で囲い、未設定なら実績と7日平均のみ描く。

`renderSettings()` の目標欄 `value="${st.startDate}"` 等は `value="${st.startDate||""}"`、`value="${st.goalDate||""}"`、`value="${st.startWeight??""}"`、`value="${st.goalWeight??""}"` に（null が文字列 "null" で出ないように）。

- [ ] **Step 5: URLハッシュでタブを開けるようにする（画面確認用・実害なし）**

タブ初期化の `renderToday();`（line 796 付近）を次に置換:
```js
{
  const tab = (location.hash||"").replace("#","");
  const btn = renders[tab] ? document.querySelector(`nav button[data-tab="${tab}"]`) : null;
  if(btn) btn.click(); else renderToday();
}
```

- [ ] **Step 6: sw.js のキャッシュ名**

`const CACHE = "gymtracker-v3";` → `const CACHE = "gymtracker-v4";`

- [ ] **Step 7: 構文チェック**

共通の構文チェック → `SYNTAX_OK`。
Run: `grep -n "gymtracker_v1\|v1.3\|gymtracker-v3" index.html sw.js` → Expected: `index.html` のコメント行1件のみ（KEY の注記）。

- [ ] **Step 8: Commit**
```bash
git add index.html sw.js
git commit -m "feat: 新規スタート（保存キー変更・初回の目標設定カード・目標未設定ガード・#tab対応）

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 5: 画面検証・手動シナリオ・公開

**Files:**
- Create: `<scratchpad>/harness.html`（localStorage に検証用データを入れて index.html へ遷移。リポジトリ外）

- [ ] **Step 1: 初期状態の4画面を撮る**

`#today` `#weight` `#settings` `#cal` を共通コマンドで撮影し Read で目視。確認点:
- today: 初回目標カードが先頭に出る／その下にメニューAの7種目／有酸素欄
- weight: 「今日の目標ペース -」「目標ペース(未設定)」で崩れなし
- settings: 目標欄が空（"null" と出ていない）、フッター `Gym Tracker v2.0`

- [ ] **Step 2: 目標設定後＋記録ありの状態を撮る**

harness.html（file:// 同一オリジンで localStorage を共有する）:
```html
<script>
localStorage.setItem("anytime_v1", JSON.stringify({
  settings:{startDate:"2026-09-14",startWeight:96.5,goalWeight:89,goalDate:"2027-01-31"},
  menus:null, sessions:[
    {date:"2026-09-11",menu:"A",exercises:[{name:"レッグプレス",kg:60,cleared:true},{name:"チェストプレス",kg:30,cleared:false}],cardioMin:20}
  ],
  weights:[{date:"2026-09-11",kg:97.2},{date:"2026-09-14",kg:96.5}]
}));
location.replace("file:///C:/cloaudecode(test)/gym-tracker/index.html#today");
</script>
```
※ `menus:null` だと `load()` の `if(d && d.menus)` で SEED に落ちて記録も消える。**`menus` は SEED と同じ内容が要る**ので、harness では `menus` を省かず、`index.html` から `SEED.menus` をコピーして貼る（Task 3 の Step 1 と同じオブジェクト）。

`--virtual-time-budget=3000` 付きで `harness.html` を撮影 → today に「前回 9/11 がメニューAだったので、今日はBの番」、レッグプレスの提案は B には無いので出ない・チェストプレスも B に無い → `#progress` で「レッグプレス 60kg ✔」が履歴に出る、`#weight` で目標ペース線と実績2点が出る、を目視。

- [ ] **Step 3: 旧データが混ざらないことの確認**

harness の `setItem` を `gymtracker_v1`（旧キー）に変えたバージョンで撮影 → today は初回目標カード付きの空状態（旧記録が出ない）。

- [ ] **Step 4: 検証で見つけた崩れを直して再撮影**

見つかった分だけ index.html を修正 → 構文チェック → 同じ画面を再撮影して確認。修正があれば
```bash
git add index.html && git commit -m "fix: 画面検証での崩れ修正

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

- [ ] **Step 5: push して公開**
```bash
cd "C:/cloaudecode(test)/gym-tracker" && git push origin main && git log --oneline -6
```
Expected: push 成功。数分後 https://nextstage2008-pixel.github.io/gym-tracker/ に反映。

- [ ] **Step 6: ユーザーへの案内**（メッセージで伝える・ファイル変更なし）
- iPhone: ホーム画面の旧アイコンを削除 → Safari で URL を開く → 「ホーム画面に追加」（アイコンは追加時に取り込まれるため削除→再追加が必要）
- 開いたら「🎯 はじめに目標を設定」に現在体重・目標体重・目標日を入れて保存
- 旧記録は端末内 `gymtracker_v1` に残っているが画面には出ない

- [ ] **Step 7: メモリ更新**

`C:\Users\nakahashi-pc\.claude\projects\c--cloaudecode-test-\memory\project_gym_tracker.md` を「エニタイムフィットネス美園店版 v2.0（2026-09-14 移籍）・紫 #6244bb・公式ロゴSVG・保存キー anytime_v1・A/B各7種目」に書き換え、`MEMORY.md` の該当行も更新する。
