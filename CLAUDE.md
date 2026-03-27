# Mario Game - プロジェクト仕様書

## 概要
ブラウザで動くSuper Mario Brosクローン。HTML/Canvas/JavaScript（1ファイル構成）。

## 技術スタック
- **描画**: HTML5 Canvas (800x450px)
- **音声**: Web Audio API（外部ライブラリなし）
- **フォント**: Press Start 2P (Google Fonts)
- **構成**: index.html 1ファイルに全コード
- **定数**: `W=800, H=450, TILE=32, GRAVITY=0.52, LW=8000`

---

## 現在の実装状況

### ステージ
| ワールド | ステージ | 関数 | 状態 |
|---|---|---|---|
| 1 | 1-1 | `buildLevel()` | ✅ 簡単・青空・入門ステージ |
| 1 | 1-2 | `buildLevel2()` | ✅ 中級・夕方背景 |
| 1 | 1-3 | `buildLevel3()` | ✅ 上級・夜背景 |
| 1 | 1-4 | `buildLevel4()` | ✅ 城テーマ・クッパボス戦（HP3・3連射ファイヤー・城門ウォール・専用BGM） |
| 地下 | - | `buildUnderground()` | ✅ ワープパイプで移行 |

### ステージ遷移（goalSlide完了ハンドラ）
```javascript
if(currentLevel===1){currentLevel=2;buildLevel2();...}
else if(currentLevel===2){currentLevel=3;buildLevel3();...}
else if(currentLevel===3){currentLevel=4;buildLevel4();...}
else{state='win';...}  // ← 2-1追加時はここを修正
```

### ステージ選択テスト画面（一時的）
- スタート画面に 1-1〜1-4 の選択ボタンがある
- ゲーム完成後に削除予定

---

## プレイヤー仕様

### マリオの状態
- `mario.big`: true=大マリオ / false=小マリオ
- `mario.power`: `'none'` / `'fire'`
- `mario.inv`: 無敵フレーム数（ダメージ後・アイテム取得時の短い保護）
- `starTimer`: スター無敵の残りフレーム（480フレーム=8秒）
  - **`mario.inv` と `starTimer` は別物**。スター取得時は `mario.inv=60`（短い保護）のみ
  - 敵との衝突判定: `if(mario.inv===0&&starTimer===0)` でダメージ、`if(starTimer>0)` で敵を倒す

### コイン
- 100枚集めるごとに自動で1UP（`updateHUD()` 内で処理）

### ヨシ
| プロパティ | 説明 |
|---|---|
| `yoshi.alive` | ヨシが存在するか |
| `yoshi.mounted` | マリオが乗っているか |
| `yoshi.eatCount` | 食べた敵の数（5体でスター発動） |
| `yoshi.runAway` | 逃走中か |
| `yoshi.runTimer` | 逃走残りフレーム（180f） |
| `yoshi.tongueMaxLen` | 舌の最大長（90px） |

- 舌は前方向に伸び、当たり判定は縦方向に広め（上方90px）→飛ぶ敵も食べられる
- `yoshi.runAway=true` の間は逃げる。runTimer終了後は停止（`runAway=false`, `vx=vy=0`）し `idleTimer=480`（8秒）をセット
- idle 中にマリオが触れれば再乗車（`idleTimer=0`でキャンセル）
- idleTimer が 0 になったら `alive=false` で消滅
- 画面外に落ちた場合も `alive=false`

### ヨシの idleTimer
```javascript
// runAway 終了時
yoshi.runAway=false; yoshi.vx=0; yoshi.vy=0; yoshi.idleTimer=480;
// idle 更新ループ
if(yoshi.idleTimer>0){yoshi.idleTimer--;if(yoshi.idleTimer<=0)yoshi.alive=false;}
// 乗車時にキャンセル
mountYoshi(); yoshi.idleTimer=0;
```

---

## 敵キャラ仕様

| 種類 | type名 | 備考 |
|---|---|---|
| クリボー | `goomba` | 踏みつけで倒す |
| ノコノコ | `koopa` | 踏みつけで甲羅に、甲羅を蹴る |
| 飛ぶノコノコ | `parakoopa` | 踏みつけ→koopa甲羅に変化。x>=2000のみ配置 |
| メット | `buzzy` | ファイアボール無効、踏みつけ→甲羅 |
| ハンマーブロス | `hammerBro` | 踏みつけで倒せる（h=41px、判定 e.h*0.4） |
| パックンフラワー | `piranha` | パイプから出現。ワープパイプには出現しない |
| キラー | `bulletBill` | 砲台から発射 |

