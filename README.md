# test10 EA

MT5用のマルチタイムフレームEAです。
相場環境・トレンド方向・エントリー条件などの判定は確定足を中心に行い、発注・決済はOnTickで現在価格を使用して実行します。

## 時間足の役割

* H1はADX、DI+、DI-、短期EMA、長期EMAから相場環境、方向、トレンド状態を判定します。
* M15はRSI、MACD Main、MACD Signal、MACD Histogram、MACDクロスを計算します。追撃許可・抑制に使用する指標はRSIです。
* M1はATR、Bollinger Bands、Stochastic、SuperTrendからエントリー・決済タイミングを判定します。
* OnTimerは新しい確定足を検出し、M1、M15、H1の順に各時間足を更新します。
* OnTickは現在価格を使用したクロス検知、発注、決済、SL更新などを実行します。

内部の配列では、`[0]`を形成中の未確定足、`[1]`を直近の確定足、`[2]`以降を過去の確定足として扱います。確定足を使用する判定では未確定足を使用しません。

指標およびM1価格系列は25本をSeries配列として取得します。価格系列は取得結果が25本未満でも、存在する有効な要素を使用します。指標取得は`CopyBuffer()`で最大3回試行し、失敗時のみ100ms待機します。取得に失敗した指標は次回OnTimerで再取得し、時間足全体の取得成功まで売買判定を実行しません。

## H1判定

ADXの強弱、DI方向、DI差、短期EMAと長期EMAの位置関係から、以下の相場環境を判定します。

* 強い上昇トレンド
* 弱い上昇トレンド
* 強い下降トレンド
* 弱い下降トレンド
* レンジ
* 不明

ADXの確定足3本の推移から、トレンドの加速、減速、再加速、維持、レンジとの移行などの状態も判定します。

H1のDI方向をトレンド売買の方向判定に使用します。

## M15判定と追撃抑制

RSIは70以上を買われすぎ、30以下を売られすぎとして扱います。

MACDのMain、Signal、Histogramおよびクロスは計算・ログ出力しますが、現時点では追撃の許可・禁止には使用しません。

強トレンドの追撃は、M15のRSIが以下の条件を満たす場合に許可します。

* BUY：RSIが70以上
* SELL：RSIが30以下

M15の新しい確定足でRSIを確認した直後のM1足では、RSI条件を満たさない方向の追撃を抑制します。

この追撃抑制は、M15確定直後のM1足だけを対象とします。そのM1足が終了して次のM1足へ移行した時点で解除します。

M15確定から次のM1足へ移行した後は、次のM15確定まで追撃抑制を継続しません。

## M1とSuperTrend

M1では以下の指標を使用します。

* Bollinger Bands：期間20、偏差2
* Stochastic：K=5、D=3、スローイング=3
* ATR：期間14
* SuperTrend：ATR期間14、ATR倍率3.0

### SuperTrend

SuperTrendはM1の確定足を使用して計算します。

起動時には過去50本のM1履歴を古い足から新しい足へ順番に処理し、直近の確定足までSuperTrendを初期計算します。

初期化に失敗してもEAを停止せず、次回の更新処理で再試行します。

初期化完了後は、保存している直近のSuperTrend計算状態を使用し、新しく確定したM1足だけを追加計算します。毎回M1履歴全体を再計算することはしません。

SuperTrendは、価格とATRから基本バンドを計算し、前足の状態を引き継いでFinal Upper BandとFinal Lower Bandを更新します。

SuperTrendの方向は、現在のFinal Bandと前足からのトレンド状態に基づいて判定します。

* 上昇方向ではLower BandをSuperTrendとして使用します。
* 下降方向ではUpper BandをSuperTrendとして使用します。
* 価格がSuperTrendの反対側へ抜けた場合、トレンド方向を反転します。

M1更新時に計算したSuperTrend値と方向は、次のM1更新まで保持します。

SuperTrendの計算値そのものを毎Tickで変更することはしません。

### Pullback

Pullbackは、トレンド方向に対して価格が逆方向へ押し戻されたことを検知します。

SuperTrendからの距離はATRを基準とし、Pullback判定幅は0.3 ATRです。

BUYの場合は、現在ASKがSuperTrendより下側のPullback判定ラインを上から下へクロスした場合にPullbackを検知します。

