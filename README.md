# map.html 取り扱い説明書

最終更新: 2025-10-18

このドキュメントは、リポジトリ内の map.html （電波伝搬計算ツール）の使い方、内部構造、主要パラメータの説明、トラブルシューティング、開発メモをまとめたものです。主にエンジニアや無線屋さんがブラウザ上で地形プロファイルに基づく伝搬損失を簡易評価できるように設計されています。

---

## 目的
- 地理院（GSI）のDEM標高データを用いて2点間の標高プロファイルを取得する
- フレネル第1ゾーン、LOS判定、ナイフエッジ回折（簡易）などを考慮して伝搬損失を評価
- Chart.js によるプロファイル（標高）グラフとレベルダイヤグラムを表示
- 地図（OpenLayers）上で Tx/Rx を選択して即座に再計算

---

## 必要条件（実行環境）
- モダンブラウザ（Chrome / Edge / Firefox 等）
- インターネット接続（CDNライブラリ、地理院タイル、地理院標高APIへアクセスするため）
- 推奨：ローカル開発サーバ（file:// で開くとブラウザのポリシーで fetch が動かない場合があるため。例: `python -m http.server`）

外部ライブラリ（CDN経由で読み込まれます）:
- OpenLayers (ol) v7.3.0
- proj4js 2.7.5
- mgrs 2.1.0
- Chart.js 4.4.0
- chartjs-plugin-annotation

利用API:
- 地理院標高API: https://cyberjapandata2.gsi.go.jp/general/dem/scripts/getelevation.php
- 地理院タイル（地図画像）: https://cyberjapandata.gsi.go.jp/xyz/...
- OSM（代替タイル）

注意: 外部CDNやAPIの変更・停止により動作しなくなる可能性があります。

---

## 起動／基本操作
1. map.html をウェブサーバでホストする（例: リポジトリをクローンして `python -m http.server`）。
2. ブラウザで http://localhost:8000/map.html を開く。
3. 左上の「伝搬計算設定」パネルでパラメータを必要に応じて設定：
   - 周波数プリセットボタンで素早く周波数を設定
   - K-ファクター（地球曲率等価）選択
   - Tx / Rx のアンテナ高、利得、ケーブル損失、送信出力（dBm）
4. 地図上をクリックして 1 点目（Tx）→ 2 点目（Rx）の順に指定。
   - 1点目をクリックすると「Tx」点が設定され、2点目でプロファイル取得と計算が走ります。
   - 設定を変更後は「設定で再計算」ボタンで再計算できます（既にプロファイルが取得済みなら再計算のみ）。
5. 画面に「伝搬プロファイルと結果サマリー」ウィンドウが表示され、グラフ・表で結果を確認できます。

（注：クリックの挙動・リセットボタンは map.html の後半に記述されたイベント処理に依存します。UI上の説明文も案内します。）

---

## UI 要素と意味
- 周波数プリセット: 30 MHz / 300 MHz / 3 GHz / 10 GHz の短縮ボタン
- K-ファクター: 1.0（幾何学的LOS）、1.333（4/3標準）、0.8（悪条件）など
- 周波数（MHz）: メインの周波数入力（getSetting('freq-mhz') で取得）
- Tx / Rx パラメータ:
  - Tx 出力 (dBm) → 隣に W 表示（updateTxPowerWatts が変換）
  - アンテナ高（m）、利得（dBi）、ケーブル損失（dB）
- 地図切替:
  - 地図タイプ（標準 / 写真）
  - タイルソース（地理院 / OSM）
- 「設定で再計算」ボタン: 既存のプロファイルデータを使って再計算（再描画）
- プロファイルポップアップ: 標高グラフ（上）、レベルダイヤグラム（下）を表示

---

## 計算の流れ（概要）
1. getElevationProfile(coord1, coord2, maxSteps)：2点間を分割し、各分割点の座標から地理院標高APIで標高を順次取得。
   - ステップ数決定ロジック: 距離に応じて間隔を調整（短距離は細かく、長距離は間隔を広げる）。
   - 取得した点は profileData に保存・地図にプロットされる。
