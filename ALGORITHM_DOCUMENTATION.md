# map.html — 伝搬計算アルゴリズムと数式ドキュメント

この文書は map.html に実装されている主要アルゴリズム、数式、変数の意味、近似・分岐ロジック、および注意点を詳細に説明します。実装に使われる定数や単位も併せて整理しています。式は実装に忠実に記述し、実装上の特殊ケースや数値処理上の扱い（クリッピングやフォールバックなど）も注記します。

---

目次
- 概要
- 定数と単位
- 座標・球面処理
  - 距離（大円距離）: Haversine 相当
  - 大円補間（球面線形補間）
  - 方位角（大円方位角）
- 標高データ処理・プロファイル離散化
  - サンプリング数（steps）決定ロジック
  - 標高取得・キャッシュ
- 基礎伝搬量（波長・有効地球半径）
- Fresnel 第1ゾーン半径
- 地球曲率補正（平滑球面上の直線線形補正）
- クリアランス・F1マージン・遮蔽率
- 損失モデル
  - 自由空間損失（FSPL）
  - ナイフエッジ回折（Lee近似による v->損失）
  - 平滑球面大地損失（実装上の簡易モデル）
  - 簡易的遮蔽損失（強遮蔽時の線形近似）
  - モデル選択ロジック（F1遮蔽率に応じた分岐）
- 受信電力計算（合成）
- 仰角・俯角計算
- グラフ表示（プロットデータ）
- 実装上の注意点・数値的安定化
- 参考

---

## 1. 概要
map.html の伝搬計算部は、地表標高プロファイル（地理院DEM）をサンプルして、Tx→Rx の経路について
- LOS 判定（視線遮蔽）
- 第1フレネルゾーンに基づく遮蔽評価（F1Margin、遮蔽率）
- 適切な損失モデル（FSPL、回折、平滑地球損失 等）の選択
- 受信電力（dBm）算出
- 仰角/俯角の算出
を行い、結果をチャートとサマリに表示します。

---

## 2. 定数と単位
- EARTH_RADIUS_M = R = 6,371,000 m（地球半径）
- 周波数: freqMhz（MHz）
- 波長: λ = c / f （m） ここで c = 299,792,458 m/s（光速）
  - 実装: lambda = 299792458 / (freqMhz * 1e6)
- 距離: distanceMeters（m）
- 高さ: m（メートル）
- 電力/利得:
  - dBm 単位（Tx 出力, 受信電力 など）
  - dBi 単位（アンテナ利得）
  - ケーブル損失: dB
- K_FACTOR: 有効地球半径倍率 K（無次元）
- Fresnel 第1ゾーン半径 F1（m）

---

## 3. 座標・球面処理

### 3.1 大円距離（ol.sphere.getDistance）
実装は Haversine 式に相当する計算：

与えられた点 c1=(lon1, lat1)、c2=(lon2, lat2)（度単位）に対して，
1. ラジアン変換: lon1_rad = lon1 * π/180, lat1_rad = lat1 * π/180（同様に lon2_rad, lat2_rad）
2. dLat = lat2_rad - lat1_rad
   dLon = lon2_rad - lon1_rad
3. a = sin^2(dLat/2) + cos(lat1_rad) * cos(lat2_rad) * sin^2(dLon/2)
4. c = 2 * atan2( sqrt(a), sqrt(1-a) )
5. distance = R * c

単位: m

（実装上は R = EARTH_RADIUS_M）

### 3.2 大円補間（ol.sphere.interpolate）
球面上で c1→c2 間の fraction（0..1）位置を返す。アルゴリズムは球面線形補間（great-circle interpolation, “slerp” の球面版）を用いる。

概略:
- d = great-circle central angle = distance(c1,c2) / R（ラジアン）
- if d ≈ 0:  単純線形補間（経度緯度を線形補間して度に戻す）
- A = sin((1-fraction)*d) / sin(d)
  B = sin(fraction * d) / sin(d)
