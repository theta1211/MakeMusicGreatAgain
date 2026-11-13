# 01. 実現性調査

対象: Android ネイティブの音楽制作アプリ(シンセ / ドラム / ギター等の音源 + Bluetooth キーボード入力)を、スマートフォンとタブレットの両方で動かす。

## 結論(サマリ)

| 項目 | 判定 | 要点 |
|---|---|---|
| 低レイテンシ音声出力 | ◎ 実現可能 | Google 公式の C++ ライブラリ **Oboe**(AAudio / OpenSL ES を自動で選ぶ)を使うのが定石。機種による差は大きい |
| シンセサイザー音源 | ◎ 実現可能 | 自前 DSP(C++)で減算合成シンセを実装。中位機種で 16〜32 音同時発音が目安 |
| ドラム音源 | ◎ 実現可能 | サンプル再生方式。CPU 負荷は小さい |
| ギター / ピアノ等の生楽器音 | ○ 実現可能 | サンプル(SoundFont / SFZ)か物理モデリング(Karplus-Strong)。音質とアプリサイズのバランスが課題 |
| Bluetooth MIDI キーボード | ○ 実現可能 | Android 6.0(API 23)以降は BLE-MIDI を標準 API(`android.media.midi`)で扱える。BLE 由来の 10〜20ms 程度の遅延・揺れは避けられない |
| Bluetooth PC 用キーボード(文字入力用) | ◎ 実現可能 | 通常の `KeyEvent` として届くので、キーを鍵盤に割り当てれば演奏できる |
| スマホ / タブレット両対応 | ◎ 実現可能 | Jetpack Compose + Window Size Class による画面サイズ別レイアウト |
| Bluetooth イヤホンへの音声出力 | △ 制約あり | A2DP は 150〜300ms 程度遅れるため、リアルタイム演奏には向かない。有線 / USB 出力を推奨し、アプリ内で警告を出す |

**総合判定: 実現可能。** 似た既存アプリ(FL Studio Mobile、Caustic、Roland Zenbeats、G-Stomper など)も同じ技術(NDK + 低レイテンシ API)で作られている。主なリスクは「機種ごとのレイテンシ差」と「BLE-MIDI の遅延」で、どちらも設計で対策する(後述)。

---

## 1. オーディオ出力(低レイテンシ)

### 1.1 使える API

| API | 特徴 | 採否 |
|---|---|---|
| `AudioTrack`(Java/Kotlin) | 手軽だが GC の影響を受けやすく、レイテンシが大きい | × |
| OpenSL ES(NDK) | 古い。Android 8.0 未満向けの低レイテンシ手段 | Oboe 経由で間接利用 |
| **AAudio**(NDK、API 26+) | 低レイテンシ。API 27 以降は MMAP(排他モード)が使える機種あり | Oboe 経由で利用 |
| **Oboe**(C++ ライブラリ、Apache 2.0) | AAudio / OpenSL ES を自動で選び、機種ごとの不具合回避も含む。Google 推奨 | **◎ 採用** |

### 1.2 レイテンシの実情

- Android の「Pro Audio」対応機種(`android.hardware.audio.pro`)は、往復レイテンシ 20ms 以下が要件。Pixel や一部 Galaxy 上位機が該当。
- `android.hardware.audio.low_latency` 対応機種なら、出力のみで 10〜30ms 程度が期待できる。
- 低価格機や一部メーカーの機種では 40〜100ms 程度かかることもある → **アプリ側で実測し、ユーザーに見せる**(設定画面の「レイテンシ測定」)。
- タッチ操作の遅延(タッチパネルのスキャン + 描画パイプラインで 20〜50ms)が音声遅延に上乗せされる。BLE-MIDI 入力でもタッチ入力でも、体感遅延の主な要因は入力経路の方。

### 1.3 レイテンシを最小にする実装方針