2. calculatePropagation(profile, txHeightM, rxHeightM, K_FACTOR, freqMhz, ...)：取得プロファイルを使って伝搬特性を評価。
   - 波長 lambda を周波数から計算
   - 各プロファイル点で LOS ライン、高さ補正（地球曲率、K-factor）を計算
   - 第1フレネルゾーン半径 F1 を計算し、F1 余裕（F1Margin）や遮蔽率を算出
   - 損失モデルの選択:
     - F1遮蔽が小さい（<=10%）: FSPL のみ
     - 10%〜40%: 平滑球面大地損失（calculateSmoothEarthLoss）
     - 40%〜80%: ナイフエッジ回折（calculateKnifeEdgeLoss）を試行
     - >80%: 簡易的な遮蔽損失モデル（深さに応じた線形近似）
   - 受信電力 rxPowerDbm を算出（Tx出力 + Tx利得 + Rx利得 - 総損失 - ケーブル損失）
   - 仰角 / 俯角、方位角も計算
3. 表示:
   - displayMeasurementResults() でサマリをHTMLに挿入（距離、LOS/F1状況、損失、受信電力など）
   - renderChartsInPopup() で Chart.js により標高プロファイルとレベルダイヤグラムを描画

---

## 主な定数・設定（ソース内）
- ELEVATION_API_URL: 地理院標高API
- DEFAULT_PROFILE_STEPS: 200（最大分割数）
- API_TIMEOUT_MS: 10000（ms） — fetch のタイムアウト
- EARTH_RADIUS_M: 地球半径 6371000 m
- DEFAULT_TX_LONLAT / DEFAULT_RX_LONLAT: 起動時のデフォルト座標
- PIXEL_RATIO, MIN_CHART_WIDTH/HEIGHT: グラフ描画サイズ関連

---

## 主要関数（開発者向け簡易リスト）
- getSetting(id, defaultValue): コントロールの数値を安全に取得（parseFloat）
- updateTxPowerWatts(): dBm を W/mW/kW 表示に変換
- convertToUtm(), convertToMGRS(): 座標系変換（proj4, mgrs ライブラリ）
- ol.sphere.getDistance(), ol.sphere.interpolate(): 大圏距離・補間（ファンクション追加）
- getElevation(coord): 地理院APIで単点の標高を取得（キャッシュ使用）
- getElevationProfile(coord1, coord2, maxSteps): 2点間標高プロファイル取得（main）
- calculateKnifeEdgeLoss(v): Lee 近似式ベースのナイフエッジ損失計算
- calculateSmoothEarthLoss(distanceMeters, frequencyMhz, K_FACTOR): 簡易平滑地モデル
- calculateFSPL(distanceMeters, frequencyMhz): 自由空間損失
- calculatePropagation(profile, ...): 伝搬計算のコア関数（LOS判定／損失選択／結果オブジェクト生成）
- displayMeasurementResults(propResults): UI サマリの更新
- renderChartsInPopup(propResults, profile): Chart.js によるプロット
- processMeasurement(): 全体の流れを制御するエントリ（プロファイル取得 → 計算 → 描画）
- executeRecalculation(): 既存 profileData を使って再計算
- resetMeasurement(): 状態のリセット
- setupResizeObserver(), executeResizeLogic(), triggerChartResize(): Chart リサイズ管理

---

## 出力（見方の説明）
- 計測・座標情報:
  - Tx / Rx の緯度経度（表記は lat, lon）と MGRS/UTM 表示
  - 方位角（Tx→Rx / Rx→Tx）
- 伝搬結果サマリー:
  - 距離 (km)
  - LOS 判定（緑 = LOS、赤 = NLOS）
  - F1 ゾーン遮蔽率（%）と F1 関連メッセージ
  - 仰角 / 俯角（Tx・Rx）
  - 空間損失（FSPL）と総損失、損失モデル名
  - 受信電力（dBm）
- 標高プロファイル:
  - 地形標高（DEM）、LOS ライン、F1 上下ラインを表示
- レベルダイヤグラム:
  - 送信出力 → ケーブルロス → EIRP → 空間損失 → Rx利得 → 受信出力 の段階を可視化

---

## キャッシュ・パフォーマンス
- elevationCache（Map）で同一座標は再フェッチを避ける
- プロファイル点の数は距離に応じて自動調整（短距離は細かく、長距離は間引き）
- API_TIMEOUT_MS は 10 秒。長い距離で多数リクエストが発生した場合は時間がかかることがある