- x = A cos(lat1) cos(lon1) + B cos(lat2) cos(lon2)
  y = A cos(lat1) sin(lon1) + B cos(lat2) sin(lon2)
  z = A sin(lat1) + B sin(lat2)
- 補間点の経度経度: lon = atan2(y,x), lat = atan2(z, sqrt(x^2+y^2))
- 返却値は度単位に戻す

この方式は球面上の最短経路（大円）上の点を返す。

### 3.3 大円方位角（calculateGreatCircleBearing）
c1→c2 の方位（真北基準の度, 0..360）を計算：

- lon, lat をラジアンに変換
- dLon = lon2 - lon1
- y = sin(dLon) * cos(lat2)
  x = cos(lat1) * sin(lat2) - sin(lat1) * cos(lat2) * cos(dLon)
- bearing = atan2(y, x)（ラジアン）を度に変換。負なら +360 して 0..360 に正規化

---

## 4. 標高プロファイルの離散化（getElevationProfile）

目的: Tx→Rx 間に均等に近い間隔で N+1 点を生成し、それぞれの標高（DEM）を取得する。

実装上のポイント:
- 距離 distanceMeters に基づき間隔最大値 maxIntervalMeters を選定:
  - distance ≤ 5 km: maxIntervalMeters = 50 m
  - 5 km < distance ≤ 50 km: maxIntervalMeters = 100 m
  - distance > 50 km: maxIntervalMeters = distance / maxSteps（maxSteps = DEFAULT_PROFILE_STEPS = 200）
- 必要ステップ数 calculatedSteps = ceil(distance / maxIntervalMeters)
- 実際 steps = clamp(calculatedSteps, 10, maxSteps)
- 実際に i = 0..steps の (steps+1) 点を生成:
  - fraction = i / steps
  - coord = interpolate(coord1, coord2, fraction)（両端は正確に coord1, coord2）
  - distance at point = distanceMeters * fraction
  - 各点で getElevation(coord) を呼んで標高を取得（API呼び出し、レスポンスは elevation（m））
- profile に { coord, webMercatorCoord, elevation, distance } を格納

注意: 実装では API タイムアウトや欠損値（-9999）を扱い、欠損は 0 に置き換えている（実務的には欠損処理は注意が必要）。

---

## 5. 基礎伝搬量

### 5.1 波長
λ = c / f
- c = 299,792,458 m/s
- f = freqMhz × 1e6 Hz
- 単位: m

実装:
lambda = 299792458 / (freqMhz * 1e6)

### 5.2 有効地球半径
Kepler の K ファクター（実装変数名 K）により拡張された地球半径:

Re = K * R
- R = EARTH_RADIUS_M = 6,371,000 m
- K はユーザ設定（1.0, 4/3, 0.8 など）

---

## 6. Fresnel 第1ゾーン半径（F1）

定義（点が Tx から d1、残りが d2、全距離 D = d1 + d2）:
F1 = sqrt( (λ * d1 * d2) / (d1 + d2) )

- 単位: m
- 実装で d1 または d2 が 0 の場合は 0 を返す。

用途:
- LOS 線（地上の視線）から F1 が占める上下幅を算定し、障害物の高さが F1 にどれだけかぶるかで遮蔽度を評価する。

---

## 7. 地球曲率補正（heightProp）

実装では「Tx と Rx を結ぶ直線（高さ線）」に地球曲率補正を入れて、経路上の各点に対する LOS 直線上の高さを算出しています。

定義:
- txTotalHeight = txElevation + txHeightM
- rxTotalHeight = rxElevation + rxHeightM
- totalDistance = D

ある点までの距離を d（Tx からの距離）とすると、
- d_rem = D - d
- correctionProp = (d * d_rem) / (2 * Re)
  - 由来: 球面と直線の幾何に基づく二次近似の補正項（中心角を用いる厳密式の近似）
- baseProp = txTotalHeight + (rxTotalHeight - txTotalHeight) * (d / D)
  - Tx→Rx を線形補間した高さ（直線）