### 踏みつけ判定
```javascript
const mBot = mario.y + mario.h;
if(mBot - mario.vy <= e.y + e.h*0.4)  // 敵の上40%に着地で踏みつけ
```
- `e.y+8` の固定値は**使わない**（ハンマーブロス・parakoopa等で判定が狭すぎる）

### parakoopa の制約
- **x < 2000 には配置しない**（ゲーム開始直後に画面外から飛んでくるため）
- baseY は `H-5*TILE`（地面から5タイル上）、±22px で上下ボブ

---

## ギミック仕様

### 溶岩炎（lavaFlames）
```javascript
// 炎オブジェクト構造
{x, y, w, maxH, curH:0, phase, period}
// phase: 開始フレームオフセット（炎ごとにずらす）
// period: 1サイクルのフレーム数
```
- **必ず `lavaFlames.length=0` をすべての buildLevelX() に書く**
  - 1-4以外のステージに炎が残るバグの原因（過去に発生）
- 描画はブロックより先（地面タイルの下に見えるように）

### 移動足場（movingPlats）
```javascript
{x, y, w:TILE*2, h:12, type:'h', ox:x, range:80, spd:2.0}  // 水平
{x, y, w:TILE*2, h:12, type:'v', oy:y, range:80, spd:1.5}  // 垂直
{x, y, w:TILE*1.5, h:12, type:'fall', fallTimer:0, falling:false, oy:y, vy:0}  // 落下型
```

### ワープパイプ・地下バリアント
```javascript
// パイプ宣言時に variant を設定
[[x, h, 'coin'], [x2, h2, false], ...].forEach(([px,ph,warp])=>{
  pipes.push({x:px, y:H-TILE-ph*TILE, w:TILE*2, h:ph*TILE, bounceOffset:0,
              isWarp:!!warp, variant:warp||null})});
// ワープパイプには piranha 出さない（isWarp チェックで skip）
```

**地下バリアント一覧（buildUnderground(variant)）:**
| variant | テーマ | 内容 |
|---|---|---|
| `'coin'` | コインの楽園（1-1） | コイン57枚＋コインブロック×3＋きのこ＋かくしスター |
| `'goomba'` | クリボーまつり（1-2前半） | クリボー10体＋コイン22枚＋かくし1UP |
| `'mushroom'` | パワーアップの部屋（1-2後半） | きのこブロック×3＋ノコノコ×3＋かくしスター |
| `'danger1up'` | 1UPチャレンジ（1-3前半） | 1UPはてな×3＋火柱6本＋クリボー6体 |
| `'star'` | スターフェスティバル（1-3後半） | スターブロック＋敵11体＋コイン26枚＋かくし1UP |

- **lavaFlames は enterUnderground/exitUnderground で自動クリア**（danger1up 用）
- 新バリアント追加時は `buildUnderground()` の switch/else に追加し、パイプ variant 文字列を設定するだけ
- **パイプを stairD の上に重ねない**（stairのブロックがパイプ内に入り進入不可になる）
  - 例: stairD(3980,5) はx=3980〜4108を使うので、パイプは x≥4150 以降に配置

---

## クッパボス（1-4専用）

### グローバル変数
```javascript
let bowser = {alive:false, x, y, w:64, h:72, hp:3, maxHp:3,
  vx:-1.5, vy:0, facing:-1, hurtTimer:0, fireTimer:130,
  jumpTimer:220, onGround:false, state:'walk', deadTimer:0};
const bowserFire = [];
```

### ボス戦エリア
- x=**6700**〜7750 の範囲で左右折り返し（境界を6200→6700に変更済み）
- 3連射ファイヤー（vx=5.5〜6.3、地面でバウンド最大6回）
- HPゼロ → `bowser.state='dead'` → `deadTimer=160` → ピーチ姫出現

### 城門ウォール
- x=6560, 6592 に縦一列レンガ壁（2列）
- y=192〜255 に2タイル分の**通路**（マリオが階段頂上からジャンプして入る）
- クッパの境界 x=6700 と組み合わせて、戦闘エリア前にクッパが出ない
- 壁を追加するには `[0,32,64,96,128,160,256,288,320,352,384].forEach(wy=>{addB(x,wy,'brick')...})` パターンを使う

