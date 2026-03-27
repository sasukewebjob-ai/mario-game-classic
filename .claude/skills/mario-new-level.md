---
name: mario-new-level
description: マリオゲームに新しいステージを追加する。ワールド・レベル番号指定で、設計→実装→接続まで行う。
---

新ステージの追加を依頼されました: $ARGUMENTS

## ステップ1: 既存コード確認（Explore Agent）

Explore サブエージェントで以下を調べてください：
- `goalSlide` の完了ハンドラ（update関数内）の現在の分岐構造
- `restartCurrentLevel()` の現在の分岐構造
- `drawBG()` の背景分岐（currentWorld/currentLevel の参照方法）
- 直前のステージ（buildLevelX）の構造（敵数・ギミック密度の参考）
- 現在の `currentWorld`, `currentLevel` の最大値

## ステップ2: ステージ設計（Plan Agent）

Plan サブエージェントで以下を設計してください：

### テーマと難易度
CLAUDE.md の難易度ガイドを参照：
- X-1: 易〜中、新ワールドの導入
- X-2: 中、ギミック組み合わせ
- X-3: 中〜難、ギャップ多め
- X-4: 難、ボス or 城テーマ

### ステージ要素の設計
1. **背景テーマ**（青空・夕方・夜・城・水中・砂漠 など）
2. **地形**: ギャップ位置と数（X-1=1〜2個、X-3=4〜6個）
3. **ブロック配置**: Zone ごとに分割（約1000px区切り）
4. **パイプ**: 本数・高さ・ワープ有無
5. **敵の密度**: Zone 1は少なめ、後半は多め
6. **ギミック**: 移動足場・バネ・砲台・チェックポイント
7. **特殊ブロック**: スター・1UP・ヨシ卵・コインブロックの位置

## ステップ3: 実装

### 3-1. 関数作成（CLAUDE.mdの命名規則に従う）
```
World 1: buildLevel(), buildLevel2(), buildLevel3(), buildLevel4()
World 2: buildLevel_2_1(), buildLevel_2_2(), buildLevel_2_3(), buildLevel_2_4()
World 3: buildLevel_3_1() など
```

### 3-2. 必須リセット（コピーして使う）
```javascript
function buildLevel_X_Y(){
  [platforms,pipes,coinItems,enemies,mushrooms,fireballs,piranhas,
   particles,scorePopups,blockAnims,movingPlats,springs,cannons,
   bulletBills,yoshiEggs,yoshiItems].forEach(a=>a.length=0);
  hammers.length=0;bowserFire.length=0;lavaFlames.length=0;
  starTimer=0;combo=0;comboTimer=0;checkpointReached=false;
  checkpoint=null;goalSlide=null;ugMode=false;savedOW=null;
  peach.alive=false;peachChase=null;
  if(!yoshi.mounted){yoshi.alive=false;yoshi.eatCount=0;}
  yoshi.runAway=false;yoshi.runTimer=0;yoshi.eggsReady=0;yoshi.idleTimer=0;
  // ...
}
```

### 3-3. goalSlide 完了ハンドラを更新
```javascript
// 例: 1-4クリア後に2-1へ
else if(currentLevel===4&&currentWorld===1){
  currentLevel=1; currentWorld=2;
  buildLevel_2_1();
  // BGM・タイマー等リセット
}
```

### 3-4. restartCurrentLevel() にケースを追加
```javascript
else if(currentWorld===2&&currentLevel===1)buildLevel_2_1();
else if(currentWorld===2&&currentLevel===2)buildLevel_2_2();
// ...
```

### 3-5. drawBG() に背景を追加（新テーマの場合）
```javascript
if(currentWorld===2&&currentLevel===1){
  // グラデーション等で背景描画
}
```

## CLAUDE.md 遵守チェック（実装後に確認）

- [ ] 全配列リセットが漏れなくあるか（lavaFlames含む）
- [ ] ヨシリセットが「ステージ遷移時」の形式か（alive=falseにしない）
- [ ] addRow と platforms.push の重複がないか
- [ ] parakoopa は x>=2000 のみか
- [ ] パイプが stairD ブロックと重複していないか
- [ ] goalSlide ハンドラを更新したか
- [ ] restartCurrentLevel() にケースを追加したか
- [ ] CLAUDE.md のステージ表を更新したか