- heightProp = baseProp + correctionProp
  - LOS 直線に地球の曲率分を足した補正高さ（経路上で直線が地球表面からどれだけ浮くかを補正）

備考:
- correctionProp は地球曲率のために中間点で LOS 線が下がる（あるいは上がる）分を表す。Re を大きくすると補正は小さくなる（平坦化）。

---

## 8. クリアランス・F1 マージン・遮蔽率

各プロファイル点 p についての定義（実装変数名に対応）:

- p.elevation: DEM による地表標高（m）
- heightProp: LOS 直線上の高さ（m）（上節参照）
- clearanceProp = (p.elevation - heightProp) * -1
  - 意味: 地形が LOS 直線より高い場合は正の値（LOS を越えている方向の符号付けに注意）
  - 実装式は (p.elevation - heightProp) * -1。従って:
    - clearanceProp > 0 → 地形は LOS の下（クリアランスあり）
    - clearanceProp < 0 → 地形は LOS を越えている（遮蔽）
  - （注意: 符号の扱いで読み替えに注意）
- p.F1: 前節の Fresnel 半径（m）
- p.F1Margin = p.clearanceProp - p.F1
  - 意味: 第1フレネルゾーンの下端からどれだけ余裕があるかを示す（m）。
  - F1Margin > 0 → F1 が確保されている
  - F1Margin < 0 → F1 が遮蔽されている
- minF1Margin = profile 全点の最小 p.F1Margin（最も厳しい点）
  - これにより経路全体の最小余裕を評価
- F1 遮蔽率（blockage ratio）:
  - 実装では点ごとに currentBlockageRatio = (F1 - clearanceProp) / F1
    - これは 0 から ≳1 になる可能性がある（F1 が 0 の場合は注意）
  - maxF1BlockageRatio = max(currentBlockageRatio)（プロファイル中の最大値）
  - blockagePercent = maxF1BlockageRatio * 100
  - 解釈:
    - currentBlockageRatio ≤ 0 → F1 下端はクリア
    - 0 < currentBlockageRatio < 1 → 一部遮蔽（割合で表現）
    - ≥1 → 完全遮蔽（LOS 直線自体が塞がれている可能性）

注意: currentBlockageRatio の計算は p.F1 が 0（λ=0やd1/d2=0の場合）だと不安定になるため、実装では F1==0 の扱いに注意が必要（コード中では暗黙に回避されているが、実務では明示的なガードが望ましい）。

---

## 9. 損失モデル（合成と分岐）

全体戦略:
- まず自由空間損失（FSPL）を計算する
- その後 F1 遮蔽度（blockagePercent）に基づき追加損失を次のように決定して合算する:
  1. blockagePercent ≤ 10%: FSPL のみ（追加損失 0）
  2. 10% < blockagePercent ≤ 40%: 平滑球面大地損失（calculateSmoothEarthLoss）
  3. 40% < blockagePercent ≤ 80%: ナイフエッジ回折損失（calculateKnifeEdgeLoss） — keIndex 点を用いる（最大遮蔽点）
  4. blockagePercent > 80%: 簡易遮蔽損失（absBlockageDepth × 1 dB/m の線形近似）

### 9.1 自由空間損失（FSPL）
FSPL(dB) = 32.45 + 20 log10(f_MHz) + 20 log10(d_km)

- d_km = distanceMeters / 1000
- 単位: dB
- 実装では distanceKm==0 を特殊扱い（0返却）

由来: FSPL の一般式 FSPL(dB) = 20 log10(4π d / λ) を MHz/km 形式に変換した形。

### 9.2 ナイフエッジ回折（calculateKnifeEdgeLoss）
実装は Lee 等の近似式に基づく区分的近似を用いて v → 回折損失 Ld (dB) を算出し、最終的に L_d = max(0, -G_d) としている（G_d は回折利得の近似）。

フレネル・キルヒホッフ回折パラメータ v の定義（実装での評価式）:
- 遮蔽物点 kePoint に対して:
  - h_clearance = kePoint.clearanceProp（符号に注意）
  - h_abs = max(0, -h_clearance)（LOSより高い分の絶対値）
  - d1 = kePoint.distance（m） — Tx から遮蔽点まで
  - d2 = totalDistance - d1