### クッパ踏みつけ判定
```javascript
// 頭上35%に着地したときのみHPダメージ（それ以外はkillMario）
if(mBot-mario.vy <= bowser.y+bowser.h*0.35 && bowser.hurtTimer===0){...}
```
- `bowser.y+12` の固定値は**使わない**（頭に乗っても必ずダメージになるバグ）

### ピーチ姫チェイス
```javascript
let peach = {alive:false, x, y, w:30, h:52, vx:0, caught:false, walkFrame:0, walkTimer:0};
let peachChase = null;  // {t, catchT}
// クッパ死亡後にpeach.alive=true、peach.vx=3.5で逃げる
// マリオが追いかけて重なったらcaught=true → 120f後 state='win'
```

---

## アイテム

| アイテム | 効果 |
|---|---|
| コイン | score+100, coins+1, 100枚で1UP |
| キノコ | 小→大マリオ |
| ファイアフラワー | ファイア状態 |
| スター | starTimer=480, mario.inv=60（短い保護のみ） |
| 1UP | lives+1 |

---

## 特殊ブロック（platforms.push 個別配置）

```javascript
// スター入りはてなブロック
platforms.push({x, y, w:TILE, h:TILE, type:'question', hit:false, hasStar:true, bounceOffset:0});
// 1UPかくしブロック
platforms.push({x, y, w:TILE, h:TILE, type:'hidden', hit:false, has1UP:true, bounceOffset:0});
// コインブロック（複数回叩ける）
platforms.push({x, y, w:TILE, h:TILE, type:'question', hit:false, coinBlock:true, hitsLeft:8, bounceOffset:0});
// ヨシ卵ブロック
platforms.push({x, y, w:TILE, h:TILE, type:'yoshiEgg', hit:false, bounceOffset:0});
// キノコ入りはてなブロック（hasMush=trueをaddRowで指定してもOK）
platforms.push({x, y, w:TILE, h:TILE, type:'question', hit:false, hasMush:true, bounceOffset:0});
```

---

## ステージ設計ルール（重要）

### ブロック配置の重複禁止
**addRow と platforms.push を同じ (x,y) に使ってはいけない（2つ重なって衝突判定が狂う）**

```javascript
// BAD: 同じ座標に両方
addRow(1280, H-5*TILE, 1, 'q');
platforms.push({x:1280, y:H-5*TILE, ..., hasStar:true});  // 重複バグ！

// GOOD: 特殊ブロックは push のみ
platforms.push({x:1280, y:H-5*TILE, ..., hasStar:true});  // pushだけ
```

### buildLevelX() の必須リセット（全項目）
```javascript
[platforms,pipes,coinItems,enemies,mushrooms,fireballs,piranhas,
 particles,scorePopups,blockAnims,movingPlats,springs,cannons,
 bulletBills,yoshiEggs,yoshiItems].forEach(a=>a.length=0);
hammers.length=0; bowserFire.length=0; lavaFlames.length=0;
starTimer=0; combo=0; comboTimer=0;
checkpointReached=false; checkpoint=null; goalSlide=null;
ugMode=false; savedOW=null; peach.alive=false; peachChase=null;
```

### ヨシのリセット（状況で異なる）
```javascript
// ステージ遷移時（乗っている場合のみ持ち越し）
if(!yoshi.mounted){yoshi.alive=false;yoshi.eatCount=0;}
yoshi.runAway=false; yoshi.runTimer=0; yoshi.eggsReady=0; yoshi.idleTimer=0;

// リスタート時（ヨシを消す）
yoshi.alive=false; yoshi.mounted=false; yoshi.eatCount=0;
yoshi.eggsReady=0; yoshi.runAway=false; yoshi.idleTimer=0;
```
- **idle（停止）状態のヨシはステージをまたいで持ち越さない**（`alive=false` にする）
- 騎乗中のみ次ステージへ持ち越し可能

### 地下ワープのルール
保存すべき配列: `platforms, pipes, coinItems, enemies, mushrooms, piranhas, movingPlats, springs, cannons, cam, mario.x, mario.y`
クリアすべき配列: `bulletBills, hammers, yoshiEggs, yoshiItems`
※ movingPlats/springs/cannons を保存し忘れると復帰後にゴーストオブジェクトが残る