SELLの場合は、現在BIDがSuperTrendより上側のPullback判定ラインを下から上へクロスした場合にPullbackを検知します。

Pullbackは単なる価格位置ではなく、前回Tickと現在Tickの価格関係によるクロスとして判定します。

### Reacceleration

Reaccelerationは、Pullback発生後に価格が再びトレンド方向へ戻ったことを検知します。

SuperTrendからの距離はATRを基準とし、Reacceleration判定幅は0.2 ATRです。

BUYの場合は、Pullback発生後、ASKがReacceleration判定ラインを下から上へクロスした場合にReaccelerationを検知します。

SELLの場合は、Pullback発生後、BIDがReacceleration判定ラインを上から下へクロスした場合にReaccelerationを検知します。

Pullbackが発生していない状態ではReaccelerationを成立させません。

PullbackとReaccelerationは同一M1足内の状態として扱います。

新しいM1足へ移行した時点で、前のM1足のPullback状態およびReacceleration状態はリセットします。

そのため、あるM1足でPullbackが発生しても、次のM1足へ移行した後にReaccelerationが発生した場合は、前のM1足のPullbackを引き継いで追撃することはありません。

## エントリー

決済シグナルと同方向の新規エントリーシグナルが同時に成立した場合は、`EnableEntryOnCloseSignal` でエントリー許可を切り替えられます。初期値は`true`です。`false`の場合は同方向の新規エントリーだけを抑制し、逆方向のエントリーは抑制しません。

### レンジ

レンジ相場では、価格がBollinger Bandsの外側へ一度抜けた後、バンド側へ戻ったことを利用してエントリーします。

BUYでは以下を満たす必要があります。

* BUY方向のBBリトレース条件
* RSIが30以下
* BB幅が最低SL幅の1.5倍以上
* 最大許容スプレッド以下

SELLでは以下を満たす必要があります。

* SELL方向のBBリトレース条件
* RSIが70以上
* BB幅が最低SL幅の1.5倍以上
* 最大許容スプレッド以下

スプレッド条件とBB幅条件は別々に判定し、両方を満たした場合のみレンジエントリーを許可します。

### トレンド

トレンド相場の新規エントリーは`EnableTrendTrading`で、レンジ相場の新規エントリーは`EnableRangeTrading`で個別に有効・無効を切り替えられます。これらの設定は既存ポジションの決済に影響しません。

トレンド時の新規エントリーは、押し目→再加速、初動ブレイク、トレンド継続の3種類に整理し、優先順位は押し目→再加速 > 初動ブレイク > トレンド継続とします。いずれもH1トレンドとM1 SuperTrendの方向性が一致した場面で評価されます。

- 押し目→再加速
  - 既存のPullback/Reacceleration判定を維持し、`g_superTrendReaccelBuy/Sell`をエントリー条件として利用する。
  - `sameDirCount == 0` でも成立し得るが、M15追撃抑制は「同一方向ポジションが存在する場合」にのみ適用する。
- 初動ブレイク
  - H1のトレンド方向とDI/EMA方向、M1 SuperTrend方向が一致し、BB幅/ATRが`initialBreakoutMinBBATR`以上のときに評価する。
  - `initialBreakoutLookback = 20`の場合、M1価格更新時に[2]～[21]の範囲（データ不足時は存在する範囲）から高値/安値を再計算する。計算可能な場合、現在ASKが最高値を上回るか、現在BIDが最安値を下回ると成立する。BB中央線からの距離は`initialBreakoutMaxExtensionATR`以内とする。
  - 現在価格で即時判定し、ブレイク維持や複数Tickの確認は行わない。
  - SuperTrend反転そのものはエントリー条件に使用しない。
- トレンド継続
  - 強い一方向トレンドでBB幅が大きく、価格がBB中央線付近に留まりながら高値圏/安値圏を維持している場合に成立する。

ロットサイズはシグナル種類ではなく、同一方向の現在ポジション数で決定します。

- 同一方向ポジション数 = 0 → `BaseLot × 1.0`
- 同一方向ポジション数 >= 1 → `BaseLot × 0.5`

通常エントリーは同一方向最大1ポジション、押し目→再加速による追撃は最大3ポジションまでを維持します。

## 決済

### レンジ

通常決済は、以下を使用します。

* BUY：価格がBB上側へ到達
* SELL：価格がBB下側へ到達

