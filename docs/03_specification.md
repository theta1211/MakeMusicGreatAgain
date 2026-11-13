# 03. 仕様設計

## 1. 技術スタック

| 層 | 採用技術 |
|---|---|
| 言語 | Kotlin(UI・アプリ)、C++17(音声エンジン) |
| UI | Jetpack Compose、Material 3、`material3-window-size-class` |
| アーキテクチャ | MVVM + 単方向データフロー(ViewModel + `StateFlow`)、Hilt(DI) |
| 音声出力 | Oboe(Apache 2.0) |
| DSP 部品 | DaisySP の MIT 部分(シンセ、電子ドラム合成、弦の物理モデル)、Freeverb |
| MIDI | `android.media.midi`(Kotlin)、AMidi(NDK、API 29+)、ktmidi(SMF 読み書き) |
| SF2 再生 | TinySoundFont(MIT) |
| 音声ファイル | dr_libs(読み込み)、MediaCodec / MediaMuxer(M4A 書き出し) |
| 対応 OS | minSdk 24 / targetSdk 36(選定理由は [05](05_sdk_selection.md)・[07](07_device_compatibility.md)) |
| 永続化 | Room(プロジェクト一覧)、kotlinx.serialization(プロジェクト本体 JSON) |
| ビルド | Gradle(Kotlin DSL)、CMake、AAB + Play Asset Delivery |
| テスト | JUnit / Compose UI Test(Kotlin)、GoogleTest(C++) |

## 2. 全体構成