- `PerformanceMode::LowLatency` と `SharingMode::Exclusive` を指定する(使えない機種では自動で Shared に戻る)。
- バッファサイズはバースト(`framesPerBurst`)の 2 倍から始め、アンダーラン(音切れ)を検知したら段階的に大きくする(Oboe の `LatencyTuner`)。
- サンプリングレートは機種の出力ネイティブ値(多くは 48kHz)に合わせ、リサンプルを避ける。
- オーディオコールバック内では**メモリ確保・ロック・ログ出力・JNI 呼び出しをしない**。UI / MIDI スレッドとはロックフリーのキュー(SPSC リングバッファ)でやり取りする。
- CPU の省電力によるクロック低下を避けるため、Oboe の `setPerformanceHint` / Android 12 以降の `PerformanceHintManager`(ADPF)を使う。

## 2. 音源

### 2.1 シンセサイザー
- 減算合成(オシレーター 2 基 + ノイズ → フィルター(ローパス / ハイパス、レゾナンス付き)→ アンプ、ADSR エンベロープ 2 基、LFO)を C++ で実装する。
- エイリアシング(折り返しノイズ)対策として PolyBLEP 方式のオシレーターを使う。
- 負荷の目安: 中位の ARM コア 1 本で、1 音あたり 1〜3% 程度。32 音でも 1 コアに収まる見込み(PoC で計測する)。

### 2.2 ドラム
- 48kHz / 16bit のサンプル(ワンショット)を再生する方式。キック、スネア、ハイハットなど 1 キット 10〜16 音。
- 1 キット数 MB。CPU 負荷は小さい。

### 2.3 ギター / ベース / ピアノなど
| 方式 | 長所 | 短所 |
|---|---|---|
| SoundFont(SF2)/ SFZ のマルチサンプル | リアルな音。外部音色の読み込みにも使える | 容量が大きい(1 楽器 10〜100MB) |
| 物理モデリング(Karplus-Strong 等) | 容量ほぼゼロ、弦らしい自然な揺らぎ | リアルさに限界、調整が難しい |

→ **初期版はサンプル方式(SF2)を基本にし、アプリ同梱は軽量な音色に絞る**。大きい音色は Play Asset Delivery で後からダウンロードする。

- SF2 再生ライブラリ候補:
  - **TinySoundFont**(MIT、ヘッダーのみ、軽量)… 第一候補
  - FluidSynth(LGPL 2.1)… 高機能だが、LGPL の条件(動的リンク・差し替え可能にする等)への対応が必要
- **音色(サンプル)のライセンス**は要注意。CC0 / 商用利用可の素材か自前収録に限る。GM 音源として流通している SF2 にも商用不可のものがある。

## 3. Bluetooth キーボード

「Bluetooth キーボード」は次の 2 種類があり、**両方に対応する**(決定済み)。

### 3.1 BLE-MIDI キーボード(楽器用の MIDI 鍵盤)
- 例: KORG microKEY Air、Roland A-49(BT アダプター)、Yamaha MD-BT01 経由の鍵盤など。
- Android 6.0(API 23)以降、`MidiManager.openBluetoothDevice()` で BLE-MIDI 機器に接続できる。
- 接続手順: `BluetoothLeScanner` で MIDI サービス UUID(`03B80E5A-EDE8-4B33-A751-6CE34EC4C700`)を持つ機器を探す → `openBluetoothDevice()` → `MidiOutputPort` を開いてノート情報を受け取る。
- 権限:
  - Android 12(API 31)以降: `BLUETOOTH_SCAN`(`neverForLocation` 指定可)、`BLUETOOTH_CONNECT`
  - Android 11 以前: `ACCESS_FINE_LOCATION`(BLE スキャンに必要)
  - `CompanionDeviceManager` を使うと、位置情報の権限なしで機器選択ダイアログを出せる(代替案)。
- 遅延: BLE の接続間隔(7.5〜15ms 単位)により **10〜20ms 程度の遅延と揺れ**が出る。`BluetoothGatt.requestConnectionPriority(HIGH)` 相当で短くできるが、`MidiManager` 経由では直接操作できないため機種差がある。
- 受け取った MIDI イベントのタイムスタンプ(`System.nanoTime` 基準)を使い、オーディオ側でバッファ内の正しい位置で発音する(サンプル単位で正確に)ことで、揺れを減らす。
- Android 10(API 29)以降は NDK の **AMidi** で C++ から直接 MIDI を読めるため、JNI を経由せず遅延を減らせる。
- USB-MIDI(USB 鍵盤)も同じ `android.media.midi` で扱えるので、追加の手間はほぼない。

