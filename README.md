# test10 EA v3.09

MT5用のマルチタイムフレームEAです。本書は、リポジトリにある`test10`の現行コードが実際に行っている処理を説明します。コード修正時の作業ルールは`.github/copilot-instructions.md`で管理します。

## 1. 処理の構成

- H1: ADX、DI+、DI-、EMA20、EMA50で相場環境、方向、TrendStateを判定する。
- M15: RSI、MACD Main、MACD Signal、MACD Histogram、MACDクロスを確定足で計算する。売買で使うM15フィルタはRSIのみ。
- M1: Bollinger Bands、Stochastic、ATR、独自計算のSuperTrendでエントリー・決済価格を計算する。
- OnTimer: 新しいH1/M15/M1足の検出、指標取得、判定、価格・実行フラグの更新を行う。
- OnTick: ポジション数の集計、決済、トレーリングSL、エントリーを現在価格で実行する。

## 2. 足番号とデータ取得

`CopyBuffer`等で取得した配列は、`CopyValuesToSeries`で反転し、EA内部では次の意味で扱います。

- `[0]`: 現在形成中の未確定足
- `[1]`: 直近の確定足
- `[2]`: 2本前の確定足
- `[3]`: 3本前の確定足

H1は4本、M15とM1は3本を取得します。H1のTrendStateは確定済みの`[3]`、`[2]`、`[1]`を比較します。M15のMACDクロスも`[2]`と`[1]`だけを比較し、`[0]`は使用しません。

将来用のダイバージェンス計算関数では、M15の価格と指標を14本取得して時系列順に並べ替えます。ただし、現バージョンではその関数を呼び出していません。

## 3. H1の相場分類

ADX、DI差、EMAの位置関係から次を分類します。

- `TREND_RANGE`: ADXが18未満
- `TREND_STRONG_UP` / `TREND_STRONG_DOWN`: ADXが40以上かつDI方向が一致
- `TREND_WEAK_UP` / `TREND_WEAK_DOWN`: ADXが25以上かつDI方向が一致
- 上記以外: `TREND_UNKNOWN`

DI方向はDI差が12以上の場合だけ、DI+優位を上昇、DI-優位を下降とします。EMA20とEMA50の位置関係は方向情報として保持し、強トレンドの追撃判定に使います。

確定ADX `[3]`、`[2]`、`[1]` の推移から、`TREND_ACCELERATE`、`TREND_DECELERATE`、`TREND_REACCELERATE`、`TREND_RECOVERY_MAINTAIN`、`TREND_STALL_MAINTAIN`、`TREND_MAINTAIN`、`RANGE_MAINTAIN`、`RANGE_TO_TREND`、`TREND_TO_RANGE`、`STATE_UNKNOWN`を分類します。

## 4. M15の判定と追撃フィルタ

RSI期間は14、MACDは12/26/9です。MACD Histogramは`Main - Signal`で計算し、クロスは確定足で判定します。RSIが70以上なら買われすぎ、30以下なら売られすぎとして状態を保持します。

M15更新時に作成するシグナルは、直後のM1足でのみ追撃判定に影響します。

- BUY追撃: RSIが70未満なら禁止、70以上なら許可
- SELL追撃: RSIが30より大きければ禁止、30以下なら許可
- M15シグナルが保留中でない場合、RSI追撃フィルタは許可扱い

MACDクロスは計算・ログ出力しますが、現行の追撃許可判定には使いません。M15ダイバージェンスも売買判定には使いません。

## 5. M1の判定

M1の設定はBB期間20・偏差2、Stochasticの`K=5`、`D=3`、slowing=3、ATR期間14です。SuperTrendのATR倍率は3.0、Pullback判定幅は0.3 ATR、Reacceleration判定幅は0.2 ATRで、いずれも内部変数です。

### レンジ

- BUYエントリー価格: BB下限
- SELLエントリー価格: BB上限
- BUY決済価格: BB上限
- SELL決済価格: BB下限
- BUYはRSIが30以下、SELLはRSIが70以上であることが必要
- 価格がいったんエントリー閾値を通過した後、閾値側へ戻ったときに発注する

### トレンド

H1がレンジまたは不明でない場合、SuperTrendの方向、反転、Pullback後の再加速を使います。通常の初撃は同一方向1ポジションまでです。強い上昇・下降トレンドで、H1のDI方向とEMA方向が一致し、SuperTrendの再加速がある場合に限り、既存ポジションへの追撃を許可します。追撃を含む同一方向の上限は3ポジションです。

## 6. 決済、SL、ロット

- レンジの通常決済は、BUYがBB上限、SELLがBB下限への到達です。
- トレンドの決済価格はSuperTrendです。Stochastic条件によりBB上限またはBB下限へ置き換わる場合があります。
- 注文時のSLは現在価格から`SL_Points=40`ポイントです。
- TPはレンジ注文だけに設定し、SL幅×`RiskRewardRatio=2.0`です。トレンド注文のTPは0です。
- トレーリングSLは、H1とM1が有効なトレンド状態でSuperTrend値を候補にします。現在SLより有利で、`MinSLUpdatePoints=1`ポイント以上離れている場合に更新します。方向確認にはコード上`g_superTrendDir[0]`を使用します。
- 基本ロットは`BaseLot=1.0`です。レンジは0.7倍、トレンド初撃は1.0倍、強トレンド追撃は0.5倍です。シンボルの最小値・刻みに合わせて丸めます。
- `MaxLot`と`RiskPercentEquity`は現行コードでロット計算に使用していません。

## 7. 発注とポジション管理

対象シンボル・Magic Numberが一致するポジションだけを集計・操作します。発注前に最大スプレッドを確認し、`MaxAllowedSpread=25`ポイント以上ではOnTickのエントリーを中止します。注文送信はFOK、許容偏差10で行います。

同一方向では、直近のM1確定足で発注した場合や、同じM1足で決済した場合の再発注を抑制します。H1がレンジでない場合は、エントリー方向とH1のDI方向が一致している必要があります。

## 8. リクエスト履歴とログ

注文送信、成行決済、SL/TP変更について、直近60秒のリクエストを時刻順に保持します。評価は「送信失敗」「実行失敗」「実行成功」に分け、履歴件数が内部閾値20件を超えたときに各件数と総数をログ出力します。

H1、M15、M1の更新結果、指標取得失敗、判定値、注文、決済、SL更新、OnTrade、OnTradeTransactionを`Print`/`PrintFormat`で出力します。取引イベントの重複ログを抑止する処理はありません。

## 9. 未実装・将来実装予定

- RSIおよびMACD Histogramのダイバージェンス判定は、enum・補助関数・計算関数を保持していますが、現バージョンでは呼び出していません。
- ダイバージェンスをエントリー、追撃、決済の条件に追加する仕様は未確定です。
- `MaxLot`による上限管理と`RiskPercentEquity`による資金連動ロット計算は未実装です。
