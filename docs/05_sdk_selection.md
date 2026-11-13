# 05. SDK・ライブラリ選定

Android で使える音声・MIDI 関連の SDK / ライブラリを比較し、採用するものを決める。
前提: **アプリは無料素材のみ・初心者向け・できるだけ多くの機種で動かす**。ライセンス費用がかからず、クローズドソースでも配布できるもの(MIT / BSD / Apache 2.0 など)を優先する。

> ライセンスや価格は 2026 年 11 月時点の Web 調査に基づく。採用前に各ライブラリの LICENSE ファイルを必ず確認すること。

## 採用一覧(結論)

| 用途 | 採用 | ライセンス | 理由 |
|---|---|---|---|
| 音声出力 | **Oboe** | Apache 2.0 | Google 公式。AAudio(8.1 以降)と OpenSL ES(それ以前)を自動で切り替え、機種ごとの不具合回避も入っている |
| シンセ / ドラム合成の DSP 部品 | **DaisySP**(MIT 部分のみ)+ 自前実装 | MIT | オシレーター、フィルター、エンベロープ、アナログ風ドラム合成、弦の物理モデルなどが揃っている |
| サンプル音源(SF2)再生 | **TinySoundFont** | MIT | ヘッダー 1 つで組み込みが簡単、軽量 |
| MIDI 入力(BLE / USB) | **android.media.midi**(標準)+ **AMidi**(NDK) | Android 標準 | 追加ライブラリ不要。BLE-MIDI は OS の Bluetooth MIDI サービスが処理する |
| BLE-MIDI の予備手段 | kshoji **BLE-MIDI-for-Android** | Apache 2.0 | 標準 API でつながらない機種向けの代替(PoC で必要性を判断) |
| MIDI ファイル(SMF)読み書き | **ktmidi** | MIT | Kotlin 製。SMF の読み書き、MIDI 2.0 にも対応 |
| WAV / FLAC 読み込み | **dr_libs**(dr_wav, dr_flac) | パブリックドメイン / MIT-0 | ヘッダーのみ。ユーザーの音声ファイル読み込みに使う |
| AAC(M4A)書き出し | **MediaCodec + MediaMuxer** | Android 標準 | 追加ライブラリ不要 |
| UI | **Jetpack Compose + Material 3 Adaptive + WindowManager** | Apache 2.0 | 画面サイズ別レイアウト、折りたたみ端末対応 |
| レイテンシ測定(開発用) | **OboeTester** | Apache 2.0 | Google 製の測定アプリ。実機検証に使う |

**ライセンス費用: 0 円**。いずれもアプリのソースコード公開義務はない(MIT / Apache は著作権表示が必要 → アプリ内の「ライセンス」画面に表示)。

---

## 1. 音声エンジン(出力)の比較

| 候補 | ライセンス / 費用 | 長所 | 短所 | 判定 |
|---|---|---|---|---|
| **Oboe** | Apache 2.0、無料 | Google 公式・推奨。低レイテンシ。機種別の回避策を内蔵 | 音を作る機能はない(出力だけ) | **◎ 採用** |
| AAudio を直接使う | Android 標準 | 依存なし | Android 8.0 未満で使えない。機種別の不具合を自分で回避する必要 | × |
| AudioTrack(Kotlin) | Android 標準 | 簡単 | 遅延が大きい、GC で音切れ | × |
| Superpowered | 独自ライセンス。無料枠ありだが条件・ライセンスキーが必要、価格は要問い合わせ | エフェクト・時間伸縮などが豊富、高速 | クローズドソース、条件変更リスク | △ 不採用(将来の時間伸縮機能で再検討) |
| JUCE 8 | AGPLv3 か有料(Starter は年収 2 万ドルまで無料、Indie は 30 万ドルまで有料) | シンセ・サンプラー・UI まで一式。iOS / PC 版にも展開しやすい | AGPL はアプリ全体のソース公開が必要。有料プランは年額費用。Android の UI は Compose と統合しにくい | △ 将来 iOS / PC 版を作るなら再検討 |
| FMOD | 年収 20 万ドル未満などの条件で無料、超えると作品ごとに有料 | ゲーム音声の定番 | ゲーム向けで、シンセ・シーケンサー用途に合わない | × |
| libpd(Pure Data) | BSD | パッチ(Pd)で音を設計できる | Pd の知識が必要、性能・デバッグ面で不利 | × |
| Csound for Android | LGPL 2.1 | 機能が非常に多い | 大きい、LGPL の扱い、学習コスト | × |
| Faust | コンパイラは GPL、生成されたコードは自由に使える | DSP を簡潔に書け、C++ に変換できる | ビルド工程が増える | ○ 一部のエフェクト開発に任意で使う |

## 2. 音源(DSP / サンプル再生)の比較

### 2.1 合成系(シンセ・電子ドラム・物理モデル)

| 候補 | ライセンス | 内容 | 判定 |
|---|---|---|---|
| **DaisySP** | MIT(一部モジュールは別リポジトリ DaisySP-LGPL に分離) | オシレーター(PolyBLEP)、SVF / ラダー系フィルター、ADSR、**アナログ風キック / スネア / ハイハット合成**、**StringVoice(弦の物理モデル)**、ディレイ、オーバードライブ | **◎ 採用**(MIT の部分だけ使う。LGPL 側は使わない) |
| Mutable Instruments(Plaits / Rings 等)のソース | MIT | 高品質なマクロオシレーター、共鳴体モデル(弦・板) | ○ 将来の追加音色候補 |
| Freeverb | パブリックドメイン | 定番リバーブ | ○ 採用 |
| Signalsmith DSP | MIT | ディレイ、フィルター、補間などの部品 | ○ 必要に応じて |
| STK(Synthesis ToolKit) | MIT 系 | 物理モデル楽器(弦・管) | △ 比較用 |

