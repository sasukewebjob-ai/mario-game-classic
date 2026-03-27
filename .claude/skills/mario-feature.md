---
name: mario-feature
description: マリオゲームに新機能を追加する。Explore→Plan→実装の3ステップで進める。
---

新機能の追加を依頼されました: $ARGUMENTS

## ステップ1: 現状把握（Explore Agent）

Explore サブエージェントで `mario-game/index.html` の関連コードを調べてください：
- 追加機能に関連する既存の変数・関数を特定する
- どこに追加コードを挿入すべきか把握する
- 既存の命名規則・コードスタイルを確認する

## ステップ2: 設計（Plan Agent）

Plan サブエージェントで実装計画を立ててください：
- 何をどこに追加するか（変数、関数、呼び出し箇所）
- 既存コードへの影響範囲
- 実装の順序

## ステップ3: 実装

CLAUDE.md のルールに従って実装してください。

### 座標系の注意（重要）
- `ctx.translate(-cam, 0)` の内側で描画する場合 → **ワールド座標**（cam を引かない）
- HUD や UI は `ctx.translate` の外 → **スクリーン座標**

### グローバル状態
- `mario.inv` = ダメージ後の無敵フレーム
- `starTimer` = スター無敵の残りフレーム（invとは別物！）
- `state` = `'start'|'intro'|'play'|'dead'|'over'|'win'`
- `currentWorld`, `currentLevel` = 現在のワールド・レベル番号

### BGM の追加・変更
- BGM配列は `THEME_NOTES`, `UG_NOTES`, `STAR_NOTES`, `CASTLE_NOTES`（1-4専用）
- `scheduleBGM` の優先順位: `ugMode > starTimer > currentLevel===4 > THEME_NOTES`
- 新ステージ専用BGMは配列を追加して `scheduleBGM` の条件分岐を伸ばす:
  ```javascript
  const notes=ugMode?UG_NOTES:(starTimer>0?STAR_NOTES:(currentLevel===4?CASTLE_NOTES:THEME_NOTES));
  ```

### 敵の踏みつけ判定（重要）
- **全ての敵（クッパ含む）**: `e.y+e.h*0.4` — 固定値ピクセルは使わない
- クッパ: `bowser.y+bowser.h*0.35`（少し厳しめ）

### ヨシ関連の注意（重要）
- `yoshi.idleTimer` が存在する。runAway 終了→停止→8秒で消滅の流れ
- **ステージ遷移時のヨシリセット**: `if(!yoshi.mounted){yoshi.alive=false;yoshi.eatCount=0;}` を必ず書く
  - 乗っている場合のみ次ステージへ持ち越し
  - idle（停止）状態のヨシは持ち越さない（永続するバグになる）
- 必ず `yoshi.idleTimer=0` もリセットに含める

### 新しい配列を追加する場合
以下の**全て**の buildLevelX 関数に `.length=0` リセットを追加する：
`buildLevel()`, `buildLevel2()`, `buildLevel3()`, `buildLevel4()`, 今後追加するもの全て

### 実装後の確認
- CLAUDE.md の「現在の実装状況」を更新する
- 既知バグ表に修正内容を記録する
