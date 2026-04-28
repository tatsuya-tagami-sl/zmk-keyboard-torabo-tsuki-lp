# トラックボール反転の調査メモ（work vs master想定）

## 1) 最有力原因

### `snippets/input-split-listener/input-split-listener.overlay`
- `f77365e` で、`input-processors` から `zip_xy_transform (INPUT_TRANSFORM_X_INVERT | INPUT_TRANSFORM_Y_INVERT)` が削除されている。
- そのため、master 側で必要だった「X/Y 反転補正」が work 側で失われ、
  ポインタ移動方向・スクロール方向が逆転して見える可能性が高い。

該当箇所（現状）:
- 通常移動: `input-processors = <&zip_temp_layer_layer1_right_non_click 1 2000>;`
- スクロール: `input-processors = <&zip_xy_to_scroll_mapper &zip_scroll_scaler 1 3>;`

## 2) 併発しうる原因

### A. `input-listener` と `input-split-listener` の適用差
- `build.yaml` の artifact ごとに適用 snippet が異なる。
- 例: `torabo_tsuki_lp_double_ball_right_central` は `input-listener` と `input-split-listener` の両方を使う。
- 片方だけ反転設定が異なると、ローカル球とリモート球で方向が食い違う。

### B. レイヤー対象の変更 (`<2 3 4>` → `<2 3>`)
- `f77365e` でスクロール対象レイヤーが 4 を除外。
- 反転問題そのものではないが、ユーザー体感として「一部レイヤーだけ挙動が違う」状態を作りやすい。

### C. 反転処理の位置（`xy_to_scroll_mapper` の前/後）
- 過去コミットで順序の修正履歴あり（`694602f`）。
- 将来再修正時に順序を戻してしまうと、再びスクロール方向のみ逆転する恐れ。

## 3) 修正候補（優先順）

1. `snippets/input-split-listener/input-split-listener.overlay`
   - 通常移動の `input-processors` に XY 反転を戻す。
   - スクロールチェーンにも XY 反転を戻す（mapper 前段）。

2. `build.yaml`
   - 反転を必要とする artifact を洗い出し、
     `input-listener` / `input-split-listener` の差分で意図せず不一致が出ていないか確認。

3. `snippets/input-listener/input-listener.overlay`
   - ローカル球と split 側で反転ポリシーを合わせる（必要なら同一設計に統一）。

## 4) すぐに確認すべき観点
- どの artifact（left/right central/peripheral, double_ball）で問題が出るか。
- 問題が「移動」だけか、「スクロール」だけか、両方か。
- レイヤー 2/3/4 で差が出るか。

上記が揃えば、反転を戻すべきノード（`pointing_listener` vs `pointing_device_split_listener`）を最短で確定できる。