### BGM ルール
- 通常BGM: `THEME_NOTES`（地上）/ `UG_NOTES`（地下）/ `CASTLE_NOTES`（1-4専用）
- スター無敵中: `STAR_NOTES`（`scheduleBGM` が自動切替）
- スター開始/終了時: `stopBGM()` → `startBGM()` でリセット必要
- scheduleBGM の優先順位: `ugMode > starTimer > currentLevel===4 > THEME_NOTES`
- 新ワールドに専用BGMを追加する場合: `CASTLE_NOTES` と同様に配列を追加し `scheduleBGM` の条件分岐を伸ばす

### killMario(force) の仕様
```javascript
killMario()        // 通常: star/inv/ヨシ/パワーアップ保護あり
killMario(true)    // 強制: 全保護スキップ（穴落下専用）
```
- 穴落下判定: `if(mario.y > H+40) killMario(true);`
- `force=true` はヨシも即消滅（`yoshi.mounted=false; yoshi.alive=false`）
- **新ステージ追加時もこの穴落下検出はそのまま使える（共通処理）**
- ⚠️ 穴落下に `killMario()` を使うと、inv>0 で弾かれて死なないバグが再発する

---

## ステージ追加方法（2-1以降）

### 命名規則
```
World 1: buildLevel() / buildLevel2() / buildLevel3() / buildLevel4()
World 2: buildLevel_2_1() / buildLevel_2_2() / buildLevel_2_3() / buildLevel_2_4()
World 3: buildLevel_3_1() など
```

### 追加手順（チェックリスト）
1. `buildLevel_2_1()` 関数を作成（`buildLevel()` を参考に）
2. 全配列リセット（上記「必須リセット」全項目）
3. `goalSlide` 完了ハンドラを更新：
   ```javascript
   // currentLevel===4 の処理を変更
   else if(currentLevel===4){currentLevel=1;currentWorld=2;buildLevel_2_1();...}
   ```
4. `restartCurrentLevel()` にケースを追加：
   ```javascript
   else if(currentWorld===2&&currentLevel===1)buildLevel_2_1();
   // ... 以下同様
   ```
5. HUD の `currentWorld+'-'+currentLevel` は自動反映される

### 背景の追加
`drawBG()` 関数に `currentWorld`/`currentLevel` の分岐を追加：
```javascript
if(currentWorld===2&&currentLevel===1){
  // 2-1用の背景描画
}
```

### ステージ設計の難易度ガイド
| ポジション | 難易度 | 特徴 |
|---|---|---|
| X-1 | 易〜中 | ワールド導入、新ギミック紹介 |
| X-2 | 中 | 敵・ギミック組み合わせ |
| X-3 | 中〜難 | ギャップ多め、スクロールの難所 |
| X-4 | 難 | 城テーマ or ボス戦 |

### ステージ設計テンプレート
```javascript
function buildLevel_2_1(){
  [platforms,pipes,coinItems,enemies,mushrooms,fireballs,piranhas,
   particles,scorePopups,blockAnims,movingPlats,springs,cannons,
   bulletBills,yoshiEggs,yoshiItems].forEach(a=>a.length=0);
  hammers.length=0;bowserFire.length=0;lavaFlames.length=0;
  starTimer=0;combo=0;comboTimer=0;checkpointReached=false;
  checkpoint=null;goalSlide=null;ugMode=false;savedOW=null;
  peach.alive=false;peachChase=null;
  yoshi.runAway=false;yoshi.runTimer=0;yoshi.eggsReady=0;

  // 地面（ギャップ定義）
  const gaps=[{s:2500,e:2650},...];
  for(let x=0;x<LW;x+=TILE)if(!gaps.some(g=>x>=g.s&&x<g.e))
    platforms.push({x,y:H-TILE,w:TILE,h:TILE,type:'ground',bounceOffset:0});

  // ブロック配置
  addRow(...); // 特殊ブロックは addRow しない

  // パイプ（ワープパイプには piranha 出さない）
  [[x,h,isWarp],...].forEach(([px,ph,warp])=>{
    pipes.push({x:px,y:H-TILE-ph*TILE,w:TILE*2,h:ph*TILE,bounceOffset:0,isWarp:warp})});
  pipes.forEach((p,i)=>{if(p.isWarp)return;
    piranhas.push({x:p.x+8,baseY:p.y,y:p.y,w:16,h:TILE,phase:i*1.5,alive:true,maxUp:TILE*1.5})});

  // コイン
  // 敵（parakoopa は x>=2000 のみ）
  // 移動足場・バネ・チェックポイント
  // 特殊ブロック（push のみ）
}
```

