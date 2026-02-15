# CYBER SHOOTER

サイバーパンク風の縦スクロールシューティングゲーム（STG）。スマホ操作に最適化された1面スコアアタック形式。

## プロジェクト概要

- **タイプ**: HTML5ゲーム（単一ファイル）
- **プラットフォーム**: スマホメイン（PC対応あり）
- **技術**: Vanilla JavaScript + Canvas API
- **依存**: なし（外部ライブラリ・画像・音声ファイル不要）

## ファイル構成

```
game-shot/
├── CLAUDE.md          # このファイル
└── index.html         # ゲーム本体（CSS/JS全て内蔵、~2000行）
```

## ゲーム仕様

### 操作方法

**スマホ（メイン）:**
- 画面どこでも指を置いて左右にスワイプ → 自機が左右移動
- 射撃は完全オートファイア（操作不要）

**PC:**
- ←→キー または A/Dキー → 左右移動
- 射撃は自動

### ゲームシステム

- **1面エンドレス** - 敵が無限に出現、難易度が徐々に上昇
- **3ライフ制** - 被弾で1ライフ減少、無敵時間あり
- **スコアアタック** - ハイスコアをlocalStorageに保存
- **コンボシステム** - 連続撃破でx1〜x255まで倍率上昇、3秒で減衰
- **ジャックポット** - コンボ50/100/150/200/255到達で大量ボーナス+演出

### ビジュアル