```
┌──────────────────────── UI 層 (Kotlin / Compose) ────────────────────────┐
│  KeyboardScreen  PadScreen  SequencerScreen  PianoRoll  Mixer  Settings  │
│        │ ViewModel (StateFlow)                                            │
└────────┼──────────────────────────────────────────────────────────────────┘
         │
┌────────▼──────────── ドメイン / アプリ層 (Kotlin) ─────────────────────────┐
│  ProjectRepository   Transport(再生/録音)   InputRouter                   │
│  MidiDeviceManager (BLE/USB)   HidKeyboardMapper   UndoManager            │
└────────┼───────────────────────────┬───────────────────────────────────────┘
         │ JNI (コマンド送信)          │ AMidi (MIDI バイト直接受け渡し)
┌────────▼───────────────────────────▼──────── 音声エンジン (C++ / NDK) ────┐
│  CommandQueue (ロックフリー SPSC)                                          │
│  AudioEngine ── Oboe AudioStream (コールバック)                            │
│     ├─ Sequencer(サンプル単位のイベント処理)                              │
│     ├─ Track[n] ── Instrument(Synth / DrumSampler / SF2Player)            │
│     │              └─ InsertFX[2]                                         │
│     ├─ SendFX(Reverb, Delay)                                              │
│     └─ MasterBus ── Limiter → 出力                                        │
│  OfflineRenderer(WAV 書き出し)                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Gradle モジュール構成

| モジュール | 内容 |
|---|---|
| `:app` | Activity、ナビゲーション、DI 設定 |
| `:feature:keyboard` `:feature:pads` `:feature:sequencer` `:feature:mixer` `:feature:settings` | 各画面 |
| `:core:ui` | 共通 Compose 部品(鍵盤、ノブ、フェーダー)、テーマ |
| `:core:model` | ドメインモデル(Project, Track, Note, Pattern …) |
| `:core:data` | 保存・読み込み、Room |
| `:core:midi` | MIDI 機器管理、BLE スキャン、HID キー割り当て |
| `:core:audio` | エンジンの Kotlin ラッパー(JNI)+ C++ ソース(`src/main/cpp`) |

## 3. 音声エンジン仕様

### 3.1 ストリーム設定
| 項目 | 値 |
|---|---|
| 方向 | 出力 |
| PerformanceMode | `LowLatency` |
| SharingMode | `Exclusive`(不可なら Shared) |
| フォーマット | Float |
| チャンネル | ステレオ |
| サンプルレート | 機種のネイティブ値(通常 48000Hz)。エンジン内部も同じ値 |
| バッファ | `framesPerBurst × 2` から開始、アンダーラン時に自動拡大(上限 burst × 8) |
| Usage | `Game` ではなく `Media`(ただし低レイテンシ経路に乗るか機種で検証) |

### 3.2 スレッドモデル
| スレッド | 役割 | 禁止事項 |
|---|---|---|
| オーディオコールバック(優先度最高) | 発音処理、シーケンサー進行、ミキシング | メモリ確保、ロック、I/O、ログ、JNI |
| MIDI 受信 | AMidi / `MidiReceiver` で受け取り、タイムスタンプ付きでキューへ | 重い処理 |
| UI(メイン) | 入力・表示。コマンドをキューへ送る | — |
| ワーカー | サンプル読み込み、書き出し、保存 | — |

- **UI/MIDI → エンジン**: 固定長コマンド(`NoteOn`, `NoteOff`, `SetParam`, `LoadInstrument` 完了通知 など)をロックフリーの SPSC キューで送る。生産者が複数(UI と MIDI)なので、キューは生産者ごとに 1 本ずつ持つ。
- **エンジン → UI**: 再生位置、レベルメーター、アンダーラン数を atomic 変数に書き、UI が 60fps で読む。
- 大きなデータ(サンプル、プロジェクト全体)は、ワーカーで準備したものをポインタの差し替えで渡す。古いデータの解放はワーカー側で行う(コールバック内で `delete` しない)。

### 3.3 イベントのタイミング
- 各コマンドは `timestampNs`(`System.nanoTime` 基準)を持つ。
- コールバックでは、バッファの出力予定時刻(`getTimestamp()` から算出)と照らし合わせ、バッファ内の該当サンプル位置で発音する。
- ライブ入力は「今すぐ」扱い(遅延を足さない)が基本。揺れ低減モード(設定で ON)では一定量(例 5ms)遅らせて揺れを吸収する。

### 3.4 シンセサイザー(SubtractiveSynth)
| ブロック | パラメータ |
|---|---|
| OSC1 / OSC2 | 波形(Sine / Saw / Square / Triangle、PolyBLEP)、オクターブ、デチューン、パルス幅、レベル |
| Noise | レベル |
| Filter | 種類(LP12 / LP24 / HP / BP、State Variable Filter)、カットオフ、レゾナンス、エンベロープ量、キートラック |
| AmpEnv / FilterEnv | A / D / S / R |
| LFO | 波形、速度(テンポ同期可)、送り先(ピッチ / カットオフ / 音量)、深さ |
| Voice | ポリ / モノ / レガート、グライド、同時発音数、ベロシティ感度 |

- パラメータ変化は 64 サンプル程度で滑らかに補間(ジッパーノイズ防止)。
- ボイススティール: 空きがない場合は、リリース中で最も古い音 → 最も古い音の順に再利用。

### 3.5 ドラム(DrumSampler)
- 16 パッド。各パッド: サンプル、音量、パン、ピッチ、減衰、チョークグループ(オープン / クローズハイハットの排他)。
- GM ドラムマップ(36=キック、38=スネア、42=クローズ HH …)に準拠し、MIDI 鍵盤からも叩ける。
- パッドの音源は 2 種類: **合成**(DaisySP のアナログ風キック / スネア / ハイハット。808・909 風キット用、素材不要)と **サンプル**(生ドラム用)。

### 3.6 ギター等(SF2Player)
- TinySoundFont でプリセットを読み込み、ノート / ベロシティ / ピッチベンドを渡す。
- ギター用: ストラム機能(和音の各音を 5〜30ms ずらして発音、上下方向)を UI 側で生成。
- サンプルはワーカーで読み込み、準備完了後にエンジンへ渡す。

### 3.7 エフェクト
| エフェクト | 方式 |
|---|---|
| Reverb | Freeverb 系(コム + オールパス) |
| Delay | テンポ同期、フィードバック、ハイカット |
| Distortion | ソフトクリップ(tanh)+ トーン、2 倍オーバーサンプリング |
| EQ | 3 バンド(バイクアッド) |
| Compressor / Limiter | RMS / ピーク、マスターは先読みリミッター |

### 3.8 JNI API(Kotlin 側から見たインターフェース)

```kotlin
interface AudioEngine {
    fun start(): EngineResult
    fun stop()
    fun noteOn(track: Int, note: Int, velocity: Int, timestampNs: Long = 0)
    fun noteOff(track: Int, note: Int, timestampNs: Long = 0)
    fun allNotesOff(track: Int? = null)
    fun pitchBend(track: Int, value: Int)          // -8192..8191
    fun controlChange(track: Int, cc: Int, value: Int)
    fun setParam(target: ParamTarget, paramId: Int, value: Float)
    fun loadInstrument(track: Int, spec: InstrumentSpec)  // 非同期、完了は Flow で通知
    fun setSequence(snapshot: SequenceSnapshot)          // 曲データの差し替え
    fun play(); fun pause(); fun seek(tick: Long)
    fun setTempo(bpm: Double)
    val status: StateFlow<EngineStatus>  // 再生位置、レベル、アンダーラン数、レイテンシ
}
```

## 4. 入力仕様

### 4.1 InputRouter
すべての演奏入力(画面鍵盤、パッド、BLE/USB-MIDI、HID キーボード)を共通の `NoteEvent` に変換し、
「現在選択中のトラック」(MIDI チャンネル割り当てがあればそのトラック)へ送る。録音中は同時に録音バッファへも記録する。

### 4.2 BLE-MIDI 接続フロー
1. 設定 > MIDI 機器 >「Bluetooth 機器を探す」
2. 権限確認(API 31+: `BLUETOOTH_SCAN` / `BLUETOOTH_CONNECT`、API 30 以下: `ACCESS_FINE_LOCATION`)
3. `BluetoothLeScanner` で MIDI サービス UUID `03B80E5A-EDE8-4B33-A751-6CE34EC4C700` をフィルタしてスキャン(最長 15 秒)
4. 一覧から選択 → `MidiManager.openBluetoothDevice()` → `MidiDevice.openOutputPort(0)`
5. API 29+: `AMidiOutputPort` で C++ 側が直接読み取り。API 28 以下: `MidiReceiver.onSend()` から JNI でエンジンへ
6. 機器のアドレスを保存し、次回起動時・画面復帰時に自動再接続
7. 切断を検知したら全ノートオフし、通知を表示

対応 MIDI メッセージ: Note On/Off、Pitch Bend、CC1(モジュレーション)、CC7(音量)、CC10(パン)、CC64(サスティン)、CC123(オールノートオフ)、Program Change(プリセット切替)。

### 4.3 HID キーボード(PC 用)のキー割り当て(初期値)

```
 黒鍵:   W  E     T  Y  U     O  P
 白鍵:  A  S  D  F  G  H  J  K  L  ;
        C  D  E  F  G  A  B  C  D  E
 Z / X : オクターブ − / +      C / V : ベロシティ − / +
 Space : 再生 / 停止            R     : 録音
 1〜8  : トラック選択