- v = h_abs * sqrt( 2 * (d1 + d2) / (λ * d1 * d2) )
  - 単位: 無次元

Lee 近似（実装での分岐）:
- v ≤ -1: G_d = 0
- -1 < v ≤ 0: G_d = 20 log10(0.5 - 0.62 v)
- 0 < v ≤ 1: G_d = 20 log10(0.5 * exp(-0.95 v))
- 1 < v ≤ 2.4: G_d = 20 log10(0.4 - sqrt(0.1184 - (0.38 - 0.1 v)^2))
  - この領域で sqrt の内部が負になる場合は（数値エラー回避で）大きな損失を返す実装（例: return 30）
- v > 2.4: G_d = 20 log10(0.225 / v)

損失:
- L_d (dB) = max(0, -G_d)
  - G_d は回折利得（dB）。負の利得は損失に対応するため -G_d を損失と見なす。

備考:
- Lee 近似は経験式であり、厳密な Fresnel-Kirchhoff 数値解ではないが、実務上の目安となる。
- v の評価で d1*d2 が小さいと v が非常に大きくなる可能性があり注意が必要。

### 9.3 平滑球面大地損失（calculateSmoothEarthLoss）
実装は ITU-R P.526 等の詳細モデルの代替としての簡易経験式：

- 入力: distanceMeters, frequencyMhz, K_FACTOR
- 内部:
  - d_km = distanceMeters / 1000
  - f_ghz = frequencyMhz / 1000  // 実装上のミスを意図的に避けるべきだが、ここでは f_ghz = freqMhz / 1000（MHz→GHz）
  - K = parseFloat(K_FACTOR)
- 近似式（実装）:
  - loss = 0 初期
  - if d_km > 10:
      loss = 10 * log10(d_km) + 20 * f_ghz - (1.333 / K - 1) * 5
      loss = max(10, loss)  // 最小 10 dB を保証
- 返却: loss (dB)

解釈:
- 距離が長くかつ周波数が高いほど損失が増える傾向を定性的に表現
- K による補正を小さな線形項で調整している

注意:
- 正確な地球曲率と回折の理論解ではないため、あくまで簡易近似。長距離や厳密評価には ITU-R の詳細式を用いるべき。

### 9.4 簡易的な遮蔽損失（blockagePercent > 80%）
実装:
- absBlockageDepth = max(0, -minF1Margin)
  - minF1Margin は F1 の中心付近からの最小余裕（m）
- additionalLossDb = absBlockageDepth * 1.0  // 1 dB/m の係数で線形増加
- 返却: additionalLossDb

備考:
- 非常に粗い近似（遮蔽深さ 1 m あたり 1 dB 増加）であり、実務的には過大/過小評価の危険あり。

### 9.5 モデル選択ロジック（まとめ）
- blockagePercent = maxF1BlockageRatio * 100
- if blockagePercent ≤ 10: additionalLoss = 0, modelName = "FSPL"
- else if blockagePercent ≤ 40: additionalLoss = calculateSmoothEarthLoss(...)
- else if blockagePercent ≤ 80:
  - if keIndex valid → calculateKnifeEdgeLoss(v) at keIndex
  - else → fall back to calculateSmoothEarthLoss(...)
- else (blockagePercent > 80): additionalLoss = abs(minF1Margin) * 1 dB/m

総損失:
- totalLossDb = freespaceLossDb + additionalLossDb

---

## 10. 受信電力の計算（dBm）

受信電力 Rx (dBm) の式:

rxPowerDbm = txPowerDbm + txGainDbi + rxGainDbi - totalLossDb - txCableLossDb - rxCableLossDb

- ここで txPowerDbm: 送信機出力（dBm）
- txGainDbi: Tx アンテナ利得（dBi）
- rxGainDbi: Rx アンテナ利得（dBi）
- totalLossDb: FSPL + 追加損失（dB）
- txCableLossDb, rxCableLossDb: ケーブル損失（dB）を差し引く