**カラーパレット:**
- シアン (#00ffff)
- マゼンタ (#ff00ff)
- ゴールド (#ffd700)
- パープル (#aa00ff)
- 背景 (#0a0a1a)

**背景（3層）:**
1. 星空（ランダムな点が下にスクロール）
2. パースペクティブグリッド（Tron風の遠近法床）
3. ネオンワイヤーフレームビル群（両サイド）

**エフェクト:**
- CRTスキャンライン（CSS overlay）
- ビネット効果（画面端が暗い）
- 大規模パーティクル爆発（火球+小粒子）
- 大量コイン/ジェム散布

### エンティティ

**プレイヤー:**
- サイズ: 大きめの戦闘機
- 位置: 画面下部固定、左右のみ移動
- パワーレベル: 0-3（弾の広がり変化）

**敵3種:**
- ドローン（HP1、直進、スコア100）
- ファイター（HP3、弾発射、サイン波移動、スコア300）
- ヘビー（HP8、3連弾、ホバー、スコア1000）

**ピックアップ:**
- コイン（+100点、50%ドロップ率）
- ジェム（+500点、10%ドロップ率）
- パワーアップ（武器強化、5%ドロップ率）
- 磁石効果で自動収集（範囲80px）

## 技術詳細

### Canvas設定

```javascript
// 内部解像度: 480x854 (9:16スマホ比率)
canvas.width = 480;
canvas.height = 854;
ctx.imageSmoothingEnabled = false; // ピクセルアート
```

### オブジェクトプーリング

高頻度生成エンティティは事前確保で再利用:
- プレイヤー弾: 100個
- 敵弾: 100個
- パーティクル: 500個
- ピックアップ: 150個

### ゲームループ

```javascript
requestAnimationFrame(gameLoop);
// dt = (timestamp - lastTime) / 16.667  // 60fps基準
// dt上限: 3 (スパイラル防止)
```

### 当たり判定

- AABB（軸平行境界ボックス）
- プレイヤー弾 vs 敵: 100×30 = 最大3000チェック
- 敵弾 vs プレイヤー: 100×1 = 100チェック
- ピックアップ vs プレイヤー: 円形判定

### オーディオ

Web Audio APIでプロシージャル生成:
- 発射音: square波、880Hz、0.06s
- 爆発音: ノイズ + sawtooth波、周波数減衰
- コイン: sine波、1200Hz→1600Hz
- ジャックポット: アルペジオファンファーレ

## 実装ガイドライン

### コード構成（index.html内）

```
<!DOCTYPE html>
<html>
<head>
  <style>
    /* CSS: 100-150行 */
    /* - フルスクリーンレイアウト */
    /* - CRTスキャンライン */
    /* - ビネット */
    /* - レスポンシブスケーリング */
  </style>
</head>
<body>
  <div id="game-container">
    <canvas id="game"></canvas>
    <div id="scanlines"></div>
    <div id="vignette"></div>
  </div>

  <script>
    // 1. 定数・設定 (80行)
    // 2. ユーティリティ関数 (60行)
    // 3. オーディオエンジン (120行)
    // 4. 入力マネージャー (50行) ← タッチ+キーボード
    // 5. オブジェクトプール (80行)
    // 6. エンティティ定義 (600行)
    //    - Player, Bullet, Enemy, Particle, Pickup, FloatingText
    // 7. 背景レンダラー (150行)
    // 8. ウェーブシステム (150行)
    // 9. スコア/コンボ/ジャックポット (100行)
    // 10. HUD (100行)
    // 11. ゲーム状態管理 (120行)
    // 12. メインループ+衝突判定 (150行)
    // 13. 初期化 (40行)
  </script>
</body>
</html>
```

### タッチ入力実装

```javascript
let touchStartX = 0;
let lastTouchX = 0;

canvas.addEventListener('touchstart', (e) => {
  touchStartX = e.touches[0].clientX;
  lastTouchX = touchStartX;
});

canvas.addEventListener('touchmove', (e) => {
  e.preventDefault();
  const touchX = e.touches[0].clientX;
  const deltaX = touchX - lastTouchX;
  player.x += deltaX * touchSensitivity;
  lastTouchX = touchX;
});
```

### パースペクティブグリッド実装

```javascript
// 地平線 35%、消失点は中央
const horizonY = HEIGHT * 0.35;
const vanishX = WIDTH / 2;

// 水平線: 二乗間隔で遠近法
for (let i = 0; i < lineCount; i++) {
  const t = ((i / lineCount + offset) % 1) ** 2;
  const y = horizonY + (HEIGHT - horizonY) * t;
  ctx.moveTo(0, y);
  ctx.lineTo(WIDTH, y);
}

// 垂直線: 消失点に収束
for (let i = 0; i <= vLineCount; i++) {
  const t = i / vLineCount;
  const bottomX = t * WIDTH;
  const topX = lerp(vanishX, bottomX, 0.15);
  ctx.moveTo(topX, horizonY);
  ctx.lineTo(bottomX, HEIGHT);
}
```

### 爆発エフェクト

```javascript
function spawnExplosion(x, y, size) {
  // 火球（大）
  particlePool.spawn(p => p.init(
    x, y, 0, 0, 30, size * 3,
    'fireball', ['#ff6600', '#ffd700']
  ));

  // 小粒子（多数）
  for (let i = 0; i < size * 10; i++) {
    const angle = Math.random() * Math.PI * 2;
    const speed = randFloat(1, 5);
    particlePool.spawn(p => p.init(
      x, y,
      Math.cos(angle) * speed,
      Math.sin(angle) * speed,
      randFloat(20, 50), randFloat(2, 6),
      'spark', ['#ff8800', '#ffd700', '#fff']
    ));
  }
}
```

## デバッグ・テスト

### ブラウザで開く

```bash
# ファイルをブラウザにドラッグ&ドロップ
# または
open index.html
```

### スマホでテスト

```bash
# ローカルサーバー起動（必要に応じて）
python3 -m http.server 8000
# スマホで http://[PC-IP]:8000/index.html
```

### デバッグモード

```javascript
// CONFIG に追加
DEBUG: false,

// デバッグ描画
if (CONFIG.DEBUG) {
  // 当たり判定枠表示
  ctx.strokeRect(enemy.x - enemy.width/2, enemy.y - enemy.height/2,
                 enemy.width, enemy.height);
}
```

## パフォーマンス目標

- 60fps維持（スマホでも）
- 最大エンティティ数: プレイヤー弾100 + 敵弾100 + 敵30 + パーティクル500 + ピックアップ150
- メモリ使用量: <50MB

## 今後の拡張案

- [ ] ボスエネミー（HPバー、複数フェーズ）
- [ ] 追加武器種（レーザー、ミサイル）
- [ ] ローカルマルチプレイヤー
- [ ] リプレイ機能
- [ ] ランキングボード（オンライン）

## 参考

- [MDN Canvas API](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API)
- [MDN Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
- [HTML5 Game Development](https://spicyyoghurt.com/tutorials/html5-javascript-game-development/)
