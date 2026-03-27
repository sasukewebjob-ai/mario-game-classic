---
name: mario-boss
description: マリオゲームにボス戦を追加する。ボスAI・状態機械・HP管理・描画・勝利条件を含む複合実装。
---

ボス戦の追加を依頼されました: $ARGUMENTS

## ステップ1: 既存コード把握（Explore Agent）

Explore サブエージェントで以下を調べてください：
- 既存の敵オブジェクト構造（goomba/koopa/hammerBro などの変数・更新・描画パターン）
- `update()` 関数内の敵処理ループの位置と構造
- `draw()` 関数内の敵描画の位置
- `goalSlide` の完了ハンドラ（ステージ遷移コード）
- `restartCurrentLevel()` の内容
- `killMario()` のロジック
- `sfx()` で使える効果音の種類

## ステップ2: ボス設計（Plan Agent）

Plan サブエージェントで以下を設計してください：

### ボスオブジェクト設計
```javascript
let boss = {
  alive: false,
  x, y, w, h,
  hp,           // 最大HP
  vx,           // 移動速度
  facing,       // 向き
  phase,        // 'idle'|'walk'|'attack'|'hurt'|'dead'
  hurtTimer,    // ダメージ無敵フレーム
  attackTimer,  // 攻撃クールダウン
  // ボス固有の追加フィールド
}
```

### ボス戦トリガー設計
- 通常の flagPole 到達でなくボス戦エリア進入で開始
- `state` に `'boss'` フェーズは追加しない（`state='play'` のまま `boss.alive` で管理）

### 勝利条件
- ボス撃破 → `state='win'` または次ステージへ遷移
- `goalSlide` ではなく独自の勝利演出

### 敗北条件
- `restartCurrentLevel()` で同ステージ再スタート時に `boss` をリセット

## ステップ3: 実装

以下の順番で実装してください：

1. **ボス変数宣言**（グローバル変数セクション）
2. **`buildLevelX()` 内でのボス初期化**（全配列リセット後に追記）
3. **ボス更新ロジック**（`update()` 内、敵ループの後）
   - 移動（左右反転・壁折り返し）
   - 攻撃パターン（タイマー管理）
   - マリオとの当たり判定（`killMario()` 呼び出し）
   - 被ダメージ判定（ファイアボール・踏みつけ）
   - 撃破処理（スコア加算・BGM停止・勝利演出）
4. **ボス描画**（`draw()` 内、敵描画セクションに追記）
   - HPバー描画
   - ボス本体（Canvas API で描画）
5. **ステージ遷移の更新**（`goalSlide` ハンドラ・`restartCurrentLevel()`）

## CLAUDE.md 遵守チェック（実装後に確認）

- [ ] `buildLevelX()` 冒頭でボス変数をリセットしているか
- [ ] `restartCurrentLevel()` にケースを追加したか
- [ ] addRow と platforms.push の重複がないか
- [ ] parakoopa を x<2000 に置いていないか
- [ ] 1ファイル（index.html）に全コードが収まっているか
- [ ] CLAUDE.md の「現在の実装状況」テーブルを更新したか

## 1-4 クッパの実装仕様（参考）

### 踏みつけ判定（重要）
```javascript
// bowser.y+bowser.h*0.35 を使う（固定値は使わない）
if(mBot-mario.vy<=bowser.y+bowser.h*0.35&&bowser.hurtTimer===0){
  bowser.hurtTimer=70;bowser.hp--;mario.vy=-11;...
}else killMario();
```
- `bowser.y+12`（固定値）は**使わない**→頭に乗ってもダメージになるバグの原因

### ボス登場エリアの制限
- x境界: `if(bowser.x<6700){bowser.x=6700;bowser.vx=Math.abs(bowser.vx)}`
- 城門ウォール（x=6560, x=6592）と組み合わせることで、戦闘エリア前にボスが出ない
- 城門ウォールのパターン:
  ```javascript
  [0,32,64,96,128,160,256,288,320,352,384].forEach(wy=>{addB(6560,wy,'brick');addB(6592,wy,'brick');});
  // y=192〜255 が通路（マリオは階段を登ってジャンプで入る）
  ```

### ステージ専用BGM
- ボスステージには専用の `CASTLE_NOTES` 配列を用意し、`scheduleBGM` で切り替える:
  ```javascript
  const notes=ugMode?UG_NOTES:(starTimer>0?STAR_NOTES:(currentLevel===4?CASTLE_NOTES:THEME_NOTES));
  ```