注: 実際の EIRP を分離して追跡する（レベルダイヤグラム用）ために、各段階のレベル配列が作成される:
- 送信出力
- Tx Ant 入力（ケーブルロス反映）
- Tx EIRP（利得を加味）
- Rx Ant 入力（空間損失を引く）
- Rx Ant 出力（Rx利得を加味）
- 受信機（最終 rxPowerDbm）

---

## 11. 仰角・俯角計算

実装では地球曲率補正を考慮した単純近似で Tx→Rx の仰角（Tx 側）と Rx→Tx の俯角（Rx 側）を算出：

- txToRxAngleRad = atan2( rxTotalHeight - txTotalHeight + (D^2) / (2 * Re), D )
- rxToTxAngleRad = atan2( txTotalHeight - rxTotalHeight + (D^2) / (2 * Re), D )
- 角度を度に変換: deg = rad * 180 / π
- 実装では rxToTxAngleDeg に -1 を掛けて俯角を負で示している（見た目表現のため）

備考:
- (D^2)/(2 Re) 項は曲率補正を高さ差に相当する形で扱っている。厳密な幾何でなく近似だが短距離での傾斜角評価に適する。

---

## 12. グラフ表示用のデータ構成
- 標高プロファイルグラフ:
  - x: 距離（km） = p.distance / 1000
  - datasets:
    - 地表標高 (DEM): p.elevation
    - F1 上端: heightProp + F1
    - F1 下端: heightProp - F1
    - LOS ライン: heightProp
- レベルダイヤグラム:
  - 折れ線上の順序化された段階ごとの dBm 値（上節 10 参照）

Y 軸の範囲はデータに応じて動的に決定：
- 標高グラフ: 最大標高 + マージン, 最小標高 - マージン、F1 上端/下端を必ずカバー
- レベルグラフ: 最小点を 10dB 刻みで切り捨てて -10 dB マージン、最大点を 10dB 刻みで切り上げて +10 dB マージン

---

## 13. 標高データ取得の実装上の注意点
- getElevation(coord): 地理院 API を使って単点ごとに標高を取得
  - URL: ELEVATION_API_URL + ?lon=...&lat=...
  - タイムアウト（API_TIMEOUT_MS）を設定
  - レスポンスの elevation フィールドを parseFloat して使用
  - 欠損値 -9999 および NaN を 0 に置換（実装現状）
  - キャッシュ：elevationCache（Map）に小数点以下 5 桁でキー化して保存
- 大量の API 呼び出しは制限やレートリミット、遅延の原因になるため、間引き（steps の制御）とキャッシュは重要

---

## 14. 離散化ステップの決定ロジック（補足）
実装の主旨:
- 近距離では高解像度（50m 間隔）でサンプリング
- 中距離（5–50 km）では 100m 間隔
- 長距離では最大ステップ数（200）を超えないように距離を等分
- 最小 steps は 10（必ずある程度の解像度を確保）

これにより API 呼び出し回数と精度のトレードオフを管理する。

---

## 15. 数値安定性と境界条件（実装上の注意）
- 距離が 0 の場合は多くの計算が不定（除算など）になるため、早期リターンしている
- F1 が 0 になりうる状況（d1=0 など）に対するガードが必要（実装は暗黙扱い）。実用上は F1<=ε の場合は blockageRatio=0 や別扱いが望ましい
- Lee 近似の sqrt 内部が負になる場合のハンドリング（実装では return 30 dB を用いる）
- log10 の引数の正値条件・ 0 近傍でのクリッピングを適用すべき（実装では暗黙に避けているが実環境ではガードを推奨）
- API のタイムアウト / エラー時は elevation=0 に置換しているが、欠損が多いと結果が大きく狂うためユーザ通知や再試行が望ましい

---

## 16. 実例（数値例）

例: Tx-Rx 間距離 10 km、周波数 1000 MHz（1 GHz）、K = 4/3、Tx/Rx 上の地形を仮定せず TxHeight = RxHeight = 30 m の単純ケース:

1. distanceKm = 10 → FSPL = 32.45 + 20 log10(1000) + 20 log10(10) = 32.45 + 60 + 20 = 112.45 dB
2. λ = 0.299792458 m
3. Fresnel 例（中間点 d1 ≈ d2 ≈ 5,000 m）:
   F1 ≈ sqrt( λ * d1 * d2 / (d1 + d2) ) = sqrt(0.2998 * 5000 * 5000 / 10000) ≈ sqrt(0.2998 * 2500) ≈ sqrt(749.5) ≈ 27.38 m
4. 仮に地形が LOS 高さより 5 m 高ければ h_abs = 5 m、v を計算し Lee 式で回折損失を推定 など。

（上は概算例。実際はプロファイル点ごとの標高を用いて minF1Margin と遮蔽率を求め、モデル選択を行う）

---

## 17. 実装での簡潔なフローチャート（擬似）
1. 2点クリックで Tx/Rx を決定
2. 距離を計算して steps を決定
3. 各点で DEM から標高取得（キャッシュ利用）
4. 各点に対して:
   - heightProp（LOS + 曲率補正）を計算
   - F1 を計算
   - clearanceProp, F1Margin を計算
5. minF1Margin, maxF1BlockageRatio, keIndex を求める
6. FSPL を計算
7. blockagePercent に応じて追加損失を決定（smooth / knife-edge / simple）
8. totalLoss, rxPower を計算
9. 角度・グラフ用データを作成して表示

---

## 18. 改善提案（実務的注意）
- 標高欠損の扱い: 現在 0 で置換しているが、補間や再リクエストを行う方が安全
- F1 や v が 0 に近い場合の明示的なガード（ゼロ除算回避）
- Lee 近似の各パラメータに対する単調性チェックと境界処理
- 平滑球面大地損失は ITU-R の公式実装に置き換えることで精度を向上可能（ITU-R P.526 など）
- 大距離（数百 km）では大気屈折、地球曲率、地形変動等のより高度なモデルが必要
- API リクエストの並列化/レート制御とローカルキャッシュ（タイル化）導入で待ち時間を短縮
- 精度検証用に既知のケース（測定データ）と比較する単体テストの追加

---

## 19. 変数一覧（実装名 → 説明）
- EARTH_RADIUS_M, R: 地球半径（m）
- DEFAULT_PROFILE_STEPS: 最大サンプル点数（200）
- lastDistanceMeters: 最後に計算した距離（m）
- profileData: 最後に取得したプロファイル配列
- profileVectorSource: プロファイルの地図描画用ソース
- points: [Tx_lonlat, Rx_lonlat]（度単位配列）
- p.elevation: 各サンプル点の標高（m）
- p.distance: Tx からの距離（m）
- p.webMercatorCoord: 表示用座標（WebMercator）
- txHeightM, rxHeightM: アンテナ高（地上高 m）
- txPowerDbm, txGainDbi, rxGainDbi, txCableLossDb, rxCableLossDb: 電力/利得/損失（dB/dBm）
- K_FACTOR: 有効地球半径 K 値（無次元）
- freqMhz: 周波数（MHz）
- minF1Margin, maxF1BlockageRatio, keIndex: 上述の評価指標

---

## 20. 参考
- ITU-R P.526 (回折および遮蔽) — 詳細モデル
- Fresnel zone の定義（電波伝搬、回折の基礎）
- Haversine / Great-circle formulas（球面幾何）
- Lee の近似（ナイフエッジ回折の近似式についての実務的取り扱い）

---

以上が map.html に実装されている主要アルゴリズムと数式の詳細ドキュメントです。必要があれば、個別関数（例: calculateKnifeEdgeLoss, calculateSmoothEarthLoss, calculatePropagation など）ごとに式の導出過程や数値例、境界テストケース（ユニットテスト用）を追加した別ドキュメントを作成します。どの関数から詳細化したいか指示してください。