**電子ドラム(808 / 909 風)は合成で作る**ので、音声ファイルが不要でアプリサイズを小さくできる。

### 2.2 サンプル再生(SF2 / SFZ)

| 候補 | ライセンス | 形式 | 長所 | 短所 | 判定 |
|---|---|---|---|---|---|
| **TinySoundFont** | MIT | SF2 | ヘッダー 1 つ、軽量、組み込み簡単 | SFZ 非対応、機能は最小限 | **◎ 採用** |
| FluidSynth | LGPL 2.1 | SF2 / SF3 | 高機能・実績多数。Oboe 出力ドライバーあり | 依存(glib 等)を含め大きい。LGPL のため静的リンク不可(.so で動的リンクし差し替え可能にする必要) | △ 予備 |
| sfizz | BSD 系(依存に Abseil 等) | SFZ | SFZ の再現度が高い | Android 向けビルドは自前で用意する必要、依存が多い | △ 将来 |
| liquidsfz | MPL 2.0 | SFZ | 比較的小さい | Android 実績が少ない | △ |

**方針:** 無料素材は SFZ 形式で配布されているものが多いが、**開発時に SF2 へ変換**(変換ツール Polyphone を開発 PC で使う)して、アプリ内は TinySoundFont 1 本にそろえる。エンジンを 1 つにすることで、テストと不具合対応が楽になる。

## 3. MIDI / Bluetooth の比較

| 候補 | ライセンス | 対応 | 長所 | 短所 | 判定 |
|---|---|---|---|---|---|
| **android.media.midi** | 標準 | API 23+、BLE-MIDI / USB-MIDI / 仮想 MIDI | 追加不要。OS の Bluetooth MIDI サービスが BLE-MIDI を処理 | 接続間隔(遅延)をアプリから制御できない | **◎ 採用** |
| **AMidi(NDK)** | 標準 | API 29+ | C++ から直接 MIDI を読める(JNI を経由しない) | Android 10 未満は使えない | **◎ 採用**(API 29 以上で使用) |
| kshoji BLE-MIDI-for-Android | Apache 2.0 | API 18+ | BLE を自前で処理するので接続パラメータを調整できる。周辺機器(Peripheral)側にもなれる | OS 標準と二重管理になる | ○ 予備(相性問題が出た機種向け) |
| ktmidi | MIT | Kotlin | SMF の読み書き、MIDI 2.0(UMP) | — | **◎ 採用**(ファイル入出力) |

補足:
- Android 13 で USB の MIDI 2.0、Android 15 で USB 以外(仮想デバイス)の MIDI 2.0 に対応した。**BLE の MIDI 2.0 は未対応**のため、本アプリは MIDI 1.0 を前提にする。
- PC 用の Bluetooth キーボード(HID)は OS の標準機能で動くため、ライブラリは不要。

## 4. その他

| 用途 | 採用 | 備考 |
|---|---|---|
| DI | Hilt | |
| 保存 | Room + kotlinx.serialization | |
| ライセンス表示 | Google の `oss-licenses-plugin` + 素材のクレジットを手動で追加 | 素材(CC-BY 等)のクレジット表示義務を満たす |
| 自動テスト(多機種) | Firebase Test Lab | クラッシュ・起動・CPU 負荷を多機種で確認(音の遅延は測れない) |
| クラッシュ収集 | (任意)Firebase Crashlytics | MVP は通信なしの方針。採用する場合はプライバシーポリシーを更新 |

## 5. 構成図(採用ライブラリの位置)

```
Kotlin / Compose UI ──── Material3 Adaptive / WindowManager
   │
   ├── android.media.midi(BLE / USB 接続管理)
   ├── KeyEvent(PC キーボード)
   ├── ktmidi(SMF 読み書き)
   ├── MediaCodec / MediaMuxer(M4A 書き出し)
   │
   └── JNI ─┐
            ▼
   C++ 音声エンジン
     ├── Oboe(出力)
     ├── AMidi(MIDI 直接受信, API 29+)
     ├── DaisySP[MIT 部分](シンセ・電子ドラム・弦モデル)
     ├── TinySoundFont(SF2: ピアノ・ギター・ベース・生ドラム等)
     ├── Freeverb / 自前エフェクト
     └── dr_libs(WAV / FLAC 読み込み)
```

## 参考資料(2026-11 調査)
- Oboe: https://developer.android.com/games/sdk/oboe / https://github.com/google/oboe
- Superpowered ライセンス: https://superpowered.com/licensing
- JUCE 8 ライセンス(各プランの収益上限): https://forum.juce.com/t/revenue-limits-for-juce-tiers/61058
- FMOD ライセンス: https://fmod.com/licensing
- FluidSynth の Android ビルドと LGPL の注意: https://github.com/FluidSynth/fluidsynth/wiki/BuildingForAndroid
- TinySoundFont: https://github.com/schellingb/TinySoundFont
- sfizz: https://github.com/sfztools/sfizz / liquidsfz: https://github.com/swesterfeld/liquidsfz
- libpd + Oboe の例(kortholt): https://github.com/simonnorberg/kortholt
- BLE-MIDI-for-Android: https://github.com/kshoji/BLE-MIDI-for-Android
- Android の MIDI 2.0 対応状況: https://atsushieno.github.io/2024/04/12/midi2-on-android.html / https://source.android.com/docs/core/audio/midi
- Google Play の targetSdk 要件: https://developer.android.com/google/play/requirements/target-sdk