### 3.2 Bluetooth PC キーボード(文字入力用、HID)
- OS が通常のキーボードとして扱うため、`onKeyDown` / `onKeyUp` の `KeyEvent` で受け取れる。ペアリングは OS の設定画面で行う。
- 例: `A S D F G H J K` を白鍵、`W E T Y U` を黒鍵に割り当てる(DAW でよくある配置)。`Z` / `X` でオクターブ切り替え。
- ベロシティ(打鍵の強さ)はないため固定値にする。キーリピートは無視する(`event.repeatCount > 0` を捨てる)。

## 4. スマートフォン / タブレット両対応

- UI は **Jetpack Compose** で作り、**Window Size Class**(Compact / Medium / Expanded)でレイアウトを切り替える。
- スマホ縦: 1 画面 1 機能(タブ切替)。スマホ横: 鍵盤を広く表示。
- タブレット / 折りたたみ機: シーケンサー + 鍵盤 + ミキサーを同時に並べる 2〜3 ペイン構成。
- マルチタッチ(10 点以上)で和音演奏。`MotionEvent` のポインター ID ごとに音を管理する。
- 音声エンジンは画面サイズと無関係なので、共通化できる(違いは UI 層だけ)。
- タブレットは CPU に余裕がある機種が多い一方、低価格タブレット(低価格 SoC + 非低レイテンシ)もあるので、性能は**機種で判定**し、画面サイズでは判定しない。

## 5. その他の技術的事項

| 項目 | 内容 |
|---|---|
| バックグラウンド再生 | `foregroundServiceType="mediaPlayback"` の Foreground Service で再生を続ける(Android 14 以降は型の宣言が必須) |
| 音声フォーカス / 割り込み | 電話着信などで `AudioFocus` を失ったら停止する。ヘッドホンが外れたら(`ACTION_AUDIO_BECOMING_NOISY`)止める |
| 出力機器の切り替え | Oboe の `ErrorDisconnected` を検知してストリームを作り直す |
| 書き出し | オフラインレンダリング(実時間より速く処理)で WAV を書き出し、AAC/M4A は `MediaCodec` で変換。MIDI ファイル(SMF)の書き出し |
| 保存 | プロジェクトは JSON(+ 音色参照)。端末内 + SAF(Storage Access Framework)で外部へ書き出し |
| APK サイズ | AAB 本体 200MB まで。大きい音色は Play Asset Delivery(on-demand)へ |

## 6. リスクと対策

| リスク | 影響 | 対策 |
|---|---|---|
| 機種ごとのレイテンシ差が大きい | 低価格機で演奏しにくい | 起動時にバッファを自動調整し、レイテンシを実測して表示。推奨機種リストを公開 |
| BLE-MIDI の遅延・揺れ | 速い演奏で違和感 | タイムスタンプによるサンプル単位の発音、USB-MIDI を代替として案内 |
| Bluetooth オーディオ出力の遅延 | 演奏が成立しない | 出力先が A2DP の場合に警告表示。再生専用の用途では遅延を補正する |
| CPU 負荷(低価格機) | 音切れ | 同時発音数の上限を機種性能で自動設定、ボイススティール(古い音から止める) |
| 音色のライセンス | 公開できない | CC0 / 商用可素材 / 自前制作に限定し、出典を台帳で管理 |
| FluidSynth の LGPL | 配布条件が複雑 | TinySoundFont(MIT)を第一候補にする |
| 端末の発熱・スロットリング | 長時間で音切れ | ADPF(Performance Hint)の利用、無音時の処理停止 |

## 7. PoC(試作)で確認すべきこと

1. Oboe でのサイン波 / シンセ音の出力と、主要機種でのレイテンシ実測(Pixel、Galaxy 中位、低価格機、タブレット各 1 台以上)
2. BLE-MIDI 鍵盤の接続・ノート受信と、打鍵から発音までの遅延実測
3. PC キーボード(HID)による演奏
4. シンセ 32 音同時発音時の CPU 使用率
5. TinySoundFont による SF2 再生とメモリ使用量
6. Compose のマルチタッチ鍵盤のタッチ遅延