---

## トラブルシューティング
- 標高データが取得できない / プロファイルが途中で止まる:
  - ネットワークや地理院APIのアクセス制限・CORSを確認
  - API_TIMEOUT_MS を延長してみる
- グラフが表示されない:
  - ブラウザのコンソールにエラーが出ていないか確認（Chart.jsのエラー、canvasサイズ）
  - ポップアップが非表示になっている場合は processMeasurement や renderChartsInPopup の引数を確認
- 受信電力が非常に小さい／大きい:
  - 設定値（Tx出力、利得、ケーブル損失、周波数）を再確認
  - 選択したモデル（FSPL / 回折 / 平滑地）と遮蔽深さにより結果が大きく変わる
- 地図のクリックで点が設定されない:
  - map.html のイベントリスナー（ファイル後半）を確認（クリックイベントが存在するか）
- 403/404 エラー:
  - タイルURLやAPI URLが正しいか、またはアクセス制限がないか確認

---

## 既知の制限と改善案
短所（既知点）
- 回折モデルは簡易実装（Lee 近似・簡易遮蔽）であり、ITU-R P.526 等の完全実装ではない
- F1 遮蔽率の計算は簡易で境界条件で誤差が出る可能性がある
- 標高APIへの多数リクエストで速度低下やレート制限を受ける可能性
- UI の一部（例: リセットボタン、2点クリックの明示的指示）が改善余地あり

改善提案
- ITU-R P.526 等の公式式の導入（より正確な回折・多重モード対応）
- 標高キャッシュを IndexedDB に保存して再利用
- プロファイル点を一括API（バッチ）で取得する手法の検討（APIが対応すれば）
- エラーメッセージのユーザーフレンドリー化（詳細ログ表示のトグル）
- 複数 Rx の一括解析、CSV/PNG 出力機能追加
- 単体テスト・ユニットテストの追加（計算関数の検証）

---

## カスタマイズ方法（よくある変更）
- 使用するDEM APIを変更する:
  - ELEVATION_API_URL 定数を書き換え
- 既定の分割数や最大ステップ数の変更:
  - DEFAULT_PROFILE_STEPS を変更
- Chart.js の見た目変更:
  - renderChartsInPopup() 内の datasets/options を編集
- タイル提供元の変更:
  - gsiStdLayer / gsiPhotoLayer / osmLayer を差し替え

---

## 開発者メモ（重要箇所と注意）
- ol.sphere.getDistance と interpolate を独自実装しているため、OpenLayers の API 変更に依存しづらいが、将来的には ol.Sphere 等のネイティブAPIを使うほうが好ましい。
- getElevationProfile() は profileVectorSource をクリアしてから点を追加します。大量の点でパフォーマンス劣化するので、可視化の間引き等を検討すると良いです。
- calculatePropagation() は profileData を直接参照し、p.F1 や p.F1Margin などの追加フィールドを profile に埋め込みます。profileData を変更しない前提の外部処理では注意してください。
- グラフのリサイズは ResizeObserver と debounced タイマーで行われます。古いブラウザでは動作しない場合があります。

---

## ライセンスと帰属
- 地理院タイルおよび地理院標高APIを利用しています。利用規約に従ってください（表示や転載条件等）。
- OpenLayers、proj4、mgrs、Chart.js 等のライブラリライセンスに従って利用してください。

---

## 付録: 主要デフォルト値（ソース参照）
- DEFAULT_PROFILE_STEPS = 200
- API_TIMEOUT_MS = 10000 (ms)
- EARTH_RADIUS_M = 6371000
- DEFAULT_TX_LONLAT = [138.68909, 35.36277]
- DEFAULT_RX_LONLAT = [138.76812, 35.36345]
- 初期周波数 = 1000 MHz
- Tx 初期出力 = 30 dBm, Tx/Rx gain = 15 dBi, ケーブル損失 = 1.5 dB, 高さ = 30 m

---

必要なら、次のような追加ドキュメントを作成できます（希望があれば実装します）:
- API応答例を含むデバッグフロー
- 詳細なアルゴリズム数式ドキュメント（Fresnel, Lee 近似の導出）
- UI のスクリーンショット付きユーザーガイド
- テストケース（単体テスト・期待値テーブル）