レンジBB逆側決済、Stochastic決済、レンジTPは入力パラメータで個別に有効・無効を切り替えられます。
Stochastic決済は、BUYでは`K[2] > 80`からのクロスダウン、SELLでは`K[2] < 20`からのクロスアップに限定します。過熱域からKが反転する既存条件も維持します。

### トレンド

通常決済にはSuperTrendを使用します。

緊急決済として、以下をOR条件で使用します。

* Stochasticクロス
* 過熱圏からの反転
* BB中帯の突破

既存のSuperTrendによる決済条件は維持し、Stochastic、SuperTrend、BB中央線は入力パラメータで個別に有効・無効を切り替えられます。同一Tickで複数条件が成立した場合は、成立した全条件を決済理由として記録します。

## SL・トレーリング

新規ポジションの初期SLは、発注時のM1確定ATRを使用し、距離を`max(ATR × 1.5, SL_Points × _Point)`として設定します。設定後はATRの変化だけを理由に初期SLを再計算しません。

トレンドポジションでは現在のM1確定ATRを使用し、`max(ATR × 1.5, SL_Points × _Point)`の距離でATRトレーリングを行います。BUYのSLは上方向、SELLのSLは下方向にのみ更新し、レンジポジションにはATRトレーリングを適用しません。

候補SLが現在SLより有利で、かつ最低更新幅以上離れている場合にSL更新を試行します。

不利な方向へのSL更新は行いません。

## リクエストと取引ログ

発注、決済、SL/TP変更のリクエストは、送信試行時刻を記録し、直近60秒分を保持します。

送信失敗、実行失敗、実行成功、および直近60秒のリクエスト数を管理します。

直近60秒のリクエスト数が20件を超えた状態でリクエストが発生した場合はログを出力します。

OnTradeは空のイベントとして残します。

## 解析CSV

バックテストごとに `EA_Analysis.csv` を新規作成し、MT5 Strategy Testerの通常のFiles領域に保存します。1行目は共通ヘッダーで、全レコードを同じ列構成にします。既存ファイルはテスト開始時に上書きします。

`RecordType` は `M1_UPDATE`、`M15_UPDATE`、`H1_UPDATE`、`ENTRY_SIGNAL`、`ENTRY`、`ENTRY_SUPPRESSION`、`EXIT_SIGNAL`、`EXIT`、`SL_UPDATE`、`POSITION_SUMMARY`、`WEEKEND`、`DEAL`、`OTHER_ANALYSIS`、`ATR2_EXTENSION_TICK` を使用します。各レコードで該当しない列は空欄です。

CSVヘッダーは次の固定列です。

```text
RecordType,DateTime,TimeMsc,Symbol,Direction,Price,ExitPrice,Bid,Ask,SpreadPoints,Lot,Ticket,Order,Deal,Position,Reason,H1BarTime,H1TrendType,H1TrendState,H1DIDirection,H1EMADirection,ADX,DIPlus,DIMinus,EMAShort,EMALong,M15BarTime,M15RSI,M15MACDMain,M15MACDSignal,M15MACDHistogram,M15MACDCross,M1BarTime,M1BBUpper,M1BBMiddle,M1BBLower,M1StochK,M1StochD,M1ATR,M1SuperTrend,M1SuperTrendDirection,EntrySignal,CloseStochastic,CloseTrendBBMiddle,EntryPrice,SL,TP,ElapsedSeconds,SameDirectionCount,LotCalculation,Balance,Equity,FreeMargin,RequiredMargin,ReasonDetail,ATR2Boundary,ATR2Extension,ATR2TickIndex,ATR2ElapsedMs,ATR2EventId,ATR2InsideBoundary,EntryExecuted,MagicNumber,Retcode,Comment,PriceDifference,RequestedPrice,ExecutionPrice,DealReason
```

確定足の指標、H1判定値、発注・決済の成功内容、IN/OUT Deal情報、決済条件、発注資金確認、ポジション数、週末処理、OnTimerが設定したENTRY/EXIT価格をCSVへ記録します。指標・価格取得失敗、無効値、発注・決済・SL更新失敗などの原因確認用メッセージは、従来どおりMT5通常ログへ残します。