```

- JIS / US 配列の違いに影響されないよう、文字ではなく**物理キー位置(`KeyEvent.scanCode`)**で判定する。
- `Activity.dispatchKeyEvent` で受け取り、テキスト入力欄にフォーカスがある時は割り当てを無効化する。
- キーリピート(`repeatCount > 0`)は無視。
- ドラム画面では `1〜4 / Q〜R / A〜F / Z〜V` を 4×4 パッドに割り当て。

## 5. 画面仕様

### 5.1 画面一覧
| 画面 | 内容 |
|---|---|
| ホーム | プロジェクト一覧、新規作成、テンプレート |
| スタジオ(メイン) | トラック一覧 + 演奏面(鍵盤 / パッド)+ トランスポート |
| ステップシーケンサー | ドラムの 16 ステップ |
| ピアノロール | メロディ / コードの編集 |
| 音色エディター | シンセのパラメータ、プリセット選択 |
| ミキサー | 音量・パン・エフェクト |
| 設定 | オーディオ、MIDI 機器、キー割り当て、言語 |

### 5.2 画面サイズ別レイアウト(Window Size Class)

| 幅クラス | 代表端末 | レイアウト |
|---|---|---|
| Compact(< 600dp) | スマホ縦 | 下部タブで「演奏 / シーケンス / ミキサー」を切替。上部にトランスポート。鍵盤は 1 オクターブ半、横スクロール |
| Medium(600–840dp) | スマホ横、小型タブレット縦、折りたたみ | 左にトラック一覧(縮小)、右に演奏面 or 編集面。鍵盤 2 オクターブ |
| Expanded(≥ 840dp) | タブレット横 | 3 ペイン: 左トラック一覧 / 上部にシーケンサー・ピアノロール / 下部に鍵盤(3〜4 オクターブ)。ミキサーは右側パネルで開閉 |

```
[Compact: スマホ縦]           [Expanded: タブレット横]
┌──────────────┐           ┌────────┬─────────────────────────────┐
│ ▶ ■ ● 120BPM │           │ ▶ ■ ●  120 BPM   4/4   ⏱ 12ms           │
├──────────────┤           ├────────┼─────────────────────────────┤
│ Track: Synth1│           │Tracks  │  ピアノロール / ステップ           │
│              │           │ Drums  │                             │
│  (パッド or  │           │ Bass   │                             │
│   ピアノロール)│           │ Synth1 │                             │
│              │           │ Guitar ├─────────────────────────────┤
├──────────────┤           │        │  鍵盤 (3〜4 オクターブ)            │
│ 鍵盤 1.5oct  │           │        │  ▮▯▮▯▮▮▯▮▯▮▯▮▮▯▮▯▮▮▯▮▯▮           │
├──────────────┤           └────────┴─────────────────────────────┘
│演奏│シーケンス│ミキサー│
└──────────────┘
```

- 鍵盤の 1 鍵の幅は最小 40dp(スマホ)/ 48dp(タブレット)。ピンチで幅を変更可能。
- 横画面固定にはしない(ユーザーが選べる)。演奏画面のみ「横向き推奨」を表示。
- 折りたたみ端末のテーブルトップ姿勢(半開き)では、上半分を編集面、下半分を鍵盤にする(Jetpack WindowManager の `FoldingFeature`)。

### 5.3 マルチタッチ鍵盤
- `pointerInput` で各ポインター ID を追跡。指が鍵をまたいで移動したら、前の鍵をオフ → 新しい鍵をオン(グリッサンド)。
- 押した位置の縦方向(鍵の手前ほど強い)でベロシティを決める(設定で固定値にもできる)。
- タッチ処理はメインスレッドで最小限にし、すぐエンジンへコマンドを送る(描画を待たない)。

## 6. データモデル

```kotlin
data class Project(
    val id: String, val name: String,
    val tempoBpm: Double, val timeSignature: TimeSig,   // 4/4 など
    val ppq: Int = 480,                                  // 4 分音符あたりの分解能
    val tracks: List<Track>,
    val arrangement: List<Clip>,                         // ソング画面(S)
    val master: MixerChannel,
    val version: Int,                                    // ファイル形式のバージョン
)
data class Track(
    val id: String, val name: String,
    val instrument: InstrumentSpec,                      // Synth(params) / Drum(kitId) / Sf2(assetId, preset)
    val patterns: List<Pattern>,
    val mixer: MixerChannel, val midiChannel: Int?,      // null = 選択中トラックとして受信
)
data class Pattern(val id: String, val lengthTicks: Long, val notes: List<Note>)
data class Note(val tick: Long, val durationTicks: Long, val pitch: Int, val velocity: Int)
data class MixerChannel(val volume: Float, val pan: Float, val mute: Boolean,
                        val solo: Boolean, val inserts: List<EffectSpec>, val sends: Map<String, Float>)