---

## 描画システム

### カメラ座標系（重要）
```javascript
ctx.save();
ctx.translate(-cam, 0);
// ← この中の描画はすべてワールド座標（cam を引かない）
ctx.restore();
```
- `bowser.x`, `peach.x`, `lavaFlames[].x` など全てワールド座標で描画
- `ctx.translate(-cam,0)` の外（HUD等）はスクリーン座標

### 背景 drawBG()
- `ugMode=true`: 地下（黒背景）
- `currentLevel===2`: 夕方
- `currentLevel===3`: 夜
- `currentLevel===4`: 城（暗い赤）
- それ以外: 青空（デフォルト）
- 新ワールドは `currentWorld` も参照するよう追記する

---

## 開発ルール
- **1ファイル維持**: index.htmlにすべて記述
- **jqは使えない**: Windowsのため。フックはNode.jsで書く
- **変数名**: camelCase（略語あり）
- **パフォーマンス**: 毎フレーム全描画、オブジェクトは配列管理
- **ステージ追加**: 必ずチェックリストに従う（リセット漏れが最多バグ原因）

---

## ファイル構成
```
mario-game/
├── index.html          （全コード）
├── CLAUDE.md           （このファイル）
└── .claude/
    ├── settings.json   （フック設定：index.html更新時にリロード通知）
    └── skills/
        ├── mario-feature.md    （汎用機能追加スキル）
        ├── mario-review.md     （コードレビュースキル）
        ├── mario-boss.md       （ボス戦追加スキル）
        └── mario-new-level.md  （新ステージ追加スキル）
```

---

## 既知バグ・過去の修正履歴

| バグ | 原因 | 修正内容 |
|---|---|---|
| 1-4以外に炎が出る | lavaFlames.length=0 が1-4以外になかった | 全buildLevelXに追加 |
| スターで敵を倒せない | star取得時mario.inv=480にしていた | mario.inv=60に（invとstarTimerは別） |
| parakoopa踏めない | 判定閾値 e.y+8 が小さすぎた | e.y+e.h*0.4 に変更 |
| ハンマーブロス踏めない | 同上 | 同上 |
| クッパ描画されない | ctx.translate(-cam)適用後にbx=bowser.x-camで二重引き算 | bx=bowser.x（ワールド座標）に修正 |
| パイプに入れない | stairDブロックがパイプ内に重複 | パイプをstairD範囲外に移動 |
| 逃げたヨシに乗れない | runTimer終了でalive=false | runAway=falseにして停止・再乗車可能に |
| ブロック重複衝突 | addRowとplatforms.pushを同座標に使用 | どちらか一方のみに統一 |
| スターで敵を倒せない（一部） | 敵当たり判定の外側条件が`mario.inv===0`のみ。スター取得直後のinv=60が残っている間は判定自体がスキップされていた | 条件を`(mario.inv===0\|\|starTimer>0)`に変更 |
| パックンフラワーがスターで倒せない | piranha衝突にstarTimer分岐がなかった | starTimer>0時にpr.alive=falseで倒す処理を追加 |
| 穴落下で死なない | killMario()冒頭のinv>0チェックで弾かれる（ヨシ降車・変身でinv=120がセットされた直後に再チェックされて通過） | killMario(force=true)を追加。穴落下はforce=trueで全保護をスキップ |
| クッパの頭を踏んでもダメージを受ける | bowser踏みつけ判定が `bowser.y+12`（固定12px）で狭すぎた。頭上部に乗っていても胴体当たり扱いになっていた | `bowser.y+bowser.h*0.35` に変更（全敵と同じ割合判定） |
| クッパが戦闘エリア前に出てくる | bowserのx境界が6200で、staircase(x=6300)より手前まで来ていた | 境界をx=6700に変更し、x=6560に城門ウォールを追加 |
| ヨシがいつまでも残る（ステージ間） | buildLevel2/3/4 で `yoshi.alive` をリセットしていなかったため、idle状態のYoshiが次のステージへ持ち越された | 遷移時に `!yoshi.mounted` なら `alive=false`。騎乗中のみ持ち越し可 |
| ヨシがいつまでも残る（同ステージ内） | runAway終了後に停止したYoshiが永続していた | `idleTimer=480` を追加。8秒以内に乗らなければ `alive=false` |