| `RecordType` | 主な出力列 |
| --- | --- |
| `M1_UPDATE` | `M1BarTime`, `M1ATR`, `M1BBUpper/Middle/Lower`, `M1StochK/D`, `M1SuperTrend/Direction` |
| `M15_UPDATE` | `M15BarTime`, `M15RSI`, `M15MACDMain/Signal/Histogram/Cross` |
| `H1_UPDATE` | `H1BarTime`, `H1TrendType/State`, `H1DIDirection`, `H1EMADirection`, `ADX`, `DIPlus/Minus`, `EMAShort/Long` |
| `ENTRY_SIGNAL` | `Direction`, `EntrySignal`, `SameDirectionCount`, `Lot`, `Bid/Ask`, `SpreadPoints`, `EntryPrice` |
| `ENTRY` | `RequestedPrice`, `ExecutionPrice`, `Lot`, `SL`, `TP`, `MagicNumber`, `Order`, `Deal`, `Position`, `Retcode`, `Comment` |
| `ENTRY_SUPPRESSION` | `Direction`, `EntryPrice`, `EntrySignal`, suppression reason |
| `EXIT_SIGNAL` | `Direction`, `Price`, stochastic/BB-middle conditions, reason, M1 indicator snapshot |
| `EXIT` | `Ticket`, `Position`, `EntryPrice`, `ExitPrice`, `Lot`, `SpreadPoints`, `PriceDifference`, `Reason`, `Retcode`, `Comment`, `Deal`, `Order` |
| `SL_UPDATE` | `Ticket`, `Position`, `Direction`, `Price`, new `SL`, `Retcode`, `Comment` |
| `POSITION_SUMMARY` | position counts and summary reason |
| `WEEKEND` | weekend state changes and force-close target snapshots |
| `DEAL` | IN/OUT deal fields, deal reason, indicator snapshot and elapsed seconds when available |
| `OTHER_ANALYSIS` | margin checks, request counts, OnTimer ENTRY/EXIT prices and other state details |
| `ATR2_EXTENSION_TICK` | per-tick quote, 2ATR boundary/extension, event ID/index/elapsed time, inside-boundary flag and entry status |

`ATR2_EXTENSION_TICK` は、BUY/SELL別に2ATR境界を初めて超えたTickから、境界内へ戻ったTickまでを記録します。イベント開始Tickの `ATR2TickIndex` は0で、境界内への復帰Tickも記録されます。`ATR2EventId` でイベントを識別し、`ATR2ElapsedMs` と `TimeMsc` でTickの経過時間・順序を確認できます。この記録追加は解析専用であり、2ATR条件を含む売買判定を変更しません。

MT5のイベント仕様により、同一取引について `DEAL` 解析レコードとIN/OUTレコードの複数行がCSVに出力される場合があります。

指標とM1価格系列の取得失敗は、ハンドル・時間足・必要本数・取得本数・無効値・エラーコードをMT5通常ログに出力します。発注、決済、SL更新の成功情報はCSVに、失敗情報はMT5通常ログに記録します。取引イベントではEA取引、SL、TPをDeal reasonで区別します。

取引解析CSVの価格、ロット、時刻、注文、ポジション、Magicは、`HistoryDealSelect()`で選択した履歴Deal情報を使用します。実約定時刻は`DEAL_TIME_MSC`を基準にし、INからOUTまでの保持時間も実約定時刻の差で計算します。`DEAL_ENTRY_INOUT`や`DEAL_ENTRY_OUT_BY`などの特殊なEntryは通常のIN/OUT保持時間計算には使用しません。

## 週末休場

`SymbolInfoSessionTrade()`からサーバー側の取引セッションを曜日・セッション番号ごとに取得します。現在セッションの終了から次回セッション開始までが24時間以上の場合だけ週末対象とし、終了3時間前から新規発注を停止し、終了10分前からこのEAの対象銘柄・Magicの全ポジションを損益に関係なく強制決済します。次回セッション開始後に停止状態を解除します。通常夜間の発注は時刻では停止せず、既存のスプレッドフィルターを使用します。

## ロット

基本ロットを基準として、以下の相場・エントリー種別ごとにロット倍率を設定します。

* レンジ
* トレンド初撃
* 強トレンド追撃

最大ロット制限および資金量連動ロット計算は現時点では使用しません。

将来的に固定方式と変動方式を選択できるようにする予定です。

## Divergence

Divergenceは将来実装予定です。

判定方法は現在検討中で、現時点では売買判定に使用しません。

将来の実装に接続できるよう、必要な計算構造は保持します。