```

### 6.1 保存形式
- `files/projects/<id>/project.json`(上記を JSON 化)+ `files/projects/<id>/samples/`(ユーザー音色の参照コピー)
- 書き出し: 共有用は `.mmga`(ZIP: project.json + 使用しているユーザー音色)
- 自動保存: 編集から 2 秒後(デバウンス)+ アプリがバックグラウンドへ移った時

## 7. 権限・マニフェスト

```xml
<uses-feature android:name="android.software.midi" android:required="false" />
<uses-feature android:name="android.hardware.bluetooth_le" android:required="false" />
<uses-feature android:name="android.hardware.audio.low_latency" android:required="false" />
<uses-feature android:name="android.hardware.touchscreen.multitouch.distinct" android:required="false" />

<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

- `required="false"` にして、MIDI / BLE 非対応端末でもインストール可能にする(該当機能だけ非表示)。
- 再生用の Foreground Service は `foregroundServiceType="mediaPlayback"`。

## 8. 端末性能の自動判定

起動時(初回 + OS 更新時)に以下を行い、設定に保存する。
1. `PackageManager.hasSystemFeature` で `audio.low_latency` / `audio.pro` を確認
2. 5 秒間のベンチマーク(シンセ 16 音をオフラインで処理し、実時間比を測る)
3. 結果から同時発音数の上限・オーバーサンプリング有無・初期バッファサイズを決める

| ランク | 条件の目安 | 同時発音数 | 初期バッファ |
|---|---|---|---|
| High | 実時間比 < 5% かつ low_latency | 64 | burst × 2 |
| Mid | 実時間比 < 15% | 32 | burst × 2〜3 |
| Low | それ以外 | 16 | burst × 4 |

## 9. テスト方針
| 種類 | 内容 |
|---|---|
| C++ 単体 | DSP(オシレーター周波数、フィルター応答、エンベロープ)、シーケンサーのタイミング(サンプル精度)をオフラインで検証 |
| 音の回帰テスト | 固定プロジェクトをオフラインレンダリングし、基準 WAV との差分を比較 |
| Kotlin 単体 | InputRouter、HID キー割り当て、プロジェクトの保存 / 読み込み、Undo |
| UI | Compose UI Test(Compact / Expanded 両方のレイアウト) |
| 実機 | レイテンシ測定(ループバック、または OboeTester)、BLE-MIDI 鍵盤での打鍵遅延、30 分連続再生での音切れ・発熱 |
| 端末マトリクス | Pixel(低レイテンシ基準)、Galaxy 中位、低価格スマホ、Galaxy Tab / Pixel Tablet、折りたたみ端末 |